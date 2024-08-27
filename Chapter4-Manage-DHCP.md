```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 章节目标

- 描述用于 IPv4 的 DHCP 协议的操作，并配置 DHCP 服务器以便为 DHCP 客户端提供 IPv4 地址池，同时为特定客户端提供保留的地址。

- 讨论 DHCPv6 在 IPv6 中的作用，将它与其他 IPv6 网络自动配置技术进行比较，并配置 DHCPv6 服务器。

- 自动配置 DHCP 服务器，为 IPv4 和 IPv6 地址提供支持。

# 使用 DHCP 配置 IPv4 地址分配

## 描述 DHCP

动态主机配置协议 (DHCP) 为系统提供了一种方法，用于自动检索其网络配置参数，如 IP 地址、默认网关、DNS 服务器和域，或者 NTP 服务器。

通过部署 DHCP 服务器，可以集中控制这些参数。可以分配 IP 地址范围以分发到客户端，也可以为特定客户端分配保留的 IP 地址。

下面是DHCP的过程

```textile
DHCP客户端启动
  │
  ├> DHCPDISCOVER (广播) ────────┐
  │                                │
  │  (广播寻找DHCP服务器)        │
  │                                │
  ├> DHCPOFFER (来自服务器) ────┘
  │
  ├> DHCPREQUEST (单播至选定服务器)
  │
  └> DHCPACK (来自服务器) ────────> 客户端开始使用IP地址
     │
     └> DHCPREQUEST (租约到期前，客户端发送续约请求)
           │
           ├> DHCPACK (续约成功) ──> 继续使用IP地址
           │
           │  (或)
           └> DHCPOFFER (续约失败，可能需要新IP地址)
                │
                └> DHCPREQUEST (客户端请求新IP地址)
                      │
                      └> DHCPACK (新IP地址分配成功)
                            客户端更新配置并继续使用
```

## 部署 DHCPv4 服务器

### 安装DHCP软件

```bash
[root@servera ~]# yum install dhcp-server -y
```

### 配置 DHCP 服务器

`dhcpd` 服务使用 `/etc/dhcp/dhcpd.conf` 配置文件。dhcp-server 软件包提供 `/usr/share/doc/dhcp-server/dhcpd.conf.example` 配置文件示例

```bash
[root@servera ~]# cp /usr/share/doc/dhcp-server/dhcpd.conf.example /etc/dhcp/dhcpd.conf
cp: overwrite '/etc/dhcp/dhcpd.conf'? y
```

```textile
authoritative;
log-facility local7;
subnet 172.25.250.0 netmask 255.255.255.0 {
  range 172.25.250.100 172.25.250.200;
  option domain-name-servers 172.25.250.254;
  option domain-name "lab.example.com";
  option routers 172.25.250.254;
  option broadcast-address 172.25.250.255;
  default-lease-time 600;
  max-lease-time 7200;
}
```

### 根据 MAC 地址保留 IP 地址

在配置文件中，`host` 声明可以将 MAC 地址绑定到 IP 地址

```textile
host lixiaohui {
  hardware ethernet 08:00:07:26:c0:a5;
  fixed-address 172.25.250.105;
}
```

### 验证配置

```textile
[root@servera ~]# dhcpd -t
Internet Systems Consortium DHCP Server 4.3.6
Copyright 2004-2017 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/
ldap_gssapi_principal is not set,GSSAPI Authentication for LDAP will not be used
Not searching LDAP since ldap-server, ldap-port and ldap-base-dn were not specified in the config file
Config file: /etc/dhcp/dhcpd.conf
Database file: /var/lib/dhcpd/dhcpd.leases
PID file: /var/run/dhcpd.pid
Source compiled to use binary-leases
```

### 启动并启用服务

```bash
[root@servera ~]# systemctl enable --now dhcpd
[root@servera ~]# firewall-cmd --permanent --add-service=dhcp
[root@servera ~]# firewall-cmd --reload
```

### DHCP客户端测试

```bash
[root@serverb ~]# nmcli device status
DEVICE  TYPE      STATE         CONNECTION
eth0    ethernet  connected     Wired connection 1
eth1    ethernet  disconnected  --
eth2    ethernet  disconnected  --
lo      loopback  unmanaged     --


