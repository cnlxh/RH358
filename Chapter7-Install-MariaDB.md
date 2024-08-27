```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 章节目标

- 安装 MariaDB 关系数据库服务器并执行其基本配置。

- 使用结构化查询语言 (SQL) 和 MariaDB 语句来检查、搜索、创建和更改数据库信息。

- 在 MariaDB 中配置数据库用户，并为它们分配访问权限。

- 安全地备份 MariaDB 数据库并恢复该备份。

- 自动化安装和配置 MariaDB 数据库服务器。

# 安装 MariaDB 数据库

## MariaDB 数据库描述

关系数据库管理系统 (RDBMS)是用于管理关系数据库的软件。大多数关系数据库管理系统都允许您使用结构化查询语言 (SQL)来查找和管理数据库中的数据。

很多人使用关系数据库来存储库存跟踪、销售和财务方面的业务信息。关系数据库管理系统也在多个应用程序堆栈中发挥了重要作用。例如，许多需要支持动态内容生成的 web 应用程序围绕 LAMP 解决方案堆栈构建，该堆栈包括：

- 用于提供基本环境的 Linux 操作系统。

- Apache HTTPS 服务器（或其他 web 服务器，如 Nginx）。

- MariaDB、MySQL 或其他关系数据库，如用于存储站点数据的 PostgreSQL。

- 由 web 服务器运行的一种编程语言，如 PHP、Python、Perl、Ruby、Java、服务器端 JavaScript 等，可以更新数据库中的数据，并使用它为用户动态构建网页。

## 安装 MariaDB

MariaDB 是一个由社区开发的 MySQL 分支，可与该数据库高度兼容以便能够轻松地从一个数据库过渡到另一个，并且使用广泛。

```bash
[root@servera ~]# yum module list mariadb
Last metadata expiration check: 0:00:36 ago on Mon 26 Aug 2024 06:12:33 PM CST.
Red Hat Enterprise Linux 8.1 AppStream (dvd)
Name                                    Stream                                   Profiles                                                  Summary
mariadb                                 10.3 [d]                                 client, server [d], galera                                MariaDB Module

Hint: [d]efault, [e]nabled, [x]disabled, [i]nstalled
```

安装服务端和客户端

```bash
[root@servera ~]# yum module install mariadb:10.3/server -y
```

只安装客户端

```bash
[root@servera ~]# yum install mariadb -y
```

## 启动服务与防火墙

```bash
[root@servera ~]# systemctl enable --now mariadb
[root@servera ~]# firewall-cmd --permanent --add-service=mysql
[root@servera ~]# firewall-cmd --reload
```

## 保护 MariaDB 安装

新 MariaDB 服务的默认配置可能具有测试数据库和一些不太安全的配置设置。运行 `mysql_secure_installation`，以配置更加安全的默认值。

此交互式脚本会提示您进行某些更改，包括：

- 设置 `root` 帐户的密码。

- 删除可以从本地主机外部访问的 `root` 帐户。

- 删除 `anonymous-user` 帐户。

- 删除用于演示的 `test` 数据库（如果存在）。

```bash
[root@servera ~]# mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

## 管理数据库的网络访问

1. 如果客户端在与服务器相同的计算机上运行，则它可以连接到特殊的套接字文件，以与 MariaDB 通信。这更加安全，因为 MariaDB 不需要侦听来自网络客户端的连接。

2. 客户端可以使用 TCP/IP 网络连接 MariaDB 服务。远程客户端以及与 MariaDB 服务器在同一主机上运行的客户端可能使用此方法。如果启用此方法，则服务器默认侦听端口 3306/TCP 上的连接。

默认情况下，这两种方法都已启用。MariaDB 在系统的所有网络地址上侦听与 3306/TCP 的连接，并且套接字文件可用。

若要完全关闭 TCP/IP 网络并依赖于本地套接字文件，或者限制 MariaDB 将使用的网络地址，需要编辑 MariaDB 配置。其主配置文件是 `/etc/my.cnf`，但该文件也会自动包含 `/etc/my.conf.d` 目录中的所有文件作为配置文件的一部分。

可以通过向 `/etc/my.cnf.d/mariadb-server.cnf` 文件的 `[mysqld]` 部分添加指令来调整服务器的网络设置。

`bind-address`

此指令指定 MariaDB 用于侦听客户端连接的网络地址。仅可输入一个选项。可能的选项包括：

