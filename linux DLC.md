# apache配置

 ```shell
 yum -y install httpd mod_ssl
 Addtype application/x-httpd-php .php .phtml .phps #开启php支持
 
 ```

## apache身份验证

```shell
#虚拟主机或httpd.conf添加以下内容  
        <Directory "/data/web_data">
                AllowOverride AuthConfig#启动认证
                AuthType Basic#认证类别 基础
                AuthName "local admin"#认证名
                AuthUserFile /data/web_data/user_pass#认证用户文件
                Require user admin#指定admin用户访问
        </Directory>    
<VirtualHost>
#cd到要设置用户认证的目录
#输入
htpasswd -c user_pass admin#添加验证用户并生成验证文件

```



## ca证书

配置https功能，https所用的证书httpd.crt、私钥httpd.key放置在/etc/httpd/ssl目录中（目录需自己创建）并使用以下命令openssl pkcs12 -export -out httpd.pfx -inkey httpd.key -in httpd.crt将证书和私钥合成pfx格式的证书给serverB的IIS服务使用

```shell
[root@servera httpd]# mkdir ssl#创建ssl目录
[root@servera httpd]# cd ssl#切换过去
[root@servera ssl]# (umask 066;openssl genrsa -out httpd.key 1024)#生成一个1024的密钥 httpd.key
Generating RSA private key, 1024 bit long modulus
..............++++++
...................++++++
e is 65537 (0x10001)
[root@servera ssl]# openssl req -new -key httpd.key -out httpd.csr#用刚才密钥生成一个证书请求文件传输到证书服务器等待颁发
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:FJ
Locality Name (eg, city) [Default City]:FZ
Organization Name (eg, company) [Default Company Ltd]:rj
Organizational Unit Name (eg, section) []:it
Common Name (eg, your name or your server's hostname) []:www.rj.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:000000
An optional company name []:ER
[root@servera ssl]# ls
httpd.csr  httpd.key
#将serverA生成的证书请求文件传输到serverb serverb在基于该请求文件颁发httpd.crt证书再传输回servera的/etc/httpd/ssl目录
[root@servera ssl]# ls
httpd.crt  httpd.csr  httpd.key
#将httpd.crt证书和httpd.key密钥合成为pfx格式证书传输到serverb给serverB的iis服务使用
[root@servera ssl]# openssl pkcs12 -export -out httpd.pfx -inkey httpd.key -in httpd.crt 
Enter Export Password:
Verifying - Enter Export Password:
[root@servera ssl]# ls
httpd.crt  httpd.csr  httpd.key  httpd.pfx
```

## 自签名证书&添加curl信任

```shell
cd /etc/pki/tls/certs/
vi ca-bundle.crt 
#把证书信息复制进去 即可添加curl信任

```





## 编写虚拟主机

l 使用www.rj.com作为域名进行访问； 

l 网站根目录为/data/web_data；

l 仅监听192.168.1XX.22 IP地址的80和443端口；

l index.html内容使用Welcome to 2019 Computer Network Application contest!

```shell
mv ./ssl.conf ssl.conf.bak
<VirtualHost 192.168.150.22:80>#监听192.168.150.22IP 80端口
        ServerName      www.rj.com#域名
        DocumentRoot    "/data/web_data"#文档目录
        SSLEngine       on#开启ssl
        SSLCertificateFile /etc/httpd/ssl/httpd.crt#证书目录
        SSLCertificateKeyFile /etc/httpd/ssl/httpd.key#密钥目录
        <Directory "/data/web_data/">
                Require all     granted
        </Directory>
</VirtualHost>
<VirtualHost 192.168.150.22:443>
        ServerName      rj.com
        ServerAlias     www.rj.com
        DocumentRoot    /data/web_data/
        <Directory "/data/web_data/">  
                Require all     granted
        </Directory>
</virtualHost>
```

## sslconf的配置

