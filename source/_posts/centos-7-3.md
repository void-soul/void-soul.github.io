---
title: centos 7.3 从零开始搭建生产环境
date: 2020-04-03 16:30:23
tags:
  - centos
  - nginx
  - mysql
  - NodeJs
  - redis
  - mongodb
categories: linux
description: centos 7.3 从零开始搭建生产环境
---

## 硬盘挂载

```
fdisk -ll
# 分区
fdisk /dev/vdb1
# 输入n，其余选项用默认值
# 格式化
mkfs.ext4 /dev/vdb1
# 创建挂载文件夹
mkdir /data
# 挂载
mount /dev/vdb1 /data
# 开机自动挂载
vi /etc/fstab
# 加入
/dev/vdb		/data		ext4		defaults	0 0
```

## 安装 nvm/nodejs/pm2/yarn

```
# 安装nvm
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.39.5/install.sh | bash
# 安装node10.2.0,此时nvm不可用时重登陆一次终端即可
nvm install 10.2.0
nvm alias default 10.2.0
# 安装全局pm2
npm install pm2 -g
# 安装yarn
curl --silent --location https://dl.yarnpkg.com/rpm/yarn.repo | sudo tee /etc/yum.repos.d/yarn.repo
curl --silent --location https://rpm.nodesource.com/setup_12.x | bash -
sudo yum install yarn
```

## 安装 htop

```
yum install epel-release -y
yum install htop -y
```

## mysql

### 卸载

```
//rpm包安装方式卸载
查包名：rpm -qa|grep -i mysql
删除命令：rpm -e –nodeps 包名

//yum安装方式下载
1.查看已安装的mysql
命令：rpm -qa | grep -i mysql
2.卸载mysql
命令：yum remove mysql-community-server-5.6.36-2.el7.x86_64
查看mysql的其它依赖：rpm -qa | grep -i mysql

//卸载依赖
yum remove mysql-libs
yum remove mysql-server
yum remove perl-DBD-MySQL
yum remove mysql
```

### 安装

```
# 基本环境
yum install -y perl perl-Module-Build net-tools autoconf libaio numactl-libs
# https://dev.mysql.com/downloads/repo/yum/ 找到linux7的最新源下载地址
# https://repo.mysql.com/yum/ 找到老版本
wget https://dev.mysql.com/get/mysql80-community-release-el7-2.noarch.rpm
# 安装mysql源
yum localinstall mysql80-community-release-el7-2.noarch.rpm
# 检查是否安装成功，出现3行红色的状态表示成功；修改vim /etc/yum.repos.d/mysql-community.repo中的enable可以启用、关闭特定版本的mysql源
yum repolist enabled | grep "mysql.*-community.*"
# 开始安装
yum install mysql-community-server
# 如果出现找不到包的情况，说明有系统默认的源被开启，禁用即可,然后再执行安装命令
yum module disable mysql
```

### 配置

> /etc/my.cnf

注意：这里有一些参数已经是最大值，超过此值 字符串将自动截断

如：group_concat_max_len

```
[client]
port=3308
socket=/data/mysql/mysql.sock
[mysql]
no-beep=
[mysqld]
port=3308
datadir=/data/mysql
default_authentication_plugin=mysql_native_password
default-storage-engine=INNODB
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION"
log-output=FILE
general-log=0
general_log_file=general.log
slow-query-log=1
slow_query_log_file=slow_query.log
long_query_time=10
log-error=/data/mysql/err.log
server-id=1
lower_case_table_names=1
max_connections=1000
table_open_cache=2000
thread_cache_size=10
myisam_max_sort_file_size=107374182400
myisam_sort_buffer_size=713031680
key_buffer_size=8388608
read_buffer_size=65536
read_rnd_buffer_size=256K
innodb_flush_log_at_trx_commit=1
innodb_log_buffer_size=1048576
innodb_buffer_pool_size=1073741824
innodb_log_file_size=50331648
innodb_thread_concurrency=9
innodb_autoextend_increment=64
innodb_buffer_pool_instances=8
innodb_concurrency_tickets=5000
innodb_old_blocks_time=1000
innodb_open_files=300
innodb_stats_on_metadata=0
innodb_file_per_table=1
back_log=80
flush_time=0
join_buffer_size=256K
max_allowed_packet=4M
max_connect_errors=100
open_files_limit=4161
sort_buffer_size=256K
table_definition_cache=1400
binlog_row_event_max_size=8K
sync_master_info=10000
sync_relay_log=10000
sync_relay_log_info=10000
loose_mysqlx_port=33080
group_concat_max_len=-1
max_allowed_packet=104857600
max_heap_table_size=1048576000
tmp_table_size=524288000
skip-log-bin
socket=/data/mysql/mysql.sock
pid-file=/data/mysql/mysqld.pid
#skip-grant-tables
```