- 一个 IPv4 地址。

- 一个 IPv6 地址。

- `::` 用于连接所有可用地址（IPv6 和 IPv4）。

- 将所有 IPv4 地址留空（或设置为 `0.0.0.0`）。

如果您希望本地客户端能够在不允许远程访问 MariaDB 的情况下使用网络连接，您可以使用 `127.0.0.1` 或 `::1` 作为网络地址。仅可使用一个 `bind-address` 条目。在具有多个地址的系统上，您可以使用此指令来选择所有地址或一个地址，但不能选择多个地址。

默认监听如下：

```bash
[root@servera ~]# ss -tulpn | grep mysqld
tcp    LISTEN   0        80                      *:3306                *:*       users:(("mysqld",pid=29476,fd=22))
```

修改监听到特定的IP

```bash
[root@servera ~]# vim /etc/my.cnf.d/server.cnf
[mysqld]
bind-address = 172.25.250.10
```

重启服务后再次获取监听地址发现已经更改

```bash
[root@servera ~]# systemctl restart mariadb.service
[root@servera ~]# ss -tulpn | grep mysqld
tcp    LISTEN   0        80          172.25.250.10:3306          0.0.0.0:*       users:(("mysqld",pid=30223,fd=22))
```

`skip-networking`

如果您在配置文件的 `[mysqld]` 部分中设置了 `skip-networking` 或 `skip-networking=1`，则网络将被禁用，且客户端必须使用套接字文件与 MariaDB 通信。

```bash
[root@servera ~]# vim /etc/my.cnf.d/server.cnf
[mysqld]
bind-address = 172.25.250.10
skip-networking = 1
```

发现已经不再监听3306

```bash
[root@servera ~]# systemctl restart mariadb.service
[root@servera ~]# ss -tulpn | grep mysqld
[root@servera ~]# ss -tulpn | grep 3306
[root@servera ~]#
```

但是还可以用套接字通信

```bash
[root@servera ~]# mysql -uroot -predhat
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 8
Server version: 10.3.17-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> exit
Bye
```

`port`

可以使用此设置来指定除 3306/TCP 之外的网络端口，比如我指定了2000，注意需要同时关注selinux

```bash
[root@servera ~]# vim /etc/my.cnf.d/server.cnf
[mysqld]
bind-address = 172.25.250.10
port = 3000
```

看到selinux只允许特定的端口范围，没有我们的3000

```bash
[root@servera ~]# semanage port -l | grep mysql
mysqld_port_t                  tcp      1186, 3306, 63132-63164
```

添加selinux允许3000端口

```bash
[root@servera ~]# semanage port -a -t mysqld_port_t -p tcp 3000
```

的确监听在3000端口

```bash
[root@servera ~]# systemctl restart mariadb
[root@servera ~]# ss -tulpn | grep mysqld
tcp    LISTEN   0        80          172.25.250.10:3000          0.0.0.0:*       users:(("mysqld",pid=30990,fd=21))
```

```bash
[root@servera ~]# mysql -uroot -predhat -P 3000
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 8
Server version: 10.3.17-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> exit;
Bye
```

# 在 MariaDB 中使用 SQL

## 访问 MariaDB 数据库

mariadb 软件包提供了命令 mysql，它支持对 MariaDB 数据库的交互式和非交互访问。

用root用户以及密码redhat在3000端口上连接localhost这个数据库服务器

```bash
[root@servera ~]# mysql -uroot -predhat -h localhost -P 3000
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 10
Server version: 10.3.17-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> exit
Bye
```

## 查询数据库列表

`mysql` 数据库是一种系统数据库，其中包含用于操作 MariaDB 的信息，包括数据库用户及其访问权限的表。另外两个数据库也由 MariaDB 用于支持其自身的操作。

SQL 语句不区分大小写，但数据库名称区分大小写。通常的做法是设置具有全部小写的名称的数据库，并使用大写的 SQL 语句将语句的 SQL 语法与语句的目标或参数区分开来，但是由于不区分大小写，语句部分小写也没问题。

<mark>在MariaDB中，记得封号结尾</mark>

```sql
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.002 sec)
```

## 切换数据库

```sql
MariaDB [(none)]> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [mysql]>
```

## 创建新数据库

通过测试，发现语句大写和小写都可以，根据自己的习惯来即可