[root@serverb ~]# nmcli connection add con-name dhcptest ifname eth1 ipv4.method auto autoconnect yes type ethernet

[root@serverb ~]# ip a s eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:01:fa:0b brd ff:ff:ff:ff:ff:ff
    inet 172.25.250.100/24 brd 172.25.250.255 scope global dynamic noprefixroute eth1
       valid_lft 595sec preferred_lft 595sec
    inet6 fe80::51d6:1bf5:3111:5ae5/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

### 查询服务器分发的租约

```bash
[root@servera ~]# cat /var/lib/dhcpd/dhcpd.leases

authoring-byte-order little-endian;

server-duid "\000\001\000\001.Za\024RT\000\000\372\012";

lease 172.25.250.100 {
  starts 4 2024/08/22 20:55:33;
  ends 4 2024/08/22 21:05:33;
  cltt 4 2024/08/22 20:55:33;
  binding state active;
  next binding state free;
  rewind binding state free;
  hardware ethernet 52:54:00:01:fa:0b;
  uid "\001RT\000\001\372\013";
  client-hostname "serverb";
}
```

### 测试IP地址保留

在配置文件中写入以下内容，并重启服务

```text
host lixiaohui {
  hardware ethernet 52:54:00:01:fa:0b;
  fixed-address 172.25.250.110;
}
```

```bash
[root@servera ~]# systemctl restart dhcpd
```

重启客户端网卡测试

```bash
[root@serverb ~]# nmcli connection down dhcptest ;nmcli connection up dhcptest
[root@serverb ~]# ip a s eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:01:fa:0b brd ff:ff:ff:ff:ff:ff
    inet 172.25.250.110/24 brd 172.25.250.255 scope global dynamic noprefixroute eth1
       valid_lft 597sec preferred_lft 597sec
    inet6 fe80::51d6:1bf5:3111:5ae5/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

# 配置 IPv6 地址分配

## IPv6 地址自动配置概述

IPv6 具有多个可用于配置网络接口的方法。

- 无状态地址自动配置 (SLAAC)

- 适用于 IPv6 的动态主机配置协议 (DHCPv6)

两者都要依赖于自动配置的本地链路地址才能发挥作用。

### 无状态地址自动配置 (SLAAC)

无状态地址自动配置 (SLAAC) 方法依赖于路由器为客户端系统提供网络配置。这可能包括 IPv6 网络的前缀（客户端可以使用它来创建地址）和 DNS 信息。对于这种方法，必须在路由器上激活和配置邻居发现协议 (NDP)。路由器公告消息仅提供网络前缀

```textile
[客户端系统启动或连接到 IPv6 网络]
      |
      v
[生成基于 MAC 地址的 EUI-64 接口标识符]
      |
      v
[使用本地链路地址发送路由器请求消息到 ff02::2]
      |
      v
[路由器接收到请求后发送路由器公告消息（包含网络前缀等参数）]
      |
      v
[客户端使用网络前缀和 EUI-64 生成全局单播地址]
      |
      v
[进行地址冲突检测（DAD）]
      |
      v
[如果 DAD 成功，客户端使用该地址进行通信]
      |
      v
[如果 DAD 失败，客户端生成新的地址并重新进行 DAD]
      |
      v
[路由器定期发送路由器公告消息以更新网络配置和宣告存在]
      |
      v
[客户端根据路由器公告更新网络配置]
```

### 监控路由器公告消息

出于故障排除目的，或者为了发现路由器所公告的参数，可以监控网络中的路由器公告消息。路由器通常以较低的频率发送这些消息，有时每隔 10 分钟发送一条消息。因此，您可能需要等待几分钟，让 `radvdump` 命令捕获其第一条消息。

我不想等，我自己安装了radvd 软件包提供 `radvd` 服务，可以通过 `/etc/radvd.conf` 配置文件进行配置，我额外拿了一个host1来做这个测试

可以明显看到prefix前缀

```textile
[root@host1 ~]# radvdump
#
# radvd configuration generated by radvdump 2.19
# based on Router Advertisement from fe80::20c:29ff:fea7:41b5
# received by interface ens160
#

