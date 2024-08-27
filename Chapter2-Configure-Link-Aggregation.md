```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 章节目标

- 使用网络合作提供链路冗余性或更高的吞吐量。

- 管理网络组接口。

- 使用 Ansible 自动创建和配置网络组。

# 配置网络合作

## 网络合作概念简介

网络合作将网络流量分布于多个网络接口，*网络合作*将多个网络接口卡在逻辑上链接在一起，以实现故障转移，或者支持更高的吞吐量。

称为*运行程序*的软件实施负载均衡和主动备份逻辑，下表总结了每个运行程序的行为：

**Teamd 运行程序**

| 运行程序           | 描述                                                                   |
| -------------- | -------------------------------------------------------------------- |
| `activebackup` | 故障转移运行程序，可监视链路变更并选择活动端口进行数据传输。                                       |
| `roundrobin`   | 此运行程序会以轮循方式传输来自每个端口的数据包。                                             |
| `broadcast`    | 此运行程序可传输来自所有端口的每个数据包。                                                |
| `loadbalance`  | 此运行程序监控流量并使用哈希函数以尝试在为数据包传输选择端口时达到完美均衡。                               |
| `lacp`         | 此运行程序实施 802.3ad 链路聚合控制协议 (LACP)。它具有与 `loadbalance` 运行程序相同的传输端口选择可能性。 |
| `random`       | 此运行程序在随机选择的端口上传输数据包。                                                 |

所有网络交互都通过*组接口*（或主接口）完成。组接口由多个*端口接口*（端口或从接口）组成，它们是组合到组中的网络接口。

使用 NetworkManager 控制组接口时，或对其进行故障排除时，您应牢记以下几点：

- 启动组接口不会自动启动其端口接口。

- 启动端口接口始终会启动组接口。

- 停止组接口也会停止端口接口。

- 不含端口的组接口可以启动静态 IP 连接。

- 在启动 DHCP 连接时，不含端口的组接口将等待端口。

- 如果组接口具有 DHCP 连接且在等待端口，则在添加具有载波信号的端口时它会完成激活。

- 如果组接口具有 DHCP 连接且在等待端口，则在添加不具有载波信号的端口时它会继续等待。

## 配置网络组

使用 `nmcli` 命令创建和管理组和端口接口。以下四个步骤用于创建和激活组接口：

1. 创建组接口。

2. 分配组接口的 IPv4 和/或 IPv6 属性。

3. 创建端口接口。

4. 启动或关闭组接口和端口接口。

#### 创建组接口

创建一个名为 `team0` 的主被模式合作接口，并为其分配地址 `192.168.0.100/24`。将网络接口 `eth1` 和 `eth2` 分配为端口接口。

1. 使用 `nmcli` 命令创建 `team0` 连接。

```bash
[root@servera ~]# nmcli con add type team con-name team0 ifname team0 \
> team.runner activebackup
```

2. 将地址 `192.168.0.100/24` 分配给 `team0` 连接。

```bash
[root@servera ~]# nmcli con mod team0 ipv4.addresses 192.168.0.100/24
[root@servera ~]# nmcli con mod team0 ipv4.method manual
```

3. 将 `eth1` and `eth2` 分配为 `team0` 连接的端口接口。

```bash
[root@servera ~]# nmcli con add type ethernet slave-type team \
con-name team0-port1 ifname eth1 master team0

[root@servera ~]# nmcli con add type ethernet slave-type team \
con-name team0-port2 ifname eth2 master team0
```

4. 激活 `team0` 连接和两个从端口

```bash
[root@servera ~]# nmcli con up team0

[root@servera ~]# nmcli con up team0-port1

[root@servera ~]# nmcli con up team0-port2
```

5. 显示合作接口的状态。

```bash
[root@servera ~]# teamdctl team0 state
setup:
  runner: activebackup
ports:
  eth1
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
  eth2
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
runner:
  active port: eth1