```sql
MariaDB [mysql]> CREATE DATABASE db1;
Query OK, 1 row affected (0.001 sec)

MariaDB [mysql]> USE db1;
Database changed
MariaDB [db1]>
```

## 查询数据库

我们现在没有数据库，在workstation上打以下lab帮我们准备

切换到数据库查询，或者用from查询都可以

```sql
MariaDB [(none)]> show tables from inventory;
+---------------------+
| Tables_in_inventory |
+---------------------+
| category            |
| manufacturer        |
| product             |
+---------------------+
3 rows in set (0.000 sec)

MariaDB [(none)]> use inventory;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [inventory]> show tables;
+---------------------+
| Tables_in_inventory |
+---------------------+
| category            |
| manufacturer        |
| product             |
+---------------------+
3 rows in set (0.000 sec)
```

## 查询数据表

### 查询表中的所有数据

表由行和列组成。在给定的表中，每个行对应一个记录，而每个列对应该记录的某个属性。

可以使用 `select` 语句从表中选择记录。在以下示例中，`SELECT *` 表示选择所有属性，`FROM` 指定要从哪一个表中进行选择。

```sql
MariaDB [inventory]> select * from product;
+----+-------------------+---------+-------+-------------+-----------------+
| id | name              | price   | stock | id_category | id_manufacturer |
+----+-------------------+---------+-------+-------------+-----------------+
|  1 | ThinkServer TS140 |  539.88 |    20 |           2 |               4 |
|  2 | ThinkServer RD630 | 2379.14 |    20 |           2 |               4 |
|  3 | RT-AC68U          |  219.99 |    10 |           1 |               3 |
|  4 | X110 64GB         |   73.84 |   100 |           3 |               1 |
+----+-------------------+---------+-------+-------------+-----------------+
4 rows in set (0.001 sec)

MariaDB [inventory]> select * from category;
+----+------------+
| id | name       |
+----+------------+
|  1 | Networking |
|  2 | Servers    |
|  3 | Ssd        |
+----+------------+
3 rows in set (0.000 sec)

MariaDB [inventory]> select * from manufacturer;
+----+----------+----------------+-------------------+
| id | name     | seller         | phone_number      |
+----+----------+----------------+-------------------+
|  1 | SanDisk  | John Miller    | +1 (941) 329-8855 |
|  2 | Kingston | Mike Taylor    | +1 (341) 375-9999 |
|  3 | Asus     | Wilson Jackson | +1 (432) 367-8899 |
|  4 | Lenovo   | Allen Scott    | +1 (876) 213-4439 |
+----+----------+----------------+-------------------+
4 rows in set (0.000 sec)
```

### 查询特定表中的特定的列

```sql
MariaDB [inventory]> select name,price,stock from product;
+-------------------+---------+-------+
| name              | price   | stock |
+-------------------+---------+-------+
| ThinkServer TS140 |  539.88 |    20 |
| ThinkServer RD630 | 2379.14 |    20 |
| RT-AC68U          |  219.99 |    10 |
| X110 64GB         |   73.84 |   100 |
+-------------------+---------+-------+
4 rows in set (0.004 sec)

MariaDB [inventory]> select * from product where price > 100;
+----+-------------------+---------+-------+-------------+-----------------+
| id | name              | price   | stock | id_category | id_manufacturer |
+----+-------------------+---------+-------+-------------+-----------------+
|  1 | ThinkServer TS140 |  539.88 |    20 |           2 |               4 |
|  2 | ThinkServer RD630 | 2379.14 |    20 |           2 |               4 |
|  3 | RT-AC68U          |  219.99 |    10 |           1 |               3 |
+----+-------------------+---------+-------+-------------+-----------------+
3 rows in set (0.001 sec)
```

常见条件运算符

| 运算符     | 描述                            |
| ------- | ----------------------------- |
| =       | 等于                            |
| < >     | 不等于注：在某些版本的 SQL 中，此运算符可能写作 != |
| >       | 大于                            |
| <       | 小于                            |
| >=      | 大于等于                          |
| <=      | 小于等于                          |
| BETWEEN | 介于某个闭区间                       |
| LIKE    | 搜索某种模式                        |
| IN      | 指定某一列的多个可能值                   |

### like语法

