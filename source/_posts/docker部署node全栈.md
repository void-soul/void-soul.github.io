---
title: docker部署eggjs实战
date: 2020-04-03 17:13:26
tags:
  - docker
categories: linux
description: 低配服务器中使用docker隔离运行环境，达到一台变多台的效果
---

# 场景

## 服务

`mysql8.x`、`mongodb4.2.x`、`eggjs服务`、文件服务、若干 nginx 映射静态服务

## 数据量

mongodb:40G+
mysql: 50G+

mysql 中涉及大量 CPU 密集计算，内存占用也很高
eggjs 有一些定时服务，耗时长、内存高（上传数据到亚马逊、从物流公司抓取轨迹等）

## 硬件条件

服务器 x 1=阿里云 C5,4U8G

系统 CentOS 8

> 其实还买过一台 t5(1U2G)运行数据库,一台 mn4(2U8G)运行 egg 服务、文件服务、nginx。但是这些服务器都是共享计算型，CPU 的计算消耗积分，每小时计算能力及其有限。在正式的环境中根本不够看，所以更换了一台 C5（成本所限，不能随心所欲）

## 要求

1. 服务的密集计算尽量不要互相影响
2. 为内存消耗大户设置占用上限，已到达自动 GC
3. 可快速迁移（等服务器有了之后）
4. 可快速水平扩容

# 安装 docker

## 卸载旧版本

```
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```

## 安装依赖

```
yum install -y yum-utils \
           device-mapper-persistent-data \
           lvm2
```

## 配置 docker 源

> 这里使用的是国内源

```
yum-config-manager \
    --add-repo \
    https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo
sed -i 's/download.docker.com/mirrors.ustc.edu.cn\/docker-ce/g' /etc/yum.repos.d/docker-ce.repo
```

官方源:(不推荐)

```
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

## 升级 containerd.io

> 安装新版 docker 需要 containerd.io>1.2.2-3,系统自带版本过低，需要先升级

[在这里下载新版](https://download.docker.com/linux/centos/7/x86_64/stable/Packages/)

```
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm
yum install containerd.io-1.2.6-3.3.el7.x86_64.rpm
```

## 执行安装(使用脚本)

```
curl -fsSL get.docker.com -o get-docker.sh
# 使用阿里云镜像
sh get-docker.sh --mirror Aliyun
```

## 启动

```
systemctl enable docker
systemctl start docker
```

## 建立用户组

> 安装完毕后会提示: Adding a user to the "docker" group will grant the ability to run containers which can be used to obtain root privileges on the docker host.

```
groupadd docker
usermod -aG docker $USER
```

## 配置国内镜像仓库

```
vi /etc/docker/daemon.json
# 写入

{
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://hub-mirror.c.163.com"
  ]
}
```

然后重启服务

```
systemctl daemon-reload
systemctl restart docker
```

## 配置内核参数

```
tee -a /etc/sysctl.conf <<-EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

重载

```
sysctl -p
```

# 分析

## 数据服务

> `mysql`、`mongodb`、`redis`

1. 这些服务不需要自己构建镜像,使用官方镜像即可
2. 同时容器也没必要持久保存,停止即删除,启动即创建
3. mysql、mongodb 必须使用已有数据目录
4. mysql 必须加载已有配置文件
5. mysql、mongodb 有外网访问需求(开发排查故障)
6. redis 不需要提供外网服务,也不需要在宿主机中访问,仅在 egg 服务镜像中访问

## egg 服务

1. 需要自己构建镜像
2. 每次更新不一定非要安装依赖，所以需要两个阶段镜像：依赖镜像和代码镜像
3. 容器需要持久保存,便于查看日志
4. 需要安装 alinode 监控服务
5. 进行监控检查
6. 不需要在宿主机中访问，仅在 nginx 中访问

## 文件服务

1. 现在文件上传都已经托管到了七牛，此服务目前作用仅仅是提供以前上传文件的访问
2. 几乎不改动，也非常稳定，几乎没有错误。因此这个服务不需要分阶段运行，也不需要持久保存容器
3. 不需要在宿主机中访问，仅在 nginx 中访问

## nginx

1. 必须加载已有配置文件
2. 需要托管静态目录、动态服务
3. 不需要容器持久

# 开始

## 创建虚拟网络

> 用于控制各容器之间互相连接，如 egg 访问 mysql、mongodb，nginx 访问 egg、文件服务。

`docker network create -d bridge dmce-net`

创建网络后，容器启动时需要加入网络

