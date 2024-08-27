```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 章节目标

- 讨论本课程的目标，并了解如何管理服务。

- 了解如何使用 NetworkManager 和相关工具配置和管理网络接口。

- 使用 Ansible 自动配置服务和网络接口。

# 控制网络服务

## Systemctl 和 Systemd 单元

红帽企业 Linux 系统的<mark>进程 ID 1</mark> 是`systemd`。此进程负责在启动过程中激活系统上的其他守护进程，并在系统停止或重新启动时将其关闭。`Systemd`*服务*通常管理一个或多个共同提供某一功能的守护进程

`systemctl` 命令用于管理各种类型的 `systemd` 对象，它们称为*单元*。可以通过 `systemctl -t help` 命令显示可用单元类型的列表。

一些常见的单元类型包括：

- *服务单元*，具有 `service` 扩展名，代表系统服务。这种单元用于管理经常访问的守护进程，如 web 服务器。当服务单元启动守护进程时，它通常会一直运行直到被 `systemd` 停止为止。

- *套接字单元*，具有 `socket` 扩展名，代表进程间通信 (IPC) 套接字。套接字的控制可以在建立客户端连接时从 `systemd` 传递给新启动的守护进程。套接字单元用于延迟系统启动时的服务启动，或者按需启动不常使用的服务。当客户端断开连接时，套接字单元可能会停止关联的服务单元。

### 服务状态

可以通过 ``systemctl status name.type`` 来查看服务的状态

```bash
[root@servera ~]# systemctl status sshd.service
● sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2020-03-25 10:26:49 EDT; 8h ago
     Docs: man:sshd(8)
           man:sshd_config(5)
 Main PID: 1081 (sshd)
    Tasks: 1 (limit: 11231)
   Memory: 6.7M
   CGroup: /system.slice/sshd.service
           └─1081 /usr/sbin/sshd -D -oCiphers=aes256-gcm@openssh.com,chacha20...

Hint: Some lines were ellipsized, use -l to show in full.
```

状态输出中可以找到表示服务状态的几个关键字：

| 关键字                | 描述                    |
| ------------------ | --------------------- |
| `loaded`           | 单元配置文件已处理。            |
| `active (running)` | 正在通过一个或多个持续进程运行。      |
| `active (exited)`  | 已成功完成一次性配置。           |
| `active (waiting)` | 运行中，但正在等待事件。          |
| `inactive`         | 不在运行。                 |
| `enabled`          | 将在系统启动时启动。            |
| `disabled`         | 不会在系统启动时启动。           |
| `static`           | 无法启用，但可以由某一启用的单元自动启动。 |

### 列出单元文件

- 查询所有单元的状态，以验证系统启动。
  
  ```bash
  [root@servera ~]# systemctl
  ```

- 仅查询服务单元的状态。
  
  ```bash
  [root@servera ~]# systemctl --type=service
  UNIT                               LOAD   ACTIVE SUB     DESCRIPTION
  accounts-daemon.service            loaded active running Accounts Service
  atd.service                        loaded active running Deferred execution scheduler
  auditd.service                     loaded active running Security Auditing Service
  avahi-daemon.service               loaded active running Avahi mDNS/DNS-SD Stack
  chronyd.service                    loaded active running NTP client/server
  colord.service                     loaded active running Manage, Install and Generate Color Profiles
  ```

- 调查处于失败或维护状态的任何单元。可选择添加 `-l` 选项以显示完整的输出。
  
  ```bash
  [root@servera ~]# systemctl status NetworkManager-wait-online -l
  ● NetworkManager-wait-online.service - Network Manager Wait Online
     Loaded: loaded (/usr/lib/systemd/system/NetworkManager-wait-online.service; enabled; preset: disabled)
     Active: active (exited) since Sun 2024-08-11 06:10:34 CST; 5min ago
       Docs: man:NetworkManager-wait-online.service(8)
    Process: 1344 ExecStart=/usr/bin/nm-online -s -q (code=exited, status=0/SUCCESS)
   Main PID: 1344 (code=exited, status=0/SUCCESS)
        CPU: 30ms
  ```

- 也可使用 `is-active` 或 `is-enabled` 参数来判断特定的单元是否活动，以及显示该单元是否已启用在系统启动时启动。其他备用命令也可显示活动和已启用状态：
  
  ```bash
  [root@servera ~]# systemctl is-active sshd
  active
  [root@servera ~]# systemctl is-enabled sshd
  enabled
  ```

- 列出所有已加载单元的活动状态。也可选择限制单元类型。`--all` 选项可加入不活动的单元。
  
  ```bash
  [root@servera ~]# systemctl list-units --type=service
  UNIT                               LOAD   ACTIVE SUB     DESCRIPTION
  accounts-daemon.service            loaded active running Accounts Service
  atd.service                        loaded active running Job spooling tools
  auditd.service                     loaded active running Security Auditing Service
  avahi-daemon.service               loaded active running Avahi mDNS/DNS-SD Stack
  ...output omitted...
  
  [root@servera ~]# systemctl list-units --type=service --all
  ...output omitted...
  ```

- 查看所有单元的已启用和已禁用设置。也可选择限制单元类型。
  
  ```bash
  [root@servera ~]# systemctl list-unit-files --type=service
  ...output omitted...  ```
  ```

