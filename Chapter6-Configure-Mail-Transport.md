```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 章节目标

- 配置服务器，以通过出站 SMTP 网关发送所有电子邮件。
- 自动配置 Postfix，以通过 SMTP 网关发送所有出站电子邮件。

# 配置仅发送电子邮件服务

在大多数包含发送电子邮件消息的服务的系统上，管理员将 Postfix 配置为空客户端。空客户端运行一个本地邮件服务，其配置为使用 SMTP 协议将其发送的所有电子邮件消息转发到出站邮件中继以进行发送。该中继根据域的 MX DNS 记录将邮件传输到收件人域的邮件服务器。收件人的邮件服务器随后将其传递到用户的邮箱。

## 使用 Postfix 发送电子邮件

Postfix 是一种强大且易于配置的邮件服务器，由 postfix RPM 软件包提供。它们通过名为 `master` 的守护进程进行协调。最重要的 Postfix 配置文件为 `/etc/postfix/main.cf`。`/etc/postfix` 目录还包含其他 Postfix 配置文件。例如，另一个重要的配置文件 `/etc/postfix/master.cf` 控制由 `master` 进程启动的子服务。

## 将 Postfix 配置为空客户端

要充当空客户端，必须将主机上的 Postfix 配置为具有如下行为：

- 必须将使用 Postfix 的 `sendmail` 帮助程序发送的所有电子邮件都移交给另一台服务器上的现有邮件中继。邮件中继处理每封电子邮件，以传输到每个邮件收件人的目标服务器。

- 本地 Postfix 服务不得接受任何电子邮件进行本地发送。这些消息必须被拒绝，或者，如果源于本地，则必须发送到邮件中继以在组织的现有中央邮件服务器上传送。

**Postfix 空客户端配置**

| 设置                | 值和用途                                                                                                                                                                                                                                                                                                 |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `inet_interfaces` | 设为 `loopback-only`。这会将 Postfix 配置为仅在地址 `127.0.0.1` 或 `::1` 上监听端口 25/TCP 上的新电子邮件消息。（因此，仅适用于由空客户端本身发送的消息。）                                                                                                                                                                                             |
| `mynetworks`      | 设为 `127.0.0.0/8 [::1]/128`。这会将 Postfix 配置为将消息提交到 `mynetworks` 设置的值列表中任何网络上的主机的邮件中继。此列表中的项目可以用逗号或空格分隔。                                                                                                                                                                                                |
| `myorigin`        | 设置为 DNS 域，此系统的用户发送的电子邮件应显示为来自该域。这会将发件人电子邮件地址的一部分重写到 `@` 符号的右侧，并帮助确保对这些电子邮件的回复转到正确的邮件服务器。例如，`myorigin = example.com` 将导致此系统上的用户 `alerts` 发送的电子邮件看起来像是由 `alerts@example.com` 发送的。                                                                                                                      |
| `relayhost`       | 设为您的邮件中继的主机名（用方括号括起）。这会将所有传出消息提交到该中继以进行发送。例如，`relayhost = [smtp.example.com]` 会将所有消息提交到 `smtp.example.com` 上的中继。默认情况下，消息将以纯文本形式提交到邮件中继上的端口 25/TCP，而不进行身份验证。如果中继通过 IP 地址对其客户端进行身份验证，并且不需要进一步身份验证，则此操作有效。（请参见下文中关于已验证中继的进一步讨论）。如果您省略邮件中继名称两边的方括号，Postfix 将尝试对包含该名称的 MX 记录执行 DNS 查找，以查找该中继，这可能不是您预期的行为。 |
| `mydestination`   | 将它设置为具有空字符串值。如果邮件发送给 `mydestination` 域中的域列表上的用户，它将被 Postfix 接受以发送到本地邮箱。由于空客户端不接受电子邮件进行本地发送，因此这应该是空的。如果您根本没有在配置文件中设置 `mydestination` （或使用 `postconf -X mydestination` 将其删除），则会将其设置为默认值，以接受发送给本地主机名或 `localhost` 的邮件，这不是您想要的行为。                                                                        |

## 将电子邮件提交到经过身份验证的中继

假设您的邮件中继接受用户名和密码身份验证，则您需要对空客户端上的 Postfix 配置进行以下更改：

- `relayhost = [smtp.example.com]:587`（将 `:587` 添加到您的 `relayhost` 值）

- `smtp_use_tls = yes` 以使用 TLS

- `smtp_sasl_auth_enable = yes` 以使用身份验证

- `smtp_sasl_security_options = noanonymous` 以要求身份验证并允许将用户名/密码用作方法

- `smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd` 使用 `/etc/postfix/sasl_passwd` 以存储用户名和密码以进行身份验证



编辑您的 `/etc/postfix/sasl_passwd` 文件，以包含 `relayhost` 的定义，以及对 MSA 进行身份验证所需的以冒号分隔的用户名和密码。在本示例中，`relayuser@example.com` 是用户名，`mysecretpassword` 是密码，用于中继邮件：

```bash
[smtp.example.com]:587 relayuser@example.com:mysecretpassword
```

运行 `postmap /etc/postfix/sasl_passwd`，以使用您的密码更新 Postfix 配置。确保 `postmap` 创建的 `/etc/postfix/sasl_passwd` 和 `/etc/postfix/sasl_passwd.db` 文件归用户 `root`、组 `root` 所有，并且仅可由 `root` 读取。

## 准备基础条件

我们没有邮箱服务器来帮我们中继邮件的发送，所以要打一个lab来自动化完成邮箱服务器的构建，下面这个lab命令可确保 DNS、SMTP 和 IMAP 服务为 `Lab.example.com` SMTP 中继服务器提供支持

```bash
[student@workstation ~]$ lab smtp-sendonly start

