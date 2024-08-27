```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 章节目标

- 使用 Apache HTTPD 配置基本的 web 服务器。

- 配置 Apache HTTPD 以提供基于 IP 和基于名称的虚拟主机。

- 配置 Apache HTTPD，以提供使用 TLS 支持 HTTPS 协议的虚拟主机。

- 配置 web 服务器，以使用 Nginx 向多个虚拟主机提供 HTTPS 访问。

- 使用 Ansible 自动配置 Apache HTTPD 和 Nginx web 服务器。

# 使用 Apache HTTPD 配置基本 Web 服务器

## 安装 Apache HTTP 服务器

Apache HTTP 服务器（有时被其用户非官方地称为“Apache”）提供了完全可配置且可扩展的 web 服务器。其功能可以通过<mark>模块扩展</mark>，即插入到 web 服务器的框架中并修改其功能的小段代码。Apache HTTP 服务器由来自 AppStream 存储库的 httpd 软件包提供

```bash
[root@servera ~]# yum install httpd -y
```

<mark>httpd-manual 软件包</mark>将在 Web 服务器上的 `http://localhost/manual` 处安装 Apache HTTP 服务器的文档。

## 配置 Apache HTTP 服务器

Apache HTTP 服务器从以下位置读取其配置：

- `/etc/httpd/conf/httpd.conf`，即主配置文件。

- `/etc/httpd/conf.d/`，它提供 `httpd.conf` 中包含的补充配置文件（如果文件名以 `.conf` 结尾）。

- `/etc/httpd/conf.modules.d/`, 它提供用于动态加载 Apache 模块的补充配置文件（如果文件名以 `.conf` 结尾）。

## 主配置文件参数解析

```textile
[root@servera ~]# grep -v -e ^$ -e ^# -e ^.*# /etc/httpd/conf/httpd.conf
ServerRoot "/etc/httpd"
Listen 80
Include conf.modules.d/*.conf
User apache
Group apache
ServerAdmin root@localhost
<Directory />
    AllowOverride none
    Require all denied
</Directory>
DocumentRoot "/var/www/html"
<Directory "/var/www">
    AllowOverride None
    Require all granted
</Directory>
<Directory "/var/www/html">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
<IfModule dir_module>
    DirectoryIndex index.html
</IfModule>
<Files ".ht*">
    Require all denied
</Files>
```

主要参数的解释：

- `ServerRoot "/etc/httpd"`：定义了Apache服务器的根目录，所有相对路径都将从这个目录开始。

- `Listen 80`：指定了Apache服务器监听的端口号，这里是80端口，也就是HTTP服务的默认端口。

- `Include conf.modules.d/*.conf`：包含`conf.modules.d`目录下的所有配置文件，这些文件通常用于加载Apache的模块。

- `User apache`：定义了Apache工作进程运行时使用的系统用户。出于安全考虑，通常不会使用root用户。

- `Group apache`：定义了Apache工作进程运行时使用的系统用户组。

- `ServerAdmin root@localhost`：定义了服务器管理员的电子邮件地址，用于接收错误报告和重要通知。

- `<Directory />`：定义了对服务器根目录（通常是`/`）的访问规则。`AllowOverride none`禁止目录中的`.htaccess`文件覆盖服务器级别的配置。`Require all denied`拒绝所有访问。

- `DocumentRoot "/var/www/html"`：定义了Apache服务的文档根目录，这是存放网站内容的默认目录。

- `<Directory "/var/www">`：定义了对`/var/www`目录的访问规则。`AllowOverride None`和`Require all granted`分别表示不允许`.htaccess`文件覆盖配置和允许所有访问。

- `<Directory "/var/www/html">`：定义了对文档根目录`/var/www/html`的访问规则。`Options Indexes FollowSymLinks`启用了索引选项，允许Apache跟随符号链接。`AllowOverride None`和`Require all granted`的含义同上。

- `<IfModule dir_module>`：如果`dir_module`模块被加载，下面的指令将被执行。`DirectoryIndex index.html`定义了默认的索引文件名，当访问一个目录时，Apache会尝试提供这个文件的内容。