- 仅查看失败的服务。
  
  ```bash
  [root@servera ~]# systemctl --failed --type=service
  ...output omitted...  ```
  ```

- 停止服务并验证其状态。
  
  ```bash
  [root@servera ~]# systemctl stop sshd.service
  [root@servera ~]# systemctl status sshd.service
  ```

- 启动服务并查看其状态。进程 ID 已经改变。
  
  ```bash
  [root@servera ~]# systemctl start sshd.service
  [root@servera ~]# systemctl status sshd.service
  ```

- 以单一命令停止服务，然后再启动该服务。
  
  ```bash
  [root@servera ~]# systemctl restart sshd.service
  [root@servera ~]# systemctl status sshd.service
  ```

- 发出指示使服务读取和重新加载其配置文件，而不完全停止和启动服务。请注意，由于没有完全停止和启动，进程 ID 未发生更改。
  
  ```bash
  [root@servera ~]# systemctl reload sshd.service
  ```

- 屏蔽服务
  
  ```bash
  [root@servera ~]# systemctl mask iptables
  [root@servera ~]# systemctl unmask iptables
  ```

- 启用或禁用服务并验证其状态
  
  ```bash
  [root@servera ~]# systemctl disable sshd.service --now 
  ```

### Systemctl 命令摘要

下面显示了一些常用的命令。若要获取更多命令，请运行 `systemctl --help`。

| 命令                                       | 任务                        |
| ---------------------------------------- | ------------------------- |
| ``systemctl status *`UNIT`*``            | 查看有关单元状态的详细信息。            |
| ``systemctl stop *`UNIT`*``              | 在运行中的系统上停止一项服务。           |
| ``systemctl start *`UNIT`*``             | 在运行中的系统上启动一项服务。           |
| ``systemctl restart *`UNIT`*``           | 在运行中的系统上重新启动一项服务。         |
| ``systemctl reload *`UNIT`*``            | 重新加载运行中服务的配置文件。           |
| ``systemctl mask *`UNIT`*``              | 彻底禁用服务，使其无法手动启动或在系统引导时启动。 |
| ``systemctl unmask *`UNIT`*``            | 取消屏蔽服务并使其可用。              |
| ``systemctl enable *`UNIT`*``            | 将服务配置为在系统引导时启动。           |
| ``systemctl disable *`UNIT`*``           | 禁止服务在系统引导时启动。             |
| ``systemctl list-dependencies *`UNIT`*`` | 列出指定单元需要的单元。              |

# 配置网络接口

## NetworkManager 简介

在 NetworkManager 中：

- *设备*是网络接口。

- *连接*是可以为设备配置的设置的集合。

- 对于任何一个设备，<mark>在同一时间只能有一个连接处于*活动*状态。可能存在多个连接</mark>，以供不同设备使用或者以便为同一设备更改配置。

- 每个连接具有一个用于标识自身的*名称*或 *ID*。

- ``/etc/sysconfig/network-scripts/ifcfg-*`name`*`` 文件存储连接的持久配置，其中 *`name`* 是连接的名称。当连接名称的名称中包含空格时，文件名中的空格将被替换为下划线。如果需要，可以手动编辑此文件。

- `nmcli` 实用程序可用于通过 shell 提示符来创建和编辑连接文件。

`nmcli dev status` 命令可显示所有网络设备的当前状态：

```bash
[root@servera ~]# nmcli dev status
DEVICE  TYPE      STATE         CONNECTION
eth0    ethernet  connected     Wired connection 1
eth1    ethernet  disconnected  --
eth2    ethernet  disconnected  --
lo      loopback  unmanaged     --
```

`nmcli con show` 命令可显示所有连接的列表。要仅列出活动的连接，可添加 `--active` 选项。

```bash
[root@servera ~]# nmcli con show
NAME                UUID                                  TYPE      DEVICE
Wired connection 1  4ae4bb9e-8f2d-3774-95f8-868d74edcc3c  ethernet  eth0
Wired connection 2  c0e6d328-fcb8-3715-8d82-f8c37cb42152  ethernet  --
Wired connection 3  9b5ac87b-572c-3632-b8a2-ca242f22733d  ethernet  --

