# 《Movie-Pilot部署与微信推送教程》

- [《Movie-Pilot部署与微信推送教程》](#movie-pilot部署与微信推送教程)
  - [一、配置VPS](#一配置vps)
  - [二、配置CookieCloud服务](#二配置cookiecloud服务)
  - [三、配置微信推送代理](#三配置微信推送代理)
  - [四、企业微信注册流程](#四企业微信注册流程)
  - [五、配置MoviePilot](#五配置moviepilot)
  - [六、配置frp相关服务](#六配置frp相关服务)
  - [七、后续操作](#七后续操作)

---

**MoviePilot**（以下简称***MP***）出现已经有一段时间了，网上的教程也非常多。MP相比于原版**NAStool**（以下简称***NT***）以及各NT发行版来说，运行速度更快，响应更快。本篇教程争取以最简短的方式来说明各环境变量应该如何按照自己意愿定制。

微信推送教程主要针对**没有公网IPv4**的情况，通过有公网IPv4地址的VPS用frp方式来固定MP的地址，以满足企业微信的固定公网的要求。

---

做一个简单说明

首先本篇教程是一篇全面配置教程，从0开始搭建自己的服务。但其实很多服务都是可以公用的，在可公用的部分我会做"可公用"标注。也就是说如果你有认识的小伙伴做过这些工作，那么你完全可以使用他的相关服务，不必自己搭建。

其次，为什么写一个双教程融合的文章呢，就是因为希望折腾一次后就不用再折腾了。如果你真的只想了解MP的配置教程，请跳转至第五节。

---

## 一、配置VPS

"可公用"说明：事实上只做页面/消息中转的话不会占用多少资源/带宽，完全可以10个人用一个。以我自己租的VPS来说，香港节点，50Mbps，25元/月。做转发很够用。

**本教程假定VPS的IP为：203.51.76.218**，注意这是一个**随便写的IP**，后续内容中需要替换为**自己的VPS地址**

VPS安装系统推荐使用Ubuntu 20/22系统。完成系统安装后以root身份登入VPS

```shell
# root身份
ssh root@203.51.76.218
# 在不支持root身份登入的VPS上
ssh 用户名203.51.76.218
```

输入密码即可。进入后为root则跳过此步骤，若不是则执行。

```shell
sudo -i
```

再次输入密码即可。在Shell中执行如下命令，更新VPS自带的软件包

```shell
apt-get update
apt-get full-upgrade
```

在VPS中安装Docker，首先安装必要的证书并允许apt通过HTTPS使用存储库

```shell
apt install apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release
```

然后，运行下列命令添加Docker官方的GPG密钥

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

添加Docker官方库

```shell
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
```

使用命令更新Ubuntu源列表并安装Docker

```shell
apt update
apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

启动Docker服务并将其设置为开机器自动

```shell
systemctl start docker
systemctl enable docker
```

使用Docker命令测试Docker是否安装正确

```shell
docker version
```

出现类似下列内容的则安装成功

```shell
Client: Docker Engine - Community
 Version:           24.0.7
 API version:       1.43
 Go version:        go1.20.10
 Git commit:        afdd53b
 Built:             Thu Oct 26 09:07:41 2023
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          24.0.7
  API version:      1.43 (minimum version 1.12)
  Go version:       go1.20.10
  Git commit:       311b9ff
  Built:            Thu Oct 26 09:07:41 2023
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.24
  GitCommit:        61f9fd88f79f081d64d6fa3bb1a0dc71ec870523
 runc:
  Version:          1.1.9
  GitCommit:        v1.1.9-0-gccaecfc
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

配置并启动Ubuntu防火墙，接下来我们每部署一个服务，都要为他打开对应的端口

```shell
# 先允许22通过防火墙，要不然SSH会断开
ufw allow 22
# 启动防火墙
ufw enable
# 查看防火墙状态
ufw status
```

这里说一下，建议大家给同一设备开通同一开头的端口。接下来的教程中端口为5开头的4位数，即5xxx均为NAS通过frp映射上来的端口；端口为9开头的4位数，即9xxx均为VPS自己配置的服务。

---

## 二、配置CookieCloud服务

"可公用"说明：自己搭建或使用MP公共CookieCloud服务或使用小伙伴搭建的均可

在云端配置自己的[CookieCloud](https://github.com/easychen/CookieCloud)服务，其实这里可以跳过，使用MP提供的公共CookieCloud服务，但总会有些人希望自己搭建，就一并写在教程里了。

Docker直接拉取

```shell
docker run -d \
    --name cookiecloud \
    --hostname cookiecloud \
    --restart=always \
    -p 9088:8088 \
    easychen/cookiecloud:latest
```

防火墙开放9088端口

```shell
ufw allow 9088
```

稍后我们将使用 http://203.51.76.218:9088 作为CookieCloud的服务地址，不自己搭建也可使用MP官方CookieCloud服务地址：[https://movie-pilot.org/cookiecloud](https://movie-pilot.org/cookiecloud)

其他具体内容，如在Edge上使用CookieCloud，可以看CookieCloud官方教程：[https://juejin.cn/post/7190963442017108027](https://juejin.cn/post/7190963442017108027)

---

## 三、配置微信推送代理

"可公用"说明：自己搭建或使用小伙伴搭建的转发地址均可

如果会配置Nginx的小伙伴可以使用MP官方推荐的设置

```nginx configuration
location /cgi-bin/gettoken {
    proxy_pass https://qyapi.weixin.qq.com;
}
location /cgi-bin/message/send {
    proxy_pass https://qyapi.weixin.qq.com;
}
location  /cgi-bin/menu/create {
    proxy_pass https://qyapi.weixin.qq.com;
}
```

如果不会的话，可以使用别人配置好的Docker容器

```shell
docker run -d \
    --name wxchat \
    --restart=always \
    -p 9080:80 \
    ddsderek/wxchat:latest
```

防火墙开放9080端口

```shell
ufw allow 9080
```

用此方法需要在配置MP时填写 http://203.51.76.218:9080 为微信代理地址

---

## 四、企业微信注册流程

本着不重复造轮子的原则，本节我是抄的，还请见谅

先登录下载个企业微信，登录要用，登录 [企业微信](https://work.weixin.qq.com/wework_admin/frame#profile)

开个记事本记下我说的这些参数

1. 企业 id

  ![1.png](https://img.pterclub.com/images/2023/11/03/16348ec32f8ea08e7.png)

2. 创建应用 拿 agentid ，secret

  ![2.png](https://img.pterclub.com/images/2023/11/03/2.png)

  <img src="https://img.pterclub.com/images/2023/11/03/3.jpg" alt="3.jpg" style="zoom:110%;" />

3. 设置可信 ip 同上页面最底部，填入 203.51.76.218

  <img src="https://img.pterclub.com/images/2023/11/03/4.png" alt="4.png" style="zoom:100%;" />

  **这里可能会存在潜在的冲突，如不设置API接收不可填入IP。如遇此中情况，就稍后来做这步。**

4. 设置接受API

  <img src="https://img.pterclub.com/images/2023/11/03/5.png" alt="5.png" style="zoom:100%;" />

  ![6.png](https://img.pterclub.com/images/2023/11/03/6.png)

  

这里URL填写：[http://203.51.76.218:5001/api/v1/message/?token=moviepilot](http://203.51.76.218:5001/api/v1/message/?token=moviepilot)

解释：5001是稍后通过frp将MP映射到VPS上的端口

这里会生成Token和EncodingAESKey，请记录下来

注意！这里点击保存是无法保存的，因为微信回调不通过，我们需要先把MP运行起来，页面不要关闭

此时，MP要求WECHAT_CORPID，WECHAT_APP_SECRET，WECHAT_APP_ID，WECHAT_TOKEN，WECHAT_ENCODING_AESKEY均已经收集完成，请填入后续MP命令行中。

---

## 五、配置MoviePilot

这是一份全量环境变量的MP配置，在需要修改的地方我都做了"需要修改"的标注，请按照要求修改。具体详情请参考[MoviePilot](https://github.com/jxxghp/MoviePilot)官方页面，需要注意的点在代码下方。

```shell
docker run -itd \
    --name MoviePilot \
    --hostname MoviePilot \
    --network=host \
    --restart=always \
    -v /share/NAS:/NAS \
    -v /share/Container/MoviePilot/config:/config \
    -v /share/Container/MoviePilot/core:/moviepilot \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    -e 'TZ=Asia/Shanghai' \
    -e 'NGINX_PORT=3000' \
    -e 'PORT=3001' \
    -e 'PUID=0' \
    -e 'PGID=0' \
    -e 'UMASK=000' \
    -e 'PROXY_HOST=' \
    -e 'MOVIEPILOT_AUTO_UPDATE=true' \
    -e 'MOVIEPILOT_AUTO_UPDATE_DEV=false' \
    -e 'SUPERUSER=需要修改' \
    -e 'SUPERUSER_PASSWORD=需要修改' \
    -e 'API_TOKEN=moviepilot' \
    -e 'TMDB_API_DOMAIN=api.tmdb.org' \
    -e 'TMDB_IMAGE_DOMAIN=image.tmdb.org' \
    -e 'WALLPAPER=tmdb' \
    -e 'SCRAP_METADATA=true' \
    -e 'SCRAP_SOURCE=themoviedb' \
    -e 'SCRAP_FOLLOW_TMDB=true' \
    -e 'TRANSFER_TYPE=link' \
    -e 'OVERWRITE_MODE=size' \
    -e 'LIBRARY_PATH=需要修改' \
    -e 'LIBRARY_MOVIE_NAME=Movie' \
    -e 'LIBRARY_TV_NAME=Teleplay' \
    -e 'LIBRARY_ANIME_NAME=Anime' \
    -e 'LIBRARY_CATEGORY=false' \
    -e 'COOKIECLOUD_HOST=需要修改' \
    -e 'COOKIECLOUD_KEY=需要修改' \
    -e 'COOKIECLOUD_PASSWORD=需要修改' \
    -e 'COOKIECLOUD_INTERVAL=60' \
    -e 'USER_AGENT=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.102 Safari/537.36' \
    -e 'SUBSCRIBE_MODE=spider' \
    -e 'SUBSCRIBE_RSS_INTERVAL=5' \
    -e 'SUBSCRIBE_SEARCH=true' \
    -e 'SEARCH_SOURCE=themoviedb' \
    -e 'AUTO_DOWNLOAD_USER=' \
    -e 'OCR_HOST=https://movie-pilot.org' \
    -e 'PLUGIN_MARKET=https://raw.githubusercontent.com/jxxghp/MoviePilot-Plugins/main/' \
    -e 'MESSAGER=wechat' \
    -e 'WECHAT_CORPID=需要更改' \
    -e 'WECHAT_APP_SECRET=需要更改' \
    -e 'WECHAT_APP_ID=需要更改' \
    -e 'WECHAT_TOKEN=需要更改' \
    -e 'WECHAT_ENCODING_AESKEY=需要更改' \
    -e 'WECHAT_ADMINS=' \
    -e 'WECHAT_PROXY=需要修改' \
    -e 'TELEGRAM_TOKEN=' \
    -e 'TELEGRAM_CHAT_ID=' \
    -e 'TELEGRAM_USERS=' \
    -e 'TELEGRAM_ADMINS=' \
    -e 'SLACK_OAUTH_TOKEN=' \
    -e 'SLACK_APP_TOKEN=' \
    -e 'SLACK_CHANNEL=' \
    -e 'SYNOLOGYCHAT_WEBHOOK=' \
    -e 'SYNOLOGYCHAT_TOKEN=' \
    -e 'DOWNLOAD_PATH=需要修改' \
    -e 'DOWNLOAD_MOVIE_PATH=需要修改/Movie' \
    -e 'DOWNLOAD_TV_PATH=需要修改/Teleplay' \
    -e 'DOWNLOAD_ANIME_PATH=需要修改/Anime' \
    -e 'DOWNLOAD_CATEGORY=false' \
    -e 'DOWNLOAD_SUBTITLE=false' \
    -e 'DOWNLOADER_MONITOR=true' \
    -e 'TORRENT_TAG=MOVIEPILOT' \
    -e 'DOWNLOADER=qbittorrent' \
    -e 'QB_HOST=需要修改' \
    -e 'QB_USER=需要修改' \
    -e 'QB_PASSWORD=需要修改' \
    -e 'QB_CATEGORY=false' \
    -e 'QB_SEQUENTIAL=true' \
    -e 'QB_FORCE_RESUME=false' \
    -e 'TR_HOST=需要修改' \
    -e 'TR_USER=需要修改' \
    -e 'TR_PASSWORD=需要修改' \
    -e 'MEDIASERVER=jellyfin' \
    -e 'EMBY_HOST=' \
    -e 'EMBY_API_KEY=' \
    -e 'JELLYFIN_HOST=需要修改' \
    -e 'JELLYFIN_API_KEY=需要修改' \
    -e 'PLEX_HOST=' \
    -e 'PLEX_TOKEN=' \
    -e 'MEDIASERVER_SYNC_INTERVAL=6' \
    -e 'MEDIASERVER_SYNC_BLACKLIST=' \
    -e 'AUTH_SITE=iyuu' \
    -e 'IYUU_SIGN=需要修改' \
    -e 'BIG_MEMORY_MODE=true' \
    -e 'MOVIE_RENAME_FORMAT={{title}}{% if year %} ({{year}}){% endif %}/{{title}}{% if year %} ({{year}}){% endif %}{% if part %}-{{part}}{% endif %}{% if videoFormat %} - {{videoFormat}}{% endif %}{{fileExt}}' \
    -e 'TV_RENAME_FORMAT={{title}}{% if year %} ({{year}}){% endif %}/Season {{season}}/{{title}} - {{season_episode}}{% if part %}-{{part}}{% endif %}{% if episode %} - 第 {{episode}} 集{% endif %}{{fileExt}}' \
    --log-driver "json-file" \
    --log-opt "max-size=5m" \
    jxxghp/moviepilot:latest

```

部分说明如下：

```shell
# （第5行）
--network=host \
```

也可以不使用host，使用：-p 3000:3000 -p 3001:3001映射web和api端口。

```shell
# （第6行）
-v /share/NAS:/NAS \
```

先描述一下我的文件目录格式，其中均为绝对路径。

![1.png](https://img.pterclub.com/images/2023/11/03/1.png)

其中在qb和tr中的映射为：-v /share/NAS:/NAS，所以qb的内部下载路径为：/NAS/Download。将Download和Media的上级目录映射进MP中，即：-v /share/NAS:/NAS，这里也是原先Nastool的推荐映射方式。

```shell
# （第7、8行）
-v /share/Container/MoviePilot/config:/config \
-v /share/Container/MoviePilot/core:/moviepilot \
```

左边需要改成你自己的路径，MP官方要求映射/config，/moviepilot是要存储浏览器内核，若不想每次更新都重新下载浏览器内核，推荐映射此文件夹。

```shell
# （第11、12行）
-e 'NGINX_PORT=3000' \
-e 'PORT=3001' \
```

默认的内部web端口和api端口，使用host模式但不希望占用3000或3001的话可更改，使用映射端口的话则不需要改

```shell
# （第19、20行）
-e 'SUPERUSER=需要修改' \
-e 'SUPERUSER_PASSWORD=需要修改' \
```

web的用户名和密码，想使用推送请使用强密码，大写+小写+数字

```shell
# （第30行）
-e 'LIBRARY_PATH=需要修改' \
```

这里要填MP内部的路径，也就是你映射进来的Media路径

```shell
# （第35、36、37行）
-e 'COOKIECLOUD_HOST=需要修改' \
-e 'COOKIECLOUD_KEY=需要修改' \
-e 'COOKIECLOUD_PASSWORD=需要修改' \
```

Cookiecloud的服务地址，用户key，端对端加密密码，要改成你自己的

```shell
# （第40行）
-e 'SUBSCRIBE_MODE=spider' \
```

这里spider是指去网站搜索订阅资源，rss是指从rss获取订阅资源，spider好像是24小时运行一次(猜的)，rss则受第41行设置影响

```shell
# （第47～54行）
-e 'MESSAGER=wechat' \
-e 'WECHAT_CORPID=需要更改' \
-e 'WECHAT_APP_SECRET=需要更改' \
-e 'WECHAT_APP_ID=需要更改' \
-e 'WECHAT_TOKEN=需要更改' \
-e 'WECHAT_ENCODING_AESKEY=需要更改' \
-e 'WECHAT_ADMINS=' \
-e 'WECHAT_PROXY=需要修改' \
```

这里是企业微信的相关内容。

```shell
# （第64～67行）
-e 'DOWNLOAD_PATH=需要修改' \
-e 'DOWNLOAD_MOVIE_PATH=需要修改/Movie' \
-e 'DOWNLOAD_TV_PATH=需要修改/Teleplay' \
-e 'DOWNLOAD_ANIME_PATH=需要修改/Anime' \
```

你qb当中的下载路径，注意这里是qb内部路径，按照上面所讲内容，我的配置是/NAS/Download

```shell
# （第73～75行）
-e 'QB_HOST=需要修改' \
-e 'QB_USER=需要修改' \
-e 'QB_PASSWORD=需要修改' \
```

qb的相关信息，HOST格式为ip:port

```shell
# （第79～81行）
-e 'TR_HOST=需要修改' \
-e 'TR_USER=需要修改' \
-e 'TR_PASSWORD=需要修改' \
```

tr相关信息，HOST格式为ip:port

```shell
# （第91、92行）
-e 'AUTH_SITE=iyuu' \
-e 'IYUU_SIGN=你的IYUU' \
```

IYUU认证信息，当然MP支持很多站点认证，但IYUU填写相对简单

修改这些信息后，将完整的内容复制到NAS的Shell中，即可运行MP。

2023年11月3日12:03，在我自己的466C上按照教程配置，测试通过，成功运行MP。

---

## 六、配置frp相关服务

"可公用"说明：frp的服务端，即frps可自己搭建或使用小伙伴搭建的转发地址均可，客户端frpc需要搭建在自己的NAS上

此时我们的MP还未固定IP地址，需要使用frp方式固定他的IP地址，方便企业微信回调（必须）

先配置frp的服务端frps.ini配置文件

```shell
mkdir -p /root/container/frp && cd /root/container/frp

cat << EOF > frps.ini
[common]
# 绑定端口
bind_port = 9000
# 启用面板
dashboard_port = 9001
# 面板登录名和密码,不建议使用admin/root这种用户名,并且一定要使用强密码
dashboard_user = admin
dashboard_pwd = password
# 使用http代理并使用9100端口进行穿透
vhost_http_port = 9100
# 使用https代理并使用9101端口进行穿透
vhost_https_port = 9101
# 服务token,相当于连接密码,一定要设置强密码
token = nidetoken
EOF
```

放行绑定端口、面板端口、http和https端口，此处只举例一个，另外几个相同操作

```shell
ufw allow 9000
```

拉取frps的Docker镜像，选择0.51.3是因为这是做后一个支持.ini配置的版本，后续都改成了.toml在网上教程当中出现的不多

```shell
docker run -d \
    --name frps \
    --restart=always \
    --network=host \
    -v /root/container/frp/frps.ini:/etc/frp/frps.ini \
    snowdreamtech/frps:0.51.3
```

此时服务端已经搭建完成了，接下来搭建客户端，要修改的部分，请按照要求修改，这里将下面的内容存储为frpc.ini并上传到NAS中容器配置所在的相关文件夹。例如，我使用的威联通TS-466C，我将Docker容器映射文件同一存储在/share/Container当中，所以我的上传位置为/share/Container/frp/frpc.ini

```shell
[common]
# vps的ip地址 要修改
server_addr = 203.51.76.218
# 刚才配置的frps的bind_port
server_port = 9000
# frps配置的token 要修改
token = nidetoken
[Movie-Pilot]
type = tcp
# 对数据进行加密
use_encryption = true
# 对数据进行压缩
use_compression = true
# MP的内网地址
local_ip = 192.168.8.4
# MP的内网端口
local_port = 3000
# 在VPS上映射的端口
remote_port = 5001
```

在NAS上拉取frp的客户端，同样需要拉取0.51.3版本

```shell
docker run -d \
    --name frpc \
    --restart=always \
    --network host \
    -v /share/Container/frp/frpc.ini:/etc/frp/frpc.ini \
    snowdreamtech/frpc:0.51.3
```

在VPS上放行映射端口

```shell
ufw allow 5001
```

此时你应该可以通过 [http://203.51.76.218:5001](http://203.51.76.218:5001) 访问到你的MP页面了。到这里就已经把MP映射到了公网，所以在MP配置时，应使用强密码，防止别人猜中。

---

## 七、后续操作

此时你的MP已经被映射到公网，可以回到API接受消息点保存了，你会发现保存正常。若之前没有设置成功可信IP，此时你应该是能设置成功了。

进入企业微信，发现创建的应用底部出现：站点、管理、订阅，点击同步站点，机器人应该会回复：开始执行 同步站点…

此时，恭喜你完成了MP的部署与wechat消息推送的部署！

---

作者：hjfzzm，协助：woshitao，转载请标明出处，谢谢！