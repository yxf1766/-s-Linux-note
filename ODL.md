# ODL安装与配置

桥接模式 虚拟机必须跟真机同一个网段

```shell
sudo vim /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static#静态获取
address 10.12.25.147#和真机网络同一个网段
netmask 255.255.255.0
gateway 10.12.25.1#网关
：wq保存
#重启网络
关闭网卡 sudo ifdown eth0
开启网卡 sudo ifup eth0
```



2）解压压缩包

```shell
sudo apt-get -y install unzip
unzip distribution-karaf-0.6.0-Carbon.zip
```



3）修改一些参数

```shell
cd distribution-karaf-0.6.0-Carbon/etc

vim org.apache.karaf.management.cfg

修改：

rmiRegistryHost=127.0.0.1

rmiServerHost = 127.0.0.1 
```



4）进入karaf，安装一些功能组件

```shell
cd distribution-karaf-0.6.0-Carbon/bin

sudo ./karaf
```



按顺序安装以下功能组件

```opendaylight
feature:install odl-restconf

feature:install odl-l2switch-switch-ui

feature:install odl-mdsal-apidocs

feature:install odl-dluxapps-applications
```

### 4）进入opendaylight

打开浏览器，输入网址：http://<yourMachineIP>:8181/index.html

用户名和密码都是admin 

![控制台账户密码都是admin](C:\Users\yfc1997\Desktop\刷就完事了\odl\控制台账户密码都是admin.png)

# J卷题目配置

l 使用Mininet和OpenVswitch构建拓扑，采用采用OVSK交换机格式，端口6653,采用openflow1.3协议，构造如下拓扑：

![拓扑图](C:\Users\yfc1997\Desktop\刷就完事了\odl\拓扑图.png)

l 在浏览器上可以访问ODL管理页面并查看网元拓扑结构。

```shell
mininet@mininet-vm:~$ sudo mn --topo=tree,2,2 --controller=remote,ip=10.12.25.147
#--topo=tree(创建树型拓扑),2(交换机有两层),2(交换机下挂两台主机) --controller=remote(远程控制器)，ip=（虚拟机内ip）
*** Creating network
*** Adding controller
Connecting to remote controller at 10.12.25.147:6653
*** Adding hosts:
h1 h2 h3 h4 
*** Adding switches:
s1 s2 s3 
*** Adding links:
(s1, s2) (s1, s3) (s2, h1) (s2, h2) (s3, h3) (s3, h4) 
*** Configuring hosts
h1 h2 h3 h4 
*** Starting controller
c0 
*** Starting 3 switches
s1 s2 s3 ...
*** Starting CLI:
mininet> pingall#测试连通性
*** Ping: testing ping reachability
h1 -> h2 h3 h4 
h2 -> h1 h3 h4 
h3 -> h1 h2 h4 
h4 -> h1 h2 h3 
*** Results: 0% dropped (12/12 received)
mininet> 
```

浏览器刷新查看拓扑

![查看拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\查看拓扑.png)