```sql
MariaDB [inventory]> select * from product where name like 'Think%';
+----+-------------------+---------+-------+-------------+-----------------+
| id | name              | price   | stock | id_category | id_manufacturer |
+----+-------------------+---------+-------+-------------+-----------------+
|  1 | ThinkServer TS140 |  539.88 |    20 |           2 |               4 |
|  2 | ThinkServer RD630 | 2379.14 |    20 |           2 |               4 |
+----+-------------------+---------+-------+-------------+-----------------+
2 rows in set (0.000 sec)
```

### 等于语法

```sql
MariaDB [inventory]> select * from product where price = 2379.14;
+----+-------------------+---------+-------+-------------+-----------------+
| id | name              | price   | stock | id_category | id_manufacturer |
+----+-------------------+---------+-------+-------------+-----------------+
|  2 | ThinkServer RD630 | 2379.14 |    20 |           2 |               4 |
+----+-------------------+---------+-------+-------------+-----------------+
1 row in set (0.004 sec)
```

### BETWEEN语法

```sql
MariaDB [inventory]> select * from product where price between 70 and 300;
+----+-----------+--------+-------+-------------+-----------------+
| id | name      | price  | stock | id_category | id_manufacturer |
+----+-----------+--------+-------+-------------+-----------------+
|  3 | RT-AC68U  | 219.99 |    10 |           1 |               3 |
|  4 | X110 64GB |  73.84 |   100 |           3 |               1 |
+----+-----------+--------+-------+-------------+-----------------+
2 rows in set (0.003 sec)
```

### IN语法

**查询特定产品ID** - 假设你想查询产品ID为 1, 2, 或 3 的记录：

```sql
MariaDB [inventory]> SELECT * FROM product WHERE id IN (1, 2, 3);
+----+-------------------+---------+-------+-------------+-----------------+
| id | name              | price   | stock | id_category | id_manufacturer |
+----+-------------------+---------+-------+-------------+-----------------+
|  1 | ThinkServer TS140 |  539.88 |    20 |           2 |               4 |
|  2 | ThinkServer RD630 | 2379.14 |    20 |           2 |               4 |
|  3 | RT-AC68U          |  219.99 |    10 |           1 |               3 |
+----+-------------------+---------+-------+-------------+-----------------+
3 rows in set (0.004 sec)
```

**查询特定名称的产品** - 如果你想找到名称为 "ThinkServer TS140" 或 "RT-AC68U" 的产品：

```sql
MariaDB [inventory]> SELECT * FROM product WHERE name IN ('ThinkServer TS140', 'RT-AC68U');
+----+-------------------+--------+-------+-------------+-----------------+
| id | name              | price  | stock | id_category | id_manufacturer |
+----+-------------------+--------+-------+-------------+-----------------+
|  1 | ThinkServer TS140 | 539.88 |    20 |           2 |               4 |
|  3 | RT-AC68U          | 219.99 |    10 |           1 |               3 |
+----+-------------------+--------+-------+-------------+-----------------+
2 rows in set (0.001 sec)
```

**查询特定制造商的产品** - 假设制造商ID为 4 的产品由特定制造商生产：

```sql
MariaDB [inventory]> SELECT * FROM product WHERE id_manufacturer IN (4);
+----+-------------------+---------+-------+-------------+-----------------+
| id | name              | price   | stock | id_category | id_manufacturer |
+----+-------------------+---------+-------+-------------+-----------------+
|  1 | ThinkServer TS140 |  539.88 |    20 |           2 |               4 |
|  2 | ThinkServer RD630 | 2379.14 |    20 |           2 |               4 |
+----+-------------------+---------+-------+-------------+-----------------+
2 rows in set (0.000 sec)
```

**排除特定产品ID** - 使用 `NOT IN` 来查询除了产品ID为 1 和 2 之外的所有产品：

```sql
MariaDB [inventory]> SELECT * FROM product WHERE id NOT IN (1, 2);
+----+-----------+--------+-------+-------------+-----------------+
| id | name      | price  | stock | id_category | id_manufacturer |
+----+-----------+--------+-------+-------------+-----------------+
|  3 | RT-AC68U  | 219.99 |    10 |           1 |               3 |
|  4 | X110 64GB |  73.84 |   100 |           3 |               1 |
+----+-----------+--------+-------+-------------+-----------------+
2 rows in set (0.001 sec)
```

### 查询表结构

`product` 表的这一输出显示：

- 表中的项有六列（六个属性）。

- `Type` 列中显示了该属性的数据必须采用何种格式。例如，`stock` 属性必须是最多 11 位数字的整数。

- `Null` 列指示此属性是否可以为空。