- `<Files ".ht*">`：定义了对所有以`.ht`开头的文件（如`.htaccess`、`.htpasswd`等）的访问规则。`Require all denied`表示禁止访问这些文件。

## 默认日志格式

```text
ErrorLog "logs/error_log"
LogLevel warn
<IfModule log_config_module>
    LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
    LogFormat "%h %l %u %t \"%r\" %>s %b" common
    <IfModule logio_module>
      LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %I %O" combinedio
    </IfModule>
    CustomLog "logs/access_log" combined
</IfModule>
```

- `ErrorLog "logs/error_log"`：指定了错误日志文件的存放路径和文件名。所有Apache服务器的错误信息将被记录在这个文件中，路径是相对于`ServerRoot`的。

- `LogLevel warn`：设置了日志记录的级别为`warn`。这意味着只有警告（warn）及以上级别的消息（如错误error和紧急emergency）会被记录到错误日志中。

- `<IfModule log_config_module>`：这是一个条件性指令，只有在加载了`log_config_module`模块时，下面的配置才会生效。

- `LogFormat`：定义了日志的格式。这里有三种格式定义：
  
  - `"%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined`：这是一个组合格式，包含了客户端主机名（%h）、错误的日志ID（%l，通常为空）、用户身份（%u）、访问时间（%t）、请求的URL（"%r"）、HTTP状态码（%>s）、响应体的字节数（%b）、引用页（%{Referer}i）、用户代理（%{User-Agent}i）。
  - `"%h %l %u %t \"%r\" %>s %b" common`：这是一个更常见的日志格式，不包括引用页和用户代理信息。
  - `"%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %I %O" combinedio`：这是另一个组合格式，额外包括了接收到的字节数（%I）和发送出去的字节数（%O）。这个格式只在加载了`logio_module`模块时可用。

- `<IfModule logio_module>`：这是另一个条件性指令，用于加载`logio_module`模块，并使用上面定义的`combinedio`日志格式。

- `CustomLog "logs/access_log" combined`：指定了访问日志文件的存放路径和文件名，以及使用的日志格式。这里使用了`combined`格式，即上面定义的包含大量信息的组合日志格式。

## 配置文件语法检测

```bash
[root@servera ~]# apachectl configtest
Syntax OK
```

## 提供简单的网站

如果只是需要简单提供内容，只需要把网页放到`/var/www/html`即可

```bash
[root@servera ~]# echo hello lixiaohui > /var/www/html/index.html
[root@servera ~]# systemctl enable httpd --now
```

访问测试

```bash
[root@servera ~]# curl http://localhost
hello lixiaohui
```

## 开通防火墙

```bash
[root@servera ~]# firewall-cmd --permanent --add-service=http --add-service=https
[root@servera ~]# firewall-cmd --reload
```

# 配置HTTPD虚拟主机

虚拟主机允许单个 web 服务器为多个网站提供内容。Web 服务器可以根据客户端所连接的服务器的特定 IP 地址或客户端 HTTP 请求中的站点名称，使用不同的配置设置来提供不同的内容。

使用 `<VirtualHost>` 块指令覆盖虚拟主机的主配置文件中的设置。每个虚拟主机都有自己的块。

良好的做法是在 `/etc/httpd/conf.d/` 中的以 `.conf` 结尾的单独配置文件中配置虚拟主机。这样可以更加轻松地部署和更新主机，而不会干扰 web 服务器配置的其他部分。

## 虚拟主机案例

别忘了httpd-manual提供的帮助

以下示例 `/etc/httpd/conf.d/site1.conf` 为 `www.lixiaohui.cn` 设置了一个虚拟主机

网页将放在`/srv/site1/www`

```text
[root@servera ~]# cat /etc/httpd/conf.d/site1.conf
<Directory /srv/site1/www>
  Require all granted
  AllowOverride None
</Directory>

<VirtualHost 172.25.250.10:80>
  DocumentRoot /srv/site1/www
  ServerName www.lixiaohui.cn
  ErrorLog "logs/site1_error_log"
  CustomLog "logs/site1_access_log" combined
</VirtualHost>
```

