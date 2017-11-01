# L2TP服务器配置

## 1.安装依赖组件
```shell
yum install -y openswan ppp xl2tpd wget
```

## 2.修改ipsec.conf配置文件
```shell
vi /etc/ipsec.conf
```

将$serverip替换为服务器IP，配置如下：

```shell
 # /etc/ipsec.conf - Libreswan IPsec configuration file
 # This file:  /etc/ipsec.conf
 #
 # Enable when using this configuration file with openswan instead of libreswan
 #version 2
 #
 # Manual:     ipsec.conf.5

 # basic configuration
 config setup
     # NAT-TRAVERSAL support, see README.NAT-Traversal
     nat_traversal=yes
     # exclude networks used on server side by adding %v4:!a.b.c.0/24
     virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16,%v4:172.16.0.0/12
     # OE is now off by default. Uncomment and change to on, to enable.
     oe=off
     # which IPsec stack to use. auto will try netkey, then klips then mast
     protostack=netkey
     force_keepalive=yes
     keep_alive=1800

 conn L2TP-PSK-NAT
     rightsubnet=vhost:%priv
     also=L2TP-PSK-noNAT

 conn L2TP-PSK-noNAT
     authby=secret
     pfs=no
     auto=add
     keyingtries=3
     rekey=no
     ikelifetime=8h
     keylife=1h
     type=transport
     left=$serverip           //$serverip替换为服务器IP
     leftid=$serverip         //$serverip替换为服务器IP
     leftprotoport=17/1701
     right=%any
     rightprotoport=17/%any
     dpddelay=40
     dpdtimeout=130
     dpdaction=clear
```

## 3.修改ipsec.secrets配置文件，即共享秘钥
```shell
vi /etc/ipsec.secrets
```

将$serverip替换为服务器ip, $mypsk替换为psk密钥（客户端连接时需要）：

```shell
#include /etc/ipsec.d/*.secrets
$serverip %any: PSK "$mypsk"      //$serverip替换为服务器ip, $mypsk替换为psk密钥
```

## 4.修改xl2tpd.conf配置文件
```shell
vi /etc/xl2tpd/xl2tpd.conf
```

将$serverip替换为服务器IP，网关和地址池范围根据实际情况修改：

```shell
;
; This is a minimal sample xl2tpd configuration file for use
; with L2TP over IPsec.
;
; The idea is to provide an L2TP daemon to which remote Windows L2TP/IPsec
; clients connect. In this example, the internal (protected) network
; is 192.168.1.0/24.  A special IP range within this network is reserved
; for the remote clients: 192.168.1.128/25
; (i.e. 192.168.1.128 ... 192.168.1.254)
;
; The listen-addr parameter can be used if you want to bind the L2TP daemon
; to a specific IP address instead of to all interfaces. For instance,
; you could bind it to the interface of the internal LAN (e.g. 192.168.1.98
; in the example below). Yet another IP address (local ip, e.g. 192.168.1.99)
; will be used by xl2tpd as its address on pppX interfaces.
[global]
; ipsec saref = yes
listen-addr = $serverip                  //$serverip替换为服务器IP
auth file = /etc/ppp/chap-secrets
port = 1701
[lns default]
ip range = 192.168.2.100-192.168.2.200   //地址池范围
local ip = 192.168.2.1                   //VPN服务网关
refuse chap = yes
refuse pap = yes
require authentication = yes
name = L2TPVPN
ppp debug = yes
pppoptfile = /etc/ppp/options.xl2tpd
length bit = yes
```

## 5.修改options.xl2tpd配置文件
```shell
vi /etc/ppp/options.xl2tpd
```

使用如下配置，无需修改:

