---
title: 与 Nginx Proxy Manager 配合使用
description: 使用 Nginx Proxy Manager 管理 Halo 服务的反向代理
---

## Halo 部署

参见 [使用 Docker Compose 部署](../docker-compose)

:::info
`「反向代理」` 部分不进行操作，保证 Halo 服务运行无误即可。
:::

## 简介

顾名思义，Nginx Proxy Manager 就是一个 Nginx 的代理管理器，它最大的特点是简单方便。

即使是没有 Nginx 基础的小伙伴，也能轻松地用它来完成反向代理的操作，而且因为自带面板，操作极其简单，非常适合配合 docker 搭建的应用使用。

Nginx Proxy Manager 后台还可以一键申请 SSL 证书，并且会自动续期，方便省心。

下面我们就来介绍如何用 Nginx Proxy Manger 来配合 Halo，实现反向代理和 HTTPS 访问。

## 安装 Nginx Proxy Manager

> 说明：默认你的服务器已经安装了 Docker 和 Docker Compose，如果你没有安装，可以参考：[使用 Docker Compose 部署](../docker-compose) 的环境搭建部分来进行安装。

点击下方链接进入 Nginx Proxy Manager（以下简称 NPM） 官网：[https://nginxproxymanager.com/](https://nginxproxymanager.com/)

我们可以直接选择 [快速安装](https://nginxproxymanager.com/guide/#quick-setup)。

首先，我们创建一个文件夹来存放 NPM 的 `docker-compose.yml` 文件：

```bash
mkdir -p ~/data/docker_data/nginxproxymanager   # 创建一个 npm 的文件夹

cd ~/data/docker_data/nginxproxymanager    # 进入该文件夹

vi docker-compose.yml
```

在英文状态的输入法下，按下 `i`，左下角出现 `--INSERT--` 后，粘贴填入下面的内容：

```yaml
version: '3'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'              # 不建议修改端口
      - '81:81'              # 可以把冒号左边的 81 端口修改成你服务器上没有被占用的端口
      - '443:443'            # 不建议修改端口
    volumes:
      - ./data:/data         # 点号表示当前文件夹，冒号左边的意思是在当前文件夹下创建一个 data 目录，用于存放数据，如果不存在的话，会自动创建
      - ./letsencrypt:/etc/letsencrypt  # 点号表示当前文件夹，冒号左边的意思是在当前文件夹下创建一个 letsencrypt 目录，用于存放证书，如果不存在的话，会自动创建
    networks:
      - halo_halo_network    # 如果你使用 docker compose 部署 halo，那么 halo 的默认网络就是 halo_halo_network

networks:
  halo_halo_network:
    external: true           # 声明使用外部网络
```

> 注意：安装了 NPM 之后，就不需要再安装 Nginx 了，否则会端口冲突（不建议修改 NPM 的 80、443 端口）。如果你的服务器安装了宝塔面板，也可以和 NPM 一起使用，只要你到软件后台把宝塔安装的 Nginx 关闭或者卸载即可。

之后，同样在英文输入法下，按一下 `esc`，然后 `:wq` 保存退出。

启动 NPM：

```bash
docker-compose up -d     # -d 表示后台运行

docker compose up -d     # 如果你用的是 docker-compose-plugin 的话，用这条命令
```

此时我们可以查看 NPM 和 halo 是否处于同一网络：

```bash
docker network inspect halo_halo_network | grep "Containers" -A30
```
可以看到 NPM 和 halo 处于同一网络，并且 halo 的容器内 IP 地址为 `172.18.0.3`。如果你选择使用容器内网络，这个 IP 地址并不需要记忆。

```bash
"Containers": {
    "0aee4d25d250082ac4771d2258ffc08b0c979dda7be02bb4b5b778b77e29f698": {
        "Name": "halo-halo-1",
        "EndpointID": "d7d6d330fbfb8362cdc51ae68ae7f9cdb354419bee16a0046a763b8475cca720",
        "MacAddress": "02:42:ac:12:00:03",
        "IPv4Address": "172.18.0.3/16",
        "IPv6Address": ""
    },
    "68fccfe5768f19152a69bb5a392d40db6d11ca9171fb7dd619d29af3a90c901b": {
        "Name": "nginxproxymanager-app-1",
        "EndpointID": "48c48d8258b288a5f37247e3f6ce277e010ec3f4a93d02bedbce843fddf5f9b8",
        "MacAddress": "02:42:ac:12:00:04",
        "IPv4Address": "172.18.0.4/16",
        "IPv6Address": ""
    },
    "d0b5fcbed69c13cb329b70e5559b086de9207074ad21000fe918e5a014139eaa": {
        "Name": "halo-halodb-1",
        "EndpointID": "fc776efd918778d37fcb5ca4d13f2a8e22a48422e84f4984bc5317f699597eb3",
        "MacAddress": "02:42:ac:12:00:02",
        "IPv4Address": "172.18.0.2/16",
        "IPv6Address": ""
    }
}
```

之后如果不出意外，此时你使用 [http://127.0.0.1:81](http://127.0.0.1:81/) 就可以访问 NPM 的网页端了。（注意把 `127.0.0.1` 替换成你实际服务器的 IP）

:::info

1. 不知道服务器 IP，可以直接在命令行输入：curl ip.sb，会显示当前服务器的 IP。
2. 遇到访问不了的情况，请再次检查在宝塔面板的防火墙和服务商的后台防火墙是否打开对应了端口。
:::

默认登录的用户名：`admin@example.com` 密码：`changeme`

第一次登录会提示更改用户名和密码，建议修改一个复杂一点的密码。

至此，我们已经完成了 Nginx Proxy Manager 的搭建，之后就可以用它给我们的 Halo 或者其他 Web 应用做反向代理了。

## 配置 Halo 的反向代理

首先我们登录网页端之后，会弹出修改用户名和密码的对话框，我们根据自己的实际来修改自己的用户名和邮箱。

![Nginx Proxy Manager 1](/img/nginx-proxy-manager/Nginx-Proxy-Manager-1.png)

保存之后，会让我们修改密码（建议用一个复杂的密码）。

![Nginx Proxy Manager 2](/img/nginx-proxy-manager/Nginx-Proxy-Manager-2.png)

接着我们就可以来给 Halo 来添加一个反向代理了。

点击 `Proxy Hosts`，

![Nginx Proxy Manager 3](/img/nginx-proxy-manager/Nginx-Proxy-Manager-3.png)

接着点击 `Add Proxy Host`，弹出如下对话框：

![Nginx Proxy Manager 4](/img/nginx-proxy-manager/Nginx-Proxy-Manager-4.png)

看起来都是英文，很复杂，但是其实很简单，我们只要用到其中的几个功能即可，这边稍微解释一下：

- `Domain Names` ：填我们 Halo 网站的域名，首先记得做好 DNS 解析，把域名绑定到我们的服务器的 IP 上
- `Scheme` ：默认 `http` 即可，除非你有自签名证书
- `Forward Hostname/IP` ：填入服务器的 IP，或者 Docker 容器内部的 IP（如果 NPM 和 Halo 搭建在同一台服务器上的话）
- `Forward Port`：填入 Halo 映射出的端口，这边默认是 `8090`
- `Cache Assets` ：缓存，可以选择打开
- `Block Common Exploits`： 阻止常见的漏洞，可以选择打开
- `Websockets Support` ：WS 支持，可以选择打开
- `Access List`： 这个是 NPM 自带的一个限制访问功能，这边我们不管，后续可以自行研究。

以下是一个样列：

![Nginx Proxy Manager 5](/img/nginx-proxy-manager/Nginx-Proxy-Manager-5.png)

此时我们可以填入服务器的 IP，如果你的 NPM 和 halo 在同一服务器上，那么可以填入 docker 容器内 halo 的 ip，或者是容器网络别名（推荐）。

可以通过下面的命令查询：

```bash
docker inspect {YOUR_HALO_CONTAINER_NAME} | grep "Networks" -A30
# 默认情况下是 docker inspect halo-halo-1 | grep "Networks" -A30
```

```bash
"Networks": {
  "halo_halo_network": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
        "halo-halo-1",
        "halo"
    ],
    "MacAddress": "02:42:ac:12:00:03",
    "DriverOpts": null,
    "NetworkID": "2b0cb49e1d004ad1814757620bf5f7fd5952ad24b2d22b41a4df3f4403829236",
    "EndpointID": "d7d6d330fbfb8362cdc51ae68ae7f9cdb354419bee16a0046a763b8475cca720",
    "Gateway": "172.18.0.1",
    "IPAddress": "172.18.0.3",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "DNSNames": [
        "halo-halo-1",
        "halo",
        "0aee4d25d250"
    ]
  }
}
```

可以看到这边 halo 在 docker 容器中的 IP 为 `172.18.0.3`，别名为 halo, halo-halo-1，此时可以不用打开防火墙的 `8090` 端口。

当然，如果你的 NPM 和 Halo 不在同一台服务上，你需要在 IP 部分填入 **你的 Halo 所在的服务器的 IP**，并在服务商（部分服务商如腾讯、阿里）的后台打开 `8090` 端口。

## 一键申请 SSL 证书

接着我们来申请一张 SSL 证书，让我们的网站支持 `https` 访问。

![Nginx Proxy Manager 6](/img/nginx-proxy-manager/Nginx-Proxy-Manager-6.png)

![Nginx Proxy Manager 7](/img/nginx-proxy-manager/Nginx-Proxy-Manager-7.png)

如图所示，记得打开强制 SSL，其他四个的功能请自行研究，这边不多做讨论。

:::info

1. 申请证书需要你提前将域名解析到 NPM 所在的服务器的 IP 上；
2. 如果你使用的是国内的服务器，默认 `80` 和 `443` 端口是关闭的，你需要备案之后才能使用；
3. 如果你使用了 CloudFlare 的 DNS 服务，记得把小黄云关闭（即不开启 CDN）。
:::

不出意外，你将成功申请到 SSL 证书，证书会三个月自动续期。

再次点开配置，查看一下，将强制 SSL 打开。

![Nginx Proxy Manager 8](/img/nginx-proxy-manager/Nginx-Proxy-Manager-8.png)

至此，你已经成功完成了 Halo 的反向代理，快尝试使用域名访问一下看看吧！

> 同样的，举一反三，试试把你的 NPM 也用一个域名来反向代理一下吧。(小提示：你需要再解析一个域名（可以是二级域名）到 NPM 所在的服务器上，反代页面需要填的 IP 可以填 docker 容器内的 IP 也可以填服务器的 IP，端口填 `81` 即可）
