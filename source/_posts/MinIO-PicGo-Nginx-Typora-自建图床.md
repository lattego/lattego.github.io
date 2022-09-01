---
title: MinIO+PicGo+Nginx+Typora 自建图床
date: 2022-09-01 14:20:54
cover: https://share.lattice.vip/lattice/latte/03.jpg
tags: [图床, MinIO, Nginx, Typora]
categories: Blog
---

### 写在前面

#### 需求背景

​	自从开始写文章后，就遇到了一个问题，`图床`；我这边是使用 Typora 编写 Markdown 格式的文章，然后发布到各个平台，有些平台会自动将你的图片文件上传到他们的服务器上，但是有的还是会使用你自己的图片文件来源。这会产生两个问题：

​	1. Typora 会默认将你的截图、复制的图片放在本地，没法多设备同步，导致我从公司回到家后打开笔记只能面对一张张“破”了的图，属实影响阅读体验

​	2. Typora 编写文章的时候如果使用第三方图床的方式，得依赖于第三方服务器的稳定性

​	所以，有没有可能自建图床，搭建一个属于自己的可控的图床方案呢？答案是，Yes!

#### 需求调研

- 首先考虑了免费图床，`SM.MS`,5G免费存储空间，网络貌似不太稳定
- 大厂提供的对象存储服务，最开始优先考虑的就是这套解决方案
    - `七牛云`：月免费额度10G,这个跟我几年前的使用体验还是有出入的，创建好桶，存储空间后就可以正常进行文件存取；但是我想的是通过自定义域名来作为图床的访问地址，又增加了自定义域名配置，至此倒也没啥大问题。使用了几天后，发现了账单里多了一笔0.12元的实时账单，询问客服后了解了来龙去脉，原来是访问我文章的外部流量，也就是我把图片URL暴露出去后，每一次的访问都会产生外网流出流量，产生一笔小小的费用
    - 客服提供的解决方案：绑定`CDN`加速域名，月免费额度 10G,好的，继续折腾；当我再次绑定好`CDN`加速域名后，配置完成咨询客服是否正确后，得到的回复是免费额度只支持HTTP,您使用的是HTTPS,好嘛，这并不是我想要的，遂抛弃此方案。如果不在意这部分支出的话，选择一些大厂的云存储方案还是很方便的，有售后保证。
    - ![image-20220901141613104](https://share.lattice.vip/lattice/typora/2022/09/01/image-20220901141613104.png)
- `MinIO`: 一款基于Go语言的高性能对象存储服务。偶然在一篇文章上看到这个开源的分布式存储介绍，就想着看能不能基于自己的云服务器搭建一套存储方案；多番查阅资料折腾后，最终实现本文的 `MinIO+PicGo+Nginx+Typora` 自建图床方案

### 你需要准备的

- 云服务器 *1
- 已备案的域名 *1

- MinIO：docker hub latest
- Nginx: 1.20.1
- Typora: 1.3.8
- PicGo: 2.3.0
- OS: CentOS 7.9
- Docker: 20.10.17

### 安装步骤

#### MinIO 部署

> 本文采用的是Docker部署方式，单节点，考虑到我的云服务器配置并不是那么高

拉取`MinIO`最新镜像

```bash
[root@VM-4-12-centos /]# docker pull minio/minio
```

容器启动`MinIO`实例

```bash
[root@VM-4-12-centos /]# docker run -p 9000:9000 -p 9001:9001 --name minio -d --restart=always -e "MINIO_ACCESS_KEY=xxx" -e "MINIO_SECRET_KEY=xxxxxxxx" -v /home/data:/data -v /home/config:/root/.minio minio/minio server --console-address ":9000" --address ":9001" /data
```

配置项说明：

- 端口`9000`: 控制台使用
- 端口`9001`: API使用
- MINIO_ACCESS_KEY=xxx：登录用户名
- MINIO_SECRET_KEY=xxxxxxxx：登录用密码
- /home/data： 宿主机映射目录卷
- /home/config： 宿主机映射配置文件目录卷

注意事项

- 将用户名密码替换你自己设置的，另外ACCESS_KEY至少3位，MINIO_SECRET_KEY至少8位，否则容器启动失败，抛出此异常

#### Nginx 配置

> 云服务器上之前有部署好的Nginx环境，所以此文就不展开说明，仅贴上具体Nginx配置文件

##### 云安全组端口开放

- `io.xxx.com`: 用于访问 MinIO Manage Web控制台,开放9000端口

- `share.xxx.com`: 用于图床API访问URL，开放9001端口

##### 云域名解析

上述提到的两个域名均解析到你对应的云服务器即可

##### Nginx设置域名配置文件

```bash
# nginx配置目录
[root@VM-4-12-centos nginx]# pwd
/etc/nginx
# nginx下使用vhost子目录 include到主的配置文件中
[root@VM-4-12-centos vhost]# pwd
/etc/nginx/vhost
[root@VM-4-12-centos vhost]# ll
-rw-r--r-- 1 root root 1014 Aug 31 14:43 io.xxx.com.conf
-rw-r--r-- 1 root root  835 Sep  1 08:42 share.xxx.com.conf
```

##### vim io.xxx.com.conf

```properties
server{
        listen 80;
        # MinIO后台管理域名
        server_name	io.xxx.com;
        # HTPP重定向到HTTPS
        return 301 https://$server_name$request_uri;
}

server{
        listen 443 ssl;
        server_name io.xxx.com;
        # nginx访问日志，错误日志配置
        access_log /usr/local/nginx/logs/io.access.log json;
        error_log /usr/local/nginx/logs/io.error.log warn;
        # SSL证书配置
        ssl_certificate      /usr/local/nginx/ssl/io.xxx.com_bundle.crt;
        ssl_certificate_key  /usr/local/nginx/ssl/io.xxx.com.key;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
        ssl_session_timeout  5m;
        ssl_prefer_server_ciphers  on;
        ssl_session_cache shared:SSL:10m;

        location / {
            proxy_redirect    off;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header  Host             $host;
            proxy_set_header  X-Real-IP        $remote_addr;
            proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
            # 目标MinIO服务
            proxy_pass http://localhost:9000;                 
        }
}
```

##### vim share.xxx.com.conf

```properties
server{
        listen 80;
        # MinIO外链访问域名
        server_name	share.xxx.com;
        # HTPP重定向到HTTPS
        return	301 https://$server_name$request_uri;
}

server{
        listen 443 ssl;
        server_name share.xxx.com;
        # nginx访问日志，错误日志配置
        access_log /usr/local/nginx/logs/share.access.log json;
        error_log /usr/local/nginx/logs/share.error.log warn;
        ssl_certificate      /usr/local/nginx/ssl/share.xxx.com_bundle.crt;
        ssl_certificate_key  /usr/local/nginx/ssl/share.xxx.com.key;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
        ssl_session_timeout  5m;
        ssl_prefer_server_ciphers  on;
        ssl_session_cache shared:SSL:10m;
   
        location / {
             proxy_set_header Host $host;
             add_header Content-Security-Policy "upgrade-insecure-requests";
             # 目标MinIO API服务
             proxy_pass http://localhost:9001;
        }
}
```

#### MinIO Manage Web 访问

##### MinIO 登录

访问地址: https://io.xxx.com, 输入你最初设置的用户密码登录即可

![image-20220901141429906](https://share.lattice.vip/lattice/typora/2022/09/01/image-20220901141429906.png)

##### Bucket（桶） 创建

![image-20220901141459185](https://share.lattice.vip/lattice/typora/2022/09/01/image-20220901141459185.png)

##### Bucket 信息

![image-20220901141522702](https://share.lattice.vip/lattice/typora/2022/09/01/image-20220901141522702.png)



至此, MinIO 整个服务端已经搭建且调试完成！

#### MinIO Client (Typora+PicGo)

##### PicGo 配置

- 安装`MinIO` 插件

![image-20220901141543044](https://share.lattice.vip/lattice/typora/2022/09/01/image-20220901141543044.png)

- MinIO 图床设置 `【】 内为注释`
    - `endPoint`: share.xxx.com 【API访问的域名】
    - `port`: 443 【端口】
    - `useSSl`: true 【使用SSL时打开】
    - `accessKey`: 【用户名】
    - `secretKey`: 【密码】
    - `bucket`: lattice 【桶名称】
    - `同名文件`：跳过 【当文件名重复时设置的策略】
    - `基础目录`：/typora【自定义子目录文件夹】
    - `自定义域名`： share.xxx.com
    - `自动归档`： true【可选择开启，PicGo程序会自动帮你按照yyyy/MM/dd的格式归档】

##### Typora 配置

- 打开typora,【文件】->【偏好设置】->【图像】

![image-20220901141709006](https://share.lattice.vip/lattice/typora/2022/09/01/image-20220901141709006.png)

- 上传图床服务成功

![image-20220901141237800](https://share.lattice.vip/lattice/typora/2022/09/01/image-20220901141237800.png)



#### 收获&问题

- 掌握了分布式存储 MinIO Docker 部署
- 了解了 Nginx 下域名配置等
- 实现“免费”图床服务，写文章再也不用考虑图片失联了
- 可充当个人网盘轻量使用

#### 拓展

- 后续可以使用 MinIO 作为业务开发中的分布式存储服务
- SpringBoot 方式整合 MinIO,JavaSDK 的使用
- 使用自己的文件服务中转，设置业务附件与 MinIO 关联，使用类似 https://share.xxx/com/UUID 这种方式访问，进而不去暴露文件名，以及文件的权限控制等等

#### 完结

​	至此，就已经全部完成了整套自建图床服务的搭建了！可能也并不是最好的方案，只能说是目前我能想到的一个比较适合我的“免费”图床方案；但是“免费”的前提也是需要一些投入的成本，我这里是已有一个闲置的域名，一台XX云的轻量服务器。

​	谢谢大家观看[我的博客](https://lattego.github.io)~下期再见

​	