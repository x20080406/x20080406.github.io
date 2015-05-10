---
layout: post
title:  试用nginx stream模块
date:   2014-04-10 23:10:30
tags:
- nginx
---

nginx1.9带了一个可以代理tcp连接的模块ngx_stream_proxy_module。可以支持负载均衡。
本地测试配置
```
stream {
    upstream backend {
        server 192.168.101.123:23432;
        server 192.168.101.125:23432;
    }
    server {
        listen 12345 so_keepalive=on;
        proxy_connect_timeout 1s;
        proxy_timeout 15s;#与连接心跳时间保持一致,15秒内无读写视为超时，被ngx和谐
        proxy_pass backend;
    }
}
```

本地启动了4个客户端，每个打开4个连接。代理效果如下：
![ngx_stream_test](/assets/article-imgs/nginx-stream-test.png)

发现nginx与后端服务器之间维持着__16__个连接。__16__个连接！假如后端服务器维持10w个连接是没有压力的情况下，nginx所在机器有那么端口代理么？如果面对是IM这样的场景nginx怎么代理呢？这种情况很正常且很普遍存在。不过仅用在服务器间通信还是不错的，弄一个网卡好点的其他配置一般的机器就行了。
