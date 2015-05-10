---
layout: post
title:  nginx多个域名监听同一端口配置
date:   2014-05-29 13:52:15
tags:
- nginx
---

###nginx多个域名监听同一端口配置###
1.
目标：domain1.com:80 访问/data/www/domain1,domain2.com:80访问/data/domain2

将domain1.com和domain2.com映射到127.0.0.1

```
sudo echo -e "127.0.0.1\tdomain1.com\n127.0.0.1\tdomain2.com" >> /etc/hosts
```

2.
默认配置。处理不在配置范围内的域名或直接用IP访问的（这里直接拒绝所有）。``default_server``可选，如果不写默认处理的是地一个（从上到下数）
```
server {
    listen 80 default_server;
    server_name *;
    location / {
          deny  all;
   }
}
```

3.
处理domain1
```
server {
    listen 80;
    server_name domain1.com;
    location / {
      root /data/www/domain1;
   }
}
```

4.
处理domain2
```
server {
    listen 80;
    server_name domain2.com;
    location / {
      root /data/www/domain2;
   }
}
```

5.
参考文章：

[ngx_http_access_module](http://nginx.org/en/docs/http/ngx_http_access_module.html),[server_names](http://nginx.org/cn/docs/http/server_names.html#miscellaneous_names)