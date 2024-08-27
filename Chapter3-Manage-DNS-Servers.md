```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 章节目标

- 描述 DNS 协议的基本操作、域和区域的定义以及一些关键的 DNS 资源记录。

- 使用 Unbound 配置更安全的缓存名称服务器。

- 使用标准的命令行实用程序对 DNS 名称解析问题进行故障排除。

- 使用 BIND 9 配置权威 DNS 名称服务器。

- 使用 Unbound 自动创建和配置缓存名称服务器，使用 BIND 9 自动创建和配置权威名称服务器。

# 描述 DNS 服务

## 域名系统

*域名系统 (DNS)* 是一个分层命名系统，充当联网主机和资源的目录。目录中的信息将网络名称映射到数据，并且在称为*资源记录*的逻辑条目中维护。DNS 层次结构从顶层的 `root` 域 (`.`) 开始，再向下划分为多个下级域。

DNS 层次结构的每个级别在域名中用“点”(.) 表示，以“`.`”作为顶级。`.com`、`.net` 和 `.org` 之类的域占据层次结构的第二级，`example.com` 和 `redhat.com` 之类的域占据第三级，以此类推。

### 域

域是资源记录的集合，该集合以通用名结尾并且表示 DNS 命名空间的整个子树，如 `example.com`。

子域是指作为另一个域的完整子树的域。例如，`lab.example.com` 是 `example.com` 的子域

### 区域

区域是指特定名称服务器直接负责或对其具有权威的某个域的组成部分。这可以是整个域，或者只是域的一部分，其部分或所有子域被委派给一个或多个其他名称服务器。

## DNS 查询分析

![](https://gitee.com/cnlxh/rh358/raw/master/images/dns/dns-lookups.svg)

## DNS 资源记录

NS 区域中的 DNS *资源记录 (RR)* 条目用于指定有关该区域中某个特定名称或对象的信息。一个资源记录包含 `TTL`、`class`、`type` 和 `data` 元素，它们按以下格式进行组织：

```textile
owner-name           TTL      class      type     data
www.example.com.     300      IN         A        192.168.1.10
```

**资源记录字段**

| 字段名称       | 内容                                        |
| ---------- | ----------------------------------------- |
| owner-name | 该资源记录的名称。                                 |
| TTL        | 资源记录的*生存时间*（秒）。这会指定 DNS 解析器应缓存此资源记录的时间长度。 |
| class      | 记录的“类”，几乎总是 `IN`（表示 Internet）。            |
| type       | 此记录存储的信息的类型。例如，A 记录将主机名映射到 IPv4 地址。       |
| data       | 此记录存储的数据。确切格式根据记录类型而有所不同。                 |

重要的资源记录类型包括：

### A 资源记录

A 资源记录将主机名映射到 IPv4 地址。

```text
host.example.com.       86400   IN      A       172.25.254.254
```

### AAAA 资源记录

AAAA 资源记录（也称为“四 A”记录）将主机名映射到 IPv6 地址。

```text
a.root-servers.net.     604800  IN      AAAA    2001:503:ba3e::2:30
```

### CNAME 资源记录

CNAME 资源记录将一个名称别名化为另一个名称（*规范名称*），其中应具有 A 或 AAAA 记录。

当 DNS 解析器为响应查询而收到 CNAME 记录时，它将使用规范名称（而非原始名称）重新发出查询。

CNAME 记录的数据字段可以指向 DNS 中任何位置的名称，无论是区域内部还是区域外部：

```text
www-dev.example.com. 30 IN CNAME lab.example.com.
www.example.com.     30 IN CNAME www.redhat.com.
```

### PTR 资源记录

PTR（即指针）资源记录将 IPv4 或 IPv6 地址映射到主机名。它们用于*反向 DNS 解析*。

PTR 记录以类似于主机名的特殊格式对 IP 地址进行编码。

- 对于 IPv4 地址，地址被逆向，最具体的部分在最前面，并且结果被视为特殊域 `in-addr.arpa` 的子域中的主机。

- 对于 IPv6 地址，为特殊域 `ip6.arpa` 的子域。

```text
4.0.41.198.in-addr.arpa. 785    IN      PTR     a.root-servers.net.
0.3.0.0.2.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.e.3.a.b.3.0.5.0.1.0.0.2.ip6.arpa. 86400 IN PTR a.root-servers.net.
```

### NS 资源记录

NS（即名称服务器）资源记录将域名映射到对其 DNS 区域具有权威的 DNS 名称服务器。区域的每个公开权威名称服务器必须具有 NS 记录。

```textile
example.com.                       86400   IN  NS    classroom.example.com.
168.192.ip-addr.arpa.              86400   IN  NS    classroom.example.com.
9.0.e.1.4.8.4.6.2.e.d.f.ip6.arpa.  86400   IN  NS    classroom.example.com.
```

### SOA 资源记录

SOA（即授权起始）资源记录提供关于 DNS 区域如何运作的信息。每个区域必须正好有一个 SOA 记录。

区域的 SOA 记录将区域的其中一个名称服务器指定为负责维护区域的资源记录的主名称服务器。它负责指定区域的管理内容的电子邮件地址。SOA 记录还指定了一个序列号和各种超时，供其他权威名称服务器用来确定何时从主名称服务器传输区域资源记录。

```textile
example.com. 86400   IN  SOA  classroom.example.com. root.classroom.example.com. 2015071700 3600 300 604800 60
```

### MX 资源记录

MX 资源记录将域名映射到接受该域的电子邮件的*邮件交换器*。

此记录类型的数据是一个优先级编号（首选最低编号）

```textile
example.com.            86400   IN      MX      10 classroom.example.com.
example.com.            86400   IN      MX      10 mail.example.com.
example.com.            86400   IN      MX      100 mailbackup.example.com.
```

### TXT 资源记录

TXT 资源记录将名称映射到编码为可打印 ASCII 字符的任意文本。它们通常用于提供各种电子邮件身份验证方案（如 SPF、DKIM 和 DMARC 等等）所用的数据

```textile
lwn.net.   27272   IN  TXT    "google-site-verification: sVlx-LS_z1es5D-fNSUNXrqr3n9Y4F7tOr7HNVMKUGs"
lwn.net.   27272   IN  TXT    "v=spf1 a:mail.lwn.net a:prod.lwn.net a:git.lwn.net a:ms.lwn.net -all"
```

### SRV 资源记录

SRV 资源记录帮助客户端查找支持域的特定服务的主机。

下例中的 SRV 记录表示有一个可以使用 TCP 传输协议 (`_tcp`) 进行联系的 LDAP 服务器 (`_ldap`)，该服务器属于域 `example.com`。该服务器为 `server0.example.com`，正在侦听端口 389，其优先级为 0，权重为 100（当客户端收到多个 SRV 记录时，它用于控制选取哪一个服务器）。

```textile
_ldap._tcp.example.com. 86400 IN SRV    0 100 389 server0.example.com.
```

# 使用 Unbound 配置缓存名称服务器

Unbound是一款轻量级的DNS（域名系统）缓存和递归解析器，通常用于缓存服务器场景，缓存名称服务器在本地缓存中存储 DNS 查询结果，并且在 TTL 到期后从缓存中删除资源记录。通常设置缓存名称服务器以代表本地网络上的客户端执行查询。这降低了 Internet 上的 DNS 流量，从而极大提高了 DNS 名称解析的效率。随着缓存的增加，缓存名称服务器从其本地缓存中应答越来越多的客户端查询，从而提高 DNS 性能。

### 安装和配置 Unbound

```bash
[root@servera ~]# yum install unbound -y
```

**定义 Unbound 将侦听的网络接口**

默认情况下，Unbound 仅侦听 `localhost` 网络接口。要使远程客户端能够使用 Unbound 作为缓存名称服务器，请使用 `/etc/unbound/unbound.conf` 的 `server` 子句中的 `interface` 选项来指定要侦听的网络接口

```textile
interface: 172.25.250.10
```

**配置ACL**

通过在 `/etc/unbound/unbound.conf` 的 `server` 子句中添加以下选项，以允许来自 `172.25.250.0/24` 子网的查询。

```bash
access-control: 172.25.250.0/24 allow
```

**配置DNSSEC**

通过在 `/etc/unbound/unbound.conf` 的 `server` 子句中添加以下选项，以使 `example.com` 区域免于进行 DNSSEC 验证。

```text
domain-insecure: "example.com"
```

**配置DNS转发**

通过在 /etc/unbound/unbound.conf 文件的结尾添加 forward-zone 子句，将所有查询转发到 172.25.250.254。

这里要注意，在配置文件中，默认有一个'*'的配置，这个要改成一个英文的点

```text
forward-zone:
  name: "."
  forward-addr: 172.25.250.254