```bash
[root@servera ~]# mkdir -p /srv/site1/www
[root@servera ~]# echo www.lixiaohui.cn content > /srv/site1/www/index.html
[root@servera ~]# semanage fcontext -a -t httpd_sys_content_t '/srv(/.*)?'
[root@servera ~]# restorecon -RvF /srv
[root@servera ~]# chown apache:apache /srv -R
[root@servera ~]# systemctl restart httpd
[root@servera ~]# echo 172.25.250.10 www.lixiaohui.cn >> /etc/hosts
[root@servera ~]# curl www.lixiaohui.cn
www.lixiaohui.cn content
```

```bash
[root@servera ~]# cat /var/log/httpd/site1_access_log
172.25.250.10 - - [26/Aug/2024:21:54:44 +0800] "GET / HTTP/1.1" 403 3985 "-" "curl/7.61.1"
172.25.250.10 - - [26/Aug/2024:21:58:07 +0800] "GET / HTTP/1.1" 200 26 "-" "curl/7.61.1"
172.25.250.10 - - [26/Aug/2024:21:59:45 +0800] "GET / HTTP/1.1" 200 26 "-" "curl/7.61.1"
```

## 控制虚拟主机的选择

如果每个虚拟主机都配置为具有自己的专用 IP 地址，则它被称为*基于 IP 的虚拟主机*。

如果多个虚拟主机共享相同的 IP 地址，则确定要将流量发送到哪一个虚拟主机的唯一方式是检查客户端的 HTTP 请求和虚拟主机的 `ServerName` 和 `ServerAlias` 指令。此配置中的虚拟主机有时称为*基于名称的虚拟主机*。

`<VirtualHost>` 指令的 IP 地址部分可以替换为星号 (*) 通配符，以匹配 web 服务器上的所有地址。

按如下所示选择用于处理客户端请求的虚拟主机块：

- 请求到达时，`httpd` 首先尝试将传入连接的地址和端口与设置了显式 IP 地址和端口的虚拟主机匹配。如果这些匹配项失败，那么将检查具有通配符 IP 地址的虚拟主机。如果正好有一个虚拟主机定义匹配，则使用基于 IP 的虚拟主机。

- 如果有多个虚拟主机定义匹配，则这是基于名称的虚拟主机。如果客户端的 HTTP 请求包含 `Host:` 标头（用于标识客户端尝试访问的服务器），则会在该列表中搜索*第一个*由配置文件加载的、在该标头中具有名称作为其 `ServerName` 或 `ServerAlias` 的虚拟主机，并且使用该虚拟主机。当您在 `/etc/httpd/conf.d` 中有多个包含虚拟主机的 `*.conf` 文件时，它们将以系统的排序顺序加载（通常按文件名字母顺序排列）。

- 如果有一个虚拟主机定义的 `<VirtualHost>` 指令的 IP 地址部分已替换为具有匹配端口的 `_default_`，并且没有其他虚拟主机匹配，则使用该虚拟主机。

- 如果没有虚拟主机定义匹配，则“主”服务器配置将为请求提供服务。

## 访问控制部分

在apache的配置文件中，`Directory`中负责授权和认证部分，`RequireAll`、`RequireAny`和`Require`是用于定义访问控制语句的指令，它们决定了如何评估多个要求条件以确定访问权限。

以下是这些指令的简要说明：

1. **Require** (使用IP地址):
   
   - `Require ip 192.168.1.100` 将只允许IP地址为192.168.1.100的客户端访问。

2. **RequireAll** (使用IP网段):
   
   - `RequireAll ip 192.168.1.0/24` 要求客户端的IP地址必须在192.168.1.0到192.168.1.255的范围内（即整个192.168.1.x的子网）。

3. **RequireAny** (结合IP地址和网段):
   
   - `RequireAny ip 192.168.1.100 ip 10.0.0.0/8` 允许IP地址为192.168.1.100或任何属于10.0.0.0/8网段（即10.x.x.x）的客户端访问。

