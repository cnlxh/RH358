```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 章节目标

- 通过使用 Varnish 缓存静态 web 内容来提高网站性能。

- 通过将 HAProxy 用作负载平衡器和您的 Varnish 缓存前面的 HTTPS 终止符来提高网站性能。

- 使用 Ansible 自动化执行 HAProxy 和 Varnish 的配置。

# 使用 Varnish 缓存静态内容

## 描述 Varnish

<mark>Varnish 缓存</mark>是在此 web 服务器之前部署的<mark> web 加速器</mark>。Web 客户端不直接访问 web 服务器，而是联系 Varnish。Varnish 代表客户端从后端 web 服务器检索并返回请求的对象。Varnish 也会缓存这些对象，以便它可以为相同对象提供请求，而无需联系 web 服务器。通过这种方式，web 服务器具有更多可用的系统资源来管理其工作负载。

下图演示了在客户端第一次请求对象时，Varnish 如何从后端 web 服务器检索该对象。操作是缓存未命中，因为该对象最初不在缓存中。

1. Web 客户端请求对象。对于客户端，Varnish 的行为方式同 web 服务器。

2. Varnish 会确定对象是否已在其缓存中。

3. 在本示例中，对象不在缓存中。Varnish 从后端 web 服务器请求对象。

4. Web 服务器将对象发回到 Varnish。

5. Varnish 将对象存储在其缓存中。

6. Varnish 在给客户端的回复中包含对象。

![](https://gitee.com/cnlxh/rh358/raw/master/images/optimizeweb/optimizeweb-varnish-miss-01.svg)

下图显示了当对象在缓存中时，Varnish 在不联系后端 web 服务器的情况下为该对象提供内容。这种行为称为缓存命中。

1. Web 客户端从 Varnish 请求对象。

2. Varnish 会确定对象是否已在缓存中且未过期。

3. 由于对象位于缓存中，因此 Varnish 将其发回到客户端，而无需联系后端 web 服务器。

请注意，web 客户端不再直接访问 web 服务器，而是查询 Varnish。通常，您需要配置您的 DNS 服务器，使 URL 的主机部分指向 Varnish 服务器。

![](https://gitee.com/cnlxh/rh358/raw/master/images/optimizeweb/optimizeweb-varnish-hit-01.svg)

## 缓存行为

为了获得最佳性能， <mark>Varnish 将对象缓存在内存中</mark>。因此，重启 Varnish 服务会清除缓存。

默认情况下，Varnish 会缓存一切内容： HTML 文档；CSS 和 JavaScript 文件;镜像和动态内容（如 PHP 程序）。这条规则有几个例外：

- Varnish 从不缓存设置 HTTP cookie 的回复。Cookie 通常存储特定于客户端的信息，如会话 ID，且 Varnish 不得为面向多个客户端的回复提供内容。

- Varnish 不会缓存 HTTP PUT 或 POST 请求。

- Varnish 可识别回复中的 `Cache-Control` HTTP 标头。在后端 web 服务器上运行的 web 应用可以设置此标头，以指示将回复缓存多久（使用 `max-age` 值），或者完全不将其缓存（使用 `no-cache` 或 `no-store` 值）。

```textile
curl -I http://www.example.com/auth.php
...
Cache-Control: no-store
```

## TTL 和清除机制

Varnish 将对象保存在其缓存中达特定的持续时间（称为生存时间 TTL）。当 Varnish 接收到针对缓存中对象的请求时，它首先确定对象的 TTL 值。如果对象已过期，则 Varnish 会丢弃其副本并从后端 web 服务器获取新版本。这种机制可防止 Varnish 从其缓存中无限期地提供陈旧内容。

缓存中的每个对象都具有 TTL 值。当后端 web 服务器的回复中包含具有特定 TTL 的 `Cache-Control` 标头时， Varnish 会将该 TTL 用于对象。否则，Varnish 将对象 TTL 设置为两分钟的默认值。

可以通过 Varnish 配置更改该默认 TTL 值。大多提供静态内容的 Web 服务器（如静态 HTML 页面和不经常更改的图像）可受益于较高的 TTL。

## 为 Web 应用提供客户端 IP 地址

一些 web 应用使用传入请求的来源 IP 地址来识别其客户端。但是，由于 Varnish 在 web 服务器前面，应用会看到来自 Varnish 服务器的请求，而不是来自单个客户端。

为解决此问题，Varnish 为其转发到后端 web 服务器的所有请求<mark>自动添加了 `X-Forwarded-For` HTTP 标头</mark>。该标头包含作为原始请求来源的客户端系统的 IP 地址：

```text
X-Forwarded-For: 172.25.250.9
```

## 部署和管理Varnish

### 安装varnish

varnish-docs 软件包在 `/usr/share/doc/varnish-docs/html/` 目录下提供了 Varnish 文档。软件包不依赖于 varnish 软件包。您可以将其部署到您的工作站上，然后使用您的 web 浏览器访问 `/usr/share/doc/varnish-docs/html/index.html` 文件。

```bash
[root@serverb ~]# yum install varnish -y
[root@serverb ~]# systemctl enable --now varnish
```

### 准备后端服务器

我打算用serverc做业务的web服务器

```bash
[root@serverc ~]# yum install httpd -y
[root@serverc ~]# echo serverc host > /var/www/html/index.html
[root@serverc ~]# systemctl enable httpd --now
[root@serverc ~]# firewall-cmd --add-service=http
[root@serverc ~]# firewall-cmd --add-service=http --permanent
```

### 设置 Varnish 守护进程命令行参数

所有配置参数可以通过以下方式获得

```bash
[root@serverb ~]# man 1 varnishd
```

使用 `systemctl cat varnish` 命令来显示当前的服务详细信息。

``-a [IP]:PORT`` 选项指定 Varnish 侦听传入客户端请求的 TCP 端口

`-s` 选项指定对象缓存的存储后端。`malloc` 参数指示 Varnish 在内存中缓存对象。前面输出中的 256 MiB 大小表示 Varnish 可用于其缓存的最大内存大小。当内存已满时，Varnish 将驱逐旧对象。

```bash
[root@serverb ~]# systemctl cat varnish
# /usr/lib/systemd/system/varnish.service
[Unit]
Description=Varnish Cache, a high-performance HTTP accelerator
After=network-online.target

