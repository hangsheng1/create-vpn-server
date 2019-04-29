# VPN搭建笔记

## 环境准备

*  服务器： 阿里云轻量级应用服务器（香港）
*  服务器操作系统： CentOS 7.3
*  服务程序： OpenVpn server 2.4.7
*  依赖程序： easy-rsa(3.x), vim , openSSL

## 配置VPN服务

### 1. 	软件下载

​			 CentOS 系统下使用 yum 管理工具安装 OpenVPN

```
yum install openvpn -y
```

​        	easy-rsa 下载

```
yum install easy-rsa
```

### 2.	密钥生成

​			

```
cp -p /usr/share/doc/easy-rsa-3.0.3/vars.example /etc/openvpn/easy-rsa/vars
cp -r /usr/share/easy-rsa/3.0.3/* /etc/openvpn/easy-rsa/
cp /usr/share/doc/openvpn-2.4.6/sample/sample-config-files/server.conf /etc/openvpn/
```



```
cd /etc/openvpn/easy-rsa/ 

./easyrsa init-pki    目录初始化，建立一个空的pki结构，生成一系列的文件和目录

./easyrsa build-ca  创建根证书，创建ca密码 和 cn那么需要记住，commonname单独定义个名字server

xxx    xxx    server   创建服务器端证书，输入commonname单独定义不要与ca一样server-uv

./easyrsa gen-req server nopass   签约服务端证书，输入ca的密码创建服务端秘钥，输入yes同意创建

./easyrsa sign server server    创建Diffie-Hellman，确保key穿越不安全网络的命令

./easyrsa gen-dh

openvpn --genkey --secret /etc/openvpn/easy-rsa/ta.key 

生成ta密钥文件（可以不需要创建），需要注释server.conf中tls-auth ta.key 0
```

创建客户端证书及key 
创建过程同服务端

```

mkdir /home/client

cd /home/client

cp -r /usr/share/easy-rsa/3.0.3/* ./

./easyrsa init-pki

./easyrsa build-ca   客户端不需要创建根证书



./easyrsa gen-req myname 
(myname) 用自己的名字，需要创建一个密码  和 cn name，自己用的 需要记住
xxxx    xxxx    client

```

现在客户端的证书要跟服务端的交互，也就是签约，这样这个用户才能使用此vpn
切换到server证书目录下

```

cd /etc/openvpn/easy-rsa/


./easyrsa import-req /root/client/pki/reqs/myname.req myname 
将得到的orangleliu.req导入然后签约证书


./easyrsa sign client myname      用户签约，根据提示输入服务端的ca密码


```

```
/etc/openvpn/easy-rsa

cp pki/ca.crt /etc/openvpn/

cp pki/private/server.key /etc/openvpn/

cp pki/issued/server.crt /etc/openvpn/

cp pki/dh.pem /etc/openvpn/

cp /etc/openvpn/easy-rsa/ta.key /etc/openvpn/    ta.key 用于防御dos攻击,tun端口攻击


```

client 证书

```
/home/client

cp /etc/openvpn/easy-rsa/pki/ca.crt /home/client/

cp pki/issued/myname.crt  /hoem/client/

cp /home/client/pki/private/myname.key /home/client/

cp /etc/openvpn/easy-rsa/ta.key /home/client/        ta.key 用于防御dos攻击,tun端口攻击
```



修改/etc/openvpn/server.conf

```
vim /etc/openvpn/server.conf
```

完整配置如下：

