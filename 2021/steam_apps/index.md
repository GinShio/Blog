# 使用 steamcmd 搭建游戏服务器


和好友联机的时候本地服务器实在是不爽，一个人起飞，其他人都是 **高PING战士** ，最开始主要是 L4D2 时各种 RPG 服务器有些不爽，为了纯净的服务器只好自己建了

事先声明，我们所有的操作在 Debian / Ubuntu 下操作，有些操作系统可能会不一样，不过大同小异，我们还是定义一些等等可能用到的变量 (主要是路径和密码之类的

```shell
steam=/home/steam
group_id=123456789
l4d2=${steam}/l4d2
l4d2_server_name="L4D2 Server"
l4d2_port=1024
valheim=${steam}/valheim
valheim_server_name="Valheim Server"
valheim_world="World"
valheim_port=1024
valheim_passwd=valheim_password
```


## SteamCMD {#steamcmd}

顾名思义，steamcmd 是一个命令行工具，同时支持 linux，是我们搭建服务器的好帮手，然而我不会用，不过这不重要，安装跑起来就好

```shell
add-apt-repository multiverse
dpkg --add-architecture i386
apt update && apt upgrade
apt install -y lib32gcc1 steamcmd
```

我们不仅要安装一个 steamcmd，还要将所有游戏服务器，存放在 `~steam` 下，使用 steam 这个用户来运行游戏

```shell
adduser --disabled-login --gecos 'Steam' steam
sudo -u steam -H ln -s /usr/games/steamcmd ${steam}/steamcmd
```


### 语法 {#语法}

这里说的语法并不是 SteamCMD 的语法，而是 Steam 中所使用的文本标记语法，这些标记标签允许您为您的留言及发帖文字添加格式，类似于 HTML

| 标签    | 释义                                | 示例                                     |
|-------|-----------------------------------|----------------------------------------|
| h       | 标题                                | [h1]一级标题[/h1]                        |
| b       | 粗体                                | [b]粗体文本[/b]                          |
| u       | 下划线                              | [u]下划线文本[/u]                        |
| l       | 斜体                                | [l]斜体[/l]                              |
| strike  | 删除线                              | [strike]删除线文本[/strike]              |
| spoiler | 隐藏                                | [spoiler]隐藏文本[/spoiler]              |
| noparse | 不解析                              | [noparse]不解析[b]标签[/b][/noparse]     |
| url     | 网络链接                            | [url=blog.ginshio.org]博客[/url]         |
|         | Youtube / 商店 / 创意工坊 URL 自动生成 Widget | <https://store.steampowered.com/app/570> |
| list    | 无序列表                            | [list] [\*] 列表 [\*] 列表 [/list]       |
| olist   | 有序列表                            | [olist] [\*] 列表 [\*] 列表 [/olist]     |
| quote   | 引用                                | [quote=author]引用文本[/quote]           |
| code    | 等宽字体 (保留空格)                 | [code]code[/code]                        |
| table   | 表格                                |                                          |


### 遇到的问题 {#遇到的问题}

如果遇到服务器无法启动，比如缺少 `sdk32/libsteam.so` 或 `sdk32/steamclient.so` 之类的错误，需要完成以下操作

```shell
sudo -u steam -H mkdir -p /home/steam/.steam/sdk32
sudo -u steam -H ln -s /home/steam/.steam/steamcmd/linux32/steamclient.so /home/steam/.steam/sdk32/steamclient.so
```

---


## L4D2 {#l4d2}

在搭建服务器之前，为了你的服务器着想，请先创建一个 Steam 组，将你们一起开黑的人都拉入组中，我们需要将服务器设置为组私有，只有组中的人才能使用

安装并对服务器进行配置，配置文件是 `${l4d2}/left4dead2/cfg/server.cfg`

```shell
sudo -u steam -H ${steam}/steamcmd +login anonymous +force_install_dir ${l4d2} +app_update 222860 validate +quit
sudo -u steam -H cat <<- EOF > ${l4d2}/left4dead2/cfg/server.cfg
hostname "${l4d2_server_name}"       // 服务器名
hostport ${l4d2_port}                // 服务器端口
sv_tags "hidde,GinShio"              // 隐藏服务器
sv_gametypes "versus,survival,coop,realism,teamversus,teamscavenge" // 游戏类型
mp_gamemode "coop"
sv_cheats 0                          // 允许作弊
sv_voiceenable 1                     // 语音服务
sv_pausable 0                        // 暂停
sv_consistency 0                     // 一致性
sv_lan 0                             // 局域网游戏
sv_allow_lobby_connect_only 0        // 大厅连接
sv_region 4                          // 区域：亚洲
sv_visiblemaxplayers 4               // 最多人数上线：4 (4~32)
mp_disable_autokick 0                // 闲置踢出
sv_steamgroup "${group_id}"          // 根据自己的steam组ID绑定服务器
sv_steamgroup_exclusive 1            // 设置组私有化
exec banned_user.cfg
exec banned_ip.cfg
heartbeat
EOF
```

对于服务器公告， `${l4d2}/left4dead2/host.txt` 可以修改服务器公告的标题，而 `${l4d2}/left4dead2/motd.txt` 修改的是公告的内容，我们可以在公告中使用之前列出的文本标记语法。对于第三方地图，我们只需要将地图存放在 `${l4d2}/left4dead2/addons/workshop` 中即可，不过记得将地图文件的权限转到 steam

个人喜欢使用 systemctl 来管理服务，这样觉得更安全些，所以我们完成一个 service 管理服务器，当然如果你想用 screen 可以自行搜索它的用法

```nil
[Unit]
Description=Left 4 Dead 2 Server
Documentation=https://left4dead.fandom.com/wiki/Left_4_Dead_Wiki
After=network.target

[Service]
Type=simple
User=steam
WorkingDirectory=/home/steam/l4d2
ExecStart=/home/steam/l4d2/srcds_run -game left4dead2 -autorestart +ip 0.0.0.0 +exec server.cfg
ExecReload=/bin/kill -HUP
Restart=on-failure
RestartPreventExitStatus=23

[Install]
WantedBy=multi-user.target
```

到这里，纯净的 l4d2 服务器就搭建好了，我并没有弄插件，也没有弄管理员，所以就到这里！

---


## Valheim: 英灵神殿 {#valheim-英灵神殿}

首先我们配置服务器，不过和 L4D2 的方式差不多，毕竟都是 SteamCMD，不废话，直接上 Shell 指令。需要注意的是，Valheim 将占用三个端口，即 `valheim_port` 到 `valheim_port + 2` ，请在防火墙开启需要的全部三个端口

```shell
sudo -u steam -H ${steam}/steamcmd +login anonymous +force_install_dir ${valheim} +app_update 896660 validate +quit
sudo -u steam -H cp ${valheim}/start_server.sh ${valheim}/start_server.sh.bkp
sudo -u steam -H cat <<- EOF > ${valheim}/start_server.sh
export templdpath=${LD_LIBRARY_PATH}
export LD_LIBRARY_PATH=${valheim}/linux64:${LD_LIBRARY_PATH}
export SteamAppId=892970
echo "Starting server PRESS CTRL-C to exit"
${valheim}/valheim_server.x86_64 -name "${valheim_server_name}" -port ${valheim_port} -world "${valheim_world}" -password "${valheim_passwd}"
export LD_LIBRARY_PATH=$templdpath
EOF
```

搞定，整一个 service 让 systemd 帮我们管理服务器

```nil
[Unit]
Description=Valheim Server
After=network.target

[Service]
Type=simple
User=steam
WorkingDirectory=/home/steam/valheim
ExecStart=bash /home/steam/valheim/start_server.sh
ExecReload=/bin/kill -HUP
Restart=on-failure
RestartPreventExitStatus=23

[Install]
WantedBy=multi-user.target
```

好了，现在问题就是，你的服务器配置，请务必 **2C4G5Mbps** 及以上配置，要求真tm高，听说开发者只有5个人，，，

