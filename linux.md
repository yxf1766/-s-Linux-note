## v	···前置配置：

```shell
yum install -y vim 
```

## LVM物理卷

创建serverA，serverB ，添加硬盘 

serverA 添加一块15G 硬盘 ，serverB添加一块 10G 硬盘 

### ServerA

新建一个15G的云硬盘，云硬盘名称为A-15，挂载到serverA;

将新加的硬盘整盘（无须分区）格式化为xfs文件系统，编辑/etc/fstab文件实现以UUID的形式开机自动挂载至/data/web_data目录。

```shell
lsblk 	#查询硬盘是否创建成功
```

#创建物理卷，创建卷组，创建逻辑卷

```shell
pvcreate /dev/sdb		#创建物理卷
vgcreate -s 8M datastore /dev/sdb 		#创建卷组
vgcreate -s 卷组尺寸  卷组名字 物理卷地址
lvcreate -L 8G	-n database datastore	#创建逻辑卷
lvcreate -L 逻辑卷大小 -n 逻辑卷名字 卷组名字
```

#格式化逻辑卷 格式为xfs文件系统 

```shell
mkfs.xfs /dev/datastore/database
```

#编辑/etc/fstab文件实现以UUID的形式将逻辑卷开机自动挂载至/data/web_data目录

```shell
mkdir -pv /data/web_data	#创建文件夹目录
mount /dev/datastore/database /data/web_data 	#将逻辑卷挂载到data/web_data目录
blkid		#查询UUID并且获取 逻辑卷挂载的 UUID
vi /etc/fstab		#在最后面添加
UUID=15cf1d62-3d2b-4436-b4e7-851292d9e672	xfs		defaults	0 0		
#UUID地址是你上面获取到的
/dev/mapper/datastore-database /data/web_data	xfs	 	defults	 0 0	#挂载
/dev/mapper/逻辑卷地址 	要挂载的地址	xfs文件格式  #后面的跟着就好了，解释太麻烦了 
reboot #重启 如正常进入系统就是可以了 
```

### ServerB

新建一个10G的云硬盘，名称为B-10，挂载到serverB；

将新加的硬盘整盘（无须分区）格式化为xfs文件系统，编辑/etc/fstab文件实现以UUID的形式开机自动挂载至/data/database目录。

```shell
lsblk		#查询是否安装成功
```

#创建物理卷，创建卷组，创建逻辑卷

```shell
pvcreate /dev/sdb		#创建物理卷
vgcreate -s 8M datastore /dev/sdb 		#创建卷组
vgcreate -s 卷组尺寸  卷组名字 物理卷地址
lvcreate -L 8G	-n database datastore	#创建逻辑卷
lvcreate -L 逻辑卷大小 -n 逻辑卷名字 卷组名字
```

#格式化逻辑卷 格式为xfs文件系统 

```shell
mkfs.xfs /dev/datastore/database
```

#编辑/etc/fstab文件实现以UUID的形式将逻辑卷开机自动挂载至/data/database目录

```shell
mkdir -pv /data/web_data	#创建文件夹目录
mount /dev/datastore/database /data/web_data 	#将逻辑卷挂载到data/web_data目录
blkid		#查询UUID并且获取 逻辑卷挂载的 UUID
vi /etc/fstab		#在最后面添加
UUID=15cf1d62-3d2b-4436-b4e7-851292d9e672	xfs		defaults	0 0		#UUID地址是你上面获取到的
/dev/mapper/datastore-database /data/web_data	xfs	 	defults	 0 0	#挂载
/dev/mapper/逻辑卷地址 	要挂载的地址	xfs文件格式  #后面的跟着就好了，解释太麻烦了  
reboot //重启 如正常进入系统就是可以了
```

## NFS

### ServerA

配置NFS服务

- 共享目录/data/web_data，以读写权限进行共享
- 共享目标：192.168.1XX.0/24

```shell
yum install -y nfs-utils		#安装nfs
systemctl start rpcbind
systemctl start nfs				#启动nfs
```

#创建共享目录/data/web_data