```
#################################################
# Sample OpenVPN 2.0 config file for            #
# multi-client server.                          #
#                                               #
# This file is for the server side              #
# of a many-clients <-> one-server              #
# OpenVPN configuration.                        #
#                                               #
# OpenVPN also supports                         #
# single-machine <-> single-machine             #
# configurations (See the Examples page         #
# on the web site for more info).               #
#                                               #
# This config should work on Windows            #
# or Linux/BSD systems.  Remember on            #
# Windows to quote pathnames and use            #
# double backslashes, e.g.:                     #
# "C:\\Program Files\\OpenVPN\\config\\foo.key" #
#                                               #
# Comments are preceded with '#' or ';'         #
#################################################

# Which local IP address should OpenVPN
# listen on? (optional)
;local a.b.c.d

# Which TCP/UDP port should OpenVPN listen on?
# If you want to run multiple OpenVPN instances
# on the same machine, use a different port
# number for each one.  You will need to
# open up this port on your firewall.
port 1194

# TCP or UDP server?
;proto tcp
proto udp

# "dev tun" will create a routed IP tunnel,
# "dev tap" will create an ethernet tunnel.
# Use "dev tap0" if you are ethernet bridging
# and have precreated a tap0 virtual interface
# and bridged it with your ethernet interface.
# If you want to control access policies
# over the VPN, you must create firewall
# rules for the the TUN/TAP interface.
# On non-Windows systems, you can give
# an explicit unit number, such as tun0.
# On Windows, use "dev-node" for this.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
;dev tap
dev tun

# Windows needs the TAP-Win32 adapter name
# from the Network Connections panel if you
# have more than one.  On XP SP2 or higher,
# you may need to selectively disable the
# Windows firewall for the TAP adapter.
# Non-Windows systems usually don't need this.
;dev-node MyTap

# SSL/TLS root certificate (ca), certificate
# (cert), and private key (key).  Each client
# and the server must have their own cert and
# key file.  The server and all clients will
# use the same ca file.
#
# See the "easy-rsa" directory for a series
# of scripts for generating RSA certificates
# and private keys.  Remember to use
# a unique Common Name for the server
# and each of the client certificates.
#
# Any X509 key management system can be used.
# OpenVPN can also use a PKCS #12 formatted key file
# (see "pkcs12" directive in man page).
ca ca.crt
cert server.crt
key server.key  # This file should be kept secret

# Diffie hellman parameters.
# Generate your own with:
#   openssl dhparam -out dh2048.pem 2048
dh dh.pem

# Network topology
# Should be subnet (addressing via IP)
# unless Windows clients v2.0.9 and lower have to
# be supported (then net30, i.e. a /30 per client)
# Defaults to net30 (not recommended)
;topology subnet

# Configure server mode and supply a VPN subnet
# for OpenVPN to draw client addresses from.
# The server will take 10.8.0.1 for itself,
# the rest will be made available to clients.
# Each client will be able to reach the server
# on 10.8.0.1. Comment this line out if you are
# ethernet bridging. See the man page for more info.
server 172.16.0.0 255.255.255.0

# Maintain a record of client <-> virtual IP address
# associations in this file.  If OpenVPN goes down or
# is restarted, reconnecting clients can be assigned
# the same virtual IP address from the pool that was
# previously assigned.
ifconfig-pool-persist ipp.txt

# Configure server mode for ethernet bridging.
# You must first use your OS's bridging capability
# to bridge the TAP interface with the ethernet
# NIC interface.  Then you must manually set the
# IP/netmask on the bridge interface, here we
# assume 10.8.0.4/255.255.255.0.  Finally we
# must set aside an IP range in this subnet
# (start=10.8.0.50 end=10.8.0.100) to allocate
# to connecting clients.  Leave this line commented
# out unless you are ethernet bridging.
;server-bridge 10.8.0.4 255.255.255.0 10.8.0.50 10.8.0.100

# Configure server mode for ethernet bridging
# using a DHCP-proxy, where clients talk
# to the OpenVPN server-side DHCP server
# to receive their IP address allocation
# and DNS server addresses.  You must first use
# your OS's bridging capability to bridge the TAP
# interface with the ethernet NIC interface.
# Note: this mode only works on clients (such as
# Windows), where the client-side TAP adapter is
# bound to a DHCP client.
;server-bridge

# Push routes to the client to allow it
# to reach other private subnets behind
# the server.  Remember that these
# private subnets will also need
# to know to route the OpenVPN client
# address pool (10.8.0.0/255.255.255.0)
# back to the OpenVPN server.
;push "route 192.168.10.0 255.255.255.0"
;push "route 192.168.20.0 255.255.255.0"

# To assign specific IP addresses to specific
# clients or if a connecting client has a private
# subnet behind it that should also have VPN access,
# use the subdirectory "ccd" for client-specific
# configuration files (see man page for more info).

# EXAMPLE: Suppose the client
# having the certificate common name "Thelonious"
# also has a small subnet behind his connecting
# machine, such as 192.168.40.128/255.255.255.248.
# First, uncomment out these lines:
;client-config-dir ccd
;route 192.168.40.128 255.255.255.248
# Then create a file ccd/Thelonious with this line:
#   iroute 192.168.40.128 255.255.255.248
# This will allow Thelonious' private subnet to
# access the VPN.  This example will only work
# if you are routing, not bridging, i.e. you are
# using "dev tun" and "server" directives.

# EXAMPLE: Suppose you want to give
# Thelonious a fixed VPN IP address of 10.9.0.1.
# First uncomment out these lines:
;client-config-dir ccd
;route 10.9.0.0 255.255.255.252
# Then add this line to ccd/Thelonious:
#   ifconfig-push 10.9.0.1 10.9.0.2

# Suppose that you want to enable different
# firewall access policies for different groups
# of clients.  There are two methods:
# (1) Run multiple OpenVPN daemons, one for each
#     group, and firewall the TUN/TAP interface
#     for each group/daemon appropriately.
# (2) (Advanced) Create a script to dynamically
#     modify the firewall in response to access
#     from different clients.  See man
#     page for more info on learn-address script.
;learn-address ./script

# If enabled, this directive will configure
# all clients to redirect their default
# network gateway through the VPN, causing
# all IP traffic such as web browsing and
# and DNS lookups to go through the VPN
# (The OpenVPN server machine may need to NAT
# or bridge the TUN/TAP interface to the internet
# in order for this to work properly).
push "redirect-gateway def1 bypass-dhcp"

# Certain Windows-specific network settings
# can be pushed to clients, such as DNS
# or WINS server addresses.  CAVEAT:
# http://openvpn.net/faq.html#dhcpcaveats
# The addresses below refer to the public
# DNS servers provided by opendns.com.
# 配置DNS服务器
push "dhcp-option DNS 208.67.222.222"
push "dhcp-option DNS 208.67.220.220"

# Uncomment this directive to allow different
# clients to be able to "see" each other.
# By default, clients will only see the server.
# To force clients to only see the server, you
# will also need to appropriately firewall the
# server's TUN/TAP interface.
client-to-client

# Uncomment this directive if multiple clients
# might connect with the same certificate/key
# files or common names.  This is recommended
# only for testing purposes.  For production use,
# each client should have its own certificate/key
# pair.
#
# IF YOU HAVE NOT GENERATED INDIVIDUAL
# CERTIFICATE/KEY PAIRS FOR EACH CLIENT,
# EACH HAVING ITS OWN UNIQUE "COMMON NAME",
# UNCOMMENT THIS LINE OUT.
;duplicate-cn

# The keepalive directive causes ping-like
# messages to be sent back and forth over
# the link so that each side knows when
# the other side has gone down.
# Ping every 10 seconds, assume that remote
# peer is down if no ping received during
# a 120 second time period.
keepalive 10 120

# For extra security beyond that provided
# by SSL/TLS, create an "HMAC firewall"
# to help block DoS attacks and UDP port flooding.
#
# Generate with:
#   openvpn --genkey --secret ta.key
#
# The server and each client must have
# a copy of this key.
# The second parameter should be '0'
# on the server and '1' on the clients.
tls-auth ta.key 0 # This file is secret

# Select a cryptographic cipher.
# This config item must be copied to
# the client config file as well.
# Note that 2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
cipher AES-256-CBC

# Enable compression on the VPN link and push the
# option to the client (2.4+ only, for earlier
# versions see below)
;compress lz4-v2
;push "compress lz4-v2"

# For compression compatible with older clients use comp-lzo
# If you enable it here, you must also
# enable it in the client config file.
#兼容低版本
comp-lzo

# The maximum number of concurrently connected
# clients we want to allow.
;max-clients 100

# It's a good idea to reduce the OpenVPN
# daemon's privileges after initialization.
#
# You can uncomment this out on
# non-Windows systems.
#防止权限而引发的问题
user nobody
group nobody

# The persist options will try to avoid
# accessing certain resources on restart
# that may no longer be accessible because
# of the privilege downgrade.
persist-key
persist-tun

# Output a short status file showing
# current connections, truncated
# and rewritten every minute.
status openvpn-status.log

# By default, log messages will go to the syslog (or
# on Windows, if running as a service, they will go to
# the "\Program Files\OpenVPN\log" directory).
# Use log or log-append to override this default.
# "log" will truncate the log file on OpenVPN startup,
# while "log-append" will append to it.  Use one
# or the other (but not both).
log         openvpn.log
;log-append  openvpn.log

# Set the appropriate level of log
# file verbosity.
#
# 0 is silent, except for fatal errors
# 4 is reasonable for general usage
# 5 and 6 can help to debug connection problems
# 9 is extremely verbose
verb 3

# Silence repeating messages.  At most 20
# sequential messages of the same message
# category will be output to the log.
;mute 20

# Notify the client that when the server restarts so it
# can automatically reconnect.
explicit-exit-notify 1
```