[root@servera ~]# nmcli con show --active
NAME                UUID                                  TYPE      DEVICE
Wired connection 1  4ae4bb9e-8f2d-3774-95f8-868d74edcc3c  ethernet  eth0
```

使用 `ip addr show` 命令显示系统上网络接口的当前配置。要仅列出单个接口，请添加接口名称作为最后一个参数：

```bash
[root@servera ~]# ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:00:fa:0a brd ff:ff:ff:ff:ff:ff
    inet 172.25.250.10/24 brd 172.25.250.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::984:87d2:dba7:1007/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

## 添加网络连接

此示例为接口 eno1 添加一个名为 eno1 的新连接，从DHCP获得地址

```bash
[root@servera ~]# nmcli con add con-name eno1 type ethernet ifname eno1
Connection 'eno1' (5a0ba475-e25f-416e-8a33-29a43fd421a4) successfully added.
```

此示例为接口 eno1 添加一个名为 static-eno1 的新连接。此连接：

- 设置 192.0.2.7/24 的静态 ipv4 地址和 192.0.2.1 的 IPv4 网关

- 设置一个静态 IPv6 地址 2001:db8:0:1::c000:207/64 和一个 IPv6 网关路由器 2001:db8:0:1::1
  
  ```bash
  [root@servera ~]# nmcli con add con-name static-eno1 type ethernet ifname eno1 \
  ipv6.address 2001:db8:0:1::c000:207/64 ipv6.gateway 2001:db8:0:1::1 \
  ipv4.address 192.0.2.7/24 ipv4.gateway 192.0.2.1
  Connection 'static-eno1' (63f9a2ef-9e76-4eae-9372-5b7c4c6143cf) successfully added.
  ```
  
  nmcli con up/down name 命令激活连接
  
  ```bash
  [root@servera ~]# nmcli con up static-eno1
  ```

``nmcli dev disconnect device`` 命令将断开与 *`device`* 网络接口的连接并将其关闭

```bash
[root@servera ~]# nmcli dev dis eno1
```

## 修改网络连接设置

``nmcli con mod name`` 命令可用于更改连接的设置

```bash
[root@servera ~]# nmcli con mod static-ens3 ipv4.address 192.0.2.2/24 \
ipv4.gateway 192.0.2.254

```

要针对 `static-ens3` 连接将 IPv6 地址设置为 `2001:db8:0:1::a00:1/64` 并将默认网关设置为 `2001:db8:0:1::1`：

```bash
[root@servera ~]# nmcli con mod static-ens3 ipv6.address 2001:db8:0:1::a00:1/64 \
ipv6.gateway 2001:db8:0:1::1
```

## nmcli 命令摘要

下表是本节中讨论的关键命令的列表。

| 命令                                            | 用途                                        |
| --------------------------------------------- | ----------------------------------------- |
| `nmcli dev status`                            | 显示所有网络接口的 NetworkManager 状态。              |
| `nmcli con show`                              | 列出所有 NetworkManager 连接。                   |
| ``nmcli con show *`name`*``                   | 列出 *`name`* 连接的当前设置。                      |
| ``nmcli con add con-name *`name`* [OPTIONS]`` | 添加一个名为 *`name`* 的新连接。                     |
| ``nmcli con mod *`name`* [OPTIONS]``          | 修改 *`name`* 连接。                           |
| `nmcli con reload`                            | 让 NetworkManager 重新读取配置文件。这在手动编辑配置文件后很有用。 |
| ``nmcli con up *`name`*``                     | 激活 *`name`* 连接。                           |
| ``nmcli dev dis *`dev`*``                     | 在 *`dev`* 网络接口上停用并断开当前连接。                 |
| ``nmcli con del *`name`*``                    | 删除 *`name`* 连接及其配置文件。                     |
| `ip addr show`                                | 显示当前网络接口地址配置。                             |

# 自动化执行服务和网络接口配置

Ansible 提供了在使用 `systemd` 服务时特别有用的三个模块：`service`、`systemd` 和 `service_facts`。