只有IP地址为172.25.250.0/24的客户端才能访问/srv/site1/www目录下的资源。其他IP地址的访问都会被拒绝。

```text
<Directory "/srv/site1/www">
    <RequireAll>
        Require ip 172.25.250.0/24
    </RequireAll>
</Directory>
<VirtualHost *:80>
  DocumentRoot /srv/site1/www
  ServerName www.lixiaohui.cn
  ErrorLog "logs/site1_error_log"
  CustomLog "logs/site1_access_log" combined
</VirtualHost>
```

只要客户端的IP地址在172.25.250.0/24网段内或者为172.25.250.10，就可以访问/srv/site1/www目录。其他IP地址的访问都会被拒绝。

```text
<Directory "/srv/site1/www">
    <RequireAny>
        Require ip 172.25.250.0/24
        Require ip 172.25.250.10
    </RequireAny>
</Directory>

<VirtualHost 172.25.250.13:80>
  DocumentRoot /srv/site1/www
  ServerName www.lixiaohui.cn
  ErrorLog "logs/site1_error_log"
  CustomLog "logs/site1_access_log" combined
</VirtualHost>

```

IP地址在172.25.250.10网段内的客户端会被拒绝访问/srv/site1/www目录。其他IP地址的访问则被允许。

```text
<Directory "/srv/site1/www">
    <RequireAll>
        Require all granted
        Require not ip 172.25.250.10
    </RequireAll>
</Directory>

<VirtualHost 172.25.250.13:80>
  DocumentRoot /srv/site1/www
  ServerName www.lixiaohui.cn
  ErrorLog "logs/site1_error_log"
  CustomLog "logs/site1_access_log" combined
</VirtualHost>
```

# 配置 HTTPS

## 描述 TLS 协议

*传输层安全性 (TLS)* 是 HTTPS 使用的协议，用于保护 Web 流量免受对其真实性、机密性和完整性的攻击。

TLS 使用*公钥加密*来设置安全的 TLS 会话。一个密钥可以加密的内容，只有其匹配密钥可以解密。

每一服务器必须安装有 TLS *证书*。此证书包含有关证书所属服务器、过期时间和作为密钥对一半的*公钥*的信息。它也由*证书颁发机构 (CA)* 进行数字签名，此签名可用于验证服务器证书的真实性。服务器还必须安装有与证书的公钥匹配的私钥。

当客户端连接到服务器并请求 TLS 会话时，它们将执行初始*握手*，以同意两者都可支持的一组加密密码。服务器为客户端提供其证书，而客户端则使用证书中的信息和 CA 的签名进行验证。然后，客户端使用公钥与服务器进行安全通信，并与它配合使用以设置可用于快速加密和解密数据的更快*会话密钥*，然后用于实际的安全会话。

## 生成TLS证书

### 生成根证书

```bash
openssl genrsa -out /etc/pki/tls/private/xiaohuiroot.key 4096

openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=xiaohuiroot" \
-key /etc/pki/tls/private/xiaohuiroot.key \
-out /etc/pki/ca-trust/source/anchors/xiaohuiroot.crt
```

### 信任根证书

```bash
update-ca-trust
```

### 生成证书请求

```bash
openssl genrsa -out /etc/pki/tls/private/lixiaohui.key 4096
openssl req -sha512 -new \
-subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=www.lixiaohui.cn" \
-key /etc/pki/tls/private/lixiaohui.key \
-out lixiaohui.csr
```

### 签发证书

给www.lixiaohui.com网站签发证书

```bash
openssl x509 -req -in lixiaohui.csr \
-CA /etc/pki/ca-trust/source/anchors/xiaohuiroot.crt \
-CAkey /etc/pki/tls/private/xiaohuiroot.key -CAcreateserial \
-out /etc/pki/tls/certs/lixiaohui.crt \
-days 3650
```

### 查询证书详情

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

## TLS 虚拟主机

服务器需要安装mod_ssl 软件包扩展模块才能激活 TLS 支持，此软件包将为侦听端口 443/TCP 的默认虚拟主机自动启用 `httpd`。此默认虚拟主机是在 `/etc/httpd/conf.d/ssl.conf` 中配置的。