- `Default` 列指示此属性是否设置了默认值（如果未指定值）。

- `Key` 列显示属性 `id` 是*主键*。主键是表中一行的唯一标识符；其他任何行都不可具有与该属性相同的值。该属性的 `Extra` 列被标记为 `auto_increment`。这意味着每次将新项插入到表中时，该条目的属性值会递增。这样更加便于保持数字主键的唯一性。

```sql
MariaDB [inventory]> DESCRIBE product;
+-----------------+--------------+------+-----+---------+----------------+
| Field           | Type         | Null | Key | Default | Extra          |
+-----------------+--------------+------+-----+---------+----------------+
| id              | int(11)      | NO   | PRI | NULL    | auto_increment |
| name            | varchar(100) | NO   |     | NULL    |                |
| price           | double       | NO   |     | NULL    |                |
| stock           | int(11)      | NO   |     | NULL    |                |
| id_category     | int(11)      | NO   |     | NULL    |                |
| id_manufacturer | int(11)      | NO   |     | NULL    |                |
+-----------------+--------------+------+-----+---------+----------------+
6 rows in set (0.002 sec)
```

## 向表中插入数据

向 `product` 表中插入一条新记录的。这条语句包含了五个字段：`name`、`price`、`stock`、`id_category` 和 `id_manufacturer`，以及它们对应的值。

SQL 插入语句的详细解释：

- `INSERT INTO product`：指定要插入数据的表名为 `product`。
- `(`：开始列名列表。
- `name`、`price`、`stock`、`id_category`、`id_manufacturer`：指定要插入数据的列名。
- `)`：结束列名列表。
- `VALUES`：指定要插入的值的开始。
- `('SDSSDP-128G-G25 2.5', 82.04, 30, 3, 1)`：为上述列指定具体的值。其中，`'SDSSDP-128G-G25 2.5'` 是产品名称，`82.04` 是产品价格，`30` 是库存数量，`3` 是分类ID，`1` 是制造商ID。
- `;`：结束 SQL 语句。

```sql
MariaDB [inventory]> INSERT INTO product (name,price,stock,id_category,id_manufacturer) VALUES ('SDSSDP-128G-G25 2.5',82.04,30,3,1);
Query OK, 1 row affected (0.004 sec)

MariaDB [inventory]> select * from product;
+----+---------------------+---------+-------+-------------+-----------------+
| id | name                | price   | stock | id_category | id_manufacturer |
+----+---------------------+---------+-------+-------------+-----------------+
|  1 | ThinkServer TS140   |  539.88 |    20 |           2 |               4 |
|  2 | ThinkServer RD630   | 2379.14 |    20 |           2 |               4 |
|  3 | RT-AC68U            |  219.99 |    10 |           1 |               3 |
|  4 | X110 64GB           |   73.84 |   100 |           3 |               1 |
|  5 | SDSSDP-128G-G25 2.5 |   82.04 |    30 |           3 |               1 |
+----+---------------------+---------+-------+-------------+-----------------+
5 rows in set (0.000 sec)
```

## 更新表中的行

```sql
MariaDB [inventory]> UPDATE product SET price=89.90, stock=60 WHERE id = 5;
Query OK, 1 row affected (0.007 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [inventory]> select * from product;
+----+---------------------+---------+-------+-------------+-----------------+
| id | name                | price   | stock | id_category | id_manufacturer |
+----+---------------------+---------+-------+-------------+-----------------+
|  1 | ThinkServer TS140   |  539.88 |    20 |           2 |               4 |
|  2 | ThinkServer RD630   | 2379.14 |    20 |           2 |               4 |
|  3 | RT-AC68U            |  219.99 |    10 |           1 |               3 |
|  4 | X110 64GB           |   73.84 |   100 |           3 |               1 |
|  5 | SDSSDP-128G-G25 2.5 |    89.9 |    60 |           3 |               1 |
+----+---------------------+---------+-------+-------------+-----------------+
5 rows in set (0.001 sec)
```

如果您使用不带 `WHERE` 子句的 `UPDATE` ，则表中的所有记录都将更新。

