---
title: nginx quick starter
date: 2015-09-06 17:18:49
tags: nginx
---

# Nginx quick starter(http server)

## Brief introduction
nginx [engine x] is an HTTP and reverse proxy server, as well as a mail proxy server, written by Igor Sysoev. For a long time, it has been running on many heavily loaded Russian sites including Yandex, Mail.Ru, VKontakte, and Rambler. According to Netcraft nginx served or proxied 17.46% busiest sites in February 2014. Here are some of the success stories: Netflix, Wordpress.com,FastMail.FM.  
## Installation
### Dependencies
you must install the dependencies, including：
gcc,openssl ,openssl-devel,pcre,pcre-devel,zlib,zlib-devel.
### Download Dependencies
You can find the appropriate package from the CD, or you can find the dependencies from the network.
### Dependencies installation
installation commands：rpm –Uvh PACKAGE_NAME.*.rpm
check wether the gcc have been installed, you should run command:
rpm –qa | grep gcc
### Nginx installation
#### down nginx tarball
here we chose the stable version：
`http://nginx.org/download/nginx-1.5.6.tar.gz`
#### Installation
configure arguments: --prefix=/data/websvr/nginx-1.5.6 --with-http_ssl_module --with-pcre --with-http_realip_module
make && make install
## The basic use of Nginx
### Nginx start and stop
start nginx:
Sbin/nginx
stop ngxin
sbin/nginx –s quit
testing configuration
    sbin/nginx -t
reload nginx configuration
sbin/nginx –s reload

### Nginx-smooth-upgrade
use these steps when upgrade version or add new modules，need compile nginx bin file first。

compiling new nginx bin file。
replace the old bin file with the new bin file.
find old nginx’s master’s pid。
kill -USR2 $pid，this command will run the new bin file。
kill -WINCH $pid，this will close old worker process smoothly。
kill -QUIT $pid,this will quit the old master process。


## Nginx common configuration
### Nginx virtual host
Nginx’s main configuration file is in nginx.conf,which in conf directory.
use server context can make virtual hosts, every server represent a virtual host. Server context is used in http context。
for instance, two virtual hosts next:
```
server {
        listen       8081;
        #server_name  somename  alias  another.alias 192.168.12.12;
        access_log    logs/port_8081.log;
        root html/8081
}
server {
        listen       8082;
        #server_name  somename  alias  another.alias 192.168.12.12;
        access_log    logs/port_8082.log;         root html/8082
}
```
the two sections above respectively represent a virtual hosts，use port 8081 and port 8082.
Nginx simple load forwarding configuration
#load balance server group
```
upstream app {
         ip_hash;
         server  192.168.132.105:80 weight=3 max-fails=3 fail_timeout=20s;
         server  192.168.132.106;
    }

#virtual host
server {
        listen       8081;
        #server_name  somename  alias  another.alias 192.168.12.12;
        access_log    logs/navi.log
        
        location /telecomnavi {
            proxy_pass http://app
        }
    }
```
the Location directive in Server context can match request uri begin with “/telecomnavi” then send it to app upstream group, this can implement simple load balancing.

the upstream directive must in http context，represents for a group of servers that providing the same service，the group including two server ,192.168.132.105 and 192.168.132.106,the priority of 192.168.132.105 is 3，maximum number of error can be toleranced is 3，timeout value is 20s.

the default priority of 192.168.132.106 is 1，if you not set the priority, the default value is 1. the configuration above is means if there are 4 requests come in，3 will be sent to server 105，1 will be sent to 106.
nginx redirection configuration
let uri begin with documents,redirction to storage
```
location /documents {
rewrite  ^/documents/(.*)  /storage/$1  redirect;
}
```
### nginx log rotate configuration
1. first create new logrotate configuration file.
touch /etc/logrotate.d/nginx
2. edit the new configuration file
vi /etc/logrotate.d/nginx
add following contents：
```
#start of logrotate file
/home/logs/hd/*.log
/home/logs/search/*.log

 {
    daily
    dateext
    rotate 99
    copytruncate
    nocompress
    missingok
    notifempty
    sharedscripts
    size 100M
    postrotate
         /bin/kill -HUP `/bin/cat /data/nginx-1.5.6/logs/nginx.pid`
    endscript
}
# end of logrotate file
```
3. create crontab job, at 0 everyday run the script。

`0 0 * * *   /usr/sbin/logrotate -f /etc/logrotate.d/nginx`

linux system time synchronize.
add the crontab job:
`0 */1 * * * /usr/sbin/ntpdate -u IP_Add;/sbin/hwclock –w`
IP_Add is the ip of time server.

## Nginx Performance Optimization Configuration
### Linux OS Optimization
#### Optimize the restriction of system resources
`ulimit –n `
The maximum number of open file descriptors.
`ulimit –u `
The maximum number of processes available to a single user.
```
vi /etc/security/limits.conf , add:
* soft nofile 1000000
* hard nofile 1000000
* soft nproc 1000000
* hard nproc 1000000
```
then reboot the system.
Optimization the disk write operation
close atime write:
open the fstab file:
vi /etc/fstab
at defaults add noatime and nodiratiome， just like down：
```
/dev/sdb1    /dataext3    defaults,noatime,nodiratime    0    0
then reboot the system, or use remount command to make it work:
mount -o defaults,noatime,nodiratime -o remount /dev/sdb1 /sdb
mount -o defaults,noatime,nodiratime  /dev/sdb1 /sdb
```
Optimization TCP kernel option
vi /etc/sysctl.conf
```
########## add by admin###########
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_keepalive_time = 30

net.core.somaxconn = 262144
net.core.rmem_default = 262144
net.core.wmem_default = 262144
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 4096 16777216
net.ipv4.tcp_wmem = 4096 4096 16777216
net.ipv4.tcp_mem = 786432 2097152 3145728
net.ipv4.tcp_max_syn_backlog = 262144
net.core.netdev_max_backlog = 262144
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_max_orphans = 262144
net.ipv4.ip_local_port_range = 1024 65535
kernel.sem = 250 256000 100 1024
net.ipv4.tcp_max_tw_buckets = 6000
net.nf_conntrack_max = 655360
# add end#
```
then save the configuration use command : sysctl -p

## Nginx configuration optimization

use epoll
after linux 2.6 kernel can support this feature.

connection optimization
worker_connections 65535
keepalive_timeout 60
client_header_buffer-size 8k (use “getconf PAGESIZE” command to get the page size)
worker_rlimit_nofile 2000000

## nginx security configuration
close server_tokens
at http context add 
` server_tokens off;`

close autoindex and ssi module
you need to recompile the nginx bin file
```
#./configure --without-http_autoindex_module --without-http_ssi_module 
# make 
# make install
```
