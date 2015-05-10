---
layout: post
title:  Ubuntu������vsftp�����û�
date:   2014-04-10 23:10:30
tags:
- linux 
---

####��װVSFTPD��

```sudo apt-get install vsftpd```

####��װdb4.8-util

```sudo apt-get install db4.8-util ```

####���������û����ݿ⡣
д��һ���ʻ���Ϣ�������д����ʻ���ż���д������롣�ڶ����˻���Ϣ�ӵ����п�ʼ���ڶ����ʻ�������ӵ����п�ʼ���Դ����� 
```
sudo mkdir -p /etc/vsftpd
vi /etc/vsftpd/users.txt
    ftpuser
    ftpuser
```

####ʹ��dbutil�������ݿ��ļ�
```
sudo db4.8_load -T -t hash -f /etc/vsftpd/users.txt /etc/vsftpd/vsftpd_login.db
sudo chmod 600 /etc/vsftpd/vsftpd_login.db
```

####����PAM�ļ�
��Ҫ.db,64λϵͳע��.SO�ļ���·������lib64�¡�__ֻ�����¼ӵ����У�����ȫɾ__
```
sudo vi /etc/pam.d/vsftpd.vu
auth required pam_userdb.so db=/etc/vsftpd/vsftpd_login
account required pam_userdb.so db=/etc/vsftpd/vsftpd_login
```

* * *
####�����û�
```
sudo mkdir -p /data/www/attachment/ftpuser
sudo groupadd www
sudo useradd vsftpd -d /data/www/attachment -s /bin/nologin -g www    ������www���û����ʣ�
sudo chown -R vsftpd:www /data/www/attachment
```

####�༭/etc/vsftpd.conf��ÿ�����ݺ������Ҫ���ո񣡣�����
```
vi /etc/vsftpd.conf
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
```

####�����û����������ļ�
```
sudo mkdir /etc/vsftpd_user_conf
cd /etc/vsftpd_user_conf
sudo vi ftpuser����Ӧ�����ļ��е��ʺţ�
 
write_enable=YES
anon_world_readable_only=NO
anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES
local_root=/data/www/attachment/ftpuser�����û�������Ŀ¼��ftpuser��ӦftpuserĿ¼�������Ķ�Ӧ�Լ���Ӧ��Ŀ¼����
```