```

**启用unbound-control管理**

这一步并不是必须的，只是为了演示启用这个管理功能

```text
[root@servera ~]# unbound-control-setup
setup in directory /etc/unbound
generating unbound_server.key
Generating RSA private key, 3072 bit long modulus (2 primes)
..........................................................++++
.....++++
e is 65537 (0x010001)
generating unbound_control.key
Generating RSA private key, 3072 bit long modulus (2 primes)
....................................++++
......................................++++
e is 65537 (0x010001)
create unbound_server.pem (self signed certificate)
create unbound_control.pem (signed client certificate)
Signature ok
subject=CN = unbound-control
Getting CA Private Key
Setup success. Certificates created. Enable in unbound.conf file to use
```

**检查是否有语法错误**

```bash
[root@servera ~]# unbound-checkconf
unbound-checkconf: no errors in /etc/unbound/unbound.conf
```

**配置防火墙**

配置防火墙以允许 DNS 流量。DNS 使用端口 53/UDP 和 53/TCP

```bash
[root@servera ~]# firewall-cmd --permanent --add-service=dns
[root@servera ~]# firewall-cmd --reload
```

**启动 unbound 服务**

这个服务启动需要较久的时间，耐心等待

```bash
[root@servera ~]# systemctl enable --now unbound
```

**查询解析**

```bash
[root@servera ~]# dig -t A servera.lab.example.com @172.25.250.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el8 <<>> -t A servera.lab.example.com @172.25.250.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57909
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;servera.lab.example.com.       IN      A

