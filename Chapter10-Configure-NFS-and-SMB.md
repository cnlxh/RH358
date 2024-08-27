```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 章节目标

- 使用 NFS 协议导出网络客户端的文件系统，基于来源 IP 地址限制访问权限。

- 使用 SMB 协议，将文件系统共享到网络客户端。

- 使用 NFS 和 SMB 自动配置网络文件系统。

# 导出 NFS 文件系统

网络文件系统 (NFS) 是一种<mark>基于文件</mark>的存储协议，通常<mark>用于Linux环境或者NAS设备</mark>使用。它提供对网络上的共享文件系统的透明远程访问。

NFS 服务器会导出要与客户端共享的文件系统目录及其内容。NFS 客户端系统在本地挂载点上挂载导出的文件系统。

## 安装NFS服务器

在服务器上，nfs-utils 软件包提供了与 NFS 共享目录所需的工具。

在服务器上，NFS 版本 4.1 和更高版本使用端口 2049/TCP。

```bash
[root@servera ~]# yum install nfs-utils -y
[root@servera ~]# systemctl enable --now nfs-server
[root@servera ~]# firewall-cmd --permanent --add-service=nfs
[root@servera ~]# firewall-cmd --reload
```

## 配置 NFS 导出

用于共享目录的主配置文件（“导出表”）为 `/etc/exports`。

目录 `/etc/exports.d` 中名称以 `.exports` 结尾的文件作为额外的导出表

在配置文件中，每一行声明一个导出点。第一个字段是要导出到客户端的目录的名称。行的其余部分列出了可以访问共享目录的客户端系统，以及授予它们的访问权限。

### 配置文件格式样例

配置文件中，都多种写法，具体如下：

DNS 可解析的主机名，如 `client1.example.com`。在以下示例中，`client1.example.com` 系统可以挂载 `/srv/myshare` 目录。

```bash
/srv/myshare client1.example.com
```

使用 `*` 通配符的 DNS 可解析主机名。以下示例允许 `example.com` 域中的所有系统访问 NFS 共享。

```bash
/srv/myshare *.example.com
```

IPv4 地址。以下示例允许从 `192.168.0.12` IP 地址访问 NFS 共享。

```bash
/srv/myshare 192.168.0.12
```

IPv4 网络。以下示例允许从 `192.168.0.0/24` 网络访问 NFS 共享。您也可以使用 `192.168.0.0/255.255.255.0` 表示法。

```bash
/srv/myshare 192.168.0.0/24
```

IPv6 地址。以下示例允许使用 `fde2:6494:1e09:2::20` IPv6 地址的客户端系统访问 NFS 共享。

```bash
/srv/myshare fde2:6494:1e09:2::20
```

IPv6 网络。以下示例允许 `fde2:6494:1e09:2::/64` IPv6 网络访问 NFS 共享。

```bash
/srv/myshare fde2:6494:1e09:2::/64
```

若要与多个客户端系统共享目录，请在目录名称后面使用空格分隔的列表：

```bash
/srv/myshare 192.168.0.0/24 client1.example.com *.example.net
```

### 导出案例

我导出了本地的/nfs给所有客户端，并允许读写

```bash
[root@servera ~]# mkdir /nfs
[root@servera ~]# semanage fcontext -a -t public_content_t '/nfs(/.*)?'
[root@servera ~]# restorecon -Rv /nfs
[root@servera ~]# chmod 777 /nfs
[root@servera ~]# vim /etc/exports.d/lxh.exports
[root@servera ~]# cat /etc/exports.d/lxh.exports
/nfs *(rw)
```

到另一个客户端上挂载测试

```bash
[root@serverd ~]# mount -t nfs servera:/nfs /mnt
```

### 导出权限

默认情况下，目录以只读模式与客户端共享。最常见的选项如下。

`rw`

此选项允许对指定的客户端进行读/写访问。在以下示例中，`client1.example.com` 具有读/写访问权限，但 `client2.example.com` 具有只读访问权限。

```bash
/srv/myshare client1.example.com(rw) client2.example.com
```

`no_root_squash`

默认情况下，当客户端上的 `root` 用户访问 NFS 导出时，服务器会将其视为 `nobody` 用户（如服务器上所定义）的访问。这意味着，如果客户端上的 `root` 在导出时创建了文件，则它将归用户 `nobody` 所有。这也意味着，如果客户端上的 `root` 试图读取用户 `nobody` 无法读取的导出中的文件，则访问将失败。

您可以通过添加 `no_root_squash` 选项来禁用该安全保护。以下示例允许 `client1.example.com` 系统进行读/写访问，并且允许实际的 `root` 用户访问 `/srv/myshare` 导出目录。

```bash
/srv/myshare client1.example.com(rw,no_root_squash)
```

### 导出权限案例

在已经挂载的客户端上，写入nfs文件，`root`写入的文件果然是nobody所有

```bash
[root@serverd ~]# touch /mnt/rootwrite
[root@servera ~]# ll /nfs/
-rw-r--r--. 1 nobody nobody 0 Aug 27 21:17 rootwrite
```

## 检查 NFS 导出

使用 `exportfs` 命令列出 NFS 服务器当前导出的目录。使用 `-v` 选项列出选项

```bash
[root@servera ~]# exportfs -v
/nfs            <world>(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```

每当您在 `/etc/exports` 或 `/etc/exports.d/*.exports` 中更改导出点时，运行 NFS 服务的 `exportfs -r` 命令，以将更改考虑在内。作为替代方案，您也可以运行 `systemctl reload nfs-server`。

# 提供 SMB 文件共享

服务器消息块 (SMB) 是用于 Microsoft Windows 服务器和客户端的标准文件共享协议。Samba 可以将 Linux 目录作为 SMB 网络文件共享来共享

将 SMB 文件共享作为客户端挂载需要安装 cifs-utils 软件包。在服务器上，samba 软件包允许使用 SMB 协议共享目录。

使用 SMB 共享目录的基本步骤如下所示：

- 安装 samba 软件包。

- 准备要共享的目录。

- 配置 `/etc/samba/smb.conf` 配置文件。

- 设置适当的 Linux 用户，并在 Samba 数据库中配置它们。

- 启动 Samba 并打开本地防火墙。

- 从客户端系统挂载 SMB 共享，以验证配置。

## 安装SMB服务

```bash
[root@servera ~]# yum install samba -y
```

## 准备SMB目录

如果需要针对特定的人或组生效权限，可以`chgrp` `chmod` `setfacl`

```bash
[root@servera ~]# mkdir /srv/smbshare
[root@servera ~]# semanage fcontext -a -t samba_share_t '/srv/smbshare(/.*)?'
[root@servera ~]# restorecon -Rv /srv/smbshare
```

## 配置 Samba

Samba 的配置文件是 `/etc/samba/smb.conf`。该文件分为几个部分。`/etc/samba/smb.conf` 配置文件以 `[global]` 部分开头。该部分提供常规服务器配置和默认值，`/etc/samba/smb.conf.example` 提供各种注释和样例

### 配置全局部分

`[global]` 

部分定义 Samba 服务器的基本配置。在该部分中，最有用的参数如下所示。

`workgroup`

`workgroup` 参数指定服务器的 Windows 工作组。此名称在客户端系统查询服务器时显示在客户端系统上。默认值为 `WORKGROUP`。

`security`

`security` 参数控制 Samba 如何对客户端进行身份验证。对于 `security = user`，客户端使用本地 Samba 在其数据库中管理的用户名和密码来登录。这是默认值。

`hostname lookups` 它用来控制 Samba 是否使用 DNS 或其他名称服务来解析主机名。当设置为 `yes` 时，Samba 将尝试使用主机名来查找相应的 IP 地址。这通常用于`hosts allow`等访问控制

`server min protocol`

`server min protocol` 参数指定服务器支持的最小 SMB 版本。默认情况下，服务器支持所有版本的协议，并与客户端协商该版本。由于第一个版本 SMB1 （或 CIFS）存在安全问题，因此红帽建议通过将 `server min protocol` 设置为 `SMB2` 来排除该版本。但是，使用该配置时，Microsoft Windows XP 或更早版本的客户端无法使用您的服务器，因为它们仅支持 SMB1。SMB 协议的当前版本是版本 3。

`smb encrypt`

`smb encrypt` 参数激活流量加密。默认情况下，服务器和客户端会协商加密。若要强制加密，请将 `smb encrypt` 参数设置为 `required` ，并将 `server min protocol` 设置为 `SMB3`。只有 SMB3 提供对加密的原生支持。Microsoft Windows 8、Microsoft Windows Server 2012 和更高版本的操作系统支持加密的 SMB3。

### 配置文件部分样例

```text
[global]
        workgroup = SAMBA
        security = user
        hostname lookups = yes
        passdb backend = tdbsam
        printing = cups
        printcap name = cups
        load printers = yes
        cups options = raw
```

### 配置共享部分

在配置文件最下面定义共享。方括号中的部分名称定义了共享的名称，还有一些常见的参数如下所示：

`path`

`path` 指令提供要在您的服务器上共享的目录的完整名称。

`writeable`

`writeable` 指令指示经过身份验证的用户是具有对共享的读/写访问权限（如果设为 `yes`）还是不具有（当设为 `no` 时）。默认设置为 `no`。

`write list`

当 `writeable` 指令的值为 `no`（默认值）时，您可使用 `write list` 指令提供一个对该共享具有读/写访问权限的以逗号分隔的用户列表。不在列表中的用户仅具有读取访问权限。在列表中，您可以通过在组名的前面加上 `@` 字符作为前缀来指定本地 Linux 组。

```textile
write list = operator1, @developers
```

`valid users`

默认情况下，所有经过身份验证的用户都可以访问该共享。如果您想要限制该访问权限，请使用 `valid users` 指令。指令采用应具有访问权限的以逗号分隔的用户列表。

`hosts allow / hosts deny`

这些选项用于基于主机名、IP 地址或 IP 范围来允许或拒绝对共享的访问

### 共享样例

```textile
[lxhshare]
        comment = suibian ceshi
        browseable = yes
        path = /srv/smbshare
        writeable = no
        write list = lxh,@group1
        valid users = lxh,zhangsan
        hosts allow = 172.25. except 172.25.250.12
```

### 配置文件测试

若要验证 `/etc/samba/smb.conf` 文件中没有错误，请运行不带任何参数的 `testparm` 命令。

```bash
[root@servera ~]# testparm
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
Server role: ROLE_STANDALONE

Press enter to see a dump of your service definitions
```

## 准备 Samba 用户

每个 Samba 帐户必须具有与相同用户名关联的 Linux 帐户。要创建仅限于 Samba 的用户帐户，请锁定其 Linux 密码，并将其登录设置为 `/sbin/nologin`。此配置可防止用户使用 SSH 或控制台登录 Linux 系统。

```bash
[root@servera ~]# useradd -s /sbin/nologin lxh
[root@servera ~]# useradd -s /sbin/nologin zhangsan
```

在创建了 Linux 帐户后，利用 samba-common-tools 软件包中的 `smbpasswd` 命令将它们添加到 Samba 数据库。

```bash
[root@servera ~]# smbpasswd -h
...
  -a                   add user
  -d                   disable user
  -e                   enable user
  -x                   delete user
```

```bash
[root@servera ~]# smbpasswd -a lxh
New SMB password:
Retype new SMB password:
Added user lxh.

[root@servera ~]# smbpasswd -a zhangsan
New SMB password:
Retype new SMB password:
Added user zhangsan.
```

除了 `smbpasswd` 命令之外，还提供了更强大的 `pdbedit` 命令。

```bash
[root@servera ~]# pdbedit -L
lxh:1002:
zhangsan:1003:
```

Samba的默认数据位置是：

```bash
[root@servera ~]# ll /var/lib/samba/private/
total 832
drwx------. 2 root root    102 Aug 27 21:46 msg.sock
-rw-------. 1 root root 421888 Aug 27 21:43 passdb.tdb
-rw-------. 1 root root 430080 Aug 27 21:42 secrets.tdb
```

现在目录也存在了、配置文件也配好了，用户也创建了，该启动服务测试了

nmb服务用于提供跨子网的网络浏览和自动发现功能，使得 Samba 服务器能够在网络邻居中被看到，并允许客户端通过 NetBIOS 名称访问服务器上的共享资源。

```bash
[root@servera ~]# systemctl enable --now smb nmb
[root@servera ~]# firewall-cmd --permanent --add-service=samba
[root@servera ~]# firewall-cmd --reload
```

## 挂载SMB

Windows 和 Linux 系统均可从 Samba 服务器访问 SMB 共享。在 Linux 系统上，安装 cifs-utils 软件包，以便您可以在本地系统上挂载 SMB 共享

安装samba客户端虽然不是必须的，但是如果安装，操作起来方便，比如在挂载之前，可以看看都共享了什么，但是在Linux中，cifs-utils是必须安装的

```bash
[root@server ~]# yum install samba-client cifs-utils -y
```

安装好了之后，看看对方给我们共享了什么

```bash
[root@serverd ~]# smbclient -L 172.25.250.10 -U lxh
Enter SAMBA\lxh's password:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        lxhshare        Disk      suibian ceshi
        IPC$            IPC       IPC Service (Samba 4.10.4)
        lxh             Disk      Home Directories
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
```

### 手动挂载

```bash
[root@serverd ~]# mount -t cifs //servera/lxhshare -o username=lxh,password=1 /mnt
[root@serverd ~]# df -h | grep mnt
//servera/lxhshare   10G  2.2G  7.8G  22% /mnt
[root@serverd ~]# mount -l | grep mnt
//servera/lxhshare on /mnt type cifs (rw,relatime,vers=3.1.1,cache=strict,username=lxh,uid=0,noforceuid,gid=0,noforcegid,addr=172.25.250.10,file_mode=0755,dir_mode=0755,soft,nounix,serverino,mapposix,rsize=4194304,wsize=4194304,bsize=1048576,echo_interval=60,actimeo=1)
```

### 自动挂载

明文写密码有点不安全，而且也不能自动挂载，仅用于临时使用，我们尝试用密码文件来自动挂载

```bash
[root@serverd ~]# vim /etc/samba/credentials
[root@serverd ~]# cat /etc/samba/credentials
username=zhangsan
password=1
[root@serverd ~]# chmod 600 /etc/samba/credentials
```

用zhangsan的身份做一个永久挂载，若要强制 SMB 流量加密，在option的地方加上`seal`选项

```bash
[root@serverd ~]# vim /etc/fstab
[root@serverd ~]# tail -n1 /etc/fstab
//servera/lxhshare /opt cifs defaults,credentials=/etc/samba/credentials 0 0
[root@serverd ~]# mount -a
[root@serverd ~]# df -h | grep opt
//servera/lxhshare   10G  2.2G  7.8G  22% /opt
[root@serverd ~]# mount -l | grep opt
//servera/lxhshare on /opt type cifs (rw,relatime,vers=3.1.1,cache=strict,username=zhangsan,uid=0,noforceuid,gid=0,noforcegid,addr=172.25.250.10,file_mode=0755,dir_mode=0755,soft,nounix,serverino,mapposix,rsize=4194304,wsize=4194304,bsize=1048576,echo_interval=60,actimeo=1)
```

## 多用户 SMB 挂载

可以让 `root` 用户使用对共享具有最少访问权限的凭据来挂载 SMB 共享。当用户登录时，他们使用 `cifscreds` 命令将他们的 SMB 密码临时添加到安全内核密钥环中。然后，客户端的 Linux 内核将使用其 SMB 凭据来确定对共享的访问权限，而不是 `root` 用于挂载该共享的凭据。想要实现非常简单，只需要在option的地方加入`multiuser`

比如我刚才的自动挂载，用的就是zhangsan的身份

```bash
[root@serverd ~]# cat /etc/samba/credentials
username=zhangsan
password=1
```

而zhangsan没有写权限

```bash
[root@servera ~]# grep -A 8 lxhshare /etc/samba/smb.conf
[lxhshare]
        comment = suibian ceshi
        browseable = yes
        path = /srv/smbshare
        writeable = no
        write list = lxh,@group1
        valid users = lxh,zhangsan
```

确保lxh用户有写入权限

```bash
[root@servera ~]# setfacl -m u:lxh:rwx /srv/smbshare/
```

我用zhangsan挂载，大家都是不能写的

```bash
[root@serverd ~]# touch /opt/writetest
touch: cannot touch '/opt/writetest': Permission denied
```

修改一下客户端的挂载，加入`multiuser`参数

```bash
[root@serverd ~]# vim /etc/fstab
[root@serverd ~]# umount /opt
[root@serverd ~]# tail -n1 /etc/fstab
//servera/lxhshare /opt cifs defaults,credentials=/etc/samba/credentials,multiuser 0 0
[root@serverd ~]# mount -a
[root@serverd ~]# df -h | grep opt
//servera/lxhshare   10G  2.2G  7.8G  22% /opt
[root@serverd ~]# mount -l | grep opt
//servera/lxhshare on /opt type cifs (rw,relatime,vers=3.1.1,cache=strict,multiuser,uid=0,noforceuid,gid=0,noforcegid,addr=172.25.250.10,file_mode=0755,dir_mode=0755,soft,nounix,serverino,mapposix,noperm,rsize=4194304,wsize=4194304,bsize=1048576,echo_interval=60,actimeo=1)
```

在客户端上创建写入用户，并尝试写入内容

```bash
[root@serverd ~]# useradd lxh
[root@serverd ~]# passwd
...
[root@serverd ~]# su - lxh
[lxh@serverd ~]$ touch /opt/writetest
touch: cannot touch '/opt/writetest': Permission denied
[lxh@serverd ~]$ cifscreds add servera
Password:
[lxh@serverd ~]$ touch /opt/writetest
[lxh@serverd ~]$ ll /opt/writetest
-rwxr-xr-x. 1 lxh lxh 0 Aug 27 22:27 /opt/writetest
[lxh@serverd ~]$
```

# 自动调配基于文件的存储

详见教材
