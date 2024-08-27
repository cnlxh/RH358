```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 章节目标

- 使用 iSCSI 协议，为网络客户端提供基于块的存储。

- 配置 iSCSI 启动器以访问基于网络的块设备，使用文件系统格式化新的 iSCSI 设备，将其配置为在启动时使用，并且能够安全地停止使用现有的 iSCSI 块设备。

- 自动配置服务器上的 iSCSI 启动器。

# 提供 iSCSI 存储

*iSCSI* 是一种<mark>基于 TCP/IP </mark>的协议，用于通过基于 IP 的网络发送 SCSI 命令。它允许通过网络将硬盘设备从客户端连接到服务器。作为 *存储区域网络 (SAN)* 协议，通常，iSCSI 与专用的 <mark>10G以太网</mark>或更好的网络搭配，以最大程度<mark>提高性能</mark>。

## iSCSI 关键术语

iSCSI 协议以客户端/服务器配置的方式运行。客户端系统将启动器软件配置为将 SCSI 命令发送到远程服务器存储目标。在客户端系统上，iSCSI 目标显示为本地 SCSI 磁盘

`启动器`

一个 iSCSI 客户端，通常以软件提供。也可以购买硬件启动器(HBA)。必须为启动器<mark>授予唯一名称</mark>（请参见 IQN）。

`目标`

iSCSI 服务器上的 iSCSI 存储资源。必须为目标授予唯一名称（请参见 IQN）。每个目标提供一个或多个块设备，或逻辑单元。在大多数情况下，目标恰好提供一个设备。单个服务器可以提供多个目标。

`IQN（iSCSI 限定名称）`

唯一的全球范围名称，用于识别启动器和目标。IQN 具有以下格式：

```textile
iqn.YYYY-MM.com.reversed.domain:name_string
```

`门户`

每个目标具有一个或多个门户，即启动器可能用来访问目标的 IP 地址和端口对。

`LUN （逻辑单元号）`

LUN 表示由目标提供的块设备。每个目标提供一个或多个 LUN。（因此，一个目标可以提供多个存储设备。）

`ACL（访问权限控制列表）`

使用启动器的 IQN 验证其访问权限的访问限制。

`TPG （目标门户组）`

TPG 是目标的完整配置，包括门户、LUN 和 ACL。几乎所有目标都使用一个 TPG，但高级配置有时可能会定义多个 TPG。

## 配置 iSCSI 目标

`targetcli` 命令既提供命令行实用程序，也提供一个交互式 shell，可以用来创建、删除和配置 iSCSI 目标。

`targetcli` 命令将目标对象整理为层级树，以便能够轻松进行浏览和上下文配置。

### 安装targetcli

```bash
[root@servera ~]# yum install targetcli -y
```

不要忘了启动服务，启用 `target` 服务，以在引导期间激活目标。

```bash
[root@servera ~]# systemctl enable --now target
```

```bash
[root@servera ~]# firewall-cmd --permanent --add-service=iscsi-target
[root@servera ~]# firewall-cmd --reload
```

### targetcli结构展示

```bash
[root@servera ~]# targetcli
Warning: Could not load preferences file /root/.targetcli/prefs.bin.
targetcli shell version 2.1.fb49
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.

/> ls /
o- / ........................................................................................................ [...]
  o- backstores ............................................................................................. [...]
  | o- block ................................................................................. [Storage Objects: 0]
  | o- fileio ................................................................................ [Storage Objects: 0]
  | o- pscsi ................................................................................. [Storage Objects: 0]
  | o- ramdisk ............................................................................... [Storage Objects: 0]
  o- iscsi ........................................................................................... [Targets: 0]
  o- loopback ........................................................................................ [Targets: 0]