[Service]
Type=forking
KillMode=process

# Maximum number of open files (for ulimit -n)
LimitNOFILE=131072

# Locked shared memory - should suffice to lock the shared memory log
# (varnishd -l argument)
# Default log size is 80MB vsl + 1M vsm + header -> 82MB
# unit is bytes
LimitMEMLOCK=85983232

# Enable this to avoid "fork failed" on reload.
TasksMax=infinity

# Maximum size of the corefile.
LimitCORE=infinity

ExecStart=/usr/sbin/varnishd -a :6081 -f /etc/varnish/default.vcl -s malloc,256m
ExecReload=/usr/sbin/varnishreload

[Install]
WantedBy=multi-user.target
```

#### 配置服务端口

默认情况下，Varnish 侦听端口 6081。由于 web 客户端通过 Varnish 访问您的 web 应用，因此您通常要将 Varnish 配置为侦听端口 80，即默认的 HTTP 端口

不带任何值的第一个 `ExecStart` 参数将清除默认配置所定义的命令。否则，`systemd` 按顺序执行所有 `ExecStart` 命令。

```bash
[root@serverb ~]# mkdir /etc/systemd/system/varnish.service.d/
[root@serverb ~]# vim /etc/systemd/system/varnish.service.d/httpport.conf
[root@serverb ~]# cat /etc/systemd/system/varnish.service.d/httpport.conf
[Service]
ExecStart=
ExecStart=/usr/sbin/varnishd -a :80 -f /etc/varnish/default.vcl -s malloc,256m
```

#### 启动vanish服务

```bash
[root@serverb ~]# systemctl daemon-reload
[root@serverb ~]# systemctl restart varnish
[root@serverb ~]# firewall-cmd --permanent --add-service=http
[root@serverb ~]# firewall-cmd --reload
```

#### 配置vanish后端

/etc/varnish/default.vcl是标准的配置文件

```bash
backend default {
    .host = "172.25.250.12";
    .port = "80";
}
```

#### 验证vanish生效

```bash
[root@serverb ~]# curl http://serverb
serverc host
```

看到正常返回后端服务器，可以监控后端日志输出，并不断访问，发现后端没有再收到新请求，这表示请求不会到达后端 web 服务器。Varnish 从其缓存返回文档。

```bash
[root@serverc ~]# tail -f /var/log/httpd/access_log
172.25.250.11 - - [27/Aug/2024:01:02:41 +0800] "GET / HTTP/1.1" 200 13 "-" "curl/7.61.1"
```

#### 使用 VCL 配置缓存行为

跳过mp3和ogg后缀的文件

```text
sub vcl_recv {
    if (req.url ~ "\.(mp3|ogg)$") {
        return (pass);
}
```

设置TTL

Varnish 通过 beresp 对象提供回复详细信息。当后端 web 服务器尝试将对象 TTL 强制设为超过一天时，以下示例可将该时间缩减为两小时。

```TEXT
sub vcl_backend_response {
  if (beresp.ttl > 1d) {
    set beresp.ttl = 2h;
  }
}
```

#### 声明访问控制列表和配置清除请求

以下示例定义了 purge_allowed ACL，然后在 vcl_recv 子例程中使用该 ACL。

```text
acl purge_allowed {
  "localhost";
  "172.25.250.12";
  "172.25.250.10";
}

sub vcl_recv {
  if (req.method == "PURGE") {
    if (!client.ip ~ purge_allowed) {
      return(synth(405, "This address is not allowed to send PURGE requests."));
    }
    return (purge);
  }
}
```

测试VCL语法

```bash
[root@serverb ~]# varnishd -C -f /etc/varnish/default.vcl
```

#### 测试缓存清除

发现无法从serverb发起清除

```bash
[root@serverb ~]# curl -X PURGE http://serverb/index.html
<!DOCTYPE html>
<html>
  <head>
    <title>405 This address is not allowed to send PURGE requests.</title>
  </head>
  <body>
    <h1>Error 405 This address is not allowed to send PURGE requests.</h1>
    <p>This address is not allowed to send PURGE requests.</p>
    <h3>Guru Meditation:</h3>
    <p>XID: 11</p>
    <hr>
    <p>Varnish cache server</p>
  </body>
</html>
```

我们用servera试试

```bash
[root@servera ~]# curl -X PURGE http://serverb/index.html
<!DOCTYPE html>
<html>
  <head>
    <title>200 Purged</title>
  </head>
```

## Varnish 的故障排除和管理

为了获得最佳性能，`varnishd` 守护进程不会写入到日志文件。取而代之，它将其日志消息写入共享内存中的缓冲区中。可选的 `varnishncsa` 服务将监控该缓冲区，并将日志消息写入到 `/var/log/varnish/varnishncsa.log` 日志文件。

```bash
[root@serverb ~]# systemctl enable --now varnishncsa
[root@serverb ~]# cat /var/log/varnish/varnishncsa.log
```

## 通过命令行管理 Varnish

利用 `varnishadm` 命令行工具，您可以监控和重新配置 Varnish，而无需重新启动守护进程。您使用该工具执行的修改在服务重启后不会持久保留，但允许您临时修改实时配置。

```text
[root@serverb ~]# varnishadm param.show default_ttl
default_ttl
        Value is: 120.000 [seconds] (default)
        Minimum is: 0.000

        The TTL assigned to objects if neither the backend nor the VCL
        code assigns one.

        NB: This parameter is evaluated only when objects are created.
        To change it for all objects, restart or ban everything.


[root@serverb ~]# varnishadm param.set default_ttl 43200

[root@serverb ~]# varnishadm param.show default_ttl
default_ttl
        Value is: 43200.000 [seconds]
        Default is: 120.000
        Minimum is: 0.000

        The TTL assigned to objects if neither the backend nor the VCL
        code assigns one.

        NB: This parameter is evaluated only when objects are created.
        To change it for all objects, restart or ban everything.
```

# HAProxy 负载平衡与HTTPS卸载

HAProxy 是一种负载平衡器，您还可以将其配置为 HTTPS 卸载，以从后端 web 服务器中卸载 TLS 处理。您在这些 web 服务器前面部署 HAProxy。Web 客户端不直接访问 web 服务器，而是联系 HAProxy。HAProxy 代表这些客户端从后端 web 服务器检索并返回请求的对象。

## 负载均衡

以下架构说明了 HAProxy 如何用作一个负载平衡器。

Web 客户端通过 HAProxy 访问 web 服务器。然后，HAProxy 在后端 web 服务器之间分发请求。这样，HAProxy 可以在 web 场中的服务器之间进行负载分布。HAProxy 还可以监控后端服务器，并停止向未响应的服务器发送请求。

![](https://gitee.com/cnlxh/rh358/raw/master/images/optimizeweb/optimizeweb-haproxy-lecture-load.svg)

HAProxy 支持多种负载平衡算法，以在后端 web 服务器之间分发请求。

`roundrobin`

此算法依次向每个 web 服务器发送请求。当 web 场提供静态内容或后端服务器上运行的 web 应用将用户会话保留在全局存储中时，您可以使用此算法。

`source`

此算法使用客户端的 IP 地址来选择后端服务器。通过这种算法，HAProxy 始终将客户端请求转发到同一服务器。当后端服务器上运行的 web 应用无法全局管理会话时，您可以使用此算法。

HAProxy 支持用于不同用例的其他算法。有关完整的列表，请参阅 `/usr/share/doc/haproxy/configuration.txt` 文档文件。

### 安装Haproxy

```bash
[root@servera ~]# yum install haproxy -y
```

### 启动 haproxy

```bash
[root@servera ~]# systemctl enable --now haproxy
```

### 准备后端服务

在serverc和serverd上部署两个web服务，用于稍后做负载均衡等后端服务器

```bash
[root@serverc ~]# yum install httpd -y
[root@serverc ~]# echo serverc > /var/www/html/index.html
[root@serverc ~]# systemctl enable httpd --now
[root@serverc ~]# firewall-cmd --add-service=http --permanent
[root@serverc ~]# firewall-cmd --reload
```

```bash
[root@serverd ~]# yum install httpd -y
[root@serverd ~]# echo serverd > /var/www/html/index.html
[root@serverd ~]# systemctl enable httpd --now
[root@serverd ~]# firewall-cmd --add-service=http --permanent
[root@serverd ~]# firewall-cmd --reload
```

### 配置文件解析

HAProxy 使用 `/etc/haproxy/haproxy.cfg` 配置文件。此文件将配置参数分为以下部分：

`global`

`global` 部分定义与 `haproxy` 进程相关的参数，如用于守护进程的 Linux 用户帐户。您通常不必更改该部分中的任何内容。

`defaults`

`defaults` 部分为其他部分的参数设置默认值。您可以在后续部分中覆盖这些参数。

`frontend name`

`frontend` 部分配置 HAProxy 中面向客户端的一侧。在此部分中，您将指定要侦听传入请求的网络端口。这也是将 HAProxy 配置为 HTTPS 终止符的位置。

`backend name`

`backend` 部分列出了所有后端 web 服务器。在此部分中，您将指定负载平衡算法和用于检查后端服务器运行状况的机制。

`listen name`

`listen` 部分支持以更紧凑的方式来定义前端和后端配置。当您的设置比较简单时，您可以使用一个`侦听`部分，而不是`前端`和`后端`部分。

### 负载均衡案例

下面的配置文件中定义了一个前端（frontend）和一个后端（backend）：

- `frontend erp` 定义了一个名为 `erp` 的前端，它绑定到所有IP地址的80端口上。
- `default_backend erpserver` 指定了当请求到达 `erp` 前端时，默认应该转发到的后端是 `erpserver`。
- `backend erpserver` 定义了一个名为 `erpserver` 的后端，它使用了轮询（roundrobin）算法来平衡负载。
- 后端 `erpserver` 包含了两个服务器：`erpserver1` 和 `erpserver2`，分别位于IP地址 `172.25.250.12` 和 `172.25.250.13`，监听80端口，并且每1秒检查一次服务器状态。

```textile
[root@servera ~]# vim /etc/haproxy/haproxy.cfg

frontend erp
    bind *:80
    default_backend erpserver
backend erpserver
    balance     roundrobin
    server erpserver1 172.25.250.12:80 check inter 1s
    server erpserver2 172.25.250.13:80 check inter 1s
```

如上面配置文件解析所说，可以改成下面的listen，效果是一样的

```text
listen erp
    bind *:80
    mode http
    balance roundrobin
    server erpserver1 172.25.250.12:80 check inter 1s
    server erpserver2 172.25.250.13:80 check inter 1s
```

重启服务后测试，的确有负载均衡效果

```bash
[root@servera ~]# systemctl restart haproxy
[root@servera ~]# curl localhost
serverc
[root@servera ~]# curl localhost
serverd
```

## HTTPS卸载

以下架构说明了如何在 web 服务器前面使用 HAProxy 作为 HTTPS 终止符。

1. Web 客户端使用 HTTPS 从 HAProxy 请求 web 文档。对于客户端，HAProxy 的行为方式同 web 服务器。

2. HAProxy 使用其 TLS 证书和私钥来解密 HTTPS 流量。

3. HAProxy 以纯文本形式将请求转发到后端 web 服务器。

4. 后端 web 服务器将其回复发送到 HAProxy。

5. HAProxy 使用其 TLS 证书和私钥来加密回复，然后将它发回到 web 客户端。

![](https://gitee.com/cnlxh/rh358/raw/master/images/optimizeweb/optimizeweb-haproxy-lecture-ssl.svg)

### HTTPS卸载案例

HAProxy 需要在单个文件中使用证书和密钥，而且我们复用负载均衡章节提供的后端服务器，先来生成所需的证书

#### 生成TLS证书

##### 生成根证书

```bash
openssl genrsa -out /etc/pki/tls/private/xiaohuiroot.key 4096

openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=xiaohuiroot" \
-key /etc/pki/tls/private/xiaohuiroot.key \
-out /etc/pki/ca-trust/source/anchors/xiaohuiroot.crt
```

##### 信任根证书

```bash
update-ca-trust
```

##### 生成证书请求

```bash
openssl genrsa -out /etc/pki/tls/private/lixiaohui.key 4096
openssl req -sha512 -new \
-subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=www.lixiaohui.cn" \
-key /etc/pki/tls/private/lixiaohui.key \
-out lixiaohui.csr
```

##### 签发证书

给www.lixiaohui.com网站签发证书

```bash
openssl x509 -req -in lixiaohui.csr \
-CA /etc/pki/ca-trust/source/anchors/xiaohuiroot.crt \
-CAkey /etc/pki/tls/private/xiaohuiroot.key -CAcreateserial \
-out /etc/pki/tls/certs/lixiaohui.crt \
-days 3650
```

##### 查询证书详情

```bash
[root@servera ~]# openssl x509 -in /etc/pki/tls/certs/lixiaohui.crt -noout -text
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number:
            1f:1b:c9:dc:15:47:28:c8:ca:9d:0e:0c:7f:df:6a:f4:ac:d1:cb:73
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = CN, ST = Shanghai, L = Shanghai, O = Company, OU = SH, CN = xiaohuiroot
        Validity
            Not Before: Aug 26 15:08:05 2024 GMT
            Not After : Aug 24 15:08:05 2034 GMT
        Subject: C = CN, ST = Shanghai, L = Shanghai, O = Company, OU = SH, CN = www.lixiaohui.cn
```

##### 生成Haproxy证书文件

HAProxy 需要在单个文件中使用证书和密钥

```bash
[root@servera ~]# cat /etc/pki/tls/certs/lixiaohui.crt \
/etc/pki/tls/private/lixiaohui.key > /etc/pki/tls/certs/haproxy.pem
```

#### 创建配置文件

配置的简要概述：

- `frontend https`: 定义了一个名为"https"的前端，用于处理进入的HTTPS请求。
- `bind *:443 ssl crt /etc/pki/tls/certs/haproxy.pem`: 监听所有IP地址的443端口，启用SSL，并指定了SSL证书文件的位置。
- `http-request add-header X-Forwarded-Proto https`: 向HTTP请求添加`X-Forwarded-Proto`头，其值为"https"，表明请求是通过HTTPS协议进行的。
- `http-request set-header X-Forwarded-For %[src]`: 向HTTP请求添加`X-Forwarded-For`头，其值为客户端的源IP地址。
- `default_backend erpserver`: 指定如果前端收到的请求没有匹配到特定的后端规则，则默认将请求转发到名为"erpserver"的后端。

```text
frontend https
    bind *:443 ssl crt /etc/pki/tls/certs/haproxy.pem
    http-request add-header X-Forwarded-Proto https
    http-request set-header X-Forwarded-For %[src]
    default_backend erpserver
backend erpserver
    balance     roundrobin
    server erpserver1 172.25.250.12:80 check inter 1s
    server erpserver2 172.25.250.13:80 check inter 1s
```

找外部测试，需要开通防火墙

```bash
[root@servera ~]# firewall-cmd --add-service=http --add-service=https --permanent
[root@servera ~]# firewall-cmd --reload
```

找另一个机器添加解析并测试

成功访问到后端

```bash
[root@foundation0 ~]# echo 172.25.250.10 www.lixiaohui.cn >> /etc/hosts
[root@foundation0 ~]# curl https://www.lixiaohui.cn -k
serverc
[root@foundation0 ~]# curl https://www.lixiaohui.cn -k
serverd
```

## Haproxy中的SELINUX

SELinux 允许 HAProxy 绑定并连接到 `http_cache_port_t` 到 `http_port_t` 端口类型

```bash
[root@servera ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
```

若要让 HAProxy 侦听或连接任何其他端口，请将 `haproxy_connect_any` SELinux 布尔值设置为 `on`。

```bash
[root@servera ~]# setsebool -P haproxy_connect_any on
```

## 监控HAProxy

HAProxy 提供 HTML 统计报告页面。在该页面中，您可以检查和监控安装的行为。如果您激活了 `admin` 模式，那么您可以临时禁用后端服务器，以便它不再接收流量。此功能可用于维护目的。

默认配置将禁用统计报告页面。若要激活它，请在 `frontend` 部分中添加以下指令。

```bash
frontend stat-page
  bind *:8080
  stats enable
  stats uri /haproxystats
  stats auth operator1:redhat123
  stats admin if TRUE
```

重启服务后，可以打开浏览器https://haproxyip/haproxystats

## 在 Varnish 前配置 HAProxy

HAProxy 使用 HTTP PROXY 协议进行其与 Varnish 的通信。该协议传输来自 web 客户端的原始请求，以及特定的代理数据，如客户端的 IP 地址。通过此信息，Varnish 会自动将正确的 X-Forwarded-For HTTP 标头插入到发送至后端 web 服务器的请求中。

下图显示了 HAProxy 终止 HTTPS 连接，然后对发送到 Varnish 服务器的请求进行负载平衡。

![](https://gitee.com/cnlxh/rh358/raw/master/images/optimizeweb/optimizeweb-haproxy-lecture-varnish.svg)

**为 PROXY 协议配置 HAProxy**

在 backend 部分下，设置 send-proxy-v2 选项，以使用 Varnish 激活 HTTP PROXY 协议。

由于 HTTP PROXY 代理协议管理 X-Forwarded-For HTTP 标头，因此请确保从您的配置中删除 defaults 和 backend 部分下的 option forwardfor 指令。

```text
backend my-varnish
  server varnish 192.168.0.42:6081 send-proxy-v2 check inter 1s
```

**为 PROXY 协议配置 Varnish**

对于 Varnish，在 `varnishd` 命令的 `-a` 选项的末尾添加 `PROXY` 关键字。为此，可通过创建一个置入文件来重新配置 `varnish` `systemd` 服务。

```bash
[root@serverb ~]# mkdir /etc/systemd/system/varnish.service.d/
[root@serverb ~]# vim /etc/systemd/system/varnish.service.d/httpport.conf
[Service]
ExecStart=
ExecStart=/usr/sbin/varnishd -a :6081,PROXY -f /etc/varnish/default.vcl -s malloc,256m
[root@serverb ~]# systemctl daemon-reload
[root@serverb ~]# systemctl restart varnish
```

# 自动化执行 Web 服务优化

详见教材