interface ens160
{
        AdvSendAdvert on;
        # Note: {Min,Max}RtrAdvInterval cannot be obtained with radvdump
        AdvManagedFlag off;
        AdvOtherConfigFlag off;
        AdvReachableTime 0;
        AdvRetransTimer 0;
        AdvCurHopLimit 64;
        AdvDefaultLifetime 0;
        AdvHomeAgentFlag off;
        AdvDefaultPreference medium;
        AdvSourceLLAddress on;

        prefix 2001:db8:1::/64
        {
                AdvValidLifetime 86400;
                AdvPreferredLifetime 14400;
                AdvOnLink on;
                AdvAutonomous on;
                AdvRouterAddr off;
        }; # End of prefix definition

}; # End of interface definition
```

路由器提供了前缀之后，我们来看看IPV6地址，成功具有正确前缀的IPV6地址

```bash
[root@host1 ~]# ip a s ens160
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:a7:41:b5 brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 192.168.8.100/24 brd 192.168.8.255 scope global noprefixroute ens160
       valid_lft forever preferred_lft forever
    inet6 2001:db8:1:0:20c:29ff:fea7:41b5/64 scope global dynamic noprefixroute
       valid_lft 86383sec preferred_lft 14383sec
    inet6 fe80::20c:29ff:fea7:41b5/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

也成功提供了网关信息

```bash
[root@host1 ~]# ip -6 route
::1 dev lo proto kernel metric 256 pref medium
2001:db8:1::/64 dev ens160 proto ra metric 100 pref medium
fe80::/64 dev ens160 proto kernel metric 1024 pref medium
```

## DHCPv6分配流程

以下是每个步骤的简要说明：

1. **启动或连接**：客户端系统启动或连接到 IPv6 网络。
2. **生成链路本地地址**：客户端生成一个临时的链路本地地址用于通信。
3. **发送 Solicit 消息**：客户端发送 Solicit 消息以发现可用的 DHCPv6 服务器。
4. **接收 Advertise 消息**：DHCPv6 服务器接收到 Solicit 消息并回复 Advertise 消息，提供配置选项。
5. **发送 Request 消息**：客户端选择一个 DHCPv6 服务器并发送 Request 消息请求地址分配。
6. **接收 Reply 消息**：DHCPv6 服务器接收到 Request 并发送 Reply 消息，包含 IPv6 地址和其他配置参数。
7. **配置地址和参数**：客户端使用 Reply 消息中的信息配置 IPv6 地址和其他网络参数。
8. **网络通信**：客户端使用配置的地址进行网络通信。
9. **续订地址**：在地址租约到期前，客户端发送 Renew 或 Rebind 消息以续订地址。
10. **更新租约**：DHCPv6 服务器响应 Renew 或 Rebind 消息，更新地址租约。

```textile
[客户端系统启动或连接到 IPv6 网络]
  |
  v
[客户端生成临时的链路本地地址（LLA）]
  |
  v
[客户端发送 DHCPv6 消息类型为 Solicit]
  |
  v
[DHCPv6 服务器接收到 Solicit 消息并发送 Advertise 消息]
  |
  v
[客户端选择一个 DHCPv6 服务器并发送 Request 消息]
  |
  v
[DHCPv6 服务器接收到 Request 消息并发送 Reply 消息]
  |
  v
[客户端接收到 Reply 消息并配置 IPv6 地址和其他参数]
  |
  v
[客户端使用配置的 IPv6 地址进行网络通信]
  |
  v
[客户端在需要时发送 Renew 或 Rebind 消息以续订地址]
  |
  v
[DHCPv6 服务器响应 Renew 或 Rebind 消息以更新租约]
```

## 实施 DHCPv6

**将路由器公告消息与 DHCPv6 进行比较**

