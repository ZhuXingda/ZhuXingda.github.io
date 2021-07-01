---
layout:       post
title: 用NGINX_HTTP_FLV_MODULE做视频流服务器
subtitle:     ""
date:         2021-06-30
author:       "ZhuXingda"
header-mask:  0.3
catalog:      true
multilingual: false
comments: true
tags:
    - Linux
    - Nginx
---
本文主要参考了[这篇博客的做法](https://www.trickyedecay.me/2019/03/17/how-to-setup-an-live-server-with-nginx-base-on-http-flv/)
## 服务端
使用nginx_http_flv_module的docker镜像来搭视频流服务器，参考[这篇博客](http://zzxin.top/post/39)

镜像下好启动之后查看linux版本，发现是
```shell
/opt/nginx # cat /etc/issue
Welcome to Alpine Linux 3.4
Kernel \r on an \m (\l)
```
alpine的应用管理器是apk，因为不会用vi所以更新apk的下载源然后装一个vim（>>添加到文件末尾，>替换文件内容）
```shell
echo "https://mirror.tuna.tsinghua.edu.cn/alpine/v3.4/main" > /etc/apk/repositories
echo "https://mirror.tuna.tsinghua.edu.cn/alpine/v3.4/community" >> /etc/apk/repositories
echo "https://mirror.tuna.tsinghua.edu.cn/alpine/edge/testing" >> /etc/apk/repositories
/opt/nginx # apk update
/opt/nginx # apk add vim
```

查找nginx.conf的位置
```shell
/ # ps -ef | grep nginx
    1 root       0:00 nginx: master process /opt/nginx/sbin/nginx
    7 nobody     0:00 nginx: worker process
    8 nobody     0:00 nginx: cache manager process
   97 root       0:00 grep nginx
/ # /opt/nginx/sbin/nginx -t
nginx: the configuration file /opt/nginx/nginx.conf syntax is ok
nginx: configuration file /opt/nginx/nginx.conf test is successful
```

修改配置文件
```shell
daemon off;

error_log /var/log/nginx/error.log warn;

events {
    worker_connections 1024;
}

rtmp {
    out_queue   4096;
    out_cork    8;
    max_streams 64;
    server {
        listen 1935; 

        application broadcast { #推流的app地址
            live on;
            gop_cache on; #open GOP cache for reducing the wating time for the first picture of video
            allow play all;
        }

        drop_idle_publisher 30s;
        ping 20s;
        ping_timeout 10s;
        meta off;
        chunk_size 4096;
        wait_video on;
        wait_key on;
    }
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    keepalive_timeout  65;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log;

    server {
        listen 80;

        location /live { #拉流的地址
            flv_live on; 
            chunked_transfer_encoding  off; 

            add_header 'Access-Control-Allow-Origin' '*'; #allow CORS
            add_header 'Access-Control-Allow-Credentials' 'true'; 
            add_header 'Access-Control-Allow-Headers' 'X-Requested-With';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE';
            add_header 'Cache-Control' 'no-cache';
        }
    }
}
```
更多配置信息可以看官方文档[nginx_http_flv_module](https://github.com/winshining/nginx-http-flv-module/blob/master/README.md)

关于CORS我之前不懂，遇到bug之后查了一下，这些文章讲得很详细
- [CORS](https://juejin.cn/post/6844904047409889288) 
- [跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)

## 采集端
下载[OBS Studio](https://obsproject.com/)，配置串流地址然后点开始推流。如果是用的笔记本处理器温度太高的话可以在设置里调低分辨率，我调到360P之后风扇声小了很多。
![OBS串流地址配置]({{ site.url }}/img/posts/OBS.jpg)

## 播放端
网页播放器用(西瓜视频的播放器)[https://v2.h5player.bytedance.com/]
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name=viewport content="width=device-width,initial-scale=1,maximum-scale=1,minimum-scale=1,user-scalable=no,minimal-ui">
    <meta name="referrer" content="no-referrer">
    <title>播放器</title>
    <style type="text/css">
        html, body {width:100%;height:100%;margin:auto;overflow: hidden;}
        body {display:flex;}
        #mse {flex:auto;}
    </style>
    <script type="text/javascript">
        window.addEventListener('resize',function(){document.getElementById('mse').style.height=window.innerHeight+'px';});
    </script>
</head>
<body>
<div id="mse"></div>
<script src="index.js" charset="utf-8"></script> //下载地址 cdn.jsdelivr.net/npm/xgplayer@2.9.6/browser/index.js
<script src="index.min.js" charset="utf-8"></script> //下载地址 cdn.jsdelivr.net/npm/xgplayer-flv/dist/index.min.js
<script type="text/javascript">
    let player = new FlvPlayer({
        id: 'mse',
        url: 'http://10.6.11.250:80/live?port=1935&app=broadcast&stream=camera', //这里拉流地址是live，拉取的推流app是broadcast，stream就是之前设置的Stream Name
        isLive: true,
        playsinline: true,
        height: window.innerHeight,
        width: window.innerWidth,
        volume: 0.5,
        autoplay: true
    });
</script>
</body>
</html>
```
这里的url一定不能填错，我不小心多打了个分号结果一直报错CORS验证出错，虽然nginx的配置文件我已经写了'Access-Control-Allow-Origin' '*'

OBS开始录制并推流之后打开这个html文件，就能看到电脑摄像头或者桌面的直播的音视频了。
内网环境延迟居然有7秒，暂时不知道是什么原因
![延迟]({{ site.url }}/img/posts/yanchi.jpg)
本来是计划做一个在内网看电影的网站，这里先用OBS试一下，后面再尝试在服务器上用FFmpeg来推电影。