/>
```

### 创建iSCSI后端

 /dev/vdb 存储设备创建成基于块的后备存储

```text
/> cd /backstores/block
/backstores/block> create lxhvdb /dev/vdb
Created block storage object lxhvdb using /dev/vdb.
/backstores/block> ls /
o- / ........................................................................................................ [...]
  o- backstores ............................................................................................. [...]
  | o- block ................................................................................. [Storage Objects: 1]
  | | o- lxhvdb ........................................................ [/dev/vdb (5.0GiB) write-thru deactivated]
  | |   o- alua .................................................................................. [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ...................................................... [ALUA state: Active/optimized]
```

### 创建iSCSI的IQN

```text
/backstores/block> cd /iscsi
/iscsi> create iqn.2024-09.com.example.lab:disk1
Created target iqn.2024-09.com.example.lab:disk1.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
/iscsi> ls /
o- / ........................................................................................................ [...]
  o- backstores ............................................................................................. [...]
  | o- block ................................................................................. [Storage Objects: 1]
  | | o- lxhvdb ........................................................ [/dev/vdb (5.0GiB) write-thru deactivated]
  | |   o- alua .................................................................................. [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ...................................................... [ALUA state: Active/optimized]
  | o- fileio ................................................................................ [Storage Objects: 0]
  | o- pscsi ................................................................................. [Storage Objects: 0]
  | o- ramdisk ............................................................................... [Storage Objects: 0]
  o- iscsi ........................................................................................... [Targets: 1]
  | o- iqn.2024-09.com.example.lab:disk1 ................................................................ [TPGs: 1]
  |   o- tpg1 .............................................................................. [no-gen-acls, no-auth]
  |     o- acls ......................................................................................... [ACLs: 0]
  |     o- luns ......................................................................................... [LUNs: 0]
  |     o- portals ................................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 .................................................................................... [OK]
  o- loopback ........................................................................................ [Targets: 0]
/iscsi>
```

### 添加后端块设备到iSCSI Luns

```text
/iscsi> cd /iscsi/iqn.2024-09.com.example.lab:disk1/tpg1/luns
/iscsi/iqn.20...sk1/tpg1/luns> create /backstores/block/lxhvdb
Created LUN 0.
/iscsi/iqn.20...sk1/tpg1/luns> ls /
o- / ........................................................................................................ [...]
  o- backstores ............................................................................................. [...]
  | o- block ................................................................................. [Storage Objects: 1]
  | | o- lxhvdb .......................................................... [/dev/vdb (5.0GiB) write-thru activated]
  | |   o- alua .................................................................................. [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ...................................................... [ALUA state: Active/optimized]
  | o- fileio ................................................................................ [Storage Objects: 0]
  | o- pscsi ................................................................................. [Storage Objects: 0]
  | o- ramdisk ............................................................................... [Storage Objects: 0]
  o- iscsi ........................................................................................... [Targets: 1]
  | o- iqn.2024-09.com.example.lab:disk1 ................................................................ [TPGs: 1]
  |   o- tpg1 .............................................................................. [no-gen-acls, no-auth]
  |     o- acls ......................................................................................... [ACLs: 0]
  |     o- luns ......................................................................................... [LUNs: 1]
  |     | o- lun0 .................................................... [block/lxhvdb (/dev/vdb) (default_tg_pt_gp)]
  |     o- portals ................................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 .................................................................................... [OK]
  o- loopback ........................................................................................ [Targets: 0]
/iscsi/iqn.20...sk1/tpg1/luns>
```

### 添加ACL

创建 ACL，以允许客户端启动器访问目标，这里要注意提前获取客户端的iqn是多少，获取办法为：

```bash
[root@serverd ~]# yum install iscsi-initiator-utils -y
[root@serverd ~]# cat /etc/iscsi/initiatorname.iscsi
InitiatorName=iqn.1994-05.com.redhat:12cfd76e8063
```

```text
/iscsi/iqn.20...sk1/tpg1/luns> cd /iscsi/iqn.2024-09.com.example.lab:disk1/tpg1/acls
/iscsi/iqn.20...sk1/tpg1/acls> create iqn.1994-05.com.redhat:12cfd76e8063
Created Node ACL for iqn.1994-05.com.redhat:12cfd76e8063
Created mapped LUN 0.
/iscsi/iqn.20...sk1/tpg1/acls> ls /
o- / ........................................................................................................ [...]
  o- backstores ............................................................................................. [...]
  | o- block ................................................................................. [Storage Objects: 1]
  | | o- lxhvdb .......................................................... [/dev/vdb (5.0GiB) write-thru activated]
  | |   o- alua .................................................................................. [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ...................................................... [ALUA state: Active/optimized]
  | o- fileio ................................................................................ [Storage Objects: 0]
  | o- pscsi ................................................................................. [Storage Objects: 0]
  | o- ramdisk ............................................................................... [Storage Objects: 0]
  o- iscsi ........................................................................................... [Targets: 1]
  | o- iqn.2024-09.com.example.lab:disk1 ................................................................ [TPGs: 1]
  |   o- tpg1 .............................................................................. [no-gen-acls, no-auth]
  |     o- acls ......................................................................................... [ACLs: 1]
  |     | o- iqn.1994-05.com.redhat:12cfd76e8063 ................................................. [Mapped LUNs: 1]
  |     |   o- mapped_lun0 ............................................................... [lun0 block/lxhvdb (rw)]
  |     o- luns ......................................................................................... [LUNs: 1]
  |     | o- lun0 .................................................... [block/lxhvdb (/dev/vdb) (default_tg_pt_gp)]
  |     o- portals ................................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 .................................................................................... [OK]
  o- loopback ........................................................................................ [Targets: 0]
```

### 配置门户

  默认门户侦听 iSCSI 服务器上的所有网络接口，这个可以改

```bash
/iscsi/iqn.20.../tpg1/portals> delete 0.0.0.0 3260
Deleted network portal 0.0.0.0:3260
/iscsi/iqn.20.../tpg1/portals> create 172.25.250.10 3260
Using default IP port 3260
Created network portal 172.25.250.10:3260.
/iscsi/iqn.20.../tpg1/portals> ls /
o- / ........................................................................................................ [...]
o- backstores ............................................................................................. [...]
| o- block ................................................................................. [Storage Objects: 1]
| | o- lxhvdb .......................................................... [/dev/vdb (5.0GiB) write-thru activated]
| |   o- alua .................................................................................. [ALUA Groups: 1]
| |     o- default_tg_pt_gp ...................................................... [ALUA state: Active/optimized]
| o- fileio ................................................................................ [Storage Objects: 0]
| o- pscsi ................................................................................. [Storage Objects: 0]
| o- ramdisk ............................................................................... [Storage Objects: 0]
o- iscsi ........................................................................................... [Targets: 1]
| o- iqn.2024-09.com.example.lab:disk1 ................................................................ [TPGs: 1]
|   o- tpg1 .............................................................................. [no-gen-acls, no-auth]
|     o- acls ......................................................................................... [ACLs: 1]
|     | o- iqn.1994-05.com.redhat:12cfd76e8063 ................................................. [Mapped LUNs: 1]
|     |   o- mapped_lun0 ............................................................... [lun0 block/lxhvdb (rw)]
|     o- luns ......................................................................................... [LUNs: 1]
|     | o- lun0 .................................................... [block/lxhvdb (/dev/vdb) (default_tg_pt_gp)]
|     o- portals ................................................................................... [Portals: 1]
|       o- 172.25.250.10:3260 .............................................................................. [OK]
o- loopback ........................................................................................ [Targets: 0]
```

### 保存并退出

  命令会将配置保存在 /etc/target/saveconfig.json 文件中。当 systemd 在引导时启动 target 服务时，它将使用该文件来配置目标。

```bash
exit
```

### 非交互式管理iSCSI

与 `targetcli` 的交互式使用不同，命令行模式不会自动将配置保存在 `/etc/target/saveconfig.json` 文件中。您必须明确地运行 `saveconfig` 子命令来保存您的配置。

```bash
[root@serverb ~]# targetcli /backstores/block create myblock1 /dev/vdb
Created block storage object myblock1 using /dev/vdb.
[root@serverb ~]# targetcli /iscsi create iqn.2014-06.com.example:disk1
Created target iqn.2014-06.com.example:disk1.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
[root@serverb ~]# targetcli /iscsi/iqn.2014-06.com.example:disk1/tpg1/luns \
> create /backstores/block/myblock1
Created LUN 0.
[root@serverb ~]# targetcli /iscsi/iqn.2014-06.com.example:disk1/tpg1/acls \
> create iqn.1994-05.com.redhat:b3d05c75ec7
Created Node ACL for iqn.1994-05.com.redhat:b3d05c75ec7
Created mapped LUN 0.
[root@serverb ~]# targetcli /iscsi/iqn.2014-06.com.example:disk1/tpg1/portals \
> delete 0.0.0.0 3260
Deleted network portal 0.0.0.0:3260
[root@serverb ~]# targetcli /iscsi/iqn.2014-06.com.example:disk1/tpg1/portals \
> create 192.168.0.10 3260
Using default IP port 3260
Created network portal 172.25.250.10:3260.
[root@serverb ~]# targetcli saveconfig
Configuration saved to /etc/target/saveconfig.json
```

# 访问 iSCSI 存储

## 安装客户端

`/etc/iscsi/initiatorname.iscsi` 文件中放的是 IQN

`/etc/iscsi/iscsid.conf` 文件包含您要连接的目标的默认设置。这些设置包括 iSCSI 超时、重试参数和身份验证用户名及密码。

```bash
[root@serverd ~]# yum install iscsi-initiator-utils -y
```

软件包安装会自动配置 `iscsi` 和 `iscsid` 服务，以便在系统引导时启动器自动重新连接到任何已发现的目标。每当您修改启动器的配置文件时，请重新启动 `iscsid` 服务。

## 发现iSCSI 目标

首先需要发现目标，然后才能连接到并使用远程设备。发现过程将目标信息和设置存储在 `/var/lib/iscsi/nodes/` 目录中，并且使用 `/etc/iscsi/iscsid.conf` 中的默认值。

参数太长，记不住就`man iscsiadm`

```bash
[root@serverd ~]# iscsiadm -m discovery -t st -p servera
172.25.250.10:3260,1 iqn.2024-09.com.example.lab:disk1
```

## 登录iSCSI目标

如果这里登录失败，要看客户端的IQN和服务器配置的ACL是否匹配

```bash
[root@serverd ~]# iscsiadm -m node -T iqn.2024-09.com.example.lab:disk1 -p servera -l
Logging in to [iface: default, target: iqn.2024-09.com.example.lab:disk1, portal: 172.25.250.10,3260]
Login to [iface: default, target: iqn.2024-09.com.example.lab:disk1, portal: 172.25.250.10,3260] successful.
```

## 查询iSCSI会话信息

```bash
[root@serverd ~]# iscsiadm -m session -P 1
Target: iqn.2024-09.com.example.lab:disk1 (non-flash)
        Current Portal: 172.25.250.10:3260,1
        Persistent Portal: 172.25.250.10:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.1994-05.com.redhat:12cfd76e8063
                Iface IPaddress: 172.25.250.13
                Iface HWaddress: default
                Iface Netdev: default
                SID: 1
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
```

## 查询iSCSI硬盘

多了一个sda

```bash
[root@serverd ~]# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0   5G  0 disk
vda    252:0    0  10G  0 disk
└─vda1 252:1    0  10G  0 part /
vdb    252:16   0   5G  0 disk
```

## 格式化挂载

使用 /etc/fstab 中的 _netdev 挂载点。由于 iSCSI 依靠网络访问远程设备，此选项可确保系统不会尝试挂载文件系统，直到网络和启动器启动为止。

在fstab中，建议用uuid

```bash
[root@serverd ~]# mkfs.xfs /dev/sda
[root@serverd ~]# mount /dev/sda /mnt
[root@serverd ~]# tail -n1 /etc/fstab
/dev/sda /mnt xfs defaults,_netdev 0 0
[root@serverd ~]# mount -a
[root@serverd ~]# df -h | grep mnt
/dev/sda            5.0G   68M  5.0G   2% /mnt
[root@serverd ~]# mount -l | grep mnt
/dev/sda on /mnt type xfs (rw,relatime,seclabel,attr2,inode64,noquota)
```

## 断开与目标的连接

- 确保没有使用目标所提供的任何设备。例如，解除挂载文件系统。

- 从 `/etc/fstab` 等位置中删除对目标的所有持久引用。

- 从 iSCSI 目标注销。

- 删除 iSCSI 目标的本地记录，使启动器在启动过程中不会自动登录到目标。

```bash
[root@serverd ~]# umount /mnt
[root@serverd ~]# vim /etc/fstab
[root@serverd ~]# iscsiadm -m node -T iqn.2024-09.com.example.lab:disk1 -p servera -u
Logging out of session [sid: 1, target: iqn.2024-09.com.example.lab:disk1, portal: 172.25.250.10,3260]
Logout of [sid: 1, target: iqn.2024-09.com.example.lab:disk1, portal: 172.25.250.10,3260] successful.
[root@serverd ~]# iscsiadm -m node -T iqn.2024-09.com.example.lab:disk1 -p servera -o delete
```

# 自动化执行 iSCSI 启动器配置

详见教材


