---
layout:     post
title:      "使用 bitwarden_rs 搭建自托管的密码服务器"
subtitle:   " \"Self hosting Password Managerwith bitarden_rs\""
date:       2019-10-27 00:00:00
author:     "yinyeya"
header-img: "img/in-post/2019-05-31-Hello-world/Start-My-Journey.jpg"
catalog: true
tags:
    - 计算机技术
    - 密码管理
    - Docker
---

![Locked With Key on Docomo 2013](https://emojipedia-us.s3.dualstack.us-west-1.amazonaws.com/thumbs/160/docomo/205/closed-lock-with-key_1f510.png)

**bitwarden_rs** 是用RUST开发的第三方Bitwarden密码管理器，相比**bitwarden**官方版本的主要优势有：

1. 大幅降低了对机器配置的要求（官方版本至少3GB内存，**bitwarden_rs** 只需要512M）；提供了 Docker 镜像并且体积很小，部署简单。
2. 官方服务器中需要付费订阅的一些功能，在这个实现中是免费的。
3. 如果按照配合其[GitHub最新教程搭建](https://github.com/dani-garcia/bitwarden_rs/wiki/Using-Docker-Compose)， 配合golang 开发的一个极简的HTTP服务器 **[Caddy](https://www.zybuluo.com/zwh8800/note/844776)**进行反向代理后，更是可以自动续期SSL证书。

具体部署流程如下。

# 部署要求
准备一个域名并解析到你的VPS服务器。

# 安装docker
### 卸载老版本（若未安装过可省略此步）
```bash
$ sudo apt-get remove docker docker-engine 
```
### 安装最新的docker （需要root权限）
```bash
$ curl -sSL https://get.docker.com/ | sh
$ systemctl start docker
$ systemctl enable docker
```
shell会提示你输入sudo的密码，然后开始执行最新的docker过程

### 确认Docker成功最新的docker
```bash
$ sudo docker run hello-world
```
# 安装docker-compose
docker-compose是一个用来**定义和运行多个 Docker 容器的应用（Defining and running multi-container Docker applications）**。经常会碰到需要多个容器相互配合来完成某项任务的情况。例如要实现一个典型的Web 项目，至少需要一个Web 服务端容器，还需要再加上后端的数据库服务容器，甚至还包括负载均衡容器等。docker 提供了一个命令行工具 docker-compose 帮助完成容器的编排，它允许用户通过编写一个`docker-compose.yml` 模板文件（[YAML](http://einverne.github.io/post/2015/08/yaml.html) 格式）来针对特定项目，定义一组相关联的应用容器。
docker-compose相关的内容和教程：

> [Compose 简介](https://yeasy.gitbooks.io/docker_practice/compose/introduction.html)                                                                                                                                                                     [使用 docker-compose 替代 docker run](https://beginor.github.io/2017/06/08/use-compose-instead-of-run.html)                                                                                                                                [docker-compose教程（安装，使用, 快速入门）](https://blog.csdn.net/pushiqiang/article/details/78682323)

## 从github上下载docker-compose二进制文件安装
### 下载最新版的docker-compose文件
```bash
$ sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
````
如果要更新 docker-compose，也可以直接使用这条命令，只需要修改版本号为最新的 docker-compose 版本号。

### 添加可执行权限
```bash
$ sudo chmod +x /usr/local/bin/docker-compose
```
### 测试安装结果
```bash
$ docker-compose --version
docker-compose version 1.16.1, build 1719ceb
```
## 放行80/443端口
绝大部分服务器都有内置防火墙并默认开启。因为配置**bitwarden_rs**服务端需要对服务器的http80/https443端口访问，因此要在防火墙下开启服务器的http/https端口。如果服务器安装了宝塔面板，直接在安全选项里放行即可。如果服务器没有可视化的控制面板，可通过以下代码实现（以CentOS 7的防火墙为例，对于CentOS的iptables，Ubuntu的UFW或者其他图形界面管理工具等防火墙的设置方法，可上网查询开启端口的相应命令）：
```bash
$ firewall-cmd --permanent --add-service=http
$ firewall-cmd --permanent --add-service=https

# 重新载入，重启防火墙
$ firewall-cmd --reload
$ systemctl restart firewalld.service
  
# 可以通过 tenlet 命令测试端口是否开启成功
tenlet IP Port
```
# 配置bitwarden_rs

### 新建配置文件`docker-compose.yml`
```bash
$ cd ~ && mkdir bitwarden && cd bitwarden && vim ~/bitwarden/docker-compose.yml
```
写入如下配置

```yaml
version: "3"
    
services:
  bitwarden:
    image: bitwardenrs/server
    restart: always
    volumes:
      - ./bw-data:/data
    environment:
      WEBSOCKET_ENABLED: "true"
      SIGNUPS_ALLOWED: "true"
    
  caddy:
    image: abiosoft/caddy
    restart: always
    volumes:
      - ./Caddyfile:/etc/Caddyfile:ro
      - caddycerts:/root/.caddy
    ports:
      - 80:80
      - 443:443
    environment:
      ACME_AGREE: "true" 
      DOMAIN: "bitwarden.koko.cat" # 将DOMAIN:后面的域名修改为你自己的即可
      EMAIL: "example@gmail.com" # 不做修改，除非要绑定域名邮箱
volumes:
  caddycerts:
```

### 新建一个Caddy的配置文件`Caddyfile`
```bash
$ nano Caddyfile
```
直接复制-粘贴下面的内容，无需改动：
```nginx
{$DOMAIN} {
        tls {$EMAIL}
    
        header / {
            Strict-Transport-Security "max-age=31536000;"
            X-XSS-Protection "1; mode=block"
            X-Frame-Options "DENY"
        }
    
        proxy /notifications/hub/negotiate bitwarden:80 {
            transparent
        }
    
        # Notifications redirected to the websockets server
        proxy /notifications/hub bitwarden:3012 {
            websocket
        }
    
        # Proxy the Root directory to Rocket
        proxy / bitwarden:80 {
            transparent
        }
    }
```
# 运行和调试

## 启动docker镜像：
```bash
$ docker-compose up -d
```
## 注册账号并关闭他人注册

在浏览器上打开服务器域名即可转入**bitwarkden**的WEB UI界面，注册一个账号。注册完成之后，可以编辑`docker-compose.yml`关闭注册功能：
```bash
$ nano ~/bitwarden/docker-compose.yml
```
改动`environment`下的SIGNUPS_ALLOWED值为false：
```yaml
SIGNUPS_ALLOWED: "false"
```
如果要配置SMTP也可以在这里一并完成，加入下面的配置：
```yaml
SMTP_HOST: "smtp.host.net"
   SMTP_FROM: "no-reply@home.example.com"
   SMTP_PORT: "587"
   SMTP_SSL: "true"
   SMTP_USERNAME: "xxx"
   SMTP_PASSWORD: "yyy"
```
停止运行，然后再启动即可应用新的环境变量：
```bash
$ docker-compose stop
$ docker-compose up -d
```
基本配置好上面这些就能满足日常使用需求了，如果需要更多配置可看官方的WIKI：
https://github.com/dani-garcia/bitwarden_rs/wiki

# 客户端的使用

**bitwarden**支持 Windows macOS 和 Linux 三大 PC 端平台和 Android，iOS 两大手机端平台，还有Chrome，Firefox，Opera等很多主流浏览器的扩展程序，也支持网页登录或者命令行管理模式，十分全面。而**bitwarden_rs**的WEB UI基本和官方版本完全相同，自建的服务和官方托管在 Azure 上的服务也拥有完全同等特性。

对于自行部署的服务器，需要首先在登录界面，点击**左上角齿轮图标**的设置按钮，在最上面的服务器地址栏中，填写自托管服务器的域名地址。然后就可以通过上面建立的账户登录同步数据。

bitwarden支持从其他主流密码管理器如LastPass，1Password，KeePass，和主流浏览器的账户密码管理器中导入数据，以此从其他平台迁移十分容易。