```shell
#require-pap
#require-chap
#require-mschap
ipcp-accept-local
ipcp-accept-remote
require-mschap-v2
ms-dns 8.8.8.8       //VPN使用的DNS服务器
ms-dns 8.8.4.4       //备用DNS服务器
asyncmap 0
auth
crtscts
lock
hide-password
modem
debug
name l2tpd
proxyarp
lcp-echo-interval 30
lcp-echo-failure 4
mtu 1400
noccp
connect-delay 5000
# To allow authentication against a Windows domain EXAMPLE, and require the
# user to be in a group "VPN Users". Requires the samba-winbind package
# require-mschap-v2
# plugin winbind.so
# ntlm_auth-helper ‘/usr/bin/ntlm_auth --helper-protocol=ntlm-server-1 --require-membership-of="EXAMPLE\VPN Users"‘
# You need to join the domain on the server, for example using samba:
# http://rootmanager.com/ubuntu-ipsec-l2tp-windows-domain-auth/setting-up-openswan-xl2tpd-with-native-windows-clients-lucid.html
```

## 6.修改chap-secrets配置文件，即用户列表及密码
```shell
vi /etc/ppp/chap-secrets
```

将$username、$password 分别替换为用户名和密码（客户端连接时使用）:

```shell
# Secrets for authentication using CHAP
# client     server     secret         IP addresses
$username    l2tpd     $password       *
```

## 7.修改IP转发系统配置
```shell
vi /etc/sysctl.conf
```

增加如下配置：
```shell
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.$eth.rp_filter = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
```

执行如下命令：
```shell
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv4.conf.all.rp_filter=0
sysctl -w net.ipv4.conf.default.rp_filter=0
sysctl -w net.ipv4.conf.$eth.rp_filter=0
sysctl -w net.ipv4.conf.all.send_redirects=0
sysctl -w net.ipv4.conf.default.send_redirects=0
sysctl -w net.ipv4.conf.all.accept_redirects=0
sysctl -w net.ipv4.conf.default.accept_redirects=0
```

## 8.修改防火墙端口配置
```shell
vi /usr/lib/firewalld/services/l2tpd.xml
```

增加如下配置：

```shell
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>l2tpd</short>
  <description>L2TP IPSec</description>
  <port protocol="udp" port="500"/>
  <port protocol="udp" port="4500"/>
  <port protocol="udp" port="1701"/>
</service>
```

重启防火墙（未开启防火墙可不执行）：
firewall-cmd --permanent --add-service=l2tpd
firewall-cmd --permanent --add-service=ipsec
firewall-cmd --permanent --add-masquerade
firewall-cmd --reload

## 9.执行允许开机启动命令
```shell
systemctl enable ipsec xl2tpd
systemctl restart ipsec xl2tpd
```

## 10.配置转发规则脚本
创建文件rc.local
```shell
vi /etc/rc.d/rc.local
```

配置如下，将$serverip替换为服务器IP：
```shell
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In contrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.

touch /var/lock/subsys/local
iptables -t nat -A POSTROUTING -s 192.168.2.0/24 -j SNAT --to-source $serverip
```

给脚本增加可执行权限
```shell
chmod +x /etc/rc.d/rc.local
```

## 11.重启服务器并验证
```shell
reboot
```

```shell
ipsec verify
```

正常情况下会打出日志：
```shell
Verifying installed system and configuration files

Version check and ipsec on-path                         [OK]
Libreswan 3.15 (netkey) on 3.10.0-327.22.2.el7.x86_64
Checking for IPsec support in kernel                    [OK]
 NETKEY: Testing XFRM related proc values
         ICMP default/send_redirects                    [OK]
         ICMP default/accept_redirects                  [OK]
         XFRM larval drop                               [OK]
Pluto ipsec.conf syntax                                 [OK]
Hardware random device                                  [N/A]
Two or more interfaces found, checking IP forwarding    [OK]
Checking rp_filter                                      [OK]
Checking that pluto is running                          [OK]
 Pluto listening for IKE on udp 500                     [OK]
 Pluto listening for IKE/NAT-T on udp 4500              [OK]
 Pluto ipsec.secret syntax                              [OK]
Checking 'ip' command                                   [OK]
Checking 'iptables' command                             [OK]
Checking 'prelink' command does not interfere with FIPSChecking for obsolete ipsec.conf options                 [OBSOLETE KEYWORD]
 Warning: ignored obsolete keyword 'force_keepalive'
Opportunistic Encryption                                [DISABLED]
```