```shell
mkdir -pv /data/web_data
chmod 755 /data/web_data		#给与目录读写权限
```

#写入配置文件 共享信息

```shell
vi /etc/exports
/data/web_data		192.168.1xx.0/24(rw,sync,no_root_squash,no_all_squash)
# /data/web_data/: 共享目录位置。
# 192.168.1XX.0/24: 客户端 IP 范围，* 代表所有，即没有限制。
# rw: 权限设置，可读可写。
# sync: 同步共享目录。
# no_root_squash: 可以使用 root 授权。
# no_all_squash: 可以使用普通用户授权。
```

```shell
systemctl restart nfs		#重启nfs 并检查是否有输入错误 如没有报错就对
```

#检查本地共享目录

```shell
showmount -e localhost
```

### ServerB

- 配置NFS服务
- 编辑/etc/fstab配置文件实现开机自动挂载serverA的NFS共享至/data/web_data目录
- 系统启动过程中网络不可用时系统将自动停止挂载操作

```shell
yum install -y nfs-utils		#安装nfs
systemctl start rpcbind			
```

#查看服务端口的共享目录

```shell
showmount -e 192.168.1xx.11
```

#创建挂载文件 ；挂载nfs

```shell
mkdir -pv /data/web_data	#创建目录
mount -t nfs 192.168.1xx.11:/data/web_data /data/web_data		#挂载
mount -t nfs 服务端ip：共享的目录 /需要挂载到的地方
mount 	//查看
```

#开机自动挂载

```shell
vi /etc/fstab		#开机挂载存放目录
192.168.100.11：/data/web_data 	/data/web_data		nfs		defaults,_netdev	0 0
#服务端ip：/共享目录		/挂载地址		文件类型	后面的默认不用管它
```

## Samba

### ServerA

​	配置Samba

- 修改工作组为WORKGROUP
- 注释[homes]和[printers]的内容
- 共享名为webdata
- 共享目录为/data/web_data，且apache用户对该目录有读写执行权限，用setfacl命令配置目录权限。
- 仅允许192.168.1XX.33的主机访问

安装samba ，samba-client

```shell
yum install -y samba samba-client
```

创建用户,设置samba密码

```shell
useradd apache		#创建用户 ，用户名为apach
smbpasswd -a apache 	#进入，设置samba密码
```

设置开机启动

```
systemctl enable --now smb nmb
```

修改配置文件

```shell
vim /etc/samba/smb.conf
[global]
		workgroup = WORKGROUP # 修改工作组为WORKGROUP
# 注释[homes]和[printers]的内容
#[homes]
#       comment = Home Directories
#       valid users = %S, %D%w%S
#       browseable = No
#       read only = No
#       inherit acls = Yes
#
#[printers]
#       comment = All Printers
#       path = /var/tmp
#       printable = Yes
#       create mask = 0600
#       browseable = No
修改名字为
[webdata]
        comment = webdata 	# 共享名为webdata
        browseable = yes
        writable = yes
        valid users = apache 	# webdata可写且仅允许用户apache访问
        write list = apache
        path = /data/web_data 	# 共享目录为/data/web_data
        hosts allow = 192.168.237.156 	# 仅允许192.168.1XX.33的主机访问
# apache用户对该目录有读写执行权限 setfacl命令配置目录权限
setfacl -m u:apache:rwx /data/web_data
systemctl restart smb nmb 		#重启服务 生效配置
```

### serverB

安装samba，安装samba-client

```shell
yum install -y samba samba-client
```

开机挂载

```shell
vim /etc/fstab  	#在最后面添加
//192.168.100.11/webdata	/data/web_data	cifs	defaults,username=apache,password=123456 0 0
#//主机ip：/创建的工作组名字	/挂载目录 		文件格式	defaults,username=创建的用户名，password=设置的smb密码
```

测试连接

```shell
smbclient -L 192.168.100.11 -U apach
#smbclient -L 服务端 -U 服务端创建的用户名
# 如连接不上
#把 /etc/resolv.conf中的nameserver 114.114.114.114注释
```

测试挂载

