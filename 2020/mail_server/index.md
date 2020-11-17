# 搭建邮箱服务器


搭建一个邮箱服务器, 方便自己之后使用 (~~还可以装逼~~

---


## Mailcow {#mailcow}

[Mailcow](https://mailcow.email/) 是一个使用docker搭建的标准邮件服务器, 集成了邮局、webmail、管理以及反垃圾邮件等功能, 过程相对全面, 不过缺点是比较吃资源, 并且不支持 **Synology/QNAP** 或 **OpenVZ** 、 **LXC** 等虚拟化方式, 并且不能使用 `CentOS 7/8` 源中的 Docker 包, 要求真多。。。

| 资源 | 需求          |
|----|-------------|
| CPU | 1GHz          |
| RAM | 最少4G (包含交换空间) |
| 硬盘 | 20GiB (不包含邮件) |

消耗资源的主要原因是 `ClamAV` 和 `Solr`, 即杀毒功能和搜索功能, 如果不需要可以关闭。

---


## 安装 {#安装}

开始搭建服务器, 以下采用域名 `example.com`, IP `1.1.1.1`, 安装在 `/mailcow`, 使用主机的nginx反向代理, 请根据自己的需求修改


### DNS {#dns}

DNS设置是一个邮件服务器的重中之重, 为了让我们可以发出邮件和收到邮件, 防止邮件被拒收或者进入垃圾箱被识别成垃圾邮件等, 当然不是配置好了就不会进垃圾邮箱, 不配置肯定会有问题。

| 类型 | 记录    | 记录值                                                                                              |
|----|-------|--------------------------------------------------------------------------------------------------|
| A   | mail    | 1.1.1.1                                                                                             |
| MX  | @       | mail.example.com (10)                                                                               |
| TXT | @       | v=spf1 a mx ip:1.1.1.1 -all                                                                         |
| TXT | \_dmarc | v=DMARC1; p=reject; rua=<mailto:admin@example.com>; ruf=<mailto:admin@example.com>; adkim=s; aspf=s |

除了上述DNS解析之外, 还需要配置 **DKIM** 和 **PTR**, DKIM在我们搭建好服务之后配置, PTR需要向运营商提交工单申请 (阿里云和腾讯云是这样的), 当然PTR可有可无, **配置了最好** 。


### 搭建 {#搭建}

在搭建之前我们首先定义一些变量, 以便之后使用, 方便根据自己的需求更改

```shell
path_to="/path/to"
mailcow_path="${path_to}/mailcow" # mailcow 所在目录
mail_host="mail.example.com"
http_port="8080"
https_port="8443"
cert_path="/path/to/cert/" # 证书存放目录
cert_file="${cert_path}/cert.pem" # 域名证书
key_file="${cert_path}/key.pem" # 域名证书密钥
ca_file="${cert_path}/intermediate_CA.pem" # 域名证书颁发者证书
root_file="${cert_path}/root.pem" # 根证书
```

现在开始正式的搭建邮箱服务器

```shell
cd ${path_to}
git clone https://github.com/mailcow/mailcow-dockerized mailcow && cd mailcow
echo ${email_host} | ./generate_config.sh
sed -ie "s/HTTP_PORT=.*/HTTP_PORT=${http_port}/" mailcow.conf # HTTP端口
sed -ie "s/HTTPS_PORT=.*/HTTPS_PORT=${https_port}/" mailcow.conf # HTTPS端口
sed -i "s/TZ=.*/TZ=Asia\/Shanghai/" mailcow.conf # 时区
sed -i "s/SKIP_LETS_ENCRYPT=.*/SKIP_LETS_ENCRYPT=y/" mailcow.conf # 证书申请, 关闭
sed -i "s/SKIP_SOGO=.*/SKIP_SOGO=n/" mailcow.conf # webmail, 开启
sed -i "s/SKIP_SOLR=.*/SKIP_SOLR=n/" mailcow.conf # 搜索, 关闭
sed -i "s/enable_ipv6: true/enable_ipv6: false/" docker-compose.yml # 关闭ipv6
```

下面给出Nginx配置文件, Apache 配置文件请参见 [官方文档](https://mailcow.github.io/mailcow-dockerized-docs/firststeps-rp/#apache-24)

```Nginx
server {
  listen 80;
  listen [::]:80;
  server_name mail.example.com;
  return 301 https://$host$request_uri;
}
server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name mail.example.com;

  ssl_certificate /ssl/domain/cert.pem;
  ssl_certificate_key /ssl/domain/key.pem;
  ssl_session_timeout 2h;
  ssl_session_cache shared:mailcow:16m;
  ssl_session_tickets off;

  # See https://ssl-config.mozilla.org/#server=nginx for the latest ssl settings recommendations
  # An example config is given below
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5:!SHA1:!kRSA;
  ssl_prefer_server_ciphers off;

  location /Microsoft-Server-ActiveSync {
    proxy_pass http://127.0.0.1:8080/Microsoft-Server-ActiveSync;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_connect_timeout 75;
    proxy_send_timeout 3650;
    proxy_read_timeout 3650;
    proxy_buffers 24 256k;
    client_body_buffer_size 512k;
    client_max_body_size 0;
  }

  location / {
    proxy_pass http://127.0.0.1:8080/;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    client_max_body_size 0;
  }
}
```

以上全部完成后, mailcow 基本配置完成, 只需要启动起服务即可, 默认用户密码 `admin` / `moohoo`

```shell
cd ${mailcow_path}
docker-compose pull
docker-compose up -d
```


### TLS {#tls}

现在我们可以为SMTP与IMAP服务加入TLS, 假设我们已经对域名 `mail.example.com` 申请了证书, 对 postfix 与 dovecot 配置证书前, 我们需要根据 postfix 文档先将我们自己的证书与提供商的证书按顺序存放在同一文件下, 并且文件后缀为 **.pem**, 并存放在mailcow的ssl文件夹下

```shell
cat ${cert_file} ${ca_file} ${root_file} > ${mailcow_path}/data/assets/ssl/cert.pem
cp ${key_file} ${mailcow_path}/data/assets/ssl/key.pem
```

证书保存完毕后, 对 postfix 与 dovecot 进行配置, 配置完成重启服务即可

```shell
# postfix
sed -i "s/smtp_tls_security_level.*/smtp_tls_security_level = dane/" data/conf/postfix/main.cf
sed -i "s/smtp_tls_CAfile.*/smtp_tls_CAfile = \/etc\/ssl\/mail\/cert.pem/" data/conf/postfix/main.cf
sed -i "s/smtp_tls_cert_file.*/smtp_tls_cert_file = \/etc\/ssl\/mail\/cert.pem/" data/conf/postfix/main.cf
sed -i "s/smtp_tls_key_file.*/smtp_tls_key_file = \/etc\/ssl\/mail\/key.pem/" data/conf/postfix/main.cf
sed -i "s/smtpd_tls_security_level.*/smtpd_tls_security_level = may/" data/conf/postfix/main.cf
sed -i "s/smtpd_tls_CAfile.*/smtpd_tls_CAfile = \/etc\/ssl\/mail\/cert.pem/" data/conf/postfix/main.cf
sed -i "s/smtpd_tls_cert_file.*/smtpd_tls_cert_file = \/etc\/ssl\/mail\/cert.pem/" data/conf/postfix/main.cf
sed -i "s/smtpd_tls_key_file.*/smtpd_tls_key_file = \/etc\/ssl\/mail\/key.pem/" data/conf/postfix/main.cf
# dovecot
sed -i "s/ssl_cert.*/ssl_cert = <\/etc\/ssl\/mail\/cert.pem/" data/conf/dovecot/dovecot.conf
sed -i "s/ssl_key.*/ssl_key = <\/etc\/ssl\/mail\/key.pem/" data/conf/dovecot/dovecot.conf
# restart
docker-compose restart postfix-mailcow dovecot-mailcow
```


### 黑名单 {#黑名单}

在互联网上发送邮件不是可以为所欲为的, 邮局服务有一套反垃圾邮件机制, 当你的IP上了黑名单时, 从这个IP发出去的邮件很容易进入垃圾邮箱或拒收, 请珍惜自己的IP, 不过可以尝试在检测上了哪些服务商的黑名单, 并尝试解除黑名单, 以下给出一些检测或申请去除反垃圾邮件网址

-   [MXToolBox](https://mxtoolbox.com/SuperTool.aspx)
-   <http://multirbl.valli.org/>
-   [Office 365](https://sender.office.com/)
-   [Barracuda](https://www.barracudacentral.org/rbl/removal-request)

---


## 安全 {#安全}

我们已经配置了TLS, 对于邮件的传输过程来说我们的邮件是安全的, 但是对于服务提供商来说还是可以随意浏览我们的邮件内容的, 如果你希望重要的内容不被服务商所浏览, 可以尝试使用对邮件加密的方式。邮件加密并不是将邮件转换为一个带密码的文件, 而是使用非对称加密套件, 在MUA中进行加密、签名等, MTA只负责传输邮件而不能检测邮件的内容。如果你想使用加密的方式向我发送邮件, 请保存以下公钥:

-   [OpenPGP](https://blog.ginshio.org/pgp%5Fpublic%5Fkey)
-   [S/MIME (iris@ginshio.org)](https://blog.ginshio.org/iris%5Fsmime%5Fpublic%5Fkey)
-   [S/MIME (ginshio78@gmail.com)](https://blog.ginshio.org/gmail%5Fsmime%5Fpublic%5Fkey)

由于加密邮件是MUA行为, 一般情况服务提供商的Webmail并不支持加密邮件, 部分提供加密功能的提供商如果需要你上传私钥到他们的服务器, 请保持警惕, 私钥可以解密你的邮件。以下列出了常见的支持加密的MUA:

-   [Microsoft Outlook](https://support.microsoft.com/zh-cn/outlook) (S/MIME)
-   [Apple Mail](https://support.apple.com/mail) (S/MIME)
-   [Mozilla Thunderbird](https://www.thunderbird.net/) (OpenPGP 和 S/MIME)
-   [KDE Kontact KMail](http://kontact.kde.org/) (OpenPGP 和 S/MIME)
-   [GNOME Evolution](https://wiki.gnome.org/Apps/Evolution) (OpenPGP 和 S/MIME)
-   [Mutt](http://www.mutt.org/) (OpenPGP 和 S/MIME)


### S/MIME {#s-mime}

安全多功能互联网邮件扩展 (S/MIME) 是基于 **PKI** 的符合 **X.509** 格式的非对称密钥协议, 提供了数字签名、加密功能。发送邮件时, 数字签名会以 `smime.p7s` 的附件跟随邮件发送, 如GMail的网页端就支持验证签名, 如果是加密邮件则整封邮件被加密后以 `smime.p7m` 的附件发送。双方互发信息之前, 如果没有对方公钥那么无法加密邮件, 需要先互相发送签名的邮件用以交换公钥, 导入公钥后可以开始发送加密邮件。你可以在 [Actalis](https://www.actalis.it/) 申请为期一年的免费 S/MIME 证书, 为你邮件加密开启第一步, 请保存好申请到的证书 (.pfx文件)、密码以及CRP。


### OpenPGP {#openpgp}

OpenPGP标准是一种非对称的非对称密钥协议, 提供了加密、签名等工程, OpenGPG是通过信任网络机制确保之间的密钥认证。相比于 S/MIME 而言, OpenGPG 在邮件方便被支持的更少, 比如Gmail可以在webmail中验证S/MIME签名, 但是并不支持 PGP/MIME。

---


## 推荐阅读 {#推荐阅读}

-   [Mailcow官方文档](https://mailcow.github.io/mailcow-dockerized-docs/)
-   [Outlook 反垃圾邮件策略指南](https://sendersupport.olc.protection.outlook.com/pm/policies.aspx)
-   [SPF 记录：原理、语法及配置方法简介](http://www.renfei.org/blog/introduction-to-spf.html)
-   [DMARC 是什么？](https://www.cnblogs.com/dmarcly/p/10947796.html)
-   [了解 S/MIME](https://docs.microsoft.com/zh-cn/previous-versions/exchange-server/exchange-server-2000/aa995740(v=exchg.65))
-   [电子邮件加密指南](https://emailselfdefense.fsf.org/zh-hans/)
-   [在 Thunderbird 中使用 OpenPGP —— 怎么做以及问题解答](https://support.mozilla.org/zh-CN/kb/thunderbird-openpgp#w%5Fthunderbird-zhi-chi-openpgp-ma)
-   [邮件服务器Poste五分钟搭建](https://newpants.top/2019/11/14/%E9%82%AE%E4%BB%B6%E6%9C%8D%E5%8A%A1%E5%99%A8Poste%E4%BA%94%E5%88%86%E9%92%9F%E6%90%AD%E5%BB%BA/)
-   [使用Mailcow自建邮件服务器](https://lala.im/4168.html)
-   [使用 mailcow:dockerized 搭建邮件服务器](https://low.bi/p/r7VbxEKo3zA)