```sql
MariaDB [inventory]> UPDATE product SET price=89.90, stock=60;
Query OK, 4 rows affected (0.003 sec)
Rows matched: 5  Changed: 4  Warnings: 0

MariaDB [inventory]> select * from product;
+----+---------------------+-------+-------+-------------+-----------------+
| id | name                | price | stock | id_category | id_manufacturer |
+----+---------------------+-------+-------+-------------+-----------------+
|  1 | ThinkServer TS140   |  89.9 |    60 |           2 |               4 |
|  2 | ThinkServer RD630   |  89.9 |    60 |           2 |               4 |
|  3 | RT-AC68U            |  89.9 |    60 |           1 |               3 |
|  4 | X110 64GB           |  89.9 |    60 |           3 |               1 |
|  5 | SDSSDP-128G-G25 2.5 |  89.9 |    60 |           3 |               1 |
+----+---------------------+-------+-------+-------------+-----------------+
5 rows in set (0.000 sec)
```

## 删除表中的数据

如果您使用不带 `WHERE` 子句的 `DELETE FROM` ，则表中的所有记录都将删除。

```bash
MariaDB [inventory]> DELETE FROM product WHERE id = 3;
Query OK, 1 row affected (0.003 sec)

MariaDB [inventory]> select * from product;
+----+---------------------+-------+-------+-------------+-----------------+
| id | name                | price | stock | id_category | id_manufacturer |
+----+---------------------+-------+-------+-------------+-----------------+
|  1 | ThinkServer TS140   |  89.9 |    60 |           2 |               4 |
|  2 | ThinkServer RD630   |  89.9 |    60 |           2 |               4 |
|  4 | X110 64GB           |  89.9 |    60 |           3 |               1 |
|  5 | SDSSDP-128G-G25 2.5 |  89.9 |    60 |           3 |               1 |
+----+---------------------+-------+-------+-------------+-----------------+
4 rows in set (0.000 sec)
```

## 创建数据表

```sql
MariaDB [inventory]> CREATE TABLE lixiaohui
    -> ( id INT(11) NOT NULL AUTO_INCREMENT,
    -> name VARCHAR(100) NOT NULL,
    -> price DOUBLE NOT NULL,
    -> stock INT(11) NOT NULL,
    -> id_category INT(11) NOT NULL,
    -> id_manufacturer INT(11) NOT NULL,
    -> CONSTRAINT id_pk PRIMARY KEY (id)
    -> );
Query OK, 0 rows affected (0.009 sec)
```

查询表结构

```sql
MariaDB [inventory]> describe lixiaohui;
+-----------------+--------------+------+-----+---------+----------------+
| Field           | Type         | Null | Key | Default | Extra          |
+-----------------+--------------+------+-----+---------+----------------+
| id              | int(11)      | NO   | PRI | NULL    | auto_increment |
| name            | varchar(100) | NO   |     | NULL    |                |
| price           | double       | NO   |     | NULL    |                |
| stock           | int(11)      | NO   |     | NULL    |                |
| id_category     | int(11)      | NO   |     | NULL    |                |
| id_manufacturer | int(11)      | NO   |     | NULL    |                |
+-----------------+--------------+------+-----+---------+----------------+
6 rows in set (0.002 sec)
```

## 删除数据表

DROP TABLE 将从数据库中删除数据和表。请谨慎使用。

```sql
MariaDB [inventory]> DROP TABLE lixiaohui;
Query OK, 0 rows affected (0.007 sec)
```

## 删除数据库

DROP DATABASE 语句将丢弃数据库中的所有表，并删除数据库。这会销毁数据库中的所有数据。只有对该数据库具有 DROP 特权的用户才能运行此语句。

这不会更改用户对数据库的特权。如果重新创建了具有该名称的数据库，则为旧数据库设置的用户特权仍然有效。

```sql
MariaDB [inventory]> DROP DATABASE inventory;
Query OK, 3 rows affected (0.012 sec)
```

# 管理 MariaDB 用户和访问权限

## 创建用户帐户

为了控制用户对数据库服务器的访问级别，您必须在 MariaDB 中设置数据库用户，并授予他们对服务器及其数据执行操作的权限。

若要创建新用户，您需要以下权限级别之一：

- 是 MariaDB `root` 用户。

- 是获得了全局 `CREATE USER` 特权的用户。

- 是在 `mysql` 数据库上获得了 `INSERT` 特权的用户。

```sql
MariaDB [(none)]> create user lixiaohui@localhost identified by 'redhat';
Query OK, 0 rows affected (0.010 sec)
```

查询用户的确存在

