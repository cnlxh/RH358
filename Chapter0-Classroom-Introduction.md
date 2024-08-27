```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 红帽服务管理和自动化

## 课程目标

- 使用红帽企业 Linux 8 随附的软件提供关键网络服务，包括使用 Unbound 和 BIND 9 的 DNS、DHCP 和 DHCPv6 、电子邮件传输、打印机服务、NFSv4 和 SMBv3 文件共享、带有 MariaDB 的 SQL 数据库服务以及使用 Apache HTTP 服务器、Nginx、Varnish 和 HAProxy 的 web 服务。

- 配置用于服务器用例（包括设备合作）的高级网络。

- 通过 Ansible，自动化处理本课程中涵盖的手动部署和配置任务。



在本课程中，用于实践学习活动的主要计算机系统是 `workstation`。名为 `bastion` 的系统<mark>必须始终处于运行状态</mark>。另有两台计算机供学员用于这些活动：`servera`、`serverb`、`serverc` 和 `serverd`. 所有系统都在 `lab.example.com` DNS 域内。



所有学员计算机系统都有一个标准用户帐户 `student`，其密码为 `student`。所有学员系统的 `root` 密码都是 `redhat`。

## 虚拟机介绍

| 计算机名称                       | IP 地址          | 角色               |
| --------------------------- | -------------- | ---------------- |
| bastion.lab.example.com     | 172.25.250.254 | 将虚拟机链接到中央服务器的路由器 |
| workstation.lab.example.com | 172.25.250.9   | 用于进行系统管理的图形工作站   |
| servera.lab.example.com     | 172.25.250.10  | 受管服务器“A”         |
| serverb.lab.example.com     | 172.25.250.11  | 受管服务器“B”         |
| serverc.lab.example.com     | 172.25.250.12  | 受管服务器“C”         |
| serverd.lab.example.com     | 172.25.250.13  | 受管服务器“D”         |

`bastion` 系统充当连接学员计算机的网络和课堂网络之间的路由器。如果 `bastion` 关闭，其他学员计算机可能无法正常工作，甚至在启动过程中也会挂起。



## Ansible部分



预先配置为允许从 `workstation`发起Ansible 剧本



- 受管服务器上的 `devops` 用户已预先配置为允许来自 `workstation` 上 `student` 帐户的基于密钥的 SSH 身份验证。

- 为了简化特权升级，`devops` 用户也可以使用 `sudo` 以 `root` 身份运行任何命令，而无需输入密码。