`--network dmce-net`

之后就可用容器名称代替 ip，例如 egg 配置数据库访问地址：

```
host: 'mysql-1',
port: 3306
```

注意：通过虚拟网络将直接连接容器，而不是宿主机，所以端口指的是容器内部的端口而不是容器映射到宿主机的端口

## egg 依赖的 Dockerfile

> 不用阶段创建方式，而是把依赖独立为一个镜像。这样在之后的服务拆分中，可以重复使用

`egg-package.Dockerfile`

**构建上下文为 egg 服务目录（确保目录中没有 node_modules，否则容器将变得很大），镜像名 `egg-package`**

```
# 使用alpine系，自带yarn
FROM node:13.12.0-alpine3.11
# 创建服务目录
RUN mkdir -p /usr/src/node-app \
# 安装alinode，为使用阿里云node监控服务准备
  && npm install nodeinstall -g \
  && nodeinstall --install-alinode v5.15.0

# 仅复制安装依赖需要的文件
COPY ./package.json /usr/src/node-app/package.json
COPY ./yarn.lock /usr/src/node-app/yarn.lock
COPY ./.npmrc /usr/src/node-app/.npmrc

# 工作目录指定
WORKDIR /usr/src/node-app
# 安装依赖
RUN yarn
```

## egg 服务的 Dockerfile

`core-service.Dockerfile`

**构建上下文为 egg 服务目录（确保目录中没有 node_modules，否则容器将变得很大），镜像名 `core-service`**

```
# 从egg依赖镜像开始构建
FROM egg-package
# 创建一个数据层，存放服务生成的pdf文件（需要在nginx中映射到域名访问）
VOLUME /data/pdf/order
# 复制代码
COPY ./ /usr/src/node-app
# 启动
CMD ["yarn", "start"]
```

## 文件服务的 Dockerfile

**构建上下文为 文件 服务目录（确保目录中没有 node_modules，否则容器将变得很大），镜像名 `file-service`**

```
FROM node:13.12.0-alpine3.11
RUN mkdir -p /usr/src/node-app
COPY ./ /usr/src/node-app
VOLUME /usr/src/node-app/file
WORKDIR /usr/src/node-app
RUN yarn
CMD ["node", "file-server.js"]
```

## shell 脚本

> shell 脚本用来快速管理服务

### 支持命令一览：

```
0：停止并启动mysql服务 1：停止mysql服务
---------------
2：停止并启动mongodb服务 3：停止mongodb服务
---------------
4：停止并启动redis服务 5：停止redis服务
---------------
6：更新service代码和依赖            7：启动service服务       8：停止service服务
9：输出service镜像日志             10：输出service框架日志  11：输出service服务日志 12：输出service错误日志
13：更新service代码并重启[丢失日志] 14：进入service容器
---------------
15：更新客户端代码
---------------
16：创建/停止并启动文件服务 17：停止并启动文件服务 18：停止文件服务
---------------
19：停止并启动公用nginx服务  20：停止公用nginx服务
---------------
```

### 脚本内容