;; ANSWER SECTION:
servera.lab.example.com. 3600   IN      A       172.25.250.10

;; Query time: 1 msec
;; SERVER: 172.25.250.10#53(172.25.250.10)
;; WHEN: Thu Aug 22 21:43:45 CST 2024
;; MSG SIZE  rcvd: 68
```

### 管理 Unbound

#### 转储和加载 Unbound 缓存

保存查询缓存

```bash
[root@servera ~]# unbound-control dump_cache > cache.txt
[root@servera ~]# cat cache.txt
START_RRSET_CACHE
;rrset 3544 1 0 8 3
bastion.lab.example.com.        3544    IN      A       172.25.250.254
;rrset 3535 1 0 8 3
servera.lab.example.com.        3535    IN      A       172.25.250.10
END_RRSET_CACHE
START_MSG_CACHE
msg servera.lab.example.com. IN A 33152 1 3535 3 1 0 0
servera.lab.example.com. IN A 0
msg bastion.lab.example.com. IN A 33152 1 3544 3 1 0 0
bastion.lab.example.com. IN A 0
END_MSG_CACHE
EOF
```

加载缓存

```bash
unbound-control load_cache < cache.txt
```

清空 Unbound 缓存

通过对记录执行 `unbound-control flush` 来强制清除过期记录，而不必等待 TTL 过期

```bash
unbound-control flush www.example.com
unbound-control flush_zone example.com
```

案例如下

```bash
[root@servera ~]# unbound-control flush bastion.lab.example.com
ok
[root@servera ~]# unbound-control dump_cache > cache.txt
[root@servera ~]# cat cache.txt
START_RRSET_CACHE
;rrset 3132 1 0 8 3
servera.lab.example.com.        3132    IN      A       172.25.250.10
END_RRSET_CACHE
START_MSG_CACHE
msg servera.lab.example.com. IN A 33152 1 3238 3 1 0 0
servera.lab.example.com. IN A 0
END_MSG_CACHE
EOF
```

# DNS 问题故障排除

行之有效的名称解析取决于以下几项是否正确：

- 客户端上的名称解析和 `/etc/resolv.conf`。

- 客户端使用的缓存名称服务器的运行。

- 向缓存名称服务器提供数据的权威名称服务器的运行。

- 权威名称服务器上的数据。

- 用于在这些系统之间进行通信的网络的配置。

### 确认名称解析问题的来源

系统的 `/etc/nsswitch.conf` 文件中的 `hosts` 行控制其如何查找主机名

```textile
[root@servera ~]# cat /etc/nsswitch.conf | grep ^hosts
hosts:      files dns myhostname
```

`getent` 命令执行名称解析的结果如果不同于 `dig` 产生的结果，那么这明确表明并非是 DNS 导致了意外的名称解析结果。

- `getent` 通常读取 `/etc/hosts` 文件和 DNS 缓存（如 `/etc/resolv.conf` 指定的 DNS 服务器）来解析主机名。
- `dig` 是一个 DNS 查找工具，它直接查询 DNS 服务器，可以指定使用特定的 DNS 服务器或递归查找。

```textile
[root@servera ~]# getent hosts bastion.lab.example.com
172.25.250.254  bastion.lab.example.com bastion
```

### dig 基本使用

默认情况下，它会查找该名称的 A 记录

结果显示，请求的状态为 `NOERROR`，这表示它已成功完成。提供了一条记录作为应答。`QUESTION SECTION` 重述查询，而 `ANSWER SECTION` 则显示返回的资源记录：`servera.lab.example.com` 具有指向 `172.25.250.10` 的 A 记录，并且具有一个生存时间 (TTL)，表示应将应答缓存 3600 秒。

```text
[root@servera ~]# dig servera.lab.example.com

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el8 <<>> servera.lab.example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12731
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;servera.lab.example.com.       IN      A

;; ANSWER SECTION:
servera.lab.example.com. 3600   IN      A       172.25.250.10