![拓扑图](C:\Users\yfc1997\Desktop\刷就完事了\odl\拓扑图.png

l 通过OVS在S1下发一条流表实现H1与H2可以互通，H3与H4可以互通，但H1、H2与H3、H4间不可以连通；

```shell
mininet@mininet-vm:~$ sudo ovs-ofctl del-flows s1#删除s1的流表
mininet@mininet-vm:~$ sudo ovs-vsctl show#展示所有流表
918037ec-b307-45d7-a75a-f1ac4337d135
    Bridge "s2"
        Controller "tcp:10.12.25.147:6653"
            is_connected: true
        Controller "ptcp:6655"
        fail_mode: secure
        Port "s2-eth1"
            Interface "s2-eth1"
        Port "s2-eth3"
            Interface "s2-eth3"
        Port "s2"
            Interface "s2"
                type: internal
        Port "s2-eth2"
            Interface "s2-eth2"
    Bridge "s1"
        Controller "tcp:10.12.25.147:6653"
            is_connected: true
        Controller "ptcp:6654"
        fail_mode: secure
        Port "s1-eth1"
            Interface "s1-eth1"
        Port "s1-eth2"
            Interface "s1-eth2"
        Port "s1"
            Interface "s1"
                type: internal
    Bridge "s3"
        Controller "tcp:10.12.25.147:6653"
            is_connected: true
        Controller "ptcp:6656"
        fail_mode: secure
        Port "s3"
            Interface "s3"
                type: internal
        Port "s3-eth1"
            Interface "s3-eth1"
        Port "s3-eth3"
            Interface "s3-eth3"
        Port "s3-eth2"
            Interface "s3-eth2"
    ovs_version: "2.0.2"
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s1 in_port=1,out_port=2,actions=drop#add-flow(下发流表) S*(下发到哪个交换机)in_port=1(进入端口1),out_port=2(输出端口2),actions=drop(操作=丢弃) 
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s2 in_port=1,out_port=2,actions=drop
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s3 in_port=1,out_port=2,actions=drop
mininet@mininet-vm:~$ sudo ovs-ofctl del-flows s2#删除s2流表
mininet@mininet-vm:~$ sudo ovs-ofctl del-flows s3#删除S3流表

mininet> pingall
*** Ping: testing ping reachability
h1 -> X X X 
h2 -> h1 X X 
h3 -> X X X 
h4 -> X X h3 
*** Results: 83% dropped (2/12 received)
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 X X 
h2 -> h1 X X 
h3 -> X X h4 
h4 -> X X h3 
*** Results: 66% dropped (4/12 received)
```



用iperf工具测试H1和H2的带宽。

```shell
mininet> iperf h1 h2#测试h1和h2的带宽
*** Iperf: testing TCP bandwidth between h1 and h2 
*** Results: ['37.1 Gbits/sec', '37.1 Gbits/sec']
```

# I卷配置

l 使用Mininet和OpenVswitch构建拓扑，连接ODL的6633端口采用Openflow1.3协议，构造形成如下拓扑：

![I卷拓扑图](C:\Users\yfc1997\Desktop\刷就完事了\odl\I卷拓扑图.png)

l 在浏览器上可以访问ODL管理页面并查看网元拓扑结构

```shell
mininet@mininet-vm:~/mininet/custom$ cd
mininet@mininet-vm:~$ cd mininet/custom/
mininet@mininet-vm:~/mininet/custom$ ls
README  topo-2sw-2host.py
mininet@mininet-vm:~/mininet/custom$ vim mytopo.py #自定义拓扑
"""Custom topology example

Two directly connected switches plus a host for each switch:

   host --- switch --- switch --- host

Adding the 'topos' dict with a key/value pair to generate our newly defined
topology enables one to pass in '--topo=mytopo' from the command line.
"""

from mininet.topo import Topo

class MyTopo( Topo ):
    "Simple topology example."

    def __init__( self ):
        "Create custom topo."

        # Initialize topology
        Topo.__init__( self )

        leftSwitch = self.addSwitch( 's1' )#添加左交换机s1
        middleSwitch = self.addSwitch( 's2' )#添加中间交换机s2
        rightSwitch = self.addSwitch( 's3' )#添加右交换机s3
        leftHost = self.addHost( 'h1' )#添加左主机h1
        middleHost = self.addHost( 'h2' )#添加中间主机h2
        rightHost = self.addHost( 'h3' )#添加右主机h3
        # Add hosts and switches
        #leftHost = self.addHost( 'h1' )
        #rightHost = self.addHost( 'h2' )
        #leftSwitch = self.addSwitch( 's3' )
        #rightSwitch = self.addSwitch( 's4' )

        # Add links
        self.addLink( leftSwitch, middleSwitch )#添加链接 左边的交换机跟中间的交换机连线
        self.addLink( rightSwitch, middleSwitch )#添加链接 右边的交换机跟中间的交换机连线
        self.addLink( leftHost, leftSwitch )#添加链接 左边的主机跟左边的交换机连线
        self.addLink( middleHost, middleSwitch )#添加链接 中间的主机跟中间的交换机连线
        self.addLink( rightHost, rightSwitch )#添加链接 右边的主机跟右边的交换机连线


topos = { 'mytopo': ( lambda: MyTopo() ) }

mininet@mininet-vm:~/mininet/custom$ sudo mn --custom mytopo.py --topo mytopo --controller=remote,ip=10.12.25.147,port=6653#--custom mytopo.py(使用自定义拓扑模板) --topo mytopo(拓扑使用mytopo) ,port=6653(设置端口号6653)
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 
*** Adding switches:
s1 s2 s3 
*** Adding links:
(h1, s1) (h2, s2) (h3, s3) (s1, s2) (s3, s2) 
*** Configuring hosts
h1 h2 h3 
*** Starting controller
c0 
*** Starting 3 switches
s1 s2 s3 ...
*** Starting CLI:
mininet> pingall#所有机器互ping测试连通性
*** Ping: testing ping reachability
h1 -> X X 
h2 -> X h3 
h3 -> h1 h2 
*** Results: 50% dropped (3/6 received)
```

![I卷查看拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\I卷查看拓扑.png)

l 通过OVS给S2下发流表，使得H2与H1、H3无法互通。

```shell
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s2 in_port=3,actions=drop

mininet> pingall
*** Ping: testing ping reachability
h1 -> X h3 
h2 -> X X 
h3 -> h1 X 
*** Results: 66% dropped (2/6 received)
```



l H1启动HTTP-Server功能，WEB端口为8080，H3作为HTTP-Client，获取H1的html网页文件目录信息。

```shell
mininet> h3 lynx h1
#配置协议
mininet@mininet-vm:~$ sudo ovs-vsctl set bridge s1 protocols=OpenFlow13
mininet@mininet-vm:~$ sudo ovs-vsctl set bridge s2 protocols=OpenFlow13
mininet@mininet-vm:~$ sudo ovs-vsctl set bridge s3 protocols=OpenFlow13
mininet@mininet-vm:~$ sudo ovs-vsctl show
918037ec-b307-45d7-a75a-f1ac4337d135
    Bridge "s2"
        Controller "tcp:10.12.25.147:6653"
            is_connected: true
        Controller "ptcp:6655"
        fail_mode: secure
        Port "s2"
            Interface "s2"
                type: internal
        Port "s2-eth1"
            Interface "s2-eth1"
        Port "s2-eth2"
            Interface "s2-eth2"
        Port "s2-eth3"
            Interface "s2-eth3"
    Bridge "s1"
        Controller "tcp:10.12.25.147:6653"
            is_connected: true
        Controller "ptcp:6654"
        fail_mode: secure
        Port "s1-eth2"
            Interface "s1-eth2"
        Port "s1"
            Interface "s1"
                type: internal
        Port "s1-eth1"
            Interface "s1-eth1"
    Bridge "s3"
        Controller "tcp:10.12.25.147:6653"
            is_connected: true
        Controller "ptcp:6656"
        fail_mode: secure
        Port "s3"
            Interface "s3"
                type: internal
        Port "s3-eth1"
            Interface "s3-eth1"
        Port "s3-eth2"
            Interface "s3-eth2"
    ovs_version: "2.0.2"
mininet@mininet-vm:~$ 
```

# H卷配置

l 使用Mininet和OpenVswitch构建拓扑，连接ODL的6653端口如下拓扑结构：

![H卷拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\H卷拓扑.png)



l 在浏览器上可以访问ODL管理页面查看网元拓扑结构。

```shell
mininet@mininet-vm:~$ cd mininet/custom/
mininet@mininet-vm:~/mininet/custom$ ls
"""Custom topology example
mininet@mininet-vm:~/mininet/custom$ vim mytopo.py
Two directly connected switches plus a host for each switch:

   host --- switch --- switch --- host

Adding the 'topos' dict with a key/value pair to generate our newly defined
topology enables one to pass in '--topo=mytopo' from the command line.
"""

from mininet.topo import Topo

class MyTopo( Topo ):
    "Simple topology example."

    def __init__( self ):
        "Create custom topo."

        # Initialize topology
        Topo.__init__( self )

        Switch = self.addSwitch( 's1' )
        leftHost = self.addHost( 'h1' )
        middleHost = self.addHost( 'h2' )
        rightHost = self.addHost( 'h3' )
        # Add hosts and switches
        #leftHost = self.addHost( 'h1' )
        #rightHost = self.addHost( 'h2' )
        #leftSwitch = self.addSwitch( 's3' )
        #rightSwitch = self.addSwitch( 's4' )

        # Add links
        self.addLink( leftHost, Switch )
        self.addLink( middleHost, Switch )
        self.addLink( rightHost, Switch )


topos = { 'mytopo': ( lambda: MyTopo() ) }

mininet@mininet-vm:~/mininet/custom$ sudo mn --custom mytopo.py --topo mytopo --controller=remote,ip=10.12.25.147,port=6653
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 
*** Adding switches:
s1 
*** Adding links:
(h1, s1) (h2, s1) (h3, s1) 
*** Configuring hosts
h1 h2 h3 
*** Starting controller
c0 
*** Starting 1 switches
s1 ...
*** Starting CLI:
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 h3 
h2 -> h1 h3 
h3 -> h1 h2 
*** Results: 0% dropped (6/6 received)
```



l 通过OVS手工下发流表，H1可以ping通H3，H1、H3无法ping通H2。

```shell
mininet@mininet-vm:~$ sudo ovs-vsctl show#显示所有流表
918037ec-b307-45d7-a75a-f1ac4337d135
    Bridge "s1"
        Controller "ptcp:6654"
        Controller "tcp:10.12.25.147:6653"
            is_connected: true
        fail_mode: secure
        Port "s1-eth1"
            Interface "s1-eth1"
        Port "s1"
            Interface "s1"
                type: internal
        Port "s1-eth2"
            Interface "s1-eth2"
        Port "s1-eth3"
            Interface "s1-eth3"
    ovs_version: "2.0.2"
mininet@mininet-vm:~$ ovs-ofctl add-flows s1 in_port=2 actions=drop#添加流表 s1 进入端口号2 操作下发
mininet> pingall#所有机器互ping测试流通性
*** Ping: testing ping reachability
h1 -> X h3 
h2 -> X X 
h3 -> h1 X 
*** Results: 66% dropped (2/6 received)
```

![H卷查看拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\H卷查看拓扑.png)

l H1启动HTTP-Server功能，WEB端口为8000，H3作为HTTP-Client，获取H1的html网页配置文件。

```shell
mininet> h1 python -m SimpleHTTPServer 8000&
mininet> h3  wget -O - http://10.0.0.1:8000
--2021-11-03 06:08:33--  http://10.0.0.1:8000/
Connecting to 10.0.0.1:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 302 [text/html]
Saving to: 鈥楽TDOUT鈥


 0% [                                       ] 0           --.-K/s              <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 3.2 Final//EN"><html>
<title>Directory listing for /</title>
<body>
<h2>Directory listing for /</h2>
<hr>
<ul>
<li><a href="mytopo.py">mytopo.py</a>
<li><a href="README">README</a>
<li><a href="topo-2sw-2host.py">topo-2sw-2host.py</a>
</ul>
<hr>
</body>
</html>
100%[======================================>] 302         --.-K/s   in 0s      

2021-11-03 06:08:33 (19.3 MB/s) - written to stdout [302/302]
```

# G卷配置

l 使用Mininet和OpenVswitch构建拓扑，采用采用OVSK交换机格式，端口6653,采用openflow1.3协议，构造如下拓扑：

![G卷拓扑图](C:\Users\yfc1997\Desktop\刷就完事了\odl\G卷拓扑图.png)

l 在浏览器上可以访问ODL管理页面并查看网元拓扑结构。

```shell
mininet@mininet-vm:~$ cd mininet/custom/#进入自定义拓扑目录
mininet@mininet-vm:~/mininet/custom$ vim mytopo.py#创建自定义拓扑脚本 
"""Custom topology example

Two directly connected switches plus a host for each switch:

   host --- switch --- switch --- host

Adding the 'topos' dict with a key/value pair to generate our newly defined
topology enables one to pass in '--topo=mytopo' from the command line.
"""

from mininet.topo import Topo

class MyTopo( Topo ):
    "Simple topology example."

    def __init__( self ):
        "Create custom topo."

        # Initialize topology
        Topo.__init__( self )
		#添加s1,s2,s3交换机
        s1 = self.addSwitch( 's1' )
        s2 = self.addSwitch( 's2' )
        s3 = self.addSwitch( 's3' )
		#添加h1,h2,h3,h4主机
        h1 = self.addHost( 'h1' )
        h2 = self.addHost( 'h2' )
        h3 = self.addHost( 'h3' )
        h4 = self.addHost( 'h4' )
        # Add hosts and switches
        #leftHost = self.addHost( 'h1' )
        #rightHost = self.addHost( 'h2' )
        #leftSwitch = self.addSwitch( 's3' )
        #rightSwitch = self.addSwitch( 's4' )

        # Add links#添加连接
   		#s2跟s1连 s1跟s3连 h1,h2跟s2连 h3,h4跟s3连
        self.addLink( s2, s1 )
        self.addLink( s1, s3 )
        self.addLink( h1, s2 )
        self.addLink( h2, s2 )
        self.addLink( h3, s3 )
        self.addLink( h4, s3 )
"mytopo.py" 45L, 1122C written                                
mininet@mininet-vm:~/mininet/custom$ sudo mn --custom mytopo.py --topo mytopo --controller=remote,ip=10.12.25.147,port=6653
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 h4 
*** Adding switches:
s1 s2 s3 
*** Adding links:
(h1, s2) (h2, s2) (h3, s3) (h4, s3) (s1, s3) (s2, s1) 
*** Configuring hosts
h1 h2 h3 h4 
*** Starting controller
c0 
*** Starting 3 switches
s1 s2 s3 ...
*** Starting CLI:
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 h3 h4 
h2 -> h1 h3 h4 
h3 -> h1 h2 h4 
h4 -> h1 h2 h3 
*** Results: 0% dropped (12/12 received)
```

![G卷查看拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\G卷查看拓扑.png)

l 通过OVS在S1下发一条流表实现H1与H2可以互通，H3与H4可以互通，但H1、H2与H3、H4间不可以连通；

```shell
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s1 in_port=1,out_port=2,actions=drop
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 X X 
h2 -> h1 X X 
h3 -> X X h4 
h4 -> X X h3 
*** Results: 66% dropped (4/12 received)
mininet> 
```



用iperf工具测试H1和H2的带宽。

```shell
mininet> iperf h1 h2
*** Iperf: testing TCP bandwidth between h1 and h2 
.*** Results: ['432 Mbits/sec', '434 Mbits/sec']
```

# F卷配置

l 使用Mininet和OpenVswitch构建拓扑，连接ODL的6653端口如下拓扑结构：

![F卷拓扑图](C:\Users\yfc1997\Desktop\刷就完事了\odl\F卷拓扑图.png)

l 在浏览器上可以访问ODL管理页面查看网元拓扑结构。

```shell
mininet@mininet-vm:~$ cd mininet/custom/
mininet@mininet-vm:~/mininet/custom$ ls
README  topo-2sw-2host.py
mininet@mininet-vm:~/mininet/custom$ vim topo-2sw-2host.py 
mininet@mininet-vm:~/mininet/custom$ vim mytopo.py
"""Custom topology example

Two directly connected switches plus a host for each switch:

   host --- switch --- switch --- host

Adding the 'topos' dict with a key/value pair to generate our newly defined
topology enables one to pass in '--topo=mytopo' from the command line.
"""

from mininet.topo import Topo

class MyTopo( Topo ):
    "Simple topology example."

    def __init__( self ):
        "Create custom topo."

        # Initialize topology
        Topo.__init__( self )

        s1 = self.addSwitch( 's1' )
        s2 = self.addSwitch( 's2' )
        s3 = self.addSwitch( 's3' )

        h1 = self.addHost( 'h1' )
        h2 = self.addHost( 'h2' )
        h3 = self.addHost( 'h3' )
        # Add hosts and switches
        #leftHost = self.addHost( 'h1' )
        #rightHost = self.addHost( 'h2' )
        #leftSwitch = self.addSwitch( 's3' )
        #rightSwitch = self.addSwitch( 's4' )

        # Add links
        self.addLink( s1, s2 )
        self.addLink( s2, s3 )
        self.addLink( h1, s1 )
        self.addLink( h2, s2 )
        self.addLink( h3, s3 )


"mytopo.py" [New] 43L, 1069C written                          
mininet@mininet-vm:~/mininet/custom$ sudo mn --custom mytopo.py --topo mytopo --controller=remote,ip=10.12.25.147,port=6653
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 
*** Adding switches:
s1 s2 s3 
*** Adding links:
(h1, s1) (h2, s2) (h3, s3) (s1, s2) (s2, s3) 
*** Configuring hosts
h1 h2 h3 
*** Starting controller
c0 
*** Starting 3 switches
s1 s2 s3 ...
*** Starting CLI:
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 h3 
h2 -> h1 h3 
h3 -> h1 h2 
*** Results: 0% dropped (6/6 received)
```

![F卷查看拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\F卷查看拓扑.png)

l 通过OVS给S2下发流表，使得H2与H1、H3无法互通。

```shell
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s2 in_port=3,actions=drop
mininet> pingall
*** Ping: testing ping reachability
h1 -> X h3 
h2 -> X X 
h3 -> h1 X 
*** Results: 66% dropped (2/6 received)
```



l H1启动HTTP-Server功能，WEB端口为8080，H3作为HTTP-Client，获取H1的html网页配置文件。

```shell
mininet> h1 python -m SimpleHTTPServer 8080&
mininet> h3 wget -O - http://10.0.0.1:8080
--2021-11-03 05:53:13--  http://10.0.0.1:8080/
Connecting to 10.0.0.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 302 [text/html]
Saving to: 鈥楽TDOUT鈥


 0% [                                       ] 0           --.-K/s              <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 3.2 Final//EN"><html>
<title>Directory listing for /</title>
<body>
<h2>Directory listing for /</h2>
<hr>
<ul>
<li><a href="mytopo.py">mytopo.py</a>
<li><a href="README">README</a>
<li><a href="topo-2sw-2host.py">topo-2sw-2host.py</a>
</ul>
<hr>
</body>
</html>
100%[======================================>] 302         --.-K/s   in 0s      

2021-11-03 05:53:13 (36.4 MB/s) - written to stdout [302/302]

```

# E卷配置

使用Mininet和OpenVswitch构建拓扑，连接ODL的6653端口如下拓扑结构

![E卷拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\E卷拓扑.png)

l 在浏览器上可以访问ODL管理页面查看网元拓扑结构。

```shell
mininet@mininet-vm:~/mininet/custom$ vim mytopo.py
"""Custom topology example

Two directly connected switches plus a host for each switch:

   host --- switch --- switch --- host

Adding the 'topos' dict with a key/value pair to generate our newly defined
topology enables one to pass in '--topo=mytopo' from the command line.
"""

from mininet.topo import Topo

class MyTopo( Topo ):
    "Simple topology example."

    def __init__( self ):
        "Create custom topo."

        # Initialize topology
        Topo.__init__( self )

        s1 = self.addSwitch( 's1' )
        s2 = self.addSwitch( 's2' )
        s3 = self.addSwitch( 's3' )

        h1 = self.addHost( 'h1' )
        h2 = self.addHost( 'h2' )
        h3 = self.addHost( 'h3' )
        h4 = self.addHost( 'h4' )
        # Add hosts and switches
        #leftHost = self.addHost( 'h1' )
        #rightHost = self.addHost( 'h2' )
        #leftSwitch = self.addSwitch( 's3' )
        #rightSwitch = self.addSwitch( 's4' )

        # Add links
        self.addLink( s2, s1 )
        self.addLink( s1, s3 )
        self.addLink( h1, s2 )
        self.addLink( h2, s2 )
        self.addLink( h3, s3 )
        self.addLink( h4, s3 )

topos = { 'mytopo': ( lambda: MyTopo() ) }
~
~
~
~
~
"mytopo.py" 44L, 1121C written                                
mininet@mininet-vm:~/mininet/custom$ sudo mn --custom mytopo.py --topo mytopo --controller=remote,ip=10.12.25.147,port=6653
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 h4 
*** Adding switches:
s1 s2 s3 
*** Adding links:
(h1, s2) (h2, s2) (h3, s3) (h4, s3) (s1, s3) (s2, s1) 
*** Configuring hosts
h1 h2 h3 h4 
*** Starting controller
c0 
*** Starting 3 switches
s1 s2 s3 ...
*** Starting CLI:
mininet> pingall
*** Ping: testing ping reachability
h1 -> X h3 h4 
h2 -> h1 h3 h4 
h3 -> h1 h2 h4 
h4 -> h1 h2 h3 
*** Results: 8% dropped (11/12 received)
mininet> 
```



l 通过OVS在S1下发一条流表实现H1与H2可以互通，H3与H4可以互通，但H1、H2与H3、H4间不可以连通；

```shell
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s1 in_port=1,out_port=2,actions=drop
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 X X 
h2 -> h1 X X 
h3 -> X X h4 
h4 -> X X h3 
*** Results: 66% dropped (4/12 received)
```



l 用iperf工具测试H1和H2的带宽。

```shell
mininet> iperf h1 h2
*** Iperf: testing TCP bandwidth between h1 and h2 
.*** Results: ['36.1 Gbits/sec', '36.1 Gbits/sec']
```



# D卷配置

l 使用Mininet和OpenVswitch构建拓扑，连接ODL的6653端口如下拓扑结构：

![D卷拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\D卷拓扑.png)



l 在浏览器上可以访问ODL管理页面查看网元拓扑结构。

```shell
mininet@mininet-vm:~$ cd mininet/custom/
mininet@mininet-vm:~/mininet/custom$ ls
README  topo-2sw-2host.py
mininet@mininet-vm:~/mininet/custom$ vim topo-2sw-2host.py 
mininet@mininet-vm:~/mininet/custom$ vim mytopo.py
"""Custom topology example

Two directly connected switches plus a host for each switch:

   host --- switch --- switch --- host

Adding the 'topos' dict with a key/value pair to generate our newly defined
topology enables one to pass in '--topo=mytopo' from the command line.
"""

from mininet.topo import Topo

class MyTopo( Topo ):
    "Simple topology example."

    def __init__( self ):
        "Create custom topo."

        # Initialize topology
        Topo.__init__( self )

        Switch = self.addSwitch( 's1' )
        leftHost = self.addHost( 'h1' )
        middleHost = self.addHost('h2')
        rightHost = self.addHost( 'h3' )
        # Add hosts and switches
        #leftHost = self.addHost( 'h1' )
        #rightHost = self.addHost( 'h2' )
        #leftSwitch = self.addSwitch( 's3' )
        #rightSwitch = self.addSwitch( 's4' )

        # Add links
        self.addLink( leftHost, Switch )
        self.addLink( middleHost, Switch )
        self.addLink( rightHost, Switch )


topos = { 'mytopo': ( lambda: MyTopo() ) }
mininet@mininet-vm:~/mininet/custom$ sudo mn --custom mytopo.py --topo mytopo --controller=remote,ip=10.12.25.147,port=6653
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 
*** Adding switches:
s1 
*** Adding links:
(h1, s1) (h2, s1) (h3, s1) 
*** Configuring hosts
h1 h2 h3 
*** Starting controller
c0 
*** Starting 1 switches
s1 ...
*** Starting CLI:
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 h3 
h2 -> h1 h3 
h3 -> h1 h2 
*** Results: 0% dropped (6/6 received)
```

![D卷查看拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\D卷查看拓扑.png)

l 通过OVS手工下发流表，优先级为50，H1可以ping通H3，H1、H3无法ping通H2。

```shell
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s1 priority=50,in_port=2,action=drop
mininet@mininet-vm:~/mininet/custom$ sudo mn --custom mytopo.py --topo mytopo --controller=remote,ip=10.12.25.147,port=6653
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 
*** Adding switches:
s1 
*** Adding links:
(h1, s1) (h2, s1) (h3, s1) 
*** Configuring hosts
h1 h2 h3 
*** Starting controller
c0 
*** Starting 1 switches
s1 ...
*** Starting CLI:
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 h3 
h2 -> h1 h3 
h3 -> h1 h2 
*** Results: 0% dropped (6/6 received)
mininet> pingall
*** Ping: testing ping reachability
h1 -> X h3 
h2 -> X X 
h3 -> h1 X 
*** Results: 66% dropped (2/6 received)
```



l H1启动HTTP-Server功能，WEB端口为8000，H3作为HTTP-Client，获取H1的html网页配置文件。

```shell
mininet> h1 python -m SimpleHTTPServer 8000&#h1开始简单http服务器
mininet> h3 wget -O - http://10.0.0.1:8000#h3获取h1的网页
--2021-11-03 05:43:07--  http://10.0.0.1:8000/
Connecting to 10.0.0.1:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 302 [text/html]
Saving to: 鈥楽TDOUT鈥


 0% [                                       ] 0           --.-K/s              <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 3.2 Final//EN"><html>
<title>Directory listing for /</title>
<body>
<h2>Directory listing for /</h2>
<hr>
<ul>
<li><a href="mytopo.py">mytopo.py</a>
<li><a href="README">README</a>
<li><a href="topo-2sw-2host.py">topo-2sw-2host.py</a>
</ul>
<hr>
</body>
</html>
100%[======================================>] 302         --.-K/s   in 0s      

2021-11-03 05:43:07 (56.5 MB/s) - written to stdout [302/302]

```

# C卷配置

l 使用Mininet和OpenVswitch构建拓扑，连接ODL的6653端口，openFlow版本为1.3，如下拓扑结构：

![C卷拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\C卷拓扑.png)

l 在浏览器上可以访问ODL管理页面查看网元拓扑结构

```shell
mininet@mininet-vm:~/mininet/custom$ sudo mn --custom mytopo.py --topo mytopo --controller=remote,ip=10.12.25.147,port=6653
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 
*** Adding switches:
s1 s2 s3 
*** Adding links:
(h1, s1) (h2, s2) (h3, s3) (s1, s2) (s2, s3) 
*** Configuring hosts
h1 h2 h3 
*** Starting controller
c0 
*** Starting 3 switches
s1 s2 s3 ...
*** Starting CLI:
mininet> pingall
*** Ping: testing ping reachability
h1 -> X h3 
h2 -> h1 h3 
h3 -> h1 h2 
*** Results: 16% dropped (5/6 received)
```

![C卷查看拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\C卷查看拓扑.png)

l 通过OVS给S2下发流表，使得H2与H1、H3无法互通。

```shell
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s2 in_port=3,actions=drop
mininet> pingall
*** Ping: testing ping reachability
h1 -> X h3 
h2 -> X X 
h3 -> h1 X 
*** Results: 66% dropped (2/6 received)
```



H1启动HTTP-Server功能，WEB端口为8088，H3作为HTTP-Client，获取H1的html网页配置文件。

```shell
mininet> h3 wget -O - http://10.0.0.1:8088
--2021-11-03 05:43:05--  http://10.0.0.1:8088/
Connecting to 10.0.0.1:8088... connected.
HTTP request sent, awaiting response... 200 OK
Length: 302 [text/html]
Saving to: 鈥楽TDOUT鈥


 0% [                                       ] 0           --.-K/s              <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 3.2 Final//EN"><html>
<title>Directory listing for /</title>
<body>
<h2>Directory listing for /</h2>
<hr>
<ul>
<li><a href="mytopo.py">mytopo.py</a>
<li><a href="README">README</a>
<li><a href="topo-2sw-2host.py">topo-2sw-2host.py</a>
</ul>
<hr>
</body>
</html>
100%[======================================>] 302         --.-K/s   in 0s      

2021-11-03 05:43:05 (42.6 MB/s) - written to stdout [302/302]

```

# B卷配置

l 使用Mininet和OpenVswitch构建拓扑，连接ODL的6653端口如下拓扑结构：

![B卷配置](C:\Users\yfc1997\Desktop\刷就完事了\odl\B卷配置.png)

l 在浏览器上可以访问ODL管理页面查看网元拓扑结构。

```shell
mininet@mininet-vm:~/mininet/custom$ vim mytopo.py
        self.addLink( h1, s2 
"""Custom topology example

Two directly connected switches plus a host for each switch:

   host --- switch --- switch --- host

Adding the 'topos' dict with a key/value pair to generate our newly defined
topology enables one to pass in '--topo=mytopo' from the command line.
"""

from mininet.topo import Topo

class MyTopo( Topo ):
    "Simple topology example."

    def __init__( self ):
        "Create custom topo."

        # Initialize topology
        Topo.__init__( self )

        s2 = self.addSwitch( 's2' )
        s1 = self.addSwitch( 's1' )
        s3 = self.addSwitch( 's3' )

        h1 = self.addHost( 'h1' )
        h2 = self.addHost( 'h2' )
        h3 = self.addHost( 'h3' )
        h4 = self.addHost( 'h4' )
        # Add hosts and switches
        leftHost = self.addHost( 'h1' )
        #rightHost = self.addHost( 'h2' )
        #leftSwitch = self.addSwitch( 's3' )
        #rightSwitch = self.addSwitch( 's4' )

        # Add links
        self.addLink( s2, s1 )
        self.addLink( s1, s3 )
        self.addLink( h1, s2 )
        self.addLink( h2, s2 )
        self.addLink( h3, s3 )
        self.addLink( h4, s3 )


topos = { 'mytopo': ( lambda: MyTopo() ) }
mininet@mininet-vm:~/mininet/custom$ sudo mn --custom mytopo.py --topo mytopo --controller=remote,ip=10.12.25.147,port=6653
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 h4 
*** Adding switches:
s1 s2 s3 
*** Adding links:
(h1, s2) (h2, s2) (h3, s3) (h4, s3) (s1, s3) (s2, s1) 
*** Configuring hosts
h1 h2 h3 h4 
*** Starting controller
c0 
*** Starting 3 switches
s1 s2 s3 ...
*** Starting CLI:
mininet> pingall
*** Ping: testing ping reachability
h1 -> X h3 h4 
h2 -> h1 h3 h4 
h3 -> h1 h2 h4 
h4 -> h1 h2 h3 
*** Results: 8% dropped (11/12 received)
```



l 通过OVS在S1下发一条流表实现H1与H2可以互通，H3与H4可以互通，但H1、H2与H3、H4间不可以连通；

```shell
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s1 in_port=1,out_port=2,actions=drop
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 X X 
h2 -> h1 X X 
h3 -> X X h4 
h4 -> X X h3 
*** Results: 66% dropped (4/12 received)
```



l 用iperf工具测试H3和H4的带宽。

```shell
mininet> iperf h3 h4
*** Iperf: testing TCP bandwidth between h3 and h4 
.*** Results: ['36.8 Gbits/sec', '36.8 Gbits/sec']
```

# A卷配置

l 使用Mininet和OpenVswitch构建拓扑，连接ODL的6653端口如下拓扑结构：

![A卷拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\A卷拓扑.png)

l 在浏览器上可以访问ODL管理页面查看网元拓扑结构。

```shell
mininet@mininet-vm:~/mininet/custom$ vim mytopo.py
"""Custom topology example

Two directly connected switches plus a host for each switch:

   host --- switch --- switch --- host

Adding the 'topos' dict with a key/value pair to generate our newly defined
topology enables one to pass in '--topo=mytopo' from the command line.
"""

from mininet.topo import Topo

class MyTopo( Topo ):
    "Simple topology example."

    def __init__( self ):
        "Create custom topo."

        # Initialize topology
        Topo.__init__( self )

        s1 = self.addSwitch( 's1' )

        h1 = self.addHost( 'h1' )
        h2 = self.addHost( 'h2' )
        h3 = self.addHost( 'h3' )
        # Add hosts and switches
        #leftHost = self.addHost( 'h1' )
        #rightHost = self.addHost( 'h2' )
        #leftSwitch = self.addSwitch( 's3' )
        #rightSwitch = self.addSwitch( 's4' )

        # Add links 
        self.addLink( h1, s1 )
        self.addLink( h2, s1 )
        self.addLink( h3, s1 )
        

topos = { 'mytopo': ( lambda: MyTopo() ) }
~
~   
~   
~
~   
~   
~   
~   
~
~
"mytopo.py" [New] 39L, 963C written                           
mininet@mininet-vm:~/mininet/custom$ sudo mn --custom mytopo.py --topo mytopo --controller=remote,ip=10.12.25.147,port=6653
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 
*** Adding switches:
s1 
*** Adding links:
(h1, s1) (h2, s1) (h3, s1) 
*** Configuring hosts
h1 h2 h3 
*** Starting controller
c0 
*** Starting 1 switches
s1 ...
*** Starting CLI:
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 h3 
h2 -> h1 h3 
h3 -> h1 h2 
*** Results: 0% dropped (6/6 received)
```

![A卷查看拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\A卷查看拓扑.png)

l 通过OVS手工下发流表，H1可以ping通H3，H1、H3无法ping通H2。

```shell
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s1 in_port=2,actions=drop
mininet> pingall
*** Ping: testing ping reachability
h1 -> X h3 
h2 -> X X 
h3 -> h1 X 
*** Results: 66% dropped (2/6 received)
```



l H1启动HTTP-Server功能，WEB端口为8080，H3作为HTTP-Client，获取H1的html网页配置文件。

```shell
mininet> h1 python -m SimpleHTTPServer 8080&
mininet> h3 wget -O - http://10.0.0.1:8080
--2021-11-03 06:04:43--  http://10.0.0.1:8080/
Connecting to 10.0.0.1:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 350 [text/html]
Saving to: 鈥楽TDOUT鈥


 0% [                                       ] 0           --.-K/s              <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 3.2 Final//EN"><html>
<title>Directory listing for /</title>
<body>
<h2>Directory listing for /</h2>
<hr>
<ul>
<li><a href=".mytopo.py.swp">.mytopo.py.swp</a>
<li><a href="mytopo.py">mytopo.py</a>
<li><a href="README">README</a>
<li><a href="topo-2sw-2host.py">topo-2sw-2host.py</a>
</ul>
<hr>
</body>
</html>
100%[======================================>] 350         --.-K/s   in 0s      

2021-11-03 06:04:43 (45.3 MB/s) - written to stdout [350/350]

```

# 18J卷配置

使用Mininet和OpenVswitch构建拓扑，连接ODL的6633端口采用Openflow1.3协议，构造形成如下拓扑：

![18J卷](C:\Users\yfc1997\Desktop\刷就完事了\odl\18J卷.png)

在浏览器上可以访问ODL管理页面并查看网元拓扑结构。

```shell
mininet@mininet-vm:~/mininet/custom$ vim mytopo.py
"""Custom topology example

Two directly connected switches plus a host for each switch:

   host --- switch --- switch --- host

Adding the 'topos' dict with a key/value pair to generate our newly defined
topology enables one to pass in '--topo=mytopo' from the command line.
"""

from mininet.topo import Topo

class MyTopo( Topo ):
    "Simple topology example."

    def __init__( self ):
        "Create custom topo."

        # Initialize topology
        Topo.__init__( self )

        s1 = self.addSwitch( 's1' )
        s2 = self.addSwitch( 's2' )

        h1 = self.addHost( 'h1' )
        h2 = self.addHost( 'h2' )
        # Add hosts and switches
        #leftHost = self.addHost( 'h1' )
        #rightHost = self.addHost( 'h2' )
        #leftSwitch = self.addSwitch( 's3' )
        #rightSwitch = self.addSwitch( 's4' )

        # Add links
        self.addLink( s1, s2 )
        self.addLink( h1, s1 )
        self.addLink( h2, s2 )


topos = { 'mytopo': ( lambda: MyTopo() ) }
mininet@mininet-vm:~/mininet/custom$ sudo mn --custom mytopo.py --topo mytopo --controller=remote,ip=10.12.25.147,port=6653
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 
*** Adding switches:
s1 s2 
*** Adding links:
(h1, s1) (h2, s2) (s1, s2) 
*** Configuring hosts
h1 h2 
*** Starting controller
c0 
*** Starting 2 switches
s1 s2 ...
*** Starting CLI:
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 
h2 -> h1 
*** Results: 0% dropped (2/2 received)
```

![18J查看拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\18J查看拓扑.png)

通过OVS给S2下发流表，使得H1与H2无法互通。

```shell
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s2 in_port=2,actions=drop
mininet> pingall
*** Ping: testing ping reachability
h1 -> X 
h2 -> X 
*** Results: 100% dropped (0/2 received)
```

# 18I卷配置

使用Mininet和OpenVswitch构建拓扑，连接ODL的6633端口，采用Openflow1.0协议，构造形成如下拓扑：

![18I卷](C:\Users\yfc1997\Desktop\刷就完事了\odl\18I卷.png)

浏览器上可以访问ODL管理页面并查看网元拓扑结构。

```shell
mininet@mininet-vm:~/mininet/custom$ vim mytopo.py
        self.addLink( leftSwitch, rightSwitch )
"""Custom topology example

Two directly connected switches plus a host for each switch:

   host --- switch --- switch --- host

Adding the 'topos' dict with a key/value pair to generate our newly defined
topology enables one to pass in '--topo=mytopo' from the command line.
"""

from mininet.topo import Topo

class MyTopo( Topo ):
    "Simple topology example."

    def __init__( self ):
        "Create custom topo."

        # Initialize topology
        Topo.__init__( self )

        s1 = self.addSwitch( 's1' )

        h1 = self.addHost( 'h1' )
        h2 = self.addHost( 'h2' )
        # Add hosts and switches
        #leftHost = self.addHost( 'h1' )
        #rightHost = self.addHost( 'h2' )
        #leftSwitch = self.addSwitch( 's3' )
        #rightSwitch = self.addSwitch( 's4' )

        # Add links
        self.addLink( h1, s1 )
        self.addLink( h2, s1 )


topos = { 'mytopo': ( lambda: MyTopo() ) }

"mytopo.py" [New] 37L, 904C written                           
mininet@mininet-vm:~/mininet/custom$ sudo mn --custom mytopo.py --topo mytopo --controller=remote,ip=10.12.25.147,port=6653
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 
*** Adding switches:
s1 
*** Adding links:
(h1, s1) (h2, s1) 
*** Configuring hosts
h1 h2 
*** Starting controller
c0 
*** Starting 1 switches
s1 ...
*** Starting CLI:
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 
h2 -> h1 
*** Results: 0% dropped (2/2 received)
```

![18I卷查看拓扑](C:\Users\yfc1997\Desktop\刷就完事了\odl\18I卷查看拓扑.png)

通过OVS手工下发流表，H1和H2互通。H1启动HTTPSERVER功能，WEB端口为80，H2作为HTTPCLIENT，获取H1的html网页文件。

```shell
#默认互通无需下发
mininet> h2 wget -O - http://10.0.0.1:80
--2021-11-03 05:44:53--  http://10.0.0.1/
Connecting to 10.0.0.1:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 302 [text/html]
Saving to: 鈥楽TDOUT鈥


 0% [                                       ] 0           --.-K/s              <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 3.2 Final//EN"><html>
<title>Directory listing for /</title>
<body>
<h2>Directory listing for /</h2>
<hr>
<ul>
<li><a href="mytopo.py">mytopo.py</a>
<li><a href="README">README</a>
<li><a href="topo-2sw-2host.py">topo-2sw-2host.py</a>
</ul>
<hr>
</body>
</html>
100%[======================================>] 302         --.-K/s   in 0s      

2021-11-03 05:44:53 (50.8 MB/s) - written to stdout [302/302]
```



下发流表使得H1和H2不通。

```shell
mininet@mininet-vm:~$ sudo ovs-ofctl add-flow s1 in_port=2,actions
mininet> pingall
*** Ping: testing ping reachability
h1 -> X 
h2 -> X 
*** Results: 100% dropped (0/2 received)

```

# 20A卷配置

启动 OpenDayLight 的 karaf 程序，并安装如下组件： 

feature:install odl-restconf 

feature:install odl-l2switch-switch-ui 

feature:install odl-mdsal-apidocs 

feature:install odl-dluxapps-applications

![image-20220117153210915](C:\Users\yfc1997\AppData\Roaming\Typora\typora-user-images\image-20220117153210915.png)

![image-20220117153224018](C:\Users\yfc1997\AppData\Roaming\Typora\typora-user-images\image-20220117153224018.png)

补充/home/mininet/mininet/custom 目录下的 topo.py 脚本，并使 

用 Mininet 通过 topo.py 脚本创建如下图 6-1 所示的拓扑，采用 ovsk 交换格 

式，连接 ODL 的远程地址为 172.16.0.10:6653,协议类型是 Openflow1.30， 

且主机 Host 的 MAC 地址从 00:00:00:00:00:01 开始顺序分配，并设置 

Host1-Host6 的 IP 地址从 10.1.1.1 10.1.1.6 顺序分配。

![image-20220117154234198](C:\Users\yfc1997\AppData\Roaming\Typora\typora-user-images\image-20220117154234198.png)

H5 启动 HTTP-Server 功能，WEB 端口为 8080，H4 作为 HTTP-Client， 

获取 H5 的 html 网页文件。

```shell
mininet> H5 python -m SimpleHTTPServer 8080&
mininet> H4 wget -O - http://10.0.0.5:8080
--2021-11-03 05:06:27--  http://10.0.0.5:8080/
Connecting to 10.0.0.5:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 298 [text/html]
Saving to: 鈥楽TDOUT鈥


 0% [                                       ] 0           --.-K/s              <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 3.2 Final//EN"><html>
<title>Directory listing for /</title>
<body>
<h2>Directory listing for /</h2>
<hr>
<ul>
<li><a href="README">README</a>
<li><a href="topo-2sw-2host.py">topo-2sw-2host.py</a>
<li><a href="topo.py">topo.py</a>
</ul>
<hr>
</body>
</html>
100%[======================================>] 298         --.-K/s   in 0s      

2021-11-03 05:06:27 (33.8 MB/s) - written to stdout [298/298]

mininet> 
```

通过 OVS 手工命令在 openflow:1 虚拟交换机下发两条流表，优先级 

均为 priority=100 实现如下需求：H2、H3、H4、H5 与 H6 之间可以互通， 

H1 不能与 H2、H3、H4、H5、H6 通信。

```shell
root@mininet-vm:~# ovs-ofctl add-flow s1 in_port=1,priority=100,actions=output:2,output:3
root@mininet-vm:~# ovs-ofctl add-flow s1 in_port=2,priority=100,actions=output:1,output:3

```