```bash
[root@servera ~]# yum install mod_ssl -y
```

配置文件如下：

```bash
[root@servera ~]# cat /etc/httpd/conf.d/site1.conf
<Directory /srv/site1/www>
  Require all granted
  AllowOverride None
</Directory>

<VirtualHost 172.25.250.10:80>
  DocumentRoot /srv/site1/www
  ServerName www.lixiaohui.cn
  ErrorLog "logs/site1_error_log"
  CustomLog "logs/site1_access_log" combined
</VirtualHost>

<VirtualHost *:443>
  DocumentRoot /srv/site1/www
  ServerName www.lixiaohui.cn
  SSLEngine on
  SSLCertificateFile /etc/pki/tls/certs/lixiaohui.crt
  SSLCertificateKeyFile /etc/pki/tls/private/lixiaohui.key
  ErrorLog "logs/site1_error_log"
  CustomLog "logs/site1_access_log" combined
</VirtualHost>
```

重启服务并验证https访问

```bash
[root@servera ~]# systemctl restart httpd
[root@servera ~]# curl https://www.lixiaohui.cn
www.lixiaohui.cn content
```

## HTTP重定向到HTTPS

自动将通过 HTTP 连接的任何客户端重定向到HTTPS

调整配置文件如下：

```text
<Directory /srv/site1/www>
  Require all granted
  AllowOverride None
</Directory>

<VirtualHost 172.25.250.10:80>
  ServerName www.lixiaohui.cn
  Redirect "/" "https://www.lixaohui.cn"
</VirtualHost>

<VirtualHost *:443>
  DocumentRoot /srv/site1/www
  ServerName www.lixiaohui.cn
  SSLEngine on
  SSLCertificateFile /etc/pki/tls/certs/lixiaohui.crt
  SSLCertificateKeyFile /etc/pki/tls/private/lixiaohui.key
  ErrorLog "logs/site1_error_log"
  CustomLog "logs/site1_access_log" combined
</VirtualHost>
```

重启服务并验证

```bash
[root@servera ~]# systemctl restart httpd
[root@servera ~]# curl http://www.lixiaohui.cn
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href="https://www.lixiaohui.cn">here</a>.</p>
</body></html>
[root@servera ~]# curl http://www.lixiaohui.cn -L
www.lixiaohui.cn content
```

# 使用 Nginx 配置 Web 服务器

Nginx 是 Apache HTTP 服务器的一种替代方案，也是互联网上使用最广泛的 web 服务器之一。其设计目标之一是提供比 Apache 更好的性能，并处理更多的并发请求。它也经常用作反向缓存代理和负载平衡器。

## 安装nginx

```bash
[root@servera ~]# yum module list nginx
Last metadata expiration check: 0:36:27 ago on Mon 26 Aug 2024 10:57:45 PM CST.
Red Hat Enterprise Linux 8.1 AppStream (dvd)
Name                     Stream                      Profiles                     Summary
nginx                    1.14 [d]                    common [d]                   nginx webserver
nginx                    1.16                        common [d]                   nginx webserver

Hint: [d]efault, [e]nabled, [x]disabled, [i]nstalled
```

```bash
[root@servera ~]# yum module install nginx:1.16 -y
[root@servera ~]# nginx -v
nginx version: nginx/1.16.1
```

## 配置 Nginx

Nginx 的默认配置根目录是 `/etc/nginx` 目录。它的主配置文件为 `/etc/nginx/nginx.conf`。此文件包含 web 服务器操作的全局设置，包括主网站的默认 `server` 块。它还会从 `/etc/nginx/conf.d` 加载名称以 `.conf` 结尾的其他配置文件。

在配置的顶级，四个称为*上下文*的特殊块将指令组合在一起，用于管理不同类型的流量：

- `events`：用于常规连接处理

- `http`：用于 HTTP 流量

- `mail`：用于电子邮件流量

- `stream`：用于 TCP 和 UDP 流量