### 3.	启动openvpn 服务

​		

```
cd /usr/share/doc/openvpn-2.4.7/sample/sample-config-files/

cp firewall.sh /etc/openvpn/

chmod 777 /etc/openvpn/firewall.sh

chmod 777 ./openvpn-startup.sh

```



修改/etc/openvpn/firewall.sh

修改此行即可

```
PRIVATE=10.0.0.0/24  ->  PRIVATE=172.16.0.0/24
```



修改openvpn-startup.sh

```
#!/bin/sh

# A sample OpenVPN startup script
# for Linux.

# openvpn config file directory
dir=/etc/openvpn

# load the firewall
$dir/firewall.sh

# load TUN/TAP kernel module
modprobe tun

# enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward # 开启路由转发

# Invoke openvpn for each VPN tunnel
# in daemon mode.  Alternatively,
# you could remove "--daemon" from
# the command line and add "daemon"
# to the config file.
#
# Each tunnel should run on a separate
# UDP port.  Use the "port" option
# to control this.  Like all of
# OpenVPN's options, you can
# specify "--port 8000" on the command
# line or "port 8000" in the config
# file.

openvpn --cd $dir --daemon --config server.conf
#openvpn --cd $dir --daemon --config vpn2.conf
#openvpn --cd $dir --daemon --config vpn2.conf
```