```sql
MariaDB [(none)]> select host,user,password from mysql.user where user = 'lixiaohui';
+-----------+-----------+-------------------------------------------+
| host      | user      | password                                  |
+-----------+-----------+-------------------------------------------+
| localhost | lixiaohui | *84BB5DF4823DA319BBF86C99624479A198E6EEE9 |
+-----------+-----------+-------------------------------------------+
1 row in set (0.004 sec)
```

用户的表达格式

| 帐户                                    | 描述                                                       |
| ------------------------------------- | -------------------------------------------------------- |
| `lixiaohui`                           | 用户 `lixiaohui` 可以从任何主机进行连接。                              |
| `lixiaohui@'%'`                       | 用户 `lixiaohui` 可以从任何主机进行连接。                              |
| `lixiaohui@'localhost'`               | 用户 `lixiaohui` 可以从 `localhost` 进行连接。                     |
| `lixiaohui@'192.168.1.5'`             | 用户 `lixiaohui` 只能从 IP 地址 `192.168.1.5` 进行连接。             |
| `lixiaohui@'192.168.1.%'`             | 用户 `lixiaohui` 可以从任何属于网络 `192.168.1.0/24` 的地址进行连接。       |
| `lixiaohui@'2001:db8:18:b51:c32:a21'` | 用户 `lixiaohui` 可以从 IP 地址 `2001:db8:18:b51:c32:a21` 进行连接。 |

## 向用户授予特权

为用户`lixiaohui`授予对数据库`lxhdb`中的所有表的特定权限。这个命令授予的权限包括：

- `SELECT`：查询表中的数据。
- `UPDATE`：更新表中的数据。
- `DELETE`：删除表中的数据。
- `INSERT`：向表中插入数据。

```sql
MariaDB [(none)]> create database lxhdb;
Query OK, 1 row affected (0.001 sec)

MariaDB [(none)]> GRANT SELECT, UPDATE, DELETE, INSERT ON lxhdb.* TO lixiaohui@localhost;
Query OK, 0 rows affected (0.001 sec)
```

查询权限

```sql
MariaDB [(none)]> show grants for lixiaohui@localhost;
+------------------------------------------------------------------------------------------------------------------+
| Grants for lixiaohui@localhost                                                                                   |
+------------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'lixiaohui'@'localhost' IDENTIFIED BY PASSWORD '*84BB5DF4823DA319BBF86C99624479A198E6EEE9' |
| GRANT SELECT, INSERT, UPDATE, DELETE ON `lxhdb`.* TO 'lixiaohui'@'localhost'                                     |
+------------------------------------------------------------------------------------------------------------------+
2 rows in set (0.002 sec)
```

**GRANT语法**

| 授权                                                             | 描述                                                |
| -------------------------------------------------------------- | ------------------------------------------------- |
| `GRANT SELECT ON database.table TO username@hostname`          | 向特定用户授予对特定数据库中特定表的 SELECT 特权。                     |
| `GRANT SELECT ON database.* TO username@hostname`              | 向特定用户授予对特定数据库中所有表的 SELECT 特权。                     |
| `GRANT SELECT ON *.* TO username@hostname`                     | 向特定用户授予对所有数据库中所有表的 SELECT 特权。                     |
| `GRANT CREATE, ALTER, DROP ON database.* to username@hostname` | 向特定用户授予在特定数据库中 CREATE、ALTER 和 DROP TABLES 特权。     |
| `GRANT ALL PRIVILEGES ON *.* to username@hostname`             | 向特定用户授予对所有数据库的所有可用特权，事实上是创建一个超级用户（类似于 `root` 用户）。 |

## 撤销用户特权

```sql
MariaDB [(none)]> show grants for lixiaohui@localhost;
+------------------------------------------------------------------------------------------------------------------+
| Grants for lixiaohui@localhost                                                                                   |
+------------------------------------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'lixiaohui'@'localhost' IDENTIFIED BY PASSWORD '*84BB5DF4823DA319BBF86C99624479A198E6EEE9' |
| GRANT SELECT, INSERT, UPDATE ON `lxhdb`.* TO 'lixiaohui'@'localhost'                                             |
+------------------------------------------------------------------------------------------------------------------+
2 rows in set (0.000 sec)
```

在修改了授权表后，最好运行 `FLUSH PRIVILEGES` 命令。

```bash
MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.001 sec)
```

删除用户账户

```sql
MariaDB [(none)]> drop user 'lixiaohui'@'localhost';
Query OK, 0 rows affected (0.001 sec)
```

