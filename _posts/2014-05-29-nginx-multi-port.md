---
layout: post
title:  nginx多个域名监听同一端口配置
date:   2014-05-29 13:52:15
tags:
- nginx
---

<h3 id="nginx">nginx多个域名监听同一端口配置</h3>
<p>1.
目标：domain1.com:80 访问/data/www/domain1,domain2.com:80访问/data/domain2</p>
<p>将domain1.com和domain2.com映射到127.0.0.1</p>
<pre><code>sudo echo -e &quot;127.0.0.1\tdomain1.com\n127.0.0.1\tdomain2.com&quot; &gt;&gt; /etc/hosts
</code></pre>

<p>2.
默认配置。处理不在配置范围内的域名或直接用IP访问的（这里直接拒绝所有）。<code>default_server</code>可选，如果不写默认处理的是地一个（从上到下数）</p>
<pre><code>server {
    listen 80 default_server;
    server_name *;
    location / {
          deny  all;
   }
}
</code></pre>

<p>3.
处理domain1</p>
<pre><code>server {
    listen 80;
    server_name domain1.com;
    location / {
      root /data/www/domain1;
   }
}
</code></pre>

<p>4.
处理domain2</p>
<pre><code>server {
    listen 80;
    server_name domain2.com;
    location / {
      root /data/www/domain2;
   }
}
</code></pre>

<p>5.
参考文章：</p>
<p><a href="http://nginx.org/en/docs/http/ngx_http_access_module.html">ngx_http_access_module</a>,<a href="http://nginx.org/cn/docs/http/server_names.html#miscellaneous_names">server_names</a></p>