| 配置参数                | 路由器公告 | DHCPv6 |
| ------------------- | ----- | ------ |
| IPv6 前缀             | 是     | 是      |
| 默认网关                | 是     | 否      |
| DNS 服务器             | 是     | 是      |
| DNS 搜索列表            | 是     | 是      |
| 其他参数，如 NIS 或 NTP 参数 | 否     | 是      |

请注意，DHCPv6 无法向客户端提供默认网关。客户端依靠路由器公告消息获取该信息。

在实践中，路由器公告和 DHCPv6 协同运作。路由器公告消息可以提供 IPv6 前缀和默认网关，并且可以指示客户端使用 DHCPv6 来检索额外的网络配置参数。

通过 `radvdump` 实用程序收集的以下路由器公告消息显示了该配置。

修改/etc/radvd.conf可以开启参数

- 客户端系统将路由器的本地链路 IPv6 地址用作其默认网关。

- 当设置为 on 时，AdvManagedFlag 参数指示客户端从 DHCPv6 检索 IPV6 地址的接口 ID。客户端不再计算地址的该部分。

- 当设置为 on 时，AdvOtherConfigFlag 参数指示客户端从 DHCPv6 检索其他网络配置参数，如 DNS 服务器和搜索列表。

```textile
[root@host1 ~]# radvdump
#
# radvd configuration generated by radvdump 2.19
# based on Router Advertisement from fe80::20c:29ff:fea7:41b5
# received by interface ens160
#

interface ens160
{
        AdvSendAdvert on;
        # Note: {Min,Max}RtrAdvInterval cannot be obtained with radvdump
        AdvManagedFlag on;
        AdvOtherConfigFlag on;
        AdvReachableTime 0;
        AdvRetransTimer 0;
        AdvCurHopLimit 64;
        AdvDefaultLifetime 0;
        AdvHomeAgentFlag off;
        AdvDefaultPreference medium;
        AdvSourceLLAddress on;

        prefix 2001:db8:1::/64
        {
                AdvValidLifetime 86400;
                AdvPreferredLifetime 14400;
                AdvOnLink on;
                AdvAutonomous on;
                AdvRouterAddr off;
        }; # End of prefix definition

}; # End of interface definition
```

### 部署 DHCPv6 服务器

DHCPv6 网络通信与 DHCPv4 类似。一个不同之处在于，所有 DHCPv6 服务器和中继都在本地链路多播地址 `ff02::1:2` 上侦听来自客户端本地链路地址的请求，因为 IPv6 不支持广播消息。

DHCPv6 服务器的系统必须有一个静态 IPv6 地址使用的前缀与 DHCPv6 服务器管理的前缀相同。所以先在workstation上打一下lab给我们提供IPv6的路由器和DNS服务器，打完lab之后，给我们提供一个 `fde2:6494:1e09:2::/64` 前缀以及路由器

```bash
[student@workstation ~]$ lab dhcp-ipv6config start
```

先给服务器自身配一个IPV6，不然无法工作

```bash
[root@servera ~]# nmcli con add con-name lxhdhcpv6 type ethernet \
ifname eth1 ipv6.addresses fde2:6494:1e09:2::a/64 ipv6.method manual
[root@servera ~]# nmcli con up lxhdhcpv6
```

确认地址配置完成

```bash
[root@servera ~]# ip -6 a s eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
    inet6 fde2:6494:1e09:2::a/64 scope global noprefixroute
       valid_lft forever preferred_lft forever
    inet6 fe80::a5:8862:cccb:2607/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

在部署 DHCP 服务器之前，捕获 IPv6 路由器定期发送的 NDP 公告，看看路由器是否在指示客户端使用DHCPV6，`radvdump`命令可能需要较长时间的等待，请耐心

- `fe80::ef0:1eaf:de3b:` 是serverd机器的本地链路地址，这是我们的路由器服务器

- `AdvSendAdvert on` 发送常规路由器公告已开启

- `AdvManagedFlag on` 指示客户端查询 DHCPv6 服务器，以获得 IPv6 地址的接口 ID，而不是自行计算该部分

- `AdvOtherConfigFlag on` 指示客户端查询 DHCPv6 服务器，以检索网络配置的其余部分，如 DNS 参数

```bash
[root@servera ~]# yum install radvd -y
[root@servera ~]# radvdump
#
# radvd configuration generated by radvdump 2.17
# based on Router Advertisement from fe80::ef0:1eaf:de3b:7a88
# received by interface eth1
#