# 创建和恢复 MariaDB 备份

有两种备份 MariaDB 的方法：

- 逻辑备份，以文本文件的形式导出信息，其中包含重新创建数据库所需的 SQL 命令。

- 物理备份，复制包含数据库内容的原始数据库目录和文件。

每种备份方法都各有利弊。

**逻辑备份特征**

- 数据库结构是通过查询数据库来检索的。

- 逻辑备份的可移植性很高，在某些情况下可以恢复到另一个数据库提供程序（如 PostgreSQL）。

- 备份过程很慢，因为服务器必须访问数据库信息并将其转换为逻辑格式。

- 在服务器联机时执行。

- 备份不包含日志和配置文件。

**物理备份特征**

- 包含数据库目录和文件夹的原始副本。

- 输出更精简。

- 备份可以包含日志和配置文件。

- 只能移植到具有类似硬件和软件的其他计算机。

- 比逻辑备份快。

- 应在服务器脱机或者数据库中所有表均锁定时执行，防止在备份期间发生更改。

# 准备数据

在workstation上打lab准备数据

```bash
[student@workstation ~]$ lab database-backups start
Starting the database-backups exercise:

 · Download exercise playbooks.................................  SUCCESS
 · Run exercise preparation playbooks..........................  SUCCESS
```

## 执行逻辑备份

查询哪些数据库存在

```sql
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| inventory          |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.000 sec)
```

使用`mysqldump`工具导出名为`inventory`的数据库到一个名为`inventory.dump`的文件

```bash
[root@servera ~]# mysqldump -uroot -predhat -P 3000 inventory > inventory.dump
[root@servera ~]# ll -h inventory.dump
-rw-r--r--. 1 root root 3.8K Aug 26 20:48 inventory.dump
```

若要以逻辑方式备份所有数据库，可使用 `--all-databases` 选项

```bash
[root@servera ~]# mysqldump -uroot -predhat -P 3000 --all-databases > alldb.dump
[root@servera ~]# ll -h
total 492K
-rw-r--r--. 1 root root 471K Aug 26 20:49 alldb.dump
-rw-r--r--. 1 root root 3.8K Aug 26 20:48 inventory.dump
```

查看备份的内容

```bash
[root@servera ~]# cat inventory.dump | tail

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2024-08-26 20:48:23
```

## 执行物理备份

默认应该安装好的，如果没安装，可以这样做

```bash
[root@servera ~]# yum install mariadb-backup -y
```

将数据库备份到/mnt

```bash
[root@servera ~]# mariabackup --backup --target-dir /mnt --user root --password redhat -P 3000
...
[00] 2024-08-26 20:53:33 completed OK!
```

```bash
[root@servera ~]# ls /mnt
aria_log.00000001  backup-my.cnf   ibdata1      inventory  performance_schema      xtrabackup_info
aria_log_control   ib_buffer_pool  ib_logfile0  mysql      xtrabackup_checkpoints
```

## 恢复逻辑备份

恢复备份时，它会使用备份内容覆盖数据库服务器的内容。

```bash
[root@servera ~]# mysql -uroot -predhat -P 3000 inventory < inventory.dump
```

## 恢复物理备份

使用 `mariabackup` 工具和下列选项之一，从备份执行物理恢复：

`--copy-back`

保留原始备份文件。

`--move-back`

将备份文件移到数据目录，然后删除原始备份文件。

1. 确保 `mariadb` 服务已停止

```bash
[root@servera ~]# systemctl stop mariadb
```

2. 确定数据目录的位置，并确保它为空：

```bash
[root@servera ~]# grep '^datadir' /etc/my.cnf.d/mariadb-server.cnf
datadir=/var/lib/mysql
[root@servera ~]# rm -rf /var/lib/mysql/*
```

3. 使用 mariabackup 恢复备份文件：
   
   ```bash
   [root@servera ~]# mariabackup --copy-back --target-dir=/mnt
   mariabackup based on MariaDB server 10.3.17-MariaDB Linux (x86_64)
   ...
   [00] 2024-08-26 21:07:45 completed OK!
   ```

4. 确保数据文件已将用户和组所有权设置为 mysql：

```bash
[root@servera ~]# chown -R mysql:mysql /var/lib/mysql/
```

5. 启动 mariadb 服务：

```bash
[root@servera ~]# systemctl start mariadb
```

# 自动部署 MariaDB

详见教材