### 启动、停止命令

`systemctl start mysqld`或`service mysqld start`

### 开机自启

`systemctl enable mysqld.service`

### 密码修改

1. 首先在 my.cnf 中打开`skip-grant-tables`
2. 在终端连接`mysql`
3. 切换数据库:`use mysql;`
4. 由于`skip-grant-tables`，所以在执行任何语句前，执行`flush privileges;`
5. 修改密码策略:`SHOW VARIABLES LIKE 'validate_password%';`其中`validate_password.policy`表示密码检测等级，默认是`MEDIUM`,将他改为`LOW`:set global validate_password.policy=LOW;
6. 先将 root 的密码清空:`update user set authentication_string='',host='%' where user='root';`
7. 修改密码`ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'aaabbb';`
8. `flush privileges;`
9. 打开云服务器的安全组：3308，然后就可以用 navicat 连接了

## mongodb

### 源

```
# 创建mongodb源文件
vim /etc/yum.repos.d/mongodb-org-4.0.6.repo
```

文件内容

```
[mongodb-org-4.2]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/6/mongodb-org/4.2/x86_64/
gpgcheck=0
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.2.asc
```

修改 gpgcheck=0, 省去 gpg 验证

更新系统环境

```
yum update
```

### 安装

```
yum -y install mongodb-org
```

### 配置

> /etc/mongod.conf

```
systemLog:
  destination: file
  logAppend: true
  path: /data/mongo/mongod.log

storage:
  dbPath: /data/mongo
  journal:
    enabled: true
  WiredTiger:
    cacheSizeGB: 4

processManagement:
  fork: true  # fork and run in background
  pidFilePath: /data/mongo/mongod.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 3306
  bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.


security:
  authorization: enabled
```

### 启动

`systemctl start mongod`或`service mongod start`

### 开机自启

> 无法`/sbin/chkconfig mongod on`自启！！！因为/dev/null 文件权限每次都会被覆盖

```
chmod 777 /dev/null
systemctl start mongod.service
```

### 密码修改

```
# 打开终端
mongo --port=3306
# 切换数据库
use admin
# 创建用户
db.createUser({user: 'root', pwd: '123456', roles: ['root']})
# 验证结果
db.auth(用户名，用户密码)
```

使用`Studio 3T`连接数据库,创建 database\专门的用户

## nginx

### 源

```
sudo yum install yum-utils
vim /etc/yum.repos.d/nginx.repo
```

文件内容

```
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
```

### 安装

```
sudo yum install nginx
```

### 配置

> /etc/nginx/nginx.conf

略

### 执行权限

```
# 如果nginx设置了提交body大小太大，提交的数据会先在 `/var/lib/nginx/tmp/client_body/` 中创建临时文件，然后nginx进程来读取这个文件。
# 但是这种方式安装的nginx，没有读取文件的权限
# 这是nginx进程的用户
ps aux | grep "nginx: worker process" | awk '{print $1}'
# 这是`/var/lib/nginx`的用户
sudo ls -al /var/lib/nginx/
# 然后将目录权限进行修改,**注意**，不一定是nobody，可以是nginx进程的其他用户
sudo chown -R nobody:nobody /var/lib/nginx/
```

### 启动

`systemctl start nginx`或`service nginx start`

### 开机自启

`systemctl enable nginx.service`

## git

### 源