interface eth1
{
        AdvSendAdvert on;
        # Note: {Min,Max}RtrAdvInterval cannot be obtained with radvdump
        AdvManagedFlag on;
        AdvOtherConfigFlag on;
        AdvReachableTime 0;
        AdvRetransTimer 0;
        AdvCurHopLimit 64;
        AdvDefaultLifetime 180;
        AdvHomeAgentFlag off;
        AdvDefaultPreference medium;
        AdvSourceLLAddress on;
}; # End of interface definition
```

看上去路由器是就绪的，我们来部署DHCPv6服务器

```bash
[root@servera ~]# yum install dhcp-server -y
```

### 配置 DHCPv6 服务器

`dhcpd6` 服务使用 `/etc/dhcp/dhcpd6.conf` 配置文件。dhcp-server 软件包提供 `/usr/share/doc/dhcp-server/dhcpd6.conf.example` 配置文件作为示例。

```bash
[root@servera ~]# cp /usr/share/doc/dhcp-server/dhcpd6.conf.example /etc/dhcp/dhcpd6.conf
cp: overwrite '/etc/dhcp/dhcpd6.conf'? y
```

**DHCPv6 服务器参数**

| 参数            | 值                                                 |
| ------------- | ------------------------------------------------- |
| 对网络段具有权威      | 是                                                 |
| IPv6 子网       | `fde2:6494:1e09:2::/64`                           |
| 提供的 IPv6 地址范围 | 从 `fde2:6494:1e09:2::20` 到 `fde2:6494:1e09:2::60` |
| DNS 服务器       | `fde2:6494:1e09:2::d`                             |
| DNS 搜索域       | `backend.lab.example.com`                         |
| 默认租用时间        | 600 秒                                             |
| 最长租用时间        | 7200 秒                                            |

preferred-lifetime 这个参数有可能会导致客户端无法获得地址，请从配置文件中删除其他内容，只保留以下参数

```textile
authoritative;

subnet6 fde2:6494:1e09:2::/64 {
  range6 fde2:6494:1e09:2::20 fde2:6494:1e09:2::60;
  option dhcp6.name-servers fde2:6494:1e09:2::d;
  option dhcp6.domain-search "backend.lab.example.com";
  default-lease-time 600;
  max-lease-time 7200;
}
```

### 验证配置

```bash
[root@servera ~]# dhcpd -t -6 -cf /etc/dhcp/dhcpd6.conf
Internet Systems Consortium DHCP Server 4.3.6
Copyright 2004-2017 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/
ldap_gssapi_principal is not set,GSSAPI Authentication for LDAP will not be used
Not searching LDAP since ldap-server, ldap-port and ldap-base-dn were not specified in the config file
Config file: /etc/dhcp/dhcpd6.conf
Database file: /var/lib/dhcpd/dhcpd6.leases
PID file: /var/run/dhcpd6.pid
```

### 启动并启用服务

```bash
[root@servera ~]# systemctl enable --now dhcpd6
[root@servera ~]# firewall-cmd --permanent --add-service=dhcpv6
[root@servera ~]# firewall-cmd --reload
```

### DHCPv6客户端测试

我们拿serverb来测试

先看看网卡哪个是闲的

```bash
[root@serverb ~]# nmcli device status
DEVICE  TYPE      STATE         CONNECTION
eth0    ethernet  connected     Wired connection 1
eth1    ethernet  disconnected  --
eth2    ethernet  disconnected  --
lo      loopback  unmanaged     --
```

要使用 SLAAC 检索网络参数，可将 `auto` 用于 `ipv6.method` 参数

```bash
[root@serverb ~]# nmcli con add con-name lxhdhcpv6test type ethernet \
ifname eth1 ipv6.method auto

