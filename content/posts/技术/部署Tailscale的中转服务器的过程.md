---
title: "部署Tailscale的中转服务器的过程"
date: 2024-12-15T14:41:50+08:00
draft: false
categories: 技术
keywords: tailscale derp ssl https
---

了解Tailscale很久，但是基本很少用，常用的场景下基本没有办法直联，官方中继延迟太高了。后来突然有一天了解到，tailscale可以自行部署中转服务器derp，于是就打算部署一个来解决延迟太高的问题.

过程中基本是按照[官方文档](https://tailscale.com/kb/1118/custom-derp-servers)来进行操作，文档本身已经写的很好了，只是正好不是我的母语，tailscale我又没有特别的深入使用，同时又有不少专属名词，导致有点东西理解，因此大概折腾了一周左右，反反复复的折腾，才算是真正的搞好和理解。

下面是具体操作步骤：

1. 打开云服务器的防火墙设置，当前教程下需要开启10443(tcp协议)，3478(UDP协议), icmp协议的防火墙策略。以上防火墙策略是必须的。

2. 在云服务上安装go应用程序，具体参考[go官网](https://go.dev/doc/install)

3. 执行一下命令，安装并编译最新的derper程序。(这个步骤可能在你的tailscale无法连接到这个中继服务器的时候重复执行，以保持与tailscale各个客户端的兼容性)
    ``` shell
    go install tailscale.com/cmd/derper@latest  
    ```
4. 安装certbot程序(其实就是Let`s Encryept)来获取https证书, 具体安装步骤见[certbot官网](https://certbot.eff.org/)

5. 执行一下命令获取https证书，一切按照默认继续下去即可。具体意思就是单独获取你域名的nginx证书，并保存到指定路径。
    ``` shell
    sudo certbot certonly --manual --preferred-challenges dns -d <你的域名>
    ```
6. 上一步获取到https证书之后，会打印出来证书保存的路径。其中fullchain.pem和privkey.pem分别代表你证书的crt和key,在很多地方使用https证书的时候会指定具体的crt和key文件路径。我们通过ln -s命令分别创建privkey.pem的副本，并重命名为.crt和.key的后缀文件。
    ``` shell
    sudo ln -s <cerbot证书保存路径>/fullchain.pem <cerbot证书保存路径>/<你的域名>.crt
    sudo ln -s <cerbot证书保存路径>/privkey.pem <cerbot证书保存路径>/<你的域名>.key
    ```
7. 启动derper程序开启中继服务
    ``` shell
    sudo <go的bin目录>/derper -a :10443 --http-port=-1  --hostname=<你的域名> -certmode manual -certdir <cerbot证书保存路径> --verify-clients
    ```    
    + -a :10443 指定https服务的监听端口为10443，且监听所有网口
    + http-port=-1 不开启http服务
    + hostname 指定你的derp中继服务的对外域名
    + certmode manual代表手动管理https证书
    + certdir 指定手动管理的https证书的具体路径
    + --verify-clients 代表需要校验tailscale的客户端，这个可以去掉(第8步无需执行)。但是就意味着你的服务谁都可以使用 

8. 在云服务器上安装tailscale客户端，并登录授权，具体参考官网。

9. 配置tailscale客户端访问策略，通过[tailscale管理平台配置](https://login.tailscale.com/admin/acls/file), 在文件的最后一个}加入一下内容。
    ``` json
    "derpMap": {
		"OmitDefaultRegions": true,
		"Regions": {
			"900": {
				"RegionID":   900,
				"RegionCode": "myderp",
				"Nodes": [
					{
						"Name":     "1",
						"RegionID": 900,
						"HostName": "<你的域名>",
						"DERPPort": 10443,
					},
				],
			},
		},
	},
    ```

    + OmitDefaultRegions代表是否需要禁用默认的中继服务器，因为我都是国内服务器，因此不需要官方的国外中继服务器，所以默认true，禁用
    + DERPPort代表你的derp服务的https端口，如需改变需要将本文中所有涉及到10443端口的地方都修改为同一个端口
    + derpMap下用户可以自定义的Regions最小开头是900，900-999是用户可以自定义的，不要随便定义。