可以使用 `service` 模块。`service` 模块提供了一组基本选项：`start`、`stop`、`restart` 和 `enable`。需要的话可以运行 `ansible-doc service` 命令



```yaml
- name: web server is started
  service:
    name: httpd
    state: started
```

```yaml
- name: reload web server
  systemd:
    name: apache2
    state: reload
    daemon-reload: yes
```

`service_facts` 模块收集有关系统上服务的信息，并将该信息存储在 `ansible_facts ['services']` 变量中。

```yaml
- name: collect service status facts
  service_facts:

- name: display whether NetworkManager is running
  debug:
    var: ansible_facts['services']['NetworkManager.service']['state']
```

## 使用网络系统角色配置网络

仓库中包括一个名为 rhel-system-roles 的软件包，其中包含用于 Linux 服务管理的 Ansible 角色集合。安装该软件包时，它会将角色放置在 `/usr/share/ansible/roles` 目录中

`rhel-system-roles.network` 角色（也可以作为 `linux-system-roles.network` 来调用）是在受管主机上配置网络设置的最简单方法。



您可以在特定主机的 `host_vars` 主机变量文件中包含以下内容，然后创建yaml来运行

```yaml
---
network_connections:
  - name: ens4
    type: ethernet
    ip:
      address:
        - 172.25.250.30/24
```

下表列出了 `network_connections` 变量的选项。

| 选项名称               | 描述                                                                                                           |
| ------------------ | ------------------------------------------------------------------------------------------------------------ |
| `name`             | 标识连接配置集。                                                                                                     |
| `state`            | 连接配置集的运行时状态：如果连接配置集应处于活动状态，为 `up`，如果应处于非活动状态，则为 `down` 。                                                     |
| `persistent_state` | `present` 设置为默认值，将创建或修改连接配置集。您还必须指定 `type` 选项。如果您不将 `state` 选项设置为 `up`，则新连接或修改后的连接将不会启动。`absent` 设置将删除连接配置集。 |
| `type`             | 标识连接类型。有效的值有 `ethernet`、`bridge`、`bond`、`team`、`vlan`、`macvlan` 和 `infiniband`。                              |
| `autoconnect`      | 确定连接是否自动启动。                                                                                                  |
| `mac`              | 将连接限制为在具有特定 MAC 地址的设备上使用。                                                                                    |
| `interface_name`   | 将连接配置集显示为供特定接口使用。                                                                                            |
| `zone`             | 为接口配置 `firewalld` 区域。                                                                                        |
| `ip`               | 确定连接的 IP 配置。支持诸如 `address`（指定静态 IP 地址）或 `dns`（配置 DNS 服务器）等选项。                                                |

创建好变量文件后，可以创建以下的yaml，会自动从host_vars中调用变量

```bash
- name: Ensure NICs have the right configuration
  hosts: webservers

  roles:
    - rhel-system-roles.network
```

## 网络配置的 Ansible 事实

当 play 执行自动事实收集或运行 `setup` 模块时，Ansible 将收集与网络配置相关的事实。如果您需要有关计算机的网络接口、IP 地址或当前配置的信息以在 play 中使用，则其中的许多事实都特别有用。

| 事实名称                                    | 描述                                                              |
| --------------------------------------- | --------------------------------------------------------------- |
| `ansible_facts['all_ipv4_addresses']`   | 包含受管主机上配置的 IPv4 地址列表，省略 `127.0.0.1`。                            |
| `ansible_facts['all_ipv6_addresses']`   | 包含受管主机上配置的 IPv6 地址的列表，省略 `::1`，但在 `fe80::/10` 中包括本地链路地址。        |
| `ansible_facts['default_ipv4']`         | 包含有关在受管主机上配置有默认 IPv4 路由的接口的事实字典，包括其名称、MAC 地址和网络设置。如果没有默认路由，则为空。 |
| `ansible_facts['default_ipv6']`         | 包含有关在受管主机上配置有默认 IPv6 路由的接口的事实字典，包括其名称、MAC 地址和网络设置。如果没有默认路由，则为空。 |
| `ansible_facts['interfaces']`           | 包含受管主机上所有网络接口的列表。                                               |
| ``ansible_facts['*`interface-name`*']`` | 包含关于 *`interface-name`* 网络接口的事实字典，包括其当前的网络配置和功能。                |

列出接口上的 IP 地址

```yaml
- name: display IPv4 address of enp11s0
  debug:
    var: ansible_facts['enp11s0']['ipv4']['address']
```