[root@serverb ~]# nmcli con up lxhdhcpv6test
```

客户端成功获得ipv6地址、dns、网关

```bash
[root@serverb ~]# ip -6 a s eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
    inet6 fde2:6494:1e09:2::60/128 scope global dynamic noprefixroute
       valid_lft 514sec preferred_lft 289sec
    inet6 fe80::1b06:e035:2b4:d985/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

```bash
[root@serverb ~]# cat /etc/resolv.conf
# Generated by NetworkManager
search lab.example.com example.com backend.lab.example.com
nameserver 172.25.250.254
nameserver fde2:6494:1e09:2::d
```

```bash
[root@serverb ~]# ip -6 route
::1 dev lo proto kernel metric 256 pref medium
fde2:6494:1e09:2::60 dev eth1 proto kernel metric 100 pref medium
fe80::/64 dev eth1 proto kernel metric 100 pref medium
fe80::/64 dev eth0 proto kernel metric 106 pref medium
default via fe80::ef0:1eaf:de3b:7a88 dev eth1 proto ra metric 100 pref medium
```

注意网关是serverd的本地链路地址

```bash
[root@serverd ~]# ip a s eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:01:fa:0d brd ff:ff:ff:ff:ff:ff
    inet6 fde2:6494:1e09:2::d/64 scope global noprefixroute
       valid_lft forever preferred_lft forever
    inet6 fe80::ef0:1eaf:de3b:7a88/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

### DHCPv6地址保留

DHCPv6 是基于DUID做的地址保留，DUID 是一种用于 DHCPv6 客户端和服务器之间通信的唯一标识符，它在 DHCPv6 协议中用于识别和跟踪客户端。DUID 可以基于多种信息生成，包括但不限于 MAC 地址，它允许客户端即使在更改其 MAC 地址或在不同网络接口上操作时也能保持其身份

1. 先让客户端从dhcpv6获取一个地址，看看duid是多少

```bash
[root@serverc ~]# nmcli con add con-name lxh-dhcpv6-keep type ethernet \
ifname eth1 ipv6.method auto
[root@serverc ~]# nmcli connection up lxh-dhcpv6-keep
```

在服务器上执行后，得到了duid，看到它分配了`fde2:6494:1e09:2::59`

```text
[root@servera ~]# journalctl -u dhcpd6.service | grep duid
Aug 23 21:02:21 servera.lab.example.com dhcpd[26745]: Reply NA: address fde2:6494:1e09:2::59 to client with duid 00:04:49:f0:60:cc:52:78:26:57:89:6f:f8:bf:35:6f:e0:7a iaid = 713252315 valid for 600 seconds
```

2. 为这个DUID分配固定地址

在配置文件中，添加以下内容，我们给它分配了一个`fde2:6494:1e09:2::33`

```text
host dhcp6-duid-keep-test {
  host-identifier option
    dhcp6.client-id 00:04:49:f0:60:cc:52:78:26:57:89:6f:f8:bf:35:6f:e0:7a;
  fixed-address6 fde2:6494:1e09:2::33;
}
```

检查配置文件

```bash
[root@servera ~]# dhcpd -t -6 -cf /etc/dhcp/dhcpd6.conf
```

在客户端上重启网卡

```bash
[root@serverc ~]# nmcli connection down lxh-dhcpv6-keep
[root@serverc ~]# nmcli connection up lxh-dhcpv6-keep
```

再次查询客户端地址，发现已经按照我们的预期分配了地址

```bash
[root@serverc ~]# ip a s eth1
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:01:fa:0c brd ff:ff:ff:ff:ff:ff
    inet6 fde2:6494:1e09:2::33/128 scope global dynamic noprefixroute
       valid_lft 591sec preferred_lft 366sec
    inet6 fe80::7991:af5c:3599:ece6/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

# 自动化执行 DHCP 配置

详见教材


