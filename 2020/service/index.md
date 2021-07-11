# 在服务器上部署一些服务


个人使用的是腾讯云的轻量服务器，系统镜像选择的是 Ubuntu 20.04，搭建的服务有 博客 [HUGO](https://gohugo.io/) 、私有网盘 [Nextcloud](https://nextcloud.com/) 以及 Git服务器 [GitLab](https://about.gitlab.com/)

一下服务搭建时，域名统一使用 example.com，请根据自己的情况修改对应的配置，用到一些基础依赖请自行安装

-   Nginx
-   Git
-   PHP
-   PostgreSQL
-   Redis

我们首先定义一些变量，以便后边使用和修改

```shell
fpm_path="/etc/php/7.4/fpm" # php-fpm 根目录
nextcloud_path="/path/to/nextcloud" # nextcloud 所在目录
nextcloud_host="example.com"
nextcloud_db_user="nextcloud" # Nextcloud 数据库用户
nextcloud_db_passwd="YourPassword" # Nextcloud 数据库密码
nextcloud_db_name="nextcloud" # Nextcloud 数据库名称
gitlab_path="/home/git/gitlab"
gitaly_path="/home/git/gitaly"
gitlab_host="example.com"
gitlab_db_passwd="YourPassword" # GitLab 数据库密码
lychee_path="/path/to/lychee" # Lychee 所在目录
lychee_version="v4.0.8" # Lychee 版本
lychee_host="example.com"
lychee_db_user="lychee" # Lychee 数据库用户
lychee_db_passwd="YourPassword" # Lychee 数据库密码
lychee_db_name="lychee" # Lychee 数据库名称
```

---


## Nextcloud {#nextcloud}

Nextcloud 是 [ownCloud](https://owncloud.com/) 项目的一个分支，一个开源的私有云盘应用，官方提供了包括 桌面以及移动系统的客户端。


### Nextcloud 依赖 {#nextcloud-依赖}

Nextcloud 依赖 PHP 运行时以及数据库 (MySQL 5.7+ / MariaDB 10.2+ 或 PostgreSQL)

```shell
apt install -y nginx postgresql \
    php php-fpm php-cli php-mysql php-pgsql php-sqlite3 php-redis \
    php-apcu php-memcached php-bcmath php-intl php-mbstring php-json php-xml \
    php-curl php-imagick php-gd php-zip php-gmp php-ctype php-dom php-iconv php-zlib
```

-   PHP: 修改 php-fpm 的配置文件

    ```shell
    cd ${fpm_path}
    # php config
    sed -i "s/memory_limit = .*/memory_limit = 512M/" php.ini
    sed -i "s/;date.timezone.*/date.timezone = UTC/" php.ini
    sed -i "s/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=1/" php.ini
    sed -i "s/upload_max_filesize = .*/upload_max_filesize = 4096M/" php.ini
    sed -i "s/post_max_size = .*/post_max_size = 4096M/" php.ini
    sed -i "s/max_input_time = .*/max_input_time = 480/" php.ini
    sed -i "s/max_execution_time = .*/max_execution_time = 360/" php.ini
    sed -i "s/pm.max_children = .*/pm.max_children = 32/" pool.d/www.conf
    sed -i "s/pm.min_spare_servers = .*/pm.min_spare_servers = 1/" pool.d/www.conf
    sed -i "s/pm.max_spare_servers = .*/pm.max_spare_servers = 8/" pool.d/www.conf
    sed -i "s/pm.start_servers = .*/pm.start_servers = 4/" pool.d/www.conf
    sed -i "s/;clear_env = no/clear_env = no/" pool.d/www.conf
    # opcache config
    sed -i "s/;opcache.enable=1/opcache.enable=1/" php.ini
    sed -i "s/;opcache.memory_consumption=128/opcache.memory_consumption=128/" php.ini
    sed -i "s/;opcache.interned_strings_buffer=8/opcache.interned_strings_buffer=8/" php.ini
    sed -i "s/;opcache.max_accelerated_files=10000/opcache.max_accelerated_files=10000/" php.ini
    sed -i "s/;opcache.revalidate_freq=2/opcache.revalidate_freq=1/" php.ini
    # apc config
    echo "[apc]" >> php.ini
    echo "apc.cache_by_default = on" >> php.ini
    echo "apc.enable_cli = off" >> php.ini
    echo "apc.enable = on" >> php.ini
    echo "apc.file_update_protection = 2" >> php.ini
    # restart
    systemctl restart php7.4-fpm
    ```

-   PostgreSQL：创建用户 **nextcloud** 和数据库 **nextcloud\_db**

    ```shell
    sudo adduser --disabled-login --gecos 'Nextcloud' ${nextcloud_db_user}
    sudo -u postgres -H psql -c "CREATE USER ${nextcloud_db_user} WITH PASSWORD '${nextcloud_db_passwd}'"
    sudo -u postgres -H psql -c "CREATE DATABASE ${nextcloud_db_name} OWNER ${nextcloud_db_user}"
    ```

-   Nginx：配置网站，修改 [官方示例配置文件](https://docs.nextcloud.com/server/20/admin%5Fmanual/installation/nginx.html#nextcloud-in-the-webroot-of-nginx) 并保存在 `/etc/nginx/sites-available/nextcloud`

    ```shell
    ln -sf /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/
    nginx -t # 检查配置文件
    systemctl restart nginx
    ```


### Nextcloud 安装 {#nextcloud-安装}

[下载](https://download.nextcloud.com/server/releases) 你需要的版本并解压到目录中

```shell
sudo chown -R www-data:www-data ${nextcloud_path}
sudo -u www-data -H mkdir -p ${nextcloud_path}
```

准备工作完成后，进入网页，设置管理员帐号和数据库。


### Nextcloud 开启邮件服务 {#nextcloud-开启邮件服务}

配置邮箱服务器前需要先修改nextcloud的代码，如下

```shell
cd /path/to/nextcloud
sed -i \
"s/\$streamContext = .*;/\$streamContext = stream_context_create(array('ssl'=>['verify_peer'=>false, 'verify_peer_name'=>false, 'allow_self_signed'=>true]));/" \
3rdparty/swiftmailer/swiftmailer/lib/classes/Swift/Transport/StreamBuffer.php
systemctl restart php7.4-fpm
```

登录管理员帐号进行邮箱服务器配置即可

| 字段  | 值                   |
|-----|---------------------|
| 发送模式 | SMTP                 |
| 加密  | SSL/TLS              |
| 来自地址 | noreply@example.com  |
| 认证方式 | 登录                 |
| 需要认证 | true                 |
| 服务器地址 | mail.example.com:465 |
| 证书  | noreply@example.com  |
| 密码  | YourPassword         |


### Nextcloud 插件 {#nextcloud-插件}

Nextcloud 不仅自带了很多功能，也提供了插件用于扩展功能，官方称作 Apps，以下个人推荐一些插件，欢迎补充

-   **Announcement center** ：公告发布
-   **Registration** ：注册功能
-   **File access control** ：文件访问控制，可以添加规则来控制管理用户对文件的操作，参见 [工作流](https://nextcloud.com/workflow)
-   **Music** ：音乐播放器
-   **Full text search** ：全文搜索

---


## GitLab {#gitlab}

GitLab 是开源的基于git的 web **DevOps生命周期工具** ，提供了 <span class="underline">Git仓库</span> 、 <span class="underline">问题追踪</span> 和 <span class="underline">CI/CD</span> 等功能。分为社区版和企业版，使用相同内核，部分功能社区版没有提供。Gitlab 相较消耗资源，官方推荐的最低要求为 **4C4G** 可以最多支持500用户， **8C8G** 最多支持1000用户，具体的使用受到 <span class="underline">用户的活跃程度</span> 、 <span class="underline">CI/CD</span> 、 <span class="underline">修改大小</span> 等因素影响。

由于暂时不需要，没有安装 Gitlab Pages，Gitlab的安装依赖 git 用户，以下是目录结构

```nil
|-- home
|    |-- git
|        |-- .ssh
|        |-- gitaly
|        |-- gitlab
|        |-- gitlab-shell
|        |-- gitlab-workhorse
|        |-- repositories
```


### Gitlab 组件 {#gitlab-组件}

{{< figure src="/blog/Applications/gitlab-architecture-simplified.png" >}}

-   Gitaly：处理所有的 git 操作
-   GitLab Shell：处理基于 SSH 的 git 会话 与 SSH密钥
-   GitLab Workhorse：反向代理服务器，处理与Rails无关的请求，Git Pull/Push 请求 和 到Rails的连接，减轻Web服务的压力，帮助整体加快Gitlab的速度
-   Unicorn / Puma：Gitlab 自身的 Web 服务器，提供面向用户的功能，Gitlab 13.0 起默认使用 Puma
-   Sidekiq：后台任务服务器，从Redis队列中提取任务并进行处理
-   GitLab Pages：允许直接从仓库发布静态网站
-   Gitlab Runner：Gitlab CI/CD 所关联的任务处理器
-   Nginx：Web服务器
-   PostgreSQL：数据库


### Gitlab 依赖 {#gitlab-依赖}

目前Gitlab最新版本为 **v13.x** ，我们安装最新版本

-   Ruby v2.6 or later
-   GoLang v1.13 or later
-   Git v2.24 (推荐 v2.28) or later
-   Node.js v10.13.0 (推荐 v12) or later
-   yarn v1.10.0 or later
-   Nginx
-   Redis v4.0 (推荐v5.0) or later
-   PostgreSQL v11 or later

安装相关依赖，建立数据库，将 Redis 设置为 Unix Domain Socket (UDS) 连接

```shell
apt update -y && apt upgrade -y
# 安装依赖
sudo apt install -y build-essential zlib1g-dev libyaml-dev libssl-dev libgdbm-dev libre2-dev \
  libreadline-dev libncurses5-dev libffi-dev curl openssh-server checkinstall libxml2-dev \
  libxslt-dev libcurl4-openssl-dev libicu-dev logrotate rsync python-docutils pkg-config \
  cmake vim runit postfix libimage-exiftool-perl golang nodejs
# Ruby
sudo gem install bundler --no-document --version '< 2'
# Node.js
curl --silent --show-error https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | \
    sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt-get update
sudo apt-get install yarn
# 数据库
sudo adduser --disabled-login --gecos 'GitLab' git
sudo -u postgres psql -d template1 -c "CREATE USER git WITH PASSWORD '${gitlab_db_passwd}' CREATEDB;"
sudo -u postgres psql -d template1 -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;"
sudo -u postgres psql -d template1 -c "CREATE EXTENSION IF NOT EXISTS btree_gist";
sudo -u postgres psql -d template1 -c "CREATE DATABASE gitlabhq_production OWNER git;"
# Redis
sudo apt install redis-server
sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.orig
sudo sed 's/^port .*/port 0/' /etc/redis/redis.conf.orig | sudo tee /etc/redis/redis.conf
echo 'unixsocket /var/run/redis/redis.sock' | sudo tee -a /etc/redis/redis.conf
echo 'unixsocketperm 770' | sudo tee -a /etc/redis/redis.conf
sudo mkdir -p /var/run/redis
sudo chown redis:redis /var/run/redis
sudo chmod 755 /var/run/redis
if [ -d /etc/tmpfiles.d ]; then
  echo 'd  /var/run/redis  0755  redis  redis  10d  -' | sudo tee -a /etc/tmpfiles.d/redis.conf
fi
sudo systemctl restart redis
sudo usermod -aG redis git
```


### Gitlab 安装 {#gitlab-安装}

以下配置完成后，Gitlab基本配置完成，登录网站设置默认管理员密码即可登录，默认管理员帐号为 **root**

```shell
# install gitlab
cd /home/git
sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-foss.git -b 13-0-stable gitlab
cd ${gitlab_path}
sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml
sudo -u git -H editor config/gitlab.yml
sudo -u git -H cp config/secrets.yml.example config/secrets.yml
sudo -u git -H chmod 0600 config/secrets.yml
sudo chown -R git log/
sudo chown -R git tmp/
sudo chmod -R u+rwX,go-w log/
sudo chmod -R u+rwX tmp/
sudo chmod -R u+rwX tmp/pids/
sudo chmod -R u+rwX tmp/sockets/
sudo -u git -H mkdir -p public/uploads/
sudo chmod 0700 public/uploads
sudo chmod -R u+rwX builds/
sudo chmod -R u+rwX shared/artifacts/
sudo chmod -R ug+rwX shared/pages/
sudo -u git -H cp config/puma.rb.example config/puma.rb
sudo -u git -H editor config/puma.rb
sudo -u git -H git config --global core.autocrlf input
sudo -u git -H git config --global gc.auto 0
sudo -u git -H git config --global repack.writeBitmaps true
sudo -u git -H git config --global receive.advertisePushOptions true
sudo -u git -H git config --global core.fsyncObjectFiles true
sudo -u git -H cp config/resque.yml.example config/resque.yml
sudo -u git -H editor config/resque.yml
sudo -u git cp config/database.yml.postgresql config/database.yml
sudo -u git -H editor config/database.yml
sudo -u git -H chmod o-rwx config/database.yml
# install gems
sudo -u git -H bundle install --deployment --without development test mysql aws kerberos
# install gitlab-shell
sudo -u git -H bundle exec rake gitlab:shell:install RAILS_ENV=production
sudo -u git -H editor /home/git/gitlab-shell/config.yml
# install gitlab-workhorse
sudo -u git -H bundle exec rake "gitlab:workhorse:install[/home/git/gitlab-workhorse]" RAILS_ENV=production
# install gitaly
cd ${gitlab_path}
sudo -u git -H bundle exec rake \
    "gitlab:gitaly:install[/home/git/gitaly,/home/git/repositories]" RAILS_ENV=production
sudo chmod 0700 ${gitlab_path}/tmp/sockets/private
sudo chown git ${gitlab_path}/tmp/sockets/private
sudo -u git -H editor ${gitaly_path}/config.toml
sudo -u git -H sh -c \
    "${gitlab_path}/bin/daemon_with_pidfile ${gitlab_path}/tmp/pids/gitaly.pid ${gitaly_path}/gitaly ${gitaly_path}/config.toml >> ${gitlab_path}/log/gitaly.log 2>&1 &"
# 初始化
sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production force=yes
sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab
sudo cp lib/support/init.d/gitlab.default.example /etc/default/gitlab
sudo update-rc.d gitlab defaults 21
sudo systemctl enable gitlab
sudo cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab
sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production
# GetText PO files
sudo -u git -H bundle exec rake gettext:compile RAILS_ENV=production
# Assets
sudo -u git -H yarn install --production --pure-lockfile
sudo -u git -H bundle exec rake gitlab:assets:compile \
    RAILS_ENV=production NODE_ENV=production \
    NODE_OPTIONS="--max_old_space_size=1024" # 内存限制在1G
if [ $? != 0 ]; then
    echo "compile assets error. desc '--max_old_space_size'"
    return 64
fi
# Nginx
sudo cp lib/support/nginx/gitlab-ssl /etc/nginx/sites-available/gitlab
sudo ln -sf /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/gitlab
sudo editor /etc/nginx/sites-available/gitlab
sudo nginx -t; if [ $? != 0 ]; then
    echo "nginx config error. editor /etc/nginx/sites-available/gitlab"
    return 64
fi
# end
sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production
sudo systemctl start gitlab
```

接下来安装 `Gitlab Runner` v13.4.1，使 CI/CD 可用

```shell
curl -LJO https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb
dpkg -i gitlab-runner_amd64.deb
sudo gitlab-runner register # 注册 runner
sudo -u gitlan-runner -H mv /home/gitlab-runner/.bash_logout /home/gitlab-runner/.bash_logout.bkp
sudo systemctl restart gitlab-runnner
sudo systemctl enable gitlab-runner
```


### GitLab 开启邮件服务 {#gitlab-开启邮件服务}

我们的GitLab使用的是源码安装，需要修改 `config/gitlab.yml` 开启 emil

```shell
gitlab_email_from="noreply@example.com"
gitlab_email_reply="noreply@example.com"
cd /home/git/gitlab
sed -i "s/email_enabled:.*/email_enabled: true/" config/gitlab.yml
sed -ie "s/email_from:.*/email_from: ${gitlab_email_from}" config/gitlab.yml
sed -ie "s/email_reply_to:.*/email_reply_to: ${gitlab_email_reply}" config/gitlab.yml
cp config/initializers/smtp_settings.rb.sample config/initializers/smtp_settings.rb
```

将email启用后，还需要配置smtp，可以参考 [官方教程](https://docs.gitlab.com/omnibus/settings/smtp.html#mailcow)，修改配置文件 **config/initializers/smtp\_settings.rb** ，将 `ActionMailer::Base.smtp_settings` 修改为以下内容

```ruby
enable: true,
address: "mail.example.com",
port: 465,
user_name: "noreply@example.com",
password: "YourPassword",
domain: "mail.example.com",
authentication: :login,
enable_starttls_auto: true,
tls: true,
openssl_verify_mode: 'none'
```

开启对邮件的 S/MIME 签名服务，将你的S/MIME私钥保存到 `${gitlab\_path}/.gitlab\_smime\_key=，公钥保存到 =${gitlab_path}/.gitlab_smime_cert`

```shell
sed -i "103s/# enabled:.*/enabled: true/" config/gitlab.yml
```

配置完成后重启服务即可，如果需要验证SMTP是否工作，可以使用以下命令

```shell
echo "Notify.test_email('${gitlab_email_reply}', 'Message Subject', 'Message Body').deliver_now" | \
sudo -u git -H bundle exec rails console -e production
```

---


## Lychee {#lychee}

Lychee 现在是由 LycheeOrg 维护的开源项目，旨在实现一个简单易用的照片管理系统，我们搭建服务将使用 4.x 版本作为示例

Lychee 是目前为数不多的支持 PostgreSQL 的图床，不过遗憾的是不支持 SVG 图片


### Lychee 依赖 {#lychee-依赖}

由于 Lychee 同样也是 PHP 开发的服务，所以已经安装 Nextcloud 的情况下，并不需要再安装 PHP 相关的其他 package (当然除了 PHP 的依赖管理器 composer)

PHP >= 7.4，依赖的扩展 (可以使用命令 `php -m` 查看已安装的扩展)

-   BCMath
-   Ctype
-   Exif
-   Ffmpeg (optional — to generate video thumbnails)
-   Fileinfo
-   GD
-   Imagick (optional — to generate better thumbnails)
-   JSON
-   Mbstring
-   OpenSSL
-   PDO
-   Tokenizer
-   XML
-   ZIP


### Lychee 安装 {#lychee-安装}

我们先创建数据库用户，Lychee 支持 MySQL (> 5.7.8) / MariaDB (> 10.2) / PostgreSQL (> 9.2)，我们继续使用 PostgreSQL 就好

```shell
sudo adduser --disabled-login --gecos 'Lychee' ${lychee_db_user}
sudo -u postgres -H psql -c "CREATE USER ${lychee_db_user} WITH PASSWORD '${lychee_db_passwd}'"
sudo -u postgres -H psql -c "CREATE DATABASE ${lychee_db_name} OWNER ${lychee_db_user}"
```

我们将安装 v4.0.8，更多详细版本信息请浏览 [更新日志](https://lycheeorg.github.io/docs/releases.html)

```shell
git clone https://www.github.com/LycheeOrg/Lychee -b ${lychee_version} ${lychee_path}
cd ${lychee_path}
composer install --no-dev
chown -R www-data:www-data ${lychee_path}
```

关于 Web 服务器的配置官方已经给出了 [Nginx](https://lycheeorg.github.io/docs/#nginx) 和 [Apache](https://lycheeorg.github.io/docs/#apache) 的相关配置


### Lychee 配置 {#lychee-配置}

Lychee 相关的环境配置在 `.env` 中

```shell
cat <<- EOF > ${lychee_path}/.env
APP_NAME=Lychee
APP_ENV=production
APP_DEBUG=false
APP_URL=http://${lychee_host}
APP_KEY=

DEBUGBAR_ENABLED=false
LOG_CHANNEL=stack

DB_CONNECTION=pgsql
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=${lychee_db_name}
DB_USERNAME=${lychee_db_user}
DB_PASSWORD=${lychee_db_passwd}
DB_LOG_SQL=false

TIMEZONE=Asia/Shanghai

BROADCAST_DRIVER=log
CACHE_DRIVER=file
SESSION_DRIVER=file
SESSION_LIFETIME=120
QUEUE_DRIVER=sync

REDIS_HOST=/var/run/redis/redis.sock
REDIS_PASSWORD=null
REDIS_PORT=0

MAIL_DRIVER=smtp
MAIL_HOST=
MAIL_PORT=
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_ENCRYPTION=

PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_APP_CLUSTER=mt1

MIX_PUSHER_APP_KEY="\${PUSHER_APP_KEY}"
MIX_PUSHER_APP_CLUSTER="\${PUSHER_APP_CLUSTER}"
EOF
chown www-data:www-data ${lychee_path}/.env
sudo -u www-data php artisan key:generate
```

最后只需要配置 Nginx 相关内容即可

