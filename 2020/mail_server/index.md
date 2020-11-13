# 搭建邮箱服务器


搭建一个邮箱服务器, 方便自己之后使用, 还可以 ~~装逼~~


## Mailcow {#mailcow}

[Mailcow](https://mailcow.email/) 是一个使用docker搭建的标准邮件服务器, 集成了邮局、webmail、管理以及反垃圾邮件等功能, 过程相对全面, 不过缺点是比较吃资源, 并且不支持 **Synology/QNAP** 或 **OpenVZ** 、 **LXC** 等虚拟化方式, 并且不能使用 `CentOS 7/8` 源中的 Docker 包, 要求真多。。。

| 资源 | 需求          |
|----|-------------|
| CPU | 1GHz          |
| RAM | 最少4G (包含交换空间) |
| 硬盘 | 20GiB (不包含邮件) |

消耗资源的主要原因是 `ClamAV` 和 `Solr`, 即杀毒功能和搜索功能, 如果不需要可以关闭。


## 安装 {#安装}

开始搭建服务器, 以下采用域名 `example.com`, IP `1.1.1.1`, 安装在 `/mailcow`, 使用主机的nginx反向代理, 请根据自己的需求修改


### DNS {#dns}

DNS设置是一个邮件服务器的重中之重, 为了让我们可以发出邮件和收到邮件, 防止邮件被拒收或者进入垃圾箱被识别成垃圾邮件等, 当然不是配置好了就不会进垃圾邮箱, 不配置肯定会有问题。

| 类型 | 记录    | 记录值                                                                                              |
|----|-------|--------------------------------------------------------------------------------------------------|
| A   | mail    | 1.1.1.1                                                                                             |
| MX  | @       | mail.example.com (10)                                                                               |
| MX  | @       | aspmx.l.google.com (15)                                                                             |
| MX  | @       | mxbiz1.qq.com (20)                                                                                  |
| TXT | @       | v=spf1 a mx ip:1.1.1.1 -all                                                                         |
| TXT | \_dmarc | v=DMARC1; p=reject; rua=<mailto:admin@example.com>; ruf=<mailto:admin@example.com>; adkim=s; aspf=s |

除了上述DNS解析之外, 还需要配置 **DKIM** 和 **PTR**, DKIM在我们搭建好服务之后配置, PTR需要向运营商提交工单申请 (阿里云和腾讯云是这样的), 当然PTR可有可无, **配置了最好** 。


### 搭建 {#搭建}

```shell
EMAIL_HOST="mail.example.com"
cd /
git clone https://github.com/mailcow/mailcow-dockerized mailcow && cd mailcow
echo ${EMAIL_HOST} | ./generate_config.sh
sed -i "s/HTTP_PORT=.*/HTTP_PORT=8080/" mailcow.conf # HTTP端口
sed -i "s/HTTPS_PORT=.*/HTTPS_PORT=8443/" mailcow.conf # HTTPS端口
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

  ssl_certificate /path/to/cert;
  ssl_certificate_key /path/tp/key;
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
cd /mailcow
docker-compose pull
docker-compose up -d
```


## 黑名单 {#黑名单}

上了黑名单的IP很容易被进入垃圾邮箱或拒收, 请珍惜自己的IP, 不过可以尝试在检测上了哪些服务商的黑名单, 并尝试解除黑名单, 以下给出一些检测或申请去除反垃圾邮件网址

-   [MXToolBox](https://mxtoolbox.com/SuperTool.aspx)
-   <http://multirbl.valli.org/>
-   [Office 365](https://sender.office.com/)
-   [Barracuda](https://www.barracudacentral.org/rbl/removal-request)


## 为其他应用开启邮件服务器 {#为其他应用开启邮件服务器}


### GitLab {#gitlab}

我们的GitLab使用的是源码安装, 需要修改 `config/gitlab.yml` 开启 emil

```shell
cd /home/git/gitlab
sed -i "s/email_enabled:.*/email_enabled: true/" config/gitlab.yml
sed -i "s/email_from:.*/email_from: noreply@example.com" config/gitlab.yml
sed -i "s/email_reply_to:.*/email_reply_to: noreply@example.com" config/gitlab.yml
cp config/initializers/smtp_settings.rb.sample config/initializers/smtp_settings.rb
```

将email启用后, 还需要配置smtp, 可以参考 [官方教程](https://docs.gitlab.com/omnibus/settings/smtp.html#mailcow), 修改配置文件 **config/initializers/smtp\_settings.rb**, 将 `ActionMailer::Base.smtp_settings` 修改为以下内容

```ruby
enable: true,
address: "mail.example.com",
port: 587,
user_name: "noreply@example.com",
password: "YourPassword",
domain: "mail.example.com",
authentication: :login,
enable_starttls_auto: true,
tls: false,
openssl_verify_mode: 'none'
```


### Nextcloud {#nextcloud}

配置邮箱服务器前需要先修改nextcloud的代码, 如下

```shell
cd /path/to/nextcloud
sed -i \
"s/\$streamContext = .*;/\$streamContext = stream_context_create(array('ssl'=>['verify_peer'=>false, 'verify_peer_name'=>false, 'allow_self_signed'=>true]));/" \
3rdparty/swiftmailer/swiftmailer/lib/classes/Swift/Transport/StreamBuffer.php
```

登录管理员帐号进行邮箱服务器配置即可

| 字段  | 值                   |
|-----|---------------------|
| 发送模式 | SMTP                 |
| 加密  | STARTTLS             |
| 来自地址 | noreply@example.com  |
| 认证方式 | 登录                 |
| 需要认证 | true                 |
| 服务器地址 | mail.example.com:587 |
| 证书  | noreply@example.com  |
| 密码  | YourPassword         |


## 推荐阅读 {#推荐阅读}

-   [Mailcow官方文档](https://mailcow.github.io/mailcow-dockerized-docs/)
-   [DMARC 是什么？](https://www.cnblogs.com/dmarcly/p/10947796.html)
-   [邮件服务器Poste五分钟搭建](https://newpants.top/2019/11/14/%E9%82%AE%E4%BB%B6%E6%9C%8D%E5%8A%A1%E5%99%A8Poste%E4%BA%94%E5%88%86%E9%92%9F%E6%90%AD%E5%BB%BA/)
-   [使用Mailcow自建邮件服务器](https://lala.im/4168.html)
-   [使用 mailcow:dockerized 搭建邮件服务器](https://low.bi/p/r7VbxEKo3zA)
-   [SPF 记录：原理、语法及配置方法简介](http://www.renfei.org/blog/introduction-to-spf.html)