```shell
[root@serverA conf.d]# vi ssl.conf
<VirtualHost *:443>
        ServerName www.rj.com
        DocumentRoot "/data/web_data"
        SSLEngine on 
        SSLCertificateFile /etc/httpd/ssl/httpd.crt     
        SSLCertificateKeyFIle /etc/httpd/ssl/httpd.key
        SSLProtocol all -SSLv2 -SSLv3
        <Directory "/data/web_data">
                AllowOverride AuthConfig
                Authtype Basic
                AuthName "local admin"
                AuthUserFile /data/web_data/user_pass
                Require user admin
        </Directory>
</VirtualHost>
```



# 配置Haproxy

\1) 配置Haproxy ，使用listen实现http代理，使用frontend、backend实现https代理，具体要求如下：

l listen的配置需求如下

² 名称：http

² 监听地址：172.16.1XX.22:80

² 后端server：serverA和serverB

l frontend的配置需求如下：

² 名称：https

² 监听地址：172.16.1XX.22:443

² 模式：tcp

² 默认后端：web_server

l backend的配置需求如下：

² 名称：web_server

² 模式：tcp

² 后端server：serverA和serverB。

```shell
listen http
        bind 172.16.120.22:80#监听172.16.120.22IP 80端口
        server s1 192.168.150.22:80#serverA IP
        server s2 192.168.150.33.80#serverB IP
frontend https
        bind 172.16.120.22:443#监听443端口
        mode tcp#tcp模式
        default_backend web_server#默认后端webserver
backend web_server
        mode tcp#模式tcp
        balance roundrobin
        server s1 192.168.150.22:443#后端serverA IP
        server s2 192.168.150.33:443#后端serverB IP
```

# DNS

## 配置view视图功能

配置view视图功能，不允许定义ACL匹配客户端来源

定义internal视图

匹配172.16.1XX.0/24、192.168.1XX.0/24、127.0.0.0/8客户端地址

允许内网主机查询和递归查询

保留根域的区域配置

rj.com的区域数据文件名为rj.com.zone.internal

为www.rj.com添加A记录解析，解析至serverA、serverB的内网IP。

为ftp.rj.com添加A记录解析，解析至serverB的内网IP。

子域gz.rj.com授权给serverB主机

定义external视图

匹配任何客户端地址

禁止外网主机查询和递归查询

rj.com的区域数据文件名为rj.com.zone.external

为www.rj.com添加A记录解析，解析至serverA和serverB的公网IP。

为ftp.rj.com添加A记录解析，解析至serverB的公网IP。

子域gz.rj.com授权给serverB主机

配置反向域数据文件名为172.16.0.zone

为serverA、serverB的公网IP添加相应的PTR解析记录

```shell
view "internal" {
        match-clients{ 172.16.120.0/24; 192.168.150.0/24; 127.0.0.0/8; };
        allow-query{ any; };
        recursion yes;
zone "." IN {
        type hint;
        file "named.ca";
};
zone "rj.com" IN {
        type master;
        file "rj.com.zone.internal";
};

view "external" {
        match-clients{ any; };
        allow-query{ none; };
        recursion no;
zone "rj.com" IN {
        type master;
        file "rj.com.zone.external";
}
zone "0.16.172.in-addr.arpa" IN {
        type master;
        file "172.16.0.zone";
        };
};


#子域授权
#添加子区域的NS记录和A 记录 比如子域是gz.rj.com
#只要输入 gz即可 不要输全名
[root@servera named]# vi rj.com.zone.internal 
$TTL 1D
@       IN SOA  @ rname.invalid. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      @
        A       172.16.120.22
gz      IN NS   ns.gz
www     IN A    172.16.120.22
www     IN A    172.16.120.33
ftp     IN A    172.16.120.33
ns.gz   IN A    172.16.120.33

```

# 配置ftp虚拟用户