在本课程中，最重要的上下文是 `http` 上下文。`/etc/nginx/conf.d` 中的 `.conf` 文件将加载到该上下文中。

### 配置虚拟服务器

先关掉httpd，避免端口冲突

```bash
[root@servera ~]# systemctl disable httpd --now
```

生成nginx配置文件，并复用网页

服务器名称可以是准确的名称，可以包含通配符，甚至正则表达式都可以

根目录中的内容必须可由运行 nginx 进程的 nginx 用户读取。Nginx 使用与 Apache HTTP 服务器相同的 SELinux 上下文。

```text
[root@servera ~]# vim /etc/nginx/conf.d/lxh.conf
server {
    listen 80;
    server_name www.lixiaohui.cn www.zhangsan.com;
    access_log /var/log/nginx/www.lixiaohui.cn_access.log main;
    error_log /var/log/nginx/www.lixiaohui.cn_error.log error;
    location / {
        root /srv/site1/www/;
        index index.html;
    }
}
```

### 测试配置文件

有两个命令可用于验证您的配置文件中的错误。

`nginx -t` 将检查您的配置文件是否有语法问题，并尝试打开配置文件所引用的任何文件。 它会在退出时提供一份简短报告。

`nginx -T` 将执行相同的操作，但在退出时，它还会将配置文件转储到标准输出。

每当您对配置文件进行更改时，您都需要重新加载 `nginx` 服务，然后更改才会生效。

启动服务并访问

```bash
[root@servera ~]# systemctl enable nginx --now
```

可以成功访问

```bash
[root@servera ~]# systemctl enable nginx --now
[root@servera ~]# echo 172.25.250.10 www.zhangsan.com >> /etc/hosts
[root@servera ~]# echo 172.25.250.10 www.lixiaohui.cn >> /etc/hosts
[root@servera ~]# curl www.lixiaohui.cn
www.lixiaohui.cn content
[root@servera ~]# curl www.zhangsan.com
www.lixiaohui.cn content
``
```

### 配置Nginx TLS

```bash
[root@servera ~]# vim /etc/nginx/conf.d/lxh.conf

server {
    listen 80;
    server_name www.lixiaohui.cn www.zhangsan.com;
    access_log /var/log/nginx/www.lixiaohui.cn_access.log main;
    error_log /var/log/nginx/www.lixiaohui.cn_error.log error;
    location / {
        root /srv/site1/www/;
        index index.html;
    }
}
server {
    listen 443 ssl;
    server_name www.lixiaohui.cn www.zhangsan.com;
    ssl_certificate /etc/pki/tls/certs/lixiaohui.crt;
    ssl_certificate_key /etc/pki/tls/private/lixiaohui.key;
    access_log /var/log/nginx/www.lixiaohui.cn_access.log main;
    error_log /var/log/nginx/www.lixiaohui.cn_error.log error;
    location / {
        root /srv/site1/www/;
        index index.html;

    }
}
```

重启服务并验证

```bash
[root@servera ~]# systemctl restart nginx
[root@servera ~]# curl https://www.lixiaohui.cn
www.lixiaohui.cn content
```

### 配置http到https重定向

配置文件

```text
server {
    listen 80;
    server_name www.lixiaohui.cn www.zhangsan.com;
    access_log /var/log/nginx/www.lixiaohui.cn_access.log main;
    error_log /var/log/nginx/www.lixiaohui.cn_error.log error;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name www.lixiaohui.cn www.zhangsan.com;
    ssl_certificate /etc/pki/tls/certs/lixiaohui.crt;
    ssl_certificate_key /etc/pki/tls/private/lixiaohui.key;
    access_log /var/log/nginx/www.lixiaohui.cn_access.log main;
    error_log /var/log/nginx/www.lixiaohui.cn_error.log error;
    location / {
        root /srv/site1/www/;
        index index.html;

    }
}
```

```bash
[root@servera ~]# curl http://www.lixiaohui.cn
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.16.1</center>
</body>
</html>
[root@servera ~]# curl http://www.lixiaohui.cn -L
www.lixiaohui.cn content
```

# 自动化执行 Web 服务器配置

详见教材
