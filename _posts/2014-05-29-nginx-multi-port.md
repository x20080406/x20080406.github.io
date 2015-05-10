---
layout: post
title:  nginx�����������ͬһ�˿�����
date:   2014-05-29 13:52:15
tags:
- nginx
---

###nginx�����������ͬһ�˿�����###
1.
Ŀ�꣺domain1.com:80 ����/data/www/domain1,domain2.com:80����/data/domain2

��domain1.com��domain2.comӳ�䵽127.0.0.1

```
sudo echo -e "127.0.0.1\tdomain1.com\n127.0.0.1\tdomain2.com" >> /etc/hosts
```

2.
Ĭ�����á����������÷�Χ�ڵ�������ֱ����IP���ʵģ�����ֱ�Ӿܾ����У���``default_server``��ѡ�������дĬ�ϴ�����ǵ�һ�������ϵ�������
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
����domain1
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
����domain2
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
�ο����£�

[ngx_http_access_module](http://nginx.org/en/docs/http/ngx_http_access_module.html),[server_names](http://nginx.org/cn/docs/http/server_names.html#miscellaneous_names)