```shell
mount -t cifs -o username=apache,password=123456 //192.168.100.11/webdata	/data/web_data
#mount -t 文件格式 usernam=用户名，passwor=smb密码 //服务端ip/共享名	/挂载目录
```

## DNS

### ServerA

安装named

```shell
yum install -y bind bind-utils
```

```shell
#修改配置文件
vim /etc/named.conf
#修改
listen-on port 53 { any; };
allow-query     { any; };
recursion yes;
#添加以下两个代码块
zone "rj.com" IN {
	type master;
	file "rj.com.zone";
	allow-update{ none; };
};
zone "0.16.172.in-addr.arpa" IN {
	type master;
	file "172.16.0.zone";
	allow-update{ none; };
};
#复制重命名配置文件 
cd /var/named
cp -p named.localhost rj.com.zone
cp -p named.loopback 172.16.0.zone
#配置正向文件
vim rj.com.zone
$TTL 1D
@       IN SOA  rj.com. rname.invalid. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      @
        A       127.0.0.1
        AAAA    ::1
        NS      ns.rj.com.
ns      IN A    172.16.0.132
www     IN A    172.16.0.132
ftp     IN A    172.16.0.133
#配置反向文件
vim 172.16.0.zone
$TTL 1D
@       IN SOA  rj.com. rname.invalid. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      @
        A       127.0.0.1
        AAAA    ::1
        PTR     localhost.
        NS      ns.rj.com.
ns       A      172.16.0.132
132     PTR     www.rj.com.
133     PTR     ftp.rj.com.
#启动服务
systemctl start dns
#修改网络 配置DNS1
BOOTPROTO=static
ONBOOT=yes
IPADDR=172.16.0.22
PREFIX=24
DNS1=172.16.0.22
#查看DNS配置
vim /etc/resolv.conf
# Generated by NetworkManager
nameserver 172.16.0.22
#验证DNS
[root@localhost named]# nslookup 
> www.rj.com
Server:         172.16.0.22
Address:        172.16.0.22#53

Name:   www.rj.com
Address: 172.16.0.132
> 172.16.0.132
132.0.16.172.in-addr.arpa       name = www.rj.com.
> ftp.rj.com
Server:         172.16.0.22
Address:        172.16.0.22#53

Name:   ftp.rj.com
Address: 172.16.0.133
> 172.16.0.133
133.0.16.172.in-addr.arpa       name = ftp.rj.com.
```

### 配置区域传送

```shell
options {
        listen-on port 53 { any; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        allow-recursion{ 172.16.120.33/24; };#允许递归的ip
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { any; };

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;#递归开启

        dnssec-enable yes;
        dnssec-validation yes;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.root.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "rj.com" IN {
        type master;
        file "rj.com.zone";
        allow-transfer{ 172.16.120.33/24; };#允许传输到那个ip上
};
```



## Nginx

### serverA

配置文件名为wordpress.conf，放置在/etc/nginx/conf.d/目录下；

使用www.rj.com作为域名进行访问；

网站根目录为/data/web_data；

提供http功能，仅监听192.168.1XX.22 IP地址；

启用FastCGI功能，让nginx能够解析php请求；

index.html内容使用Welcome to 2019 Computer Network Application contest!

```shell
#安装nginx，解压，进入文件目录
yum install -y nginx* php*
#进人nginx/conf.d 创建wordpress.conf
#配置wordpress.conf
server {
        listen          192.168.100.11:80;
        server_name     www.rj.com;
        root            /data/web_data;
		location \ {
			index index.html;
		}
        location ~ \.php$ {
                root    /data/web_data;
                fastcgi_pass    127.0.0.1:9000;
                fastcgi_index   index.php;
                fastcgi_param   SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include         fastcgi_params;
        }

}

```

### ServerB

```shell
#创建wordpress.conf
server {
        listen  192.168.100.12:80;
        server_name     www.rj.com;
        root    /data/web_data;
		location /{
			index index.php;
		}
        location ~ \.php$ {
                root    /data/web_data;
                fastcgi_pass    127.0.0.1:9000;
                fastcgi_index   index.php;
                fastcgi_param   SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include         fastcgi_params;
        }



}
```