```

### 验证网络通信

通过 team0 接口对 192.168.0.0/24 网络上的网络网关 192.168.0.254 进行 Ping 操作

```bash
[root@servera ~]# ping -I team0 -c 4 192.168.0.254
```

使用 tcpdump 命令来监控活动的接口

```bash
[root@servera ~]# tcpdump -i eth1
```

# 管理网络合作

## 网络合作配置文件

NetworkManager 在 /etc/sysconfig/network-scripts 目录中为网络合作创建配置文件

```ini
[root@servera ~]# cat /etc/sysconfig/network-scripts/ifcfg-team0
TEAM_CONFIG="{ \"runner\": { \"name\": \"activebackup\" } }"
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=team0
UUID=5a5503fd-61ea-4c20-888a-f0f2f1117819
DEVICE=team0
ONBOOT=yes
DEVICETYPE=Team
IPADDR=192.168.0.100
PREFIX=24
```

定义辅助端口

```ini
[root@servera ~]# cat /etc/sysconfig/network-scripts/ifcfg-team0-port1
NAME=team0-port1
UUID=0d56248e-dd59-4bbb-927a-2ce772927393
DEVICE=eth1
ONBOOT=yes
TEAM_MASTER=team0
DEVICETYPE=TeamPort
```

## 设置和调整组配置

nmcli con mod 命令将分配不同的运行程序参数。

```bash
[root@servera ~]# nmcli con mod team0 team.runner activebackup
[root@servera ~]# teamdctl team0 state
setup:
  runner: activebackup
```

## 故障排除

显示 team0 组接口的端口接口：

```bash
[root@servera ~]# teamnl team0 ports
 4: eth2: up 4294967295Mbit FD
 3: eth1: up 4294967295Mbit FD
```

 显示 team0 的当前活动端口

```bash
[root@servera ~]# teamnl team0 getoption activeport
3
```

更改 team0 的活动端口

```bash
[root@servera ~]# teamdctl team0 state item set runner.active_port eth2
[root@servera ~]# teamnl team0 getoption activeport
4
```

显示 team0 的当前状态

```bash
[root@servera ~]# teamdctl team0 state
setup:
  runner: activebackup
ports:
  eth1
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
  eth2
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
runner:
  active port: eth2
```

显示 team0 的当前 JSON 配置

```json
[root@servera ~]# teamdctl team0 config dump
{
    "device": "team0",
    "mcast_rejoin": {
        "count": 1
    },
    "notify_peers": {
        "count": 1
    },
    "ports": {
        "eth1": {
            "link_watch": {
                "name": "ethtool"
            }
        },
        "eth2": {
            "link_watch": {
                "name": "ethtool"
            }
        }
    },
    "runner": {
        "name": "activebackup"
    }
}
```

# 自动化网络合作

## 安装系统角色软件包

```bash
[root@servera ~]# yum install rhel-system-roles -y
```

## 创建网络组

定义一个 network_connections 变量，用于创建名为 team0 的组配置集，其接口名为 team0。为该团队分配静态 IP 地址 192.168.0.100/24，并禁用 DHCPv4 和 DHCPv6

```yaml
---

- name: Configure team network device
  hosts: servers
  become: true

  vars:
    network_connections:

      # Create a team profile
      - name: team0
        state: up
        type: team
        interface_name: team0
        ip:
          dhcp4: no
          auto6: no
          address:
            - "192.168.0.100/24"

  roles:
    - rhel-system-roles.network
```

## 创建从接口

第一个从接口应命名为 `team0-port1`，并绑定到 `eth1` 以太网接口。第二个从接口应命名为 `team0-port2`，并绑定到 `eth2` 以太网接口。

```yaml
---

- name: Configure team network device
  hosts: servers
  become: true

  vars:
    network_connections:

      # Create a team profile
      - name: team0
        state: up
        type: team
        interface_name: team0
        ip:
          dhcp4: no
          auto6: no
          address:
            - "192.168.0.100/24"

      # enslave an ethernet to the team
      - name: team0-port1
        state: up
        type: ethernet
        interface_name: eth1
        master: team0

      # enslave an ethernet to the team
      - name: team0-port2
        state: up
        type: ethernet
        interface_name: eth2
        master: team0

  roles:
    - rhel-system-roles.network
```

检查 `playbook.yml` 文件中的语法错误，然后运行该 playbook。

```bash
ansible-playbook --syntax-check playbook.yml
ansible-playbook playbook.yml
```

## 调整运行程序

```yaml
---

- name: Tune team network device
  hosts: servers
  become: true

  tasks:
    - name: Tune team runner to activebackup
      command: nmcli con mod team0 team.runner activebackup
```