```shell
##使用虚拟用户认证方式，创建用户virtftp， shell为/sbin/nologin，并将虚拟用户映射至virtftp用户
[root@localhost ~]# useradd  -M -d /data/ftp_data -s /sbin/nologin virtftp # 
[root@localhost ~]# cat /etc/passwd | grep virtftp
virtftp:x:1000:1000::/data/ftp_data:/sbin/nologin
[root@localhost ~]# mkdir /data/ftp_data #创建ftp_data 文件夹
[root@localhost ~]# cd !$
cd /data/ftp_data
[root@localhost ftp_data]# touch ftp_test #创建ftp_test文件
##将/data/ftp_data属主属组更改为virtftp，并且属主有写权限
[root@localhost ftp_data]# chown -R virtftp:virtftp /data/ftp_data
[root@localhost ftp_data]# chmod u+w !$
chmod u+w /data/ftp_data
[root@localhost ftp_data]# setfacl -m u:virtftp:rwx /data/ftp_data/
[root@localhost ftp_data]# getfacl !$
getfacl /data/ftp_data/
getfacl: Removing leading '/' from absolute path names
# file: data/ftp_data/
# owner: virtftp
# group: virtftp
user::rwx
user:virtftp:rwx
group::r-x
mask::rwx
other::r-x
###/etc/pam.d/vsftpd.vu，（pam配置文件）
[root@localhost vsftpd]# cd /etc/pam.d/
[root@localhost pam.d]# vim vsftpd.vu #创建pam配置文件 
[root@localhost pam.d]# cat vsftpd.vu 
auth                required     pam_userdb.so   db=/etc/vsftpd/vlogin 
account             required     pam_userdb.so   db=/etc/vsftpd/vlogin
[root@localhost pam.d]# 

[root@localhost vsftpd]# grep ^[^#] /etc/vsftpd/vsftpd.conf
anonymous_enable=YES
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
listen=NO
#下面几乎都要添加
listen_port=21#监听21端口
listen_ipv6=YES#默认配置 无需添加 监听ipv6地址
pam_service_name=vsftpd.vu#默认配置 无需添加 pam配置文件默认vsftpd 改为vsftpd.vu
userlist_enable=YES#启用用户列表
tcp_wrappers=YES#I DON KNOW what means of this config this is a recommend config without edit 
guest_enable=YES#启用虚拟用户
guest_username=virtftp#虚拟用户对应账户名
user_config_dir=/etc/vsftpd/ftp_user#虚拟用户配置文件目录
allow_writeable_chroot=YES
pasv_promiscuous=YES
max_clients=100#最大100客户端
max_per_ip=3#最大ip3个
pasv_enable=YES#启动被动模式
pasv_min_port=9000#被动模式最小端口
pasv_max_port=9200#被动模式最大端口
##/etc/vsftpd/ftp_user （该目录下ftp用户权限配置目录）
[root@localhost vsftpd]# tail /etc/vsftpd/ftp_user/*
==> /etc/vsftpd/ftp_user/ftpadmin <==
anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES
anon_umask=022

==> /etc/vsftpd/ftp_user/ftpuser <==
anon_upload_enable=YES
[root@localhost vsftpd]# 
##/etc/vsftpd/vlogin.db, （用户数据库）
[root@localhost vsftpd]# vim vlogin.txt#创建账户列表文档 存放账号密码信息 稍后转换成虚拟用户数据库db格式
[root@localhost vsftpd]# cat vlogin.txt 
ftpuser#账户
123456#密码
ftpadmin
123456
[root@localhost vsftpd]# db_load -T -t hash -f /etc/vsftpd/vlogin.txt  /etc/vsftpd/vlogin.db#将刚刚的账户列表转换成数据库
[root@localhost vsftpd]# chmod 700 /etc/vsftpd/vlogin.db 
[root@localhost vsftpd]# chmod 777 /data/ftp_data/
####验证测试
[root@localhost vsftpd]# ftp 192.168.71.129
Connected to 192.168.71.129 (192.168.71.129).
220 (vsFTPd 3.0.2)
Name (192.168.71.129:root): ftpuser
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
227 Entering Passive Mode (192,168,71,129,35,158).
150 Here comes the directory listing.
-rw-r--r--    1 1000     1000            0 Sep 12 16:03 ftp_test
226 Directory send OK.
ftp> put vlogin.db 
local: vlogin.db remote: vlogin.db
227 Entering Passive Mode (192,168,71,129,35,204).
150 Ok to send data.
226 Transfer complete.
12288 bytes sent in 3.1e-05 secs (396387.10 Kbytes/sec)
ftp> ls
227 Entering Passive Mode (192,168,71,129,35,134).
150 Here comes the directory listing.
-rw-r--r--    1 1000     1000            0 Sep 12 16:03 ftp_test
-rw-------    1 1000     1000        12288 Sep 12 17:24 vlogin.db
226 Directory send OK.
[root@localhost vsftpd]# ftp 192.168.71.129
Connected to 192.168.71.129 (192.168.71.129).
220 (vsFTPd 3.0.2)
Name (192.168.71.129:root): ftpadmin
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
227 Entering Passive Mode (192,168,71,129,35,100).
150 Here comes the directory listing.
-rw-r--r--    1 1000     1000            0 Sep 12 16:03 ftp_test
-rw-------    1 1000     1000        12288 Sep 12 17:24 vlogin.db
226 Directory send OK.
ftp> rename vlogin.db rename.txt
350 Ready for RNTO.
250 Rename successful.
ftp> ls
227 Entering Passive Mode (192,168,71,129,35,69).
150 Here comes the directory listing.
-rw-r--r--    1 1000     1000            0 Sep 12 16:03 ftp_test
-rw-------    1 1000     1000        12288 Sep 12 17:24 rename.txt
226 Directory send OK.
ftp> 
# db_load -T -t hash -f /etc/vsftpd/vuser.txt /etc/vsftpd/vlogin.db
```