## Nginx 代理（proxy）

### serverA

```shell
vim proxy.conf
upstream web{
        server 192.168.100.11;
        server 192.168.100.12;
}
server {
        listen 172.16.110.22:80;
        server_name www.rj.com;
        location /{
        proxy_pass http://web;
        }
}
server {
        listen 172.16.110.22:443 ssl http2 default_server;
        server_name www.rj.com;
        ssl_certificate "/etc/nginx/ssl/nginx.crt";
        ssl_certificate_key "/etc/nginx/ssl/nginx.key"; 
        ssl_session_cache  shared:SSL:1m;
        ssl_session_timeout     10m;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        location / {
                proxy_pass http://web;
        }
}
```

## CA

```shell
#进入/etc/pki/CA/private
#输入以下内容
[root@serverB private]# (umask 077;openssl genrsa -out cakey.pem 4096)
[root@serverB private]# openssl req -new -x509 -key cakey.pem -out /etc/pki/CA/cacert.pem -days 3650
[root@serverB CA]# touch /etc/pki/CA/index.txt
[root@serverB CA]# echo 01 > serial
#在A上生成请求文件传到B上认证#
[root@serverA nginx]# mkdir ssl
[root@serverA ssl]# (umask 066;openssl genrsa -out nginx.key 1024)
[root@serverA ssl]# openssl req -new -key nginx.key -out nginx.csr
[root@serverA ssl]# scp ./nginx.csr root@172.16.0.139:/data
#认证从A上传过来的请求文件 生成证书#
[root@serverB CA]# openssl ca -in /data/nginx.csr -out /etc/pki/CA/certs/nginx.crt -days 365
[root@serverB CA]# cat index.txt
[root@serverB certs]# scp ./nginx.crt root@172.16.0.138:/etc/nginx/ssl
```

## Mariadb

### serverB

```shell
#安装 mariadb数据库
yum install -y mariadb mariadb-server
#启动数据库
systemctl start mariadb mariadb.service
#把/data/database权限给mysql
chown -R mysql /data/database
#修改/etc/my.cnf文档内容
[mysqld]
datadir=/data/database
socket=/var/lib/mysql/mysql.sock
skip_name_resolve=1
innodb_file_per_table=1
bind-address=192.168.100.12
#进入数据库 初始化数据库 设置用户等；
mysql -u root -pCRE
#创建数据库
 mysql_secure_installation 
MariaDB [(none)]> CREATE DATABASE wordpress;
MariaDB [(none)]> CREATE USER 'wordpress'@'localhost' IDENTIFIED BY '123456';
MariaDB [(none)]> FLUSH privileges;
MariaDB [(none)]> show databases;
#修改默认密码
MariaDB [(none)]> use mysql;
MariaDB [mysql]> UPDATE user SET password=password('123456') WHERE user='root';
MariaDB [mysql]> flush privileges;
#重启数据库 如重启成功 进入数据库 输入以下内容
MariaDB [(none)]> GRANT ALL ON wordpress.* to wordpress@'192.168.100.%' IDENTIFIED BY '123456';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'192.168.100.%'IDENTIFIED BY '123456' WITH GRANT OPTION;
```

## HTTP

serverA

```
<VirtualHost 192.168.23.100:80>
        ServerName      www.rj.com
        DocumentRoot    "/data/web_data/"
        SSLEngine on
        SSLCertificateFile /etc/httpd/ssl/httpd.crt
        SSLCertificateKeyFile /etc/httpd/ssl/httpd.key
        <Directory "/data/web_data/">
                Options FollowSymLinks
                AllowOverride   All
                Require all     granted
        </Directory>
</VirtualHost>
#Listen 443
Listen 192.168.23.100:443 https
<VirtualHost *:443>
        ServerName      rj.com
        ServerAlias     www.rj.com
        DocumentRoot    /data/web_data/
#        ErrorLog        /data/web_data/log/error.log 
        <Directory "/data/web_data/">
                Options FollowSymLinks
                AllowOverride   All
                Require all     granted
        </Directory>
</VirtualHost>
```

