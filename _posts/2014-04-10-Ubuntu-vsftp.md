---
layout: post
title:  Ubuntu下配置vsftp虚拟用户
date:   2014-04-10 23:10:30
tags:
- linux 
---

<h4 id="vsftpd">安装VSFTPD。</h4>
<p><code>sudo apt-get install vsftpd</code></p>
<h4 id="db48-util">安装db4.8-util</h4>
<p><code>sudo apt-get install db4.8-util</code></p>
<h4 id="">制作虚拟用户数据库。</h4>
<p>写入一个帐户信息。奇数行代表帐户，偶数行代表密码。第二个账户信息从第三行开始，第二个帐户的密码从第四行开始。以此类推 </p>
<pre><code>sudo mkdir -p /etc/vsftpd
vi /etc/vsftpd/users.txt
    ftpuser
    ftpuser
</code></pre>

<h4 id="dbutil">使用dbutil生成数据库文件</h4>
<pre><code>sudo db4.8_load -T -t hash -f /etc/vsftpd/users.txt /etc/vsftpd/vsftpd_login.db
sudo chmod 600 /etc/vsftpd/vsftpd_login.db
</code></pre>

<h4 id="pam">配置PAM文件</h4>
<p>不要.db,64位系统注意.SO文件的路径，在lib64下。<strong>只保留新加的两行，其他全删</strong></p>
<pre><code>sudo vi /etc/pam.d/vsftpd.vu
auth required pam_userdb.so db=/etc/vsftpd/vsftpd_login
account required pam_userdb.so db=/etc/vsftpd/vsftpd_login
</code></pre>

<hr />
<h4 id="_1">创建用户</h4>
<pre><code>sudo mkdir -p /data/www/attachment/ftpuser
sudo groupadd www
sudo useradd vsftpd -d /data/www/attachment -s /bin/nologin -g www    （方便www组用户访问）
sudo chown -R vsftpd:www /data/www/attachment
</code></pre>

<h4 id="etcvsftpdconf">编辑/etc/vsftpd.conf（每行内容后面均不要带空格！！！）</h4>
<pre><code>vi /etc/vsftpd.conf
listen=YES
anonymous_enable=NO
local_enable=YES

write_enable=NO
anon_upload_enable=NO
anon_mkdir_write_enable=NO
anon_other_write_enable=NO
dirmessage_enable=YES
xferlog_enable=YES
xferlog_file=/var/log/vsftpd.log
xferlog_std_format=YES

chroot_list_enable=YES
chroot_list_file=/etc/vsftpd/chroot_list

chroot_local_user=YES

guest_enable=YES
guest_username=vsftpd
pam_service_name=vsftpd.vu
user_config_dir=/etc/vsftpd/vsftpd_user_conf
</code></pre>

<h4 id="_2">创建用户个人配置文件</h4>
<pre><code>sudo mkdir /etc/vsftpd_user_conf
cd /etc/vsftpd_user_conf
sudo vi ftpuser（对应密码文件中的帐号）

write_enable=YES
anon_world_readable_only=NO
anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES
local_root=/data/www/attachment/ftpuser（按用户划分主目录，ftpuser对应ftpuser目录，其他的对应自己相应的目录。）
</code></pre>