```
#!/bin/sh

# 每个命令执行都延迟5秒，留出后悔时间
time_out=5

# 各服务参数配置
# 包括使用内存、交换内存、CPU、CPU分数、端口以及是否持久容器
# 注意：交换内存参数=内存+交换内存
# 注意：CPU按数字编号，从0开始
# 注意：CPU分数越大，权重越高，默认都是1024
mysql_memory=2G
mysql_memory_swap=4G
mysql_cpu=0,1
mysql_cpu_share=1024
mysql_port=3308
mysql_rm=--rm

mongo_memory=2G
mongo_memory_swap=4G
mongo_cpu=0,1
mongo_cpu_share=512
mongo_port=3306
mongo_rm=--rm

redis_memory=300M
redis_memory_swap=1G
redis_cpu=2
redis_cpu_share=512
redis_rm=--rm

core_service_memory=2G
core_service_memory_swap=4G
core_service_cpu=2,3
core_service_cpu_share=1024
core_service_rm=

file_service_memory=500M
file_service_memory_swap=1G
file_service_cpu=2
file_service_cpu_share=512
file_service_rm=--rm

nginx_service_memory=300M
nginx_service_memory_swap=1G
nginx_service_cpu=2
nginx_service_cpu_share=1024
nginx_service_rm=--rm

# 后悔缓冲函数
function dotime(){
  for i in `seq -w $time_out -1 1`
    do
      echo -e "$i秒后执行 \c"
      for j in $(seq 1 $i)
      do
      if [ $j = $i ]; then
      echo "后悔来得及"
      else
      echo -e "后悔来得及  \c"
      fi
      done
      sleep 1
  done
  echo "来不及了~~~"
}

# mysql容器创建函数
function startMysql(){
  # 无论如何都先停止
  docker stop mysql-1
  docker run \
  # 挂载配置文件
  -v /data/docker/my.cnf:/etc/mysql/conf.d/my.cnf:ro \
  # 挂载数据目录
  -v /data/mysql:/var/lib/mysql \
  # 除了最终服务启动用户，其余都用root
  --user root \
  # 端口映射，这里不加IP表示映射到所有IP
  -p $mysql_port:3306 \
  # 容器名称
  --name mysql-1 \
  # d=后台运行
  -d \
  # 内存限制
  -m $mysql_memory \
  # 交换内存限制=内存+交换内存
  --memory-swap $mysql_memory_swap \
  # cpu限制
  --cpuset-cpus $mysql_cpu \
  # cpu分数
  --cpu-shares $mysql_cpu_share \
  # 持久or临时容器
  $mysql_rm \
  # 加入虚拟网络
  --network dmce-net \
  mysql:8.0.19
}

# mongo 容器创建
function startMongodb(){
  docker stop mongo-1
  docker run \
  --name mongo-1 \
  -v /data/mongo:/data/db \
  --user root \
  -p $mongo_port:27017 \
  -dit \
  -m $mongo_memory \
  --memory-swap $mongo_memory_swap \
  --cpuset-cpus $mongo_cpu \
  --cpu-shares $mongo_cpu_share \
  $mongo_rm \
  --network dmce-net \
  mongo:4.2.5-bionic \
  # mongo 自身内存限制
  --wiredTigerCacheSizeGB 1.5 \
  # 绑定所有ip
  --bind_ip_all \
  # 最大连接数
  --maxConns 1000 \
  # 需要授权访问
  --auth
}

# redis 容器创建
function startRedis(){
  docker stop redis-1
  docker run \
  --name redis-1 \
  -d \
  -m $redis_memory \
  --memory-swap $redis_memory_swap \
  --cpuset-cpus $redis_cpu \
  --cpu-shares $redis_cpu_share \
  $redis_rm \
  --network dmce-net \
  redis
}

# 构建egg依赖镜像
function buildEggService(){
  cd /data/services/dmce-erp
  git pull
  docker image rm egg-package
  echo "【注意：】此镜像需要安装node依赖,安装完成后,docker要整理文件碎片,会有一段时间等待"
  docker build \
  # 镜像名称
  -t egg-package \
  # 指定dockerfile
  -f /data/docker/egg-package.Dockerfile \
  # 指定上下文
  /data/services/dmce-erp/service-dist
}

# 构建 egg服务镜像
function buildCoreService(){
  cd /data/services/dmce-erp
  git pull
  docker stop core-service-1
  docker rm core-service-1
  docker build \
  -t core-service \
  -f /data/docker/core-service.Dockerfile \
  /data/services/dmce-erp/service-dist
}
# 启动egg服务容器
function startCoreService(){
  docker stop core-service-1
  docker rm core-service-1
  docker run \
  --name core-service-1 \
  -v /data/pdf/order:/data/pdf/order \
  -d \
  -m $core_service_memory \
  --memory-swap $core_service_memory_swap \
  --cpuset-cpus $core_service_cpu \
  --cpu-shares $core_service_cpu_share \
  --network dmce-net \
  $core_service_rm \
  core-service
}
# 启动文件服务容器
function startFileService(){
  docker stop file-service-1
  docker run \
  --name file-service-1 \
  -v /data/services/file-services/files:/data/services/file-services/files \
  -d \
  -m $file_service_memory \
  --memory-swap $file_service_memory_swap \
  --cpuset-cpus $file_service_cpu \
  --cpu-shares $file_service_cpu_share \
  $file_service_rm \
  --network dmce-net \
  file-service
}
# 构建文件服务镜像
function buildFileService(){
  docker stop file-service-1
  docker image rm file-service
  docker build \
  -t file-service \
  -f /data/docker/file-service.Dockerfile \
  /data/services/dmce-erp/file-service
  startFileService
}
# 启动nginx容器
function startNginx(){
  docker stop nginx-1
  docker run \
  --name nginx-1 \
  -dit \
  # 挂载配置文件
  -v /data/docker/nginx.conf:/etc/nginx/nginx.conf:ro \
  # 挂载静态目录
  -v /data/valid-files:/data/valid-files:ro \
  -v /data/pdf:/data/pdf:ro \
  -v /data/site/blog:/data/site/blog:ro \
  -v /data/site/jingrise.com:/data/site/jingrise.com:ro \
  -v /data/docker/cert:/etc/nginx/cert:ro \
  $nginx_service_rm \
  -p 80:80 \
  -p 443:443 \
  -m $nginx_service_memory \
  --memory-swap $nginx_service_memory_swap \
  --cpuset-cpus $nginx_service_cpu \
  --cpu-shares $nginx_service_cpu_share \
  --network dmce-net \
  nginx
}

echo "-----------------------"
echo "|    部署辅助脚本      |"
echo "-----------------------"
read -p "请输入命令:
0：停止并启动mysql服务 1：停止mysql服务
---------------
2：停止并启动mongodb服务 3：停止mongodb服务
---------------
4：停止并启动redis服务 5：停止redis服务
---------------
6：更新service代码和依赖            7：启动service服务       8：停止service服务
9：输出service镜像日志             10：输出service框架日志  11：输出service服务日志 12：输出service错误日志
13：更新service代码并重启[丢失日志] 14：进入service容器
---------------
15：更新客户端代码
---------------
16：创建/停止并启动文件服务 17：停止并启动文件服务 18：停止文件服务
---------------
19：停止并启动公用nginx服务  20：停止公用nginx服务
---------------
请输入要做何操作? " type

echo $type

if [ $type = '0' ]; then
  echo "您选择了 [0：停止并启动mysql服务]"
  dotime
  startMysql
elif [ $type = '1' ]; then
  echo "您选择了 [1：停止mysql服务]"
  dotime
  docker stop mysql-1


elif [ $type = '2' ]; then
  echo "您选择了 [2：停止并启动mongodb服务]"
  dotime
  startMongodb
elif [ $type = '3' ]; then
  echo "您选择了 [3：停止mongodb服务]"
  dotime
  docker stop mongo-1


elif [ $type = '4' ]; then
  echo "您选择了 [4：停止并启动redis服务]"
  dotime
  startRedis
elif [ $type = '5' ]; then
  echo "您选择了 [5：停止redis服务]"
  dotime
  docker stop redis-1


elif [ $type = '6' ]; then
  echo "您选择了 [6：更新service代码和依赖]"
  dotime
  buildEggService
  buildCoreService
  startCoreService
elif [ $type = '7' ]; then
  echo "您选择了 [7：启动service服务]"
  dotime
  docker start core-service-1
elif [ $type = '8' ]; then
  echo "您选择了 [8：停止service服务]"
  dotime
  docker stop core-service-1
elif [ $type = '9' ]; then
  docker logs core-service-1
elif [ $type = '10' ]; then
  # 执行容器中命令
  docker exec core-service-1 tail -fn 500 /root/logs/service/egg-web.log
elif [ $type = '11' ]; then
  docker exec core-service-1 tail -fn 500 /root/logs/service/service-web.log
elif [ $type = '12' ]; then
  docker exec core-service-1 tail -fn 500 /root/logs/service/common-error.log
elif [ $type = '13' ]; then
  echo "您选择了 [13：更新service代码并重启[丢失日志]]"
  dotime
  buildCoreService
  startCoreService
elif [ $type = '14' ]; then
  echo "您选择了 [14：进入service容器]"
  # 进入容器，i=保持可输入命令状态 t=打开终端
  docker exec -it core-service-1 sh

elif [ $type = '15' ]; then
  echo "您选择了 [15：更新客户端代码]"
  dotime
  cd /data/services/dmce-erp
  git pull
  docker stop nginx-1
  docker start nginx-1


elif [ $type = '16' ]; then
  echo "您选择了 [16：创建/停止并启动文件服务]"
  dotime
  buildFileService
elif [ $type = '17' ]; then
  echo "您选择了 [17：停止并启动文件服务]"
  dotime
  startFileService
elif [ $type = '18' ]; then
  echo "您选择了 [18：停止文件服务]"
  dotime
  docker stop file-service-1


elif [ $type = '19' ]; then
  echo "您选择了 [19：停止并启动公用nginx服务]"
  dotime
  startNginx
elif [ $type = '20' ]; then
  echo "您选择了 [20：停止公用nginx服务]"
  dotime
  docker stop nginx-1

fi

```