Starting smtp-sendonly exercise.

 · Installing the rhel-system-roles package on workstation.....  SUCCESS
 · Download exercise playbooks.................................  SUCCESS
 · Run exercise preparation playbook...........................  SUCCESS
```

## 安装postfix

```bash
[root@servera ~]# yum install postfix -y
```

## 语法介绍

使用 `postconf -e setting = value` 命令更改 `/etc/postfix/main.cf` 设置。此命令使用指定的值编辑现有设置，否则它将使用新设置在配置文件的底部添加一行。

## 配置中继服务器

使用 `relayhost` 指令配置 Postfix，以将所有消息发送到公司邮件中继。将公司邮件中继的主机名括在方括号中，以防止使用 DNS 服务器来查找 MX 记录。

```bash
[root@servera ~]# postconf -e 'relayhost=[smtp.lab.example.com]'
[root@servera ~]# postconf | grep ^relayhost
relayhost = [smtp.lab.example.com]
```

## 配置监听地址

将 Postfix 配置为仅侦听环回接口

```bash
[root@servera ~]# postconf -e 'inet_interfaces=loopback-only'
[root@servera ~]# postconf | grep ^inet_interfaces
inet_interfaces = loopback-only
```

## 配置信任网络源

这个参数通常用于确定从哪些网络来源的连接被认为是“内部”或“可信任”的，从而允许这些网络的客户端发送邮件

```bash
[root@servera ~]# postconf -e 'mynetworks=127.0.0.0/8 [::1]/128'
[root@servera ~]# postconf | grep ^mynetworks
mynetworks = 127.0.0.0/8 [::1]/128
```

## 配置邮件域名

发件人电子邮件地址重后缀为lab.example.com

```bash
[root@servera ~]# postconf -e 'myorigin=lab.example.com'
[root@servera ~]# postconf | grep ^myorigin
myorigin = lab.example.com
```

## 配置不允许发送到本地

禁止 Postfix 空客户端将任何邮件发送到本地帐户，将空客户端配置为将所有邮件转发到中继服务器。将 mydestination 选项设置为空值以实现此目的。

```bash
[root@servera ~]# postconf | grep ^mydestination
mydestination =
```

## 启动postfix服务

```bash
[root@servera ~]# systemctl enable --now postfix
```

## 查询所有配置

运行不带参数的 postconf 将显示所有 /etc/postfix/main.cf 设置。

```bash
[root@servera ~]# postconf
```

当然，后面跟上参数可以查询具体的值

```bash
[root@servera ~]# postconf inet_interfaces myorigin mydestination mynetworks relayhost
inet_interfaces = loopback-only
myorigin = lab.example.com
mydestination =
mynetworks = 127.0.0.0/8 [::1]/128
relayhost = [smtp.lab.example.com]
```

## 发送邮件

需要注意的是，当你输入了命令后，会出现类似卡着不动的的情况，这是在等你输入邮件的正文，输入正文后回车，并输入英文的句号，即可完成邮件的书写并发送

```bash
[root@servera ~]# mail -s 'hello lixiaohui' student@lab.example.com
hi i'm Xiaohui Li, My WeXin is lxh_chat
.
EOT
```

## 接收邮件

使用 IMAPS 协议，通过 mutt 命令行电子邮件客户端，在 imap.lab.example.com 上读取 student 的邮件。
此命令会提示您接受主机 SSL 证书，然后提示您输入用户名和密码。键入 a 以永久接受该证书。

```bash
[root@servera ~]# mutt -f imaps://imap.lab.example.com
...
Username at imap.lab.example.com: student
Password for student@imap.lab.example.com:
...
q:Quit  d:Del  u:Undel  s:Save  m:Mail  r:Reply  g:Group  ?:Help
   1 N   Aug 26 root            (0.6K) hello lixiaohui
# 这里可以看到邮件，回车即可看到邮件内容，看完之后，按q即可退出邮件客户端
i:Exit  -:PrevPg  <Space>:NextPg v:View Attachm.  d:Del  r:Reply  j:Next ?:Help
Date: Mon, 26 Aug 2024 17:46:47 +0800
From: root <root@lab.example.com>
To: student@lab.example.com
Subject: hello lixiaohui
User-Agent: Heirloom mailx 12.5 7/5/10

hi i'm Xiaohui Li, My WeXin is lxh_chat
```

# 自动化执行 Postfix 配置

详见教材