运行openvpn-startup.sh

```
./openvpn-startup.sh
```

进入阿里云控制台，设置服务器安全组，添加防火墙规则  	协议(tun)  	端口号(1194)

这时候服务端全部配置完成了



### 4.	客户端配置(Win10) 

​		

客户端下载：

（Win10)

 https://swupdate.openvpn.org/community/releases/openvpn-install-2.4.7-I607-Win10.exe  

(Win7)

https://swupdate.openvpn.org/community/releases/openvpn-install-2.4.7-I607-Win7.exe

因为国内无法访问此网站，于是要用服务器（服务器部署在境外的才行，如阿里云香港服务器）去下载此安装包

```
wget https://swupdate.openvpn.org/community/releases/openvpn-install-2.4.7-I607-  Win10.exe  
```

然后将此安装包从服务器传到本地电脑里

客户端配置文件编写

openvpn 客户端配置文件后缀为*.ovpn 

配置如下：

```
##############################################
# Sample client-side OpenVPN 2.0 config file #
# for connecting to multi-client server.     #
#                                            #
# This configuration can be used by multiple #
# clients, however each client should have   #
# its own cert and key files.                #
#                                            #
# On Windows, you might want to rename this  #
# file so it has a .ovpn extension           #
##############################################

# Specify that we are a client and that we
# will be pulling certain config file directives
# from the server.
client

# Use the same setting as you are using on
# the server.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
;dev tap
dev tun

# Windows needs the TAP-Win32 adapter name
# from the Network Connections panel
# if you have more than one.  On XP SP2,
# you may need to disable the firewall
# for the TAP adapter.
;dev-node MyTap

# Are we connecting to a TCP or
# UDP server?  Use the same setting as
# on the server.
;proto tcp
proto udp

# The hostname/IP and port of the server.
# You can have multiple remote entries
# to load balance between the servers.
remote my-server-1 1194 #你的服务器地址
;remote my-server-2 1194

# Choose a random host from the remote
# list for load-balancing.  Otherwise
# try hosts in the order specified.
;remote-random

# Keep trying indefinitely to resolve the
# host name of the OpenVPN server.  Very useful
# on machines which are not permanently connected
# to the internet such as laptops.
resolv-retry infinite

# Most clients don't need to bind to
# a specific local port number.
nobind

# Downgrade privileges after initialization (non-Windows only)
;user nobody
;group nobody

# Try to preserve some state across restarts.
persist-key
persist-tun

# If you are connecting through an
# HTTP proxy to reach the actual OpenVPN
# server, put the proxy server/IP and
# port number here.  See the man page
# if your proxy server requires
# authentication.
;http-proxy-retry # retry on connection failures
;http-proxy [proxy server] [proxy port #]

# Wireless networks often produce a lot
# of duplicate packets.  Set this flag
# to silence duplicate packet warnings.
;mute-replay-warnings

# SSL/TLS parms.
# See the server config file for more
# description.  It's best to use
# a separate .crt/.key file pair
# for each client.  A single ca
# file can be used for all clients.
ca ca.crt
cert myname.crt
key myname.key

# Verify server certificate by checking that the
# certicate has the correct key usage set.
# This is an important precaution to protect against
# a potential attack discussed here:
#  http://openvpn.net/howto.html#mitm
#
# To use this feature, you will need to generate
# your server certificates with the keyUsage set to
#   digitalSignature, keyEncipherment
# and the extendedKeyUsage to
#   serverAuth
# EasyRSA can do this for you.
remote-cert-tls server

# If a tls-auth key is used on the server
# then every client must also have the key.
tls-auth ta.key 1

# Select a cryptographic cipher.
# If the cipher option is used on the server
# then you must also specify it here.
# Note that 2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
cipher AES-256-CBC

# Enable compression on the VPN link.
# Don't enable this unless it is also
# enabled in the server config file.
comp-lzo

# Set log file verbosity.
verb 3

# Silence repeating messages
;mute 20

```



从服务器将生成的客户端相关证书下载到本地电脑里

```
ca.crt

myname.crt

myname.key

ta.key #此文件用于防御dos攻击

```



安装客户端

管理员身份运行客户端

系统托盘出现小图标即启动成功

右击小图标

导入配置文件

将证书文件放在配置文件同一目录下

点击connect

若图标变绿，即表示连接成功，这时候就可以访问国外的网站了，如谷歌，facebook,  youtube 等等。

还有一点就是sublime text3 的插件可以在线安装了，，