```
# 基本环境
# 因为center-os的git版本太老，所以使用第三方源库：https://ius.io/GettingStarted/ ，这里找到ius-release对center7的包(因为centeros可以忽略epel-release)
# 下载安装
wget https://centos7.iuscommunity.org/ius-release.rpm
# 安装源
rpm -i ius-release.rpm
# 开始安装,git需要在后面追加2u,表示使用刚才源库的地址
yum install git2u
```

### 配置

```
git config --global credential.helper store
```

然后进行 clone、pull 等操作，输入一次用户名密码后，以后无需再次输入了

## redis

```
# 基本环境
# 因为center-os的redis版本太老，所以使用第三方源库：https://mirrors.tuna.tsinghua.edu.cn/remi/ ，这里找到对center8的包
# 下载安装
yum install https://mirrors.tuna.tsinghua.edu.cn/remi/enterprise/remi-release-8.rpm
# 开始安装
yum --enablerepo=remi install redis
# 安装之后可以查看创建的文件
rpm -ql redis
# 开机启动
systemctl enable redis.service
```

### 启动、停止命令

`service redis start`或者`systemctl start redis`

### 开机自启

`systemctl enable redis.service`

### 配置

```
vim /etc/redis.conf
```

## java

```
# 官网找rpm安装包
# java8 https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
# 最新 https://www.oracle.com/technetwork/java/javase/downloads/index.html
# 将安装包上传到服务器

# 检查是否有旧版java、openjdk

rpm -qa | grep java

# 如果有则卸载,例如上面命令输出

tzdata-java-2014i-1.el7.noarch
java-1.7.0-openjdk-headless-1.7.0.71-2.5.3.1.el7_0.x86_64
java-1.7.0-openjdk-1.7.0.71-2.5.3.1.el7_0.x86_64

# 表示有3个java需要卸载:

rpm -e --nodeps tzdata-java-2014i-1.el7.noarch
rpm -e --nodeps java-1.7.0-openjdk-headless-1.7.0.71-2.5.3.1.el7_0.x86_64
rpm -e --nodeps java-1.7.0-openjdk-1.7.0.71-2.5.3.1.el7_0.x86_64

# 安装

rpm -ivh jdk-8u191-linux-x64.rpm

# 配置环境变量：

vim /etc/profile


# 添加内容：

export JAVA_HOME=/usr/java/default
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

# 使配置生效

source /etc/profile

```

# 内存优化

## 交换内存

```
# 查看Swap 分区大小
free -m
# 创建目录
mkdir /swap
cd /swap
# count单位是KB
sudo dd if=/dev/zero of=swapfile bs=1024 count=2000000
# 转换文件为swap文件
sudo mkswap -f swapfile
# 激活文件
sudo swapon swapfile
# 反激活文件
sudo swapoff swapfile
# 永久保持此空间
vi /etc/fstab
# 加入
/swap/swapfile /swap swap defaults 0 0

# 虚拟内存与物理内存关系
cat /proc/sys/vm/swappiness

# 此值越大，表示越依赖虚拟内存，取值0-100
# 临时修改
sysctl vm.swappiness=10
# 永久修改
vi /etc/sysctl.conf
# 加入
vm.swappiness=20

```

## 交换内存的大小

1. 原则上是物理内存的 2 倍
2. 规则 1：物理内存小于 2G=2 倍到 3 倍，2G-8G=1 倍到 2 倍，8-64G=4G 到 1.5 倍 64G 以上=4G 以上
3.

## 用户限制

### 查看当前设置

```
ulimit -a
# 设定core文件的最大值，单位为区块。
core file size          (blocks, -c) unlimited
# 程序数据节区的最大值，单位为KB。
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
# **重要**
file size               (blocks, -f) unlimited
pending signals                 (-i) 31129
max locked memory       (kbytes, -l) 16384
# **重要**
max memory size         (kbytes, -m) unlimited
# **重要**
open files                      (-n) 65535
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
# **重要**
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
# **重要**
max user processes              (-u) 31129
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

### 修改进程数

`vi /etc/security/limits.d/90-nproc.conf`

### 修改可打开的最大文件数

`vi /etc/security/limits.conf`

```
# 开启任务
nohup 命令 > log.log  2>&1 &
# 任务列表
jobs
# 关闭

gb %n
```