# RAID10

```shell
#添加4块4G硬盘 用lsblk查分区和硬盘列表
[root@serverb ~]# lsblk
sdb               8:16   0    2G  0 disk 
sdc               8:32   0    2G  0 disk 
sdd               8:48   0    2G  0 disk 
sde               8:64   0    4G  0 disk 
#分区完 p查看分区表 w保存并写入到硬盘
[root@serverb ~]# lsblk
sdb               8:16   0    2G  0 disk 
└─sdb1            8:17   0    2G  0 part 
sdc               8:32   0    2G  0 disk 
└─sdc1            8:33   0    2G  0 part 
sdd               8:48   0    2G  0 disk 
└─sdd1            8:49   0    2G  0 part 
sde               8:64   0    4G  0 disk 
├─sde1            8:65   0    1K  0 part 
├─sde5            8:69   0    2G  0 part 
└─sde6            8:70   0    2G  0 part 
#yum 安装mdadm工具 创建 raid10
[root@serverb ~]# yum -y install mdadm
#-C 创建 v显示创建过程 /dev/md10#创建的raid10的设备名 -a yes #检测设备名称 -n 要使用的磁盘数 -l 要创建的raid级别 要用到的硬盘硬盘名
mdadm -Cv /dev/md10 -a yes -n 4 -l 10 /dev/sdb /dev/sdc /dev/sdd /dev/sde
#lsblk 查看
[root@serverb ~]# lsblk
sdb               8:16   0    2G  0 disk   
├─sdb1            8:17   0    2G  0 part   
└─md10            9:10   0    4G  0 raid10 
sdc               8:32   0    2G  0 disk   
├─sdc1            8:33   0    2G  0 part   
└─md10            9:10   0    4G  0 raid10 
sdd               8:48   0    2G  0 disk   
├─sdd1            8:49   0    2G  0 part   
└─md10            9:10   0    4G  0 raid10 
sde               8:64   0    4G  0 disk   
├─sde1            8:65   0    1K  0 part   
├─sde5            8:69   0    2G  0 part   
├─sde6            8:70   0    2G  0 part   
└─md10            9:10   0    4G  0 raid10 
#raid10 创建成功
#raid 10 分出1G 格式化为swap 分区 设置为开机自动生效
[root@serverb ~]# fdisk /dev/md10
命令(输入 m 获取帮助)：n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
分区号 (1-4，默认 1)：
起始 扇区 (2048-8376319，默认为 2048)：
将使用默认值 2048
Last 扇区, +扇区 or +size{K,M,G} (2048-8376319，默认为 8376319)：+1G
分区 1 已设置为 Linux 类型，大小设为 1 GiB
令(输入 m 获取帮助)：w
The partition table has been altered!
Calling ioctl() to re-read partition table.
正在同步磁盘。
[root@serverb ~]# lsblk
sdb               8:16   0    2G  0 disk   
├─sdb1            8:17   0    2G  0 part   
└─md10            9:10   0    4G  0 raid10 
  └─md10p1      259:0    0    1G  0 md     
sdc               8:32   0    2G  0 disk   
├─sdc1            8:33   0    2G  0 part   
└─md10            9:10   0    4G  0 raid10 
  └─md10p1      259:0    0    1G  0 md     
sdd               8:48   0    2G  0 disk   
├─sdd1            8:49   0    2G  0 part   
└─md10            9:10   0    4G  0 raid10 
  └─md10p1      259:0    0    1G  0 md     
sde               8:64   0    4G  0 disk   
├─sde1            8:65   0    1K  0 part   
├─sde5            8:69   0    2G  0 part   
├─sde6            8:70   0    2G  0 part   
└─md10            9:10   0    4G  0 raid10 
  └─md10p1      259:0    0    1G  0 md  
#成功创建了 1G分区 md10p1
#将md10p1 格式化为swap
[root@serverb ~]# mkswap /dev/md10p1
正在设置交换空间版本 1，大小 = 1048572 KiB
无标签，UUID=7f618ee5-a6db-429d-9574-52677e93d0ae
#设置开机自动生效
[root@serverb ~]# vi /etc/fstab 
/dev/md10p1 swap                               swap    defaults        0 0
#给md10创建物理卷 卷组
[root@serverb ~]#  pvcreate /dev/md10
  Physical volume "/dev/md10" successfully created.
[root@serverb ~]# vgcreate vg1 /dev/md10
  Volume group "vg1" successfully created
  #创建逻辑卷lv0 lv1 格式化 xfs格式 创建/lvm0 /lvm1目录 使这两个逻辑卷开机挂载这两个目录
[root@serverb ~]# lvcreate -L 1.5G -n lv0 vg1
  Logical volume "lv0" created.
[root@serverb ~]# lvcreate -L 1.5G -n lv1 vg1
  Logical volume "lv1" created.
[root@serverb ~]# mkfs.xfs /dev/mapper/vg1-lv0
meta-data=/dev/mapper/vg1-lv0    isize=512    agcount=8, agsize=49024 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=392192, imaxpct=25
         =                       sunit=128    swidth=256 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@serverb ~]# mkfs.xfs /dev/mapper/vg1-lv1
meta-data=/dev/mapper/vg1-lv1    isize=512    agcount=8, agsize=49024 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=392192, imaxpct=25
         =                       sunit=128    swidth=256 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@serverb ~]# mkdir /lvm0
[root@serverb ~]# mkdir /lvm1
"/etc/fstab" 14L, 649C written
[root@serverb ~]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Sat Dec 25 22:51:49 2021
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/vg1-lv0 /lvm0                       xfs     defaults        0 0
/dev/mapper/vg1-lv1 /lvm1                       xfs     defaults        0 0
#将/dev/sde6分区 扩张到物理卷组vg1里 再给逻辑卷/dev/mapper/vg1-lv0 扩展 实现挂载的lvm0目录容量达到3.5G
#raid10做物理卷扩展先要把要扩展的盘设为在移出阵列重做分区表重新分区即可扩展 因为组成raid10的硬盘视为一个硬盘/组 是找不到之前的硬盘分区的 所以报设备被过滤器排除的错误
[root@serverb ~]# vgextend vg1 /dev/sde6
  Device /dev/sde6 excluded by a filter.
[root@serverb ~]# mdadm /dev/md10 -f /dev/sde
mdadm: set /dev/sde faulty in /dev/md10
[root@serverb ~]# mdadm /dev/md10 -r /dev/sde
mdadm: hot removed /dev/sde from /dev/md10
[root@serverb ~]# parted /dev/sde
GNU Parted 3.1
使用 /dev/sde
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel msdos                                                    
警告: The existing disk label on /dev/sde will be destroyed and all data on this disk will be lost. Do you
want to continue?
是/Yes/否/No? yes                                                         
(parted) quit    
[root@serverb ~]# vgextend vg1 /dev/sde6
  Physical volume "/dev/sde6" successfully created.
  Volume group "vg1" successfully extended
[root@serverb ~]# lvextend /dev/mapper/vg1-lv0 /dev/sde6
  Size of logical volume vg1/lv0 changed from 1.50 GiB (384 extents) to <3.50 GiB (895 extents).
  Logical volume vg1/lv0 successfully resized.
[root@serverb /]# lvdisplay /dev/mapper/vg1-lv0 
  --- Logical volume ---
  LV Path                /dev/vg1/lv0
  LV Name                lv0
  VG Name                vg1
  LV UUID                6nxOeu-l0CB-ql0U-r05N-QBq1-4VzQ-Hx6DgQ
  LV Write Access        read/write
  LV Creation host, time serverb, 2021-12-31 10:06:18 +0800
  LV Status              available
  # open                 1
  LV Size                <3.50 GiB
  Current LE             895
  Segments               2
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:3
   
```