;; Query time: 2 msec
;; SERVER: 172.25.250.254#53(172.25.250.254)
;; WHEN: Thu Aug 22 22:13:24 CST 2024
;; MSG SIZE  rcvd: 68
```

**查询特定类型**

```bash
[root@servera ~]# dig -t a bastion.lab.example.com
```

`dig` 使用 `/etc/resolv.conf` 中列出的 DNS 名称服务器。也可以通过在命令行中添加 `@` 后跟服务器的名称来指定不同的名称服务器

```textile
[root@servera ~]# dig @172.25.250.254 -t a bastion.lab.example.com
```

### DNS 状态代码

```bash
[root@servera ~]# dig @172.25.250.254 -t a bastion.lab.example.com

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el8 <<>> @172.25.250.254 -t a bastion.lab.example.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29903
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
```

| 代码         | 含义                          |
| ---------- | --------------------------- |
| `SERVFAIL` | 名称服务器在处理查询时遇到问题。            |
| `NXDOMAIN` | 查询名称在区域中不存在。                |
| `REFUSED`  | 由于策略限制，名称服务器拒绝了客户端的 DNS 请求。 |

### TTL 设置

权威名称服务器管理员应将记录的 TTL 设低一些，如果您管理的缓存名称服务器具有过时、未过期的记录，请使用 ``unbound-control flush *`record-name`*`` 命令来清除

### 识别循环 CNAME 记录

无法解析的 CNAME 循环

```textile
test.example.com. IN CNAME lab.example.com.
lab.example.com.  IN CNAME test.example.com.
```

# 使用 BIND 9 配置权威名称服务器

BIND 9 是一个强大且广泛使用的 DNS 服务软件，适用于多种应用场景，并且拥有活跃的社区支持和持续的更新

## 为权威名称服务器设计架构

通过 BIND，可以将权威服务器配置为区域的<mark>主要或次要服务器</mark>。

次要服务器通过请求区域传送定期从主服务器下载最新版本的区域信息。它们多久执行一次区域传送以及如何判断其数据是否过期都由区域的 SOA 资源记录控制。

为了确保可靠性，您应至少具有两个公共 DNS 服务器，并且它们应位于不同的站点，以避免由于网络故障而造成的停机。

并非所有权威服务器都必须是公共服务器。例如，您可能决定仅使用您的主服务器来管理区域文件，并将区域信息发布到权威次要服务器。主服务器可以是私有服务器，但次要服务器将是面向公共的，为外部客户端提供权威应答。这可帮助您保护您的主服务器免受攻击。

下图显示了外部主机如何使用其缓存名称服务器和权威名称服务器对 `example.com` 中的记录执行 DNS 查找，假设尚未缓存任何记录：

在本示例中，主名称服务器实际上不是公共的，而是可以从主服务器执行区域传送的次要名称服务器，以便它们拥有关于 `example.com` 区域的最新数据。

![](https://gitee.com/cnlxh/rh358/raw/master/images/dns/bind-external-client.svg)

## 安装配置bind

### 安装bind

通过安装 bind 软件包来安装 BIND。名称服务器本身作为 `named` 服务运行。bind 软件包也会以 HTML 和 PDF 格式在 `/usr/share/doc/bind/` 目录中安装综合 BIND 文档。

```bash
[root@servera ~]# yum install bind -y
```

### 启动bind服务

```bash
[root@servera ~]# systemctl enable named.service --now
```

### 开通防火墙

```bash
[root@servera ~]# firewall-cmd --add-service=dns --permanent
[root@servera ~]# firewall-cmd --reload
```

### 配置 主BIND服务器

`named` 使用的主配置文件是 `/etc/named.conf`。此文件控制 BIND 的基本操作。它<mark>应由 `root` 用户、`named` 组所有</mark>，具有八进制权限 `0640`，并且具有 `named_conf_t` SELinux 类型。<mark>此文件一般以封号结尾</mark>

配置文件还指定由权威服务器管理的每个区域的区域文件的位置，这些文件通常保存在 `/var/named` 中。

配置 DNS 服务器需要以下步骤：

- 定义地址匹配列表，以便于维护。

- 配置`named` 侦听的 IP 地址。

- 配置客户端的访问控制。

- 指定 `zone` 指令。

- 在 `/etc/named.conf` 配置文件的外部，写入区域文件。

#### 定义地址匹配列表

以使用 `acl` 指令来定义地址匹配列表，这个acl位于options的上方

条目可以是由尾部点 (`192.168.0.`) 或 CIDR 表示法（`192.168.0/24` 或 `2001:db8::/32`）表示的完整 IP 地址或网络，或者是之前定义的地址匹配列表的名称。

我定义了一个名为 `trusted` 的 ACL，其中只包含多个 IP 地址 `172.25.0/16`

又定义了一个名为 `classroom` 的 ACL，它包含了一个网络段 `192.168.0.0/24`以及trusted里的所有IP

```textile
acl trusted         { 172.25.0/16; };
acl classroom       { 192.168.0.0/24; trusted; };
```

`named` 中内置有四个预定义的 ACL：

| `ACL`       | `描述`                   |
| ----------- | ---------------------- |
| `none`      | 无匹配主机。                 |
| `any`       | 匹配全部主机。                |
| `localhost` | 匹配 DNS 服务器的全部 IP 地址。   |
| `localnets` | 匹配 DNS 服务器所在本地子网的全部主机。 |

#### 配置服务器接口

`listen-on` 和 `listen-on-v6`，用于指示 `named` 侦听的接口和端口，可以用封号表达多个，甚至直接使用你的acl名称也可以

```textile
acl trusted         { 172.25.0/16; };
acl classroom       { 192.168.0.0/24; trusted; };
options {
        listen-on port 53 { 127.0.0.1; 172.25.250.10; trusted; };
```

#### 限制访问

`/etc/named.conf` 中的三个附加 `options` 指令对于控制访问非常重要：

- `allow-query` 控制所有查询。

- `allow-recursion` 控制递归查询。

- `allow-transfer` 控制区域传送。

默认情况下， `allow-query` 设置为 `localhost`，根据需求更改，例如any

```textile
allow-query     { localhost; };
```

权威服务器不应允许递归查询。如果您必须允许受信任的客户端执行递归，您可以打开递归，并为这些特定的主机或网络设置 `allow-recursion`：

权威 DNS 服务器是为其管理的特定域名提供确切答案的服务器。它们不需要执行递归查询，因为它们本身就是所查询域的权威数据源

递归 DNS 服务器通常用于客户端查询，它们会代表客户端向其他 DNS 服务器查询以获取查询结果，并可能将结果缓存起来以加快响应速度。

```textile
recursion yes;
allow-recursion { trusted; };
```

#### 声明权威区域

`/etc/named.conf` 中最下面的内容定义了区域文件，不过建议在include的文件中修改

```textile
include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

```vim
[root@servera ~]# vim /etc/named.rfc1912.zones
```

`file` 指令指定相对路径名称。这些文件的位置由 `options` 块中的 `directory` 设置决定（默认为 `/var/named/`）。始终将 `named.conf` 配置为将次要区域文件保存在 `slaves/` 子目录中，其 SELinux 类型为 `named_cache_t`。SELinux 会阻止 `named` 在其他位置创建文件。

这里提前添加了辅助服务器需要的参数

```textile
zone "lixiaohui.cn" IN {
        type master;
        file "lixiaohui.cn.zone";
        allow-update { none; };
        allow-transfer { 172.25.250.11; };
        notify yes;
        also-notify { 172.25.250.11; };
};
zone "250.25.172.in-addr.arpa" IN {
        type master;
        file "lixiaohui.cn.ptr";
        allow-update { none; };
        allow-transfer { 172.25.250.11; };
        notify yes;
        also-notify { 172.25.250.11; };
};
```

#### 编辑区域文件

区域文件的位置由 `options` 块中的 `directory` 指令和 `/etc/named.conf` 中的区域配置中的 `file` 指令控制。

如果您将次要区域文件保存在 `/var/named/slaves` 中，那么当次要服务器启动时，它会将该区域的缓存版本与主服务器上的当前版本进行比较，如果是最新状态，则使用它。如果区域不是最新的，或者文件不存在，则 `named` 会执行区域传送，并将结果缓存到该文件中。

区域文件应由 `root` 用户、`named` 组所有，具有八进制权限 `0640`，并且具有 `named_conf_t` SELinux 类型。

```textile
[root@servera ~]# cd /var/named/
[root@servera named]# ls
data  dynamic  named.ca  named.empty  named.localhost  named.loopback  slaves
[root@servera named]# cp named.localhost lixiaohui.cn.zone
[root@servera named]# cp named.loopback lixiaohui.cn.ptr
```

```text
[root@servera named]# chmod 640 /var/named/*.zone
[root@servera named]# chmod 640 /var/named/*.ptr
[root@servera named]# chcon -t named_zone_t /var/named/*.zone
[root@servera named]# chcon -t named_zone_t /var/named/*.ptr
[root@servera named]# chown root:named /var/named/*.zone
[root@servera named]# chown root:named /var/named/*.ptr
```

#### 区域文件格式

BIND 区域文件是一种文本文件，每行包含一个指令或资源记录。如果资源记录的数据中包含圆括号，它可以跨越多行。在同一物理行上分号 (`;`) 右侧的所有内容都将被注释掉。

区域文件可能以 `$TTL` 指令开头，该指令可为没有列出 TTL 的任何资源记录设置默认 TTL。这使您可以一次调整多个资源记录的 TTL，而无需编辑整个文件。如果 TTL 是一个数字，它将以秒为计量单位。

```textile
$TTL 3600
```

默认是秒，这个数字后可以跟一个字母，以指定更长的时间段：

- `M` 表示分钟（`1M` 为 `60`）

- `H` 表示小时（`1H` 为 `3600`）

- `D` 表示天（`1D` 为 `86400`）

- `W` 表示周 （`1W` 为 `604800`）

每个区域文件都包含一个 SOA （授权起始）资源记录，如下所示

```textile
$TTL 1D
lixiaohui.cn    IN SOA  ns.lixiaohui.cn. 939958092.qq.com. (
                                        1       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
```

- 第一行规定了TTL默认为1天

- 第二行最开始写了本区域的域名，也可以缩写为一个@，随后的IN SOA是表明这是互联网的SOA记录

- 第二行中的ns.lixiaohui.cn<mark>.</mark>部分是本区域的NS主机，注意以英文的句号结尾

- 第二行中的939958092.qq.com<mark>.</mark> 是本区域维护人的邮箱地址，我写了我的QQ邮箱，但是要把@变为英文的点，因为@在文件中有特别的意思

- serial代表文件版本的序列号。每次文件发生更改时，该序列号都必须增加。<mark>每次在主服务器上更新区域文件时，您必须增加序列号，并重新加载 `named`。</mark>

- refresh次要服务器应多久查询一次主服务器，以查看是否需要进行区域刷新。

- retry 如果主服务器停机导致刷新失败，次要服务器应多久尝试一次重新连接。

- expire 在放弃之前，次要服务器应尝试重新连接到不响应的主服务器的时长。如果发生超时，次要服务器将假设该区域不再存在，并停止对其查询的应答。

- minimum 其他名称服务器应缓存该区域负（“无此类记录”）响应的时长。例如，如果 `minimum` 设置为 `3H`，这表示对于该区域不存在的记录，DNS 服务器将缓存这个 NXDOMAIN 的结果长达 3 小时。在这 3 小时内，即使再次查询相同的不存在的记录，DNS 服务器也会直接返回缓存中 NXDOMAIN 的结果，而不需要再次发起查询请求

#### 将记录添加到区域文件

正常来说，区域文件必须具有：

- 一条 SOA 记录。

- 每个公共名称服务器的 NS 记录。

- 区域的其他 A、AAAA、CNAME、MX、SRV 和 TXT 记录。

**正向区域文件**

```textile
[root@servera ~]# vim /var/named/lixiaohui.cn.zone
```

一般来说，不需要在记录前面指定TTL，因为第一行写了1D

```text
$TTL 1D
lixiaohui.cn.    IN SOA  ns.lixiaohui.cn. 939958092.qq.com. (
                                        1       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        1D      IN      NS      ns.lixiaohui.cn.
ns      1D      IN      A       172.25.250.10
lxh     30      IN      A       172.25.250.100
        30      IN      AAAA    2001:db8:2020::5300
@       20      IN      MX      10 mail.lixiaohui.cn.
mail    30      IN      A       172.25.250.10
```

查询测试

发现TTL为30，的确是生效了

```text
[root@servera named]# dig -t a lxh.lixiaohui.cn @172.25.250.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el8 <<>> -t a lxh.lixiaohui.cn @172.25.250.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 63979
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 4756cdaceb0d67483c8a646766c763c2179ad76011e94218 (good)
;; QUESTION SECTION:
;lxh.lixiaohui.cn.              IN      A

;; ANSWER SECTION:
lxh.lixiaohui.cn.       30      IN      A       192.168.8.100

;; AUTHORITY SECTION:
lixiaohui.cn.           86400   IN      NS      ns.lixiaohui.cn.

;; ADDITIONAL SECTION:
ns.lixiaohui.cn.        86400   IN      A       172.25.250.10

;; Query time: 0 msec
;; SERVER: 172.25.250.10#53(172.25.250.10)
;; WHEN: Fri Aug 23 00:13:54 CST 2024
;; MSG SIZE  rcvd: 122
```

**反向区域文件**

```textile
[root@servera ~]# vim /var/named/lixiaohui.cn.ptr
```

```text
$TTL 1D
@       IN SOA  ns.lixiaohui.cn. 939958092.qq.com. (
                                        1       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        IN      NS      ns.lixiaohui.cn.
10      IN      PTR     ns.lixiaohui.cn.
100     IN      PTR     lxh.lixiaohui.cn.
```

查询测试

```text
[root@servera named]# dig -x 172.25.250.10 @172.25.250.10

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el8 <<>> -x 172.25.250.10 @172.25.250.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 13818
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 5cfd05c2de214996511d482266c7633236d99e1de8817760 (good)
;; QUESTION SECTION:
;10.250.25.172.in-addr.arpa.    IN      PTR

;; ANSWER SECTION:
10.250.25.172.in-addr.arpa. 86400 IN    PTR     ns.lixiaohui.cn.

;; AUTHORITY SECTION:
250.25.172.in-addr.arpa. 86400  IN      NS      ns.lixiaohui.cn.

;; ADDITIONAL SECTION:
ns.lixiaohui.cn.        86400   IN      A       172.25.250.10

;; Query time: 0 msec
;; SERVER: 172.25.250.10#53(172.25.250.10)
;; WHEN: Fri Aug 23 00:11:30 CST 2024
;; MSG SIZE  rcvd: 142
```

#### 子域委托

可以将整个子域作为含有其父域的区域的一部分进行设置和管理，例如添加 `support.online.lixiaohui.cn` 和 `test.online.lixiaohui.cn` 的记录：

```text
$TTL 1D
lixiaohui.cn.   IN SOA  ns.lixiaohui.cn. 939958092.qq.com. (
                                        1       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        1D      IN      NS      ns.lixiaohui.cn.
ns      1D      IN      A       172.25.250.10
lxh     30      IN      A       192.168.8.100
        30      IN      AAAA    2001:db8:2020::5300
@       20      IN      MX      10 mail.lixiaohui.cn.
mail    30      IN      A       172.25.250.10
support.online  1H      IN      A       172.25.250.10
```

查询发现DNS解析成功

```text
[root@servera named]# dig -t a support.online.lixiaohui.cn @localhost

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el8 <<>> -t a support.online.lixiaohui.cn @localhost
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 61584
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: d5d561bd5d71672046a5414966c765e14864ded5735acf36 (good)
;; QUESTION SECTION:
;support.online.lixiaohui.cn.   IN      A

;; ANSWER SECTION:
support.online.lixiaohui.cn. 3600 IN    A       172.25.250.10

;; AUTHORITY SECTION:
lixiaohui.cn.           86400   IN      NS      ns.lixiaohui.cn.

;; ADDITIONAL SECTION:
ns.lixiaohui.cn.        86400   IN      A       172.25.250.10

;; Query time: 0 msec
;; SERVER: ::1#53(::1)
;; WHEN: Fri Aug 23 00:22:57 CST 2024
;; MSG SIZE  rcvd: 133
```

#### 验证配置文件

在重新加载或重新启动 `named` 之前，您应验证 `/etc/named.conf` 和区域文件的语法。

- `named-checkconf` 验证 `/etc/named.conf`。

- `named-checkzone zone zone-file` 验证 `zone-file` 中 `zone` 的区域文件。

```bash
[root@servera named]# named-checkconf /etc/named.conf
[root@servera named]# named-checkconf /etc/named.rfc1912.zones
[root@servera named]# named-checkzone lixiaohui.cn /var/named/lixiaohui.cn.zone
zone lixiaohui.cn/IN: loaded serial 1
OK
```

如果服务起不来或者想看到详细内容，可以用下面的方法查看详情

```bash
[root@servera named]# journalctl -xeu named
```

### 配置辅助bind服务器

#### 安装bind

```textile
[root@serverb ~]# yum install bind -y
```

#### 配置主配置文件

将 BIND 配置为具有与主 BIND 服务器相同的安全配置。从 `主服务器` 复制 `/etc/named.conf` 配置

```bash
[root@serverb ~]# scp servera:/etc/named.conf /etc/named.conf
[root@serverb ~]# scp servera:/etc/named.rfc1912.zones /etc/named.rfc1912.zones
```

#### 配置区域文件

在刚才scp中，我们把主服务器的配置复制过来了，但是里面的类型是主服务器，我们需要修改为次要的，需要注意的是辅助区域在 `slaves/` 子目录中创建区域文件

```textile
[root@serverb ~]# vim /etc/named.rfc1912.zones
```

```textile
zone "lixiaohui.cn" IN {
        type slave;
        file "slaves/lixiaohui.cn.zone";
        masters { 172.25.250.10; };
};
zone "250.25.172.in-addr.arpa" IN {
        type slave;
        file "slaves/lixiaohui.cn.ptr";
        masters { 172.25.250.10; };
};
```

#### 配置文件权限

```bash
[root@serverb ~]# chown :named /etc/named.conf /etc/named.rfc1912.zones
[root@serverb ~]# chmod 0640 /etc/named.conf /etc/named.rfc1912.zones
```

#### 启动服务

```text
[root@serverb ~]# systemctl enable --now named
```

#### 开通防火墙

```text
[root@serverb ~]# firewall-cmd --add-service=dns --permanent
[root@serverb ~]# firewall-cmd --reload
```

#### 查询同步状态

```bash
[root@serverb ~]# journalctl -xeu named
```

```textile
Aug 23 01:01:51 serverb.lab.example.com named[26633]: zone lixiaohui.cn/IN: Transfer started.
Aug 23 01:01:51 serverb.lab.example.com named[26633]: transfer of 'lixiaohui.cn/IN' from 172.25.250.10#53: connected using 172.25.250.11#52091
Aug 23 01:01:51 serverb.lab.example.com named[26633]: zone lixiaohui.cn/IN: transferred serial 1
Aug 23 01:01:51 serverb.lab.example.com named[26633]: transfer of 'lixiaohui.cn/IN' from 172.25.250.10#53: Transfer status: success
Aug 23 01:01:51 serverb.lab.example.com named[26633]: transfer of 'lixiaohui.cn/IN' from 172.25.250.10#53: Transfer completed: 1 messages, 9 records, 281 bytes, 0.002 secs (140500 bytes/sec)
Aug 23 01:01:51 serverb.lab.example.com named[26633]: zone 250.25.172.in-addr.arpa/IN: Transfer started.
Aug 23 01:01:51 serverb.lab.example.com named[26633]: transfer of '250.25.172.in-addr.arpa/IN' from 172.25.250.10#53: connected using 172.25.250.11#33001
Aug 23 01:01:51 serverb.lab.example.com named[26633]: zone 250.25.172.in-addr.arpa/IN: transferred serial 1
Aug 23 01:01:51 serverb.lab.example.com named[26633]: transfer of '250.25.172.in-addr.arpa/IN' from 172.25.250.10#53: Transfer status: success
Aug 23 01:01:51 serverb.lab.example.com named[26633]: transfer of '250.25.172.in-addr.arpa/IN' from 172.25.250.10#53: Transfer completed: 1 messages, 5 records, 197 bytes, 0.001 secs (197000 bytes/sec)
Aug 23 01:02:01 serverb.lab.example.com named[26633]: resolver priming query complete
```

#### 查询辅助DNS服务器

```bash
[root@serverb ~]# ll /var/named/slaves/
total 8
-rw-r--r--. 1 named named 326 Aug 23 01:01 lixiaohui.cn.ptr
-rw-r--r--. 1 named named 480 Aug 23 01:01 lixiaohui.cn.zone
```

默认情况下无法用cat查询以上两个文件，因为是性能更好的raw格式，如果需要查询，可以将masterfile-format text;参数加入slave配置文件

```bash
[root@serverb ~]# cat /etc/named.rfc1912.zones
zone "lixiaohui.cn" IN {
        type slave;
        file "slaves/lixiaohui.cn.zone";
        masterfile-format text;
        masters { 172.25.250.10; };
};
zone "250.25.172.in-addr.arpa" IN {
        type slave;
        masterfile-format text;
        file "slaves/lixiaohui.cn.ptr";
        masters { 172.25.250.10; };
};
```

```text
[root@serverb ~]# cat /var/named/slaves/lixiaohui.cn.zone
$ORIGIN .
$TTL 86400      ; 1 day
lixiaohui.cn            IN SOA  ns.lixiaohui.cn. 939958092.qq.com. (
                                1          ; serial
                                86400      ; refresh (1 day)
                                3600       ; retry (1 hour)
                                604800     ; expire (1 week)
                                10800      ; minimum (3 hours)
                                )
                        NS      ns.lixiaohui.cn.
$TTL 20 ; 20 seconds
                        MX      10 mail.lixiaohui.cn.
$ORIGIN lixiaohui.cn.
abc                     NS      ns.zhangsan.com.
$TTL 30 ; 30 seconds
lxh                     A       192.168.8.100
                        AAAA    2001:db8:2020::5300
$TTL 86400      ; 1 day
ns                      A       172.25.250.10
$TTL 3600       ; 1 hour
support.online          A       172.25.250.10
```

正式用辅助DNS服务器查询

```bash
[root@serverb slaves]# dig -x 172.25.250.10 @172.25.250.11

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el8 <<>> -x 172.25.250.10 @172.25.250.11
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28337
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 6d630cc38d0004131424d42c66c77727a4c53994dcef3b99 (good)
;; QUESTION SECTION:
;10.250.25.172.in-addr.arpa.    IN      PTR

;; ANSWER SECTION:
10.250.25.172.in-addr.arpa. 86400 IN    PTR     ns.lixiaohui.cn.

;; AUTHORITY SECTION:
250.25.172.in-addr.arpa. 86400  IN      NS      ns.lixiaohui.cn.

;; ADDITIONAL SECTION:
ns.lixiaohui.cn.        86400   IN      A       172.25.250.10

;; Query time: 0 msec
;; SERVER: 172.25.250.11#53(172.25.250.11)
;; WHEN: Fri Aug 23 01:36:39 CST 2024
;; MSG SIZE  rcvd: 142
```

#### 触发主辅服务器同步

在主服务器更新区域文件，并序号加1

```textile
[root@servera ~]# cd /var/named/
[root@servera named]# vim lixiaohui.cn.zone
[root@servera named]# cat lixiaohui.cn.zone
$TTL 1D
lixiaohui.cn.   IN SOA  ns.lixiaohui.cn. 939958092.qq.com. (
                                        2       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        1D      IN      NS      ns.lixiaohui.cn.
ns      1D      IN      A       172.25.250.10
lxh     30      IN      A       192.168.8.100
lxh-2     30      IN      A       192.168.8.100
        30      IN      AAAA    2001:db8:2020::5300
@       20      IN      MX      10 mail.lixiaohui.cn.
mail    30      IN      A       172.25.250.10
support.online  1H      IN      A       172.25.250.10

[root@servera named]# systemctl reload named
```

在辅助服务器查看是否自动更新

```textile
[root@serverb ~]# cd /var/named/slaves/
[root@serverb slaves]# cat lixiaohui.cn.zone
$ORIGIN .
$TTL 86400      ; 1 day
lixiaohui.cn            IN SOA  ns.lixiaohui.cn. 939958092.qq.com. (
                                2          ; serial
                                86400      ; refresh (1 day)
                                3600       ; retry (1 hour)
                                604800     ; expire (1 week)
                                10800      ; minimum (3 hours)
                                )
                        NS      ns.lixiaohui.cn.
$TTL 20 ; 20 seconds
                        MX      10 mail.lixiaohui.cn.
$ORIGIN lixiaohui.cn.
$TTL 30 ; 30 seconds
lxh                     A       192.168.8.100
lxh-2                   A       192.168.8.100
                        AAAA    2001:db8:2020::5300
mail                    A       172.25.250.10
$TTL 86400      ; 1 day
ns                      A       172.25.250.10
$TTL 3600       ; 1 hour
support.online          A       172.25.250.10
```

确认日志消息正确

```text
Aug 23 04:05:50 serverb.lab.example.com named[864]: client @0x7fe308044c90 172.25.250.10#53572: received notify for zone 'lixiaohui.cn'
Aug 23 04:05:50 serverb.lab.example.com named[864]: zone lixiaohui.cn/IN: notify from 172.25.250.10#53572: serial 2
Aug 23 04:05:50 serverb.lab.example.com named[864]: zone lixiaohui.cn/IN: Transfer started.
Aug 23 04:05:50 serverb.lab.example.com named[864]: transfer of 'lixiaohui.cn/IN' from 172.25.250.10#53: connected using 172.25.250.11#46605
Aug 23 04:05:50 serverb.lab.example.com named[864]: zone lixiaohui.cn/IN: transferred serial 2
Aug 23 04:05:50 serverb.lab.example.com named[864]: transfer of 'lixiaohui.cn/IN' from 172.25.250.10#53: Transfer status: success
Aug 23 04:05:50 serverb.lab.example.com named[864]: transfer of 'lixiaohui.cn/IN' from 172.25.250.10#53: Transfer completed: 1 messages, 10 records, 289 bytes, 0.001 secs (289000 bytes/sec)
```

# 自动化执行名称服务器配置

详见本章练习
