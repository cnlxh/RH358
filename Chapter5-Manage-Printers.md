```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 章节目标

- 创建和管理网络打印机的打印机队列，并使用命令行工具打印文件和管理打印机队列。

- 使用 Ansible 自动执行网络打印机配置。

# 配置和管理打印机

## CUPS 打印架构

CUPS 可以使用多个协议与打印机和打印服务器通信。在大多数情况下， *Internet 打印协议* (IPP) 是使用 CUPS 与打印机通信的首选机制。此协议是对 HTTP/1.1 的修改，它受到大多数现代网络和 USB 打印机的本地支持，通常使用 TCP 端口 631。CUPS 可以支持直接连接的打印机（例如，使用并行、串行或 USB 通信），并且可以使用 LPD 等较旧的网络协议。

CUPS 提供了一组命令行工具和一个 web 界面，用于管理 CUPS 和提交打印作业。它还提供了一个守护进程 (`cupsd`)，用于管理每个已配置打印机的作业队列。打印机的每个队列都与 PostScript 打印机描述 (PPD) 文件关联，该文件描述了打印机功能以及 CUPS 应如何为作业做好在该打印机上打印的准备。

## 使用 IPP Everywhere 发现网络打印机

大多数网络打印机都支持 IPP Everywhere标准，该标准允许您查找、配置和打印到网络打印机，无需特定于硬件的驱动程序或其他软件。

- 打印机通过本地多播 DNS (mDNS) 使用 DNS 服务发现 (DNS-SD) 公布其存在性和基本功能。

- 当您首次配置打印机队列时，CUPS 使用此信息来自动发现打印机功能，并为该打印机构建自定义 PPD。

- 当您的客户端为打印机准备打印作业时，CUPS 首先根据创建队列时从打印机获取的列表将其格式化为打印机理解的页面描述语言 (PDL)。这通常是 PDF 或为打印机分辨率优化的光栅图像。

- 您的客户端使用 IPP 协议向打印机发送格式化的作业。

## 安装MDNS客户端

由于 IPP Everywhere 打印机使用 mDNS 公布其本地网络存在性和打印机功能，因此请安装 avahi 软件包，以便能够接收 mDNS 消息。cups-ipptool 软件包提供了一个实用程序，可用于在随处发现本地 IPP Everywhere 打印机 `ippfind`。

```bash
[root@servera ~]# yum install avahi cups-ipptool -y
```

调整防火墙规则，以允许传入的 mDNS 数据包：

```bash
[root@servera ~]# firewall-cmd --add-service=mdns --permanent
[root@servera ~]# firewall-cmd --reload
```

至此，发现打印机的客户端就安装和准备好了，我们没有真的打印机，需要打lab命令帮我们准备一个打印机出来

## 准备打印机

lab命令可确保 serverc 上已配置 IPP Everywhere 打印机。您可以在 web 浏览器中通过以下 URL 查看发送到此打印机的打印作业：http://serverd/ippserver/ipp-everywhere-pdf。

```bash
[student@workstation ~]$ lab printing-config start
```

使用 ippfind 命令，发现本地网络上可用的 IPP Everywhere 打印机,-T可以执行超时时间，可以看到，成功发现了打印机，但是这个名称在本地不能解析，你可以换成serverc的fqdn或ip地址

```bash
[root@servera ~]# ippfind -T 20
ipp://serverc.local:631/printers/rht-printer
```

## 创建打印队列

在确定了网络打印机的 URI 后，在系统上安装 CUPS 并为其创建打印队列。

```bash
[root@servera ~]# yum install cups -y
[root@servera ~]# systemctl enable --now cups
```

使用 `lpadmin` 命令创建打印队列

`-p`  定义打印机队列的名称

`-v ` 指定打印机的 URI

`-m everywhere` 指定您想要使用 IPP Everywhere 作为您的“驱动程序”

`-E` 可立即启用打印机队列，并允许它开始处理新作业

```bash
[root@servera ~]# lpadmin -p lxh-printer \
-v ipp://serverc.lab.example.com:631/printers/rht-printer \
-m everywhere -E
```

使用 `lpstat -v` 命令确认已定义了打印机队列。此命令显示可用打印机队列的列表，以及各自打印机的设备 URI。

```bash
[root@servera ~]# lpstat -v
device for lxh-printer: ipp://serverc.lab.example.com:631/printers/rht-printer
```

当有多个打印队列的时候，可以用`-d`设置默认，不想要了，可以`-x`删除整个队列

```bash
[root@servera ~]# lpadmin -d lxh-printer
```

## 打印文件和管理打印作业

使用 `lp` 命令，从 shell 打印文件。作业将发送到默认队列，除非您使用 `-d` 选项指定不同的打印机。

```bash
[root@servera ~]# ll
total 16
-rw-------. 1 root root 6971 Oct 30  2019 anaconda-ks.cfg
drwxr-xr-x. 2 root root   82 Aug 23 19:04 dhcp-ipv6config
-rw-------. 1 root root 6774 Oct 30  2019 original-ks.cfg
[root@servera ~]# ll | lp
request id is lxh-printer-1 (0 file(s))
[root@servera ~]# lp anaconda-ks.cfg
request id is lxh-printer-2 (1 file(s))
```

查询队列里的内容

```bash
[root@servera ~]# lp anaconda-ks.cfg
request id is lxh-printer-3 (1 file(s))
[root@servera ~]# lpstat
lxh-printer-3           root              7168   Fri 23 Aug 2024 09:52:44 PM CST
```

使用 cancel 命令取消打印机队列中的作业。指定您要取消的作业 ID，或者用-a取消队列里的所有作业

不过动作要快，不然就像我的意义，任务都完成了，无法取消

```bash
[root@servera ~]# cancel lxh-printer-3
cancel: cancel-job failed: Job #3 is already completed - can't cancel.
```

## 管理打印机队列

如果您需要执行维护，您可能希望停止或重新启动打印机队列的处理。

### 暂停和恢复打印

```bash
[root@servera ~]# cupsdisable -r 'lxh pause' lxh-printer
[root@servera ~]# ll | lp
request id is lxh-printer-4 (0 file(s))[root@servera ~]# lpstat -p
printer lxh-printer disabled since Fri 23 Aug 2024 09:55:47 PM CST -
        lxh pause
```

```bash
[root@servera ~]# cupsenable lxh-printer
[root@servera ~]# lpstat -p
printer lxh-printer now printing lxh-printer-4.  enabled since Fri 23 Aug 2024 09:57:22 PM CST
        Printing.
```

### 拒绝和接受打印

```bash
[root@servera ~]# cupsreject lxh-printer
[root@servera ~]# ll | lp
lp: Destination "lxh-printer" is not accepting jobs.

[root@servera ~]# cupsaccept lxh-printer
[root@servera ~]# ll | lp
request id is lxh-printer-6 (0 file(s))
[root@servera ~]# lpstat -p
printer lxh-printer is idle.  enabled since Fri 23 Aug 2024 09:58:29 PM CST
```

# 自动化执行打印机配置

详见教材