# 配置kvm虚拟机

配置kvm虚拟机前 要先检查物理机的cpu支不支持虚拟化

在去虚拟机设置里把虚拟化打开

物理机要安装xmanager工具

来联接linux kvm图形化界面

crt 会话选项把 x11转发打开

```shell
#查看系统版本
[root@serverb ~]# cat /etc/centos-release
CentOS Linux release 7.9.2009 (Core)
#查看cpu是否支持虚拟化（要需要虚拟机虚拟化设置打开）
[root@serverb ~]# cat /proc/cpuinfo | egrep 'vmx|svm'
#有出现 vmx 或 svm 字样即代表支持虚拟化
#查看是否加载kvm
[root@serverb ~]# lsmod | grep kvm
kvm_amd              2177304  0 
kvm                   637289  1 kvm_amd
irqbypass              13503  1 kvm
#注意关闭防火墙
#安装kvm相关软件包
[root@serverb ~]# yum install qemu-kvm qemu-img \
virt-manager libvirt libvirt-python virt-manager \
libvirt-client virt-install virt-viewer -y
#kvm虚拟机字体包（防止出现乱码）
yum install dejavu-lgc-sans-fonts
[root@serverb ~]# yum -y install xorg-x11-font-utils
#启动libvirt  并设置开机自启动
[root@serverb~]# systemctl start libvirtd
[root@serverb ~]# systemctl enable libvirtd
#创建两个目录一个放镜像 一个放虚拟机
[root@serverb ~]# mkdir /home/iso
[root@serverb ~]# mkdir /home/images
#关闭NetworkManager服务
[root@serverb ~]# chkconfig NetworkManager off
[root@serverb ~]# service NetworkManager stop
#设置网络桥接 虚拟网卡与 ens33桥接
[root@serverb ~]# virsh iface-bridge ens33 br0
#查看桥接是否成功
[root@serverb ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.000c2924274b       yes             ens33
virbr0          8000.52540027900c       yes             virbr0-nic

#安装虚拟机图形化工具包
 yum groupinstall "Virtualization Hypervisor" "Virutalization Client","Virutalization Platform","Virtualization Tools"
#配置服务器的sshd，重启服务
# vi /etc/ssh/sshd_config
X11Forwarding yes
#systemctl restart sshd
#reboot 重启 再输入virt-manager 即可进入虚拟机图形化界面


```

