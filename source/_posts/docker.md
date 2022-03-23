```
title: docker学习
date: 2022年3月23日22:28:46
tags: [docker]
categories: docker
```

#### 虚拟机和容器的区别

- Docker容器是在操作系统层面上实现虚拟化，直接复用本地主机的操作系统，而传统虚拟机则是在硬件层面实现虚拟化。docker优势体现为启动速度快，占用体积小。

##### 不同之处

- 传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整的操作系统，在该系统上再运行所需应用进程
- 容器内的应用进程直接运行与宿主的内核，容器内没有自己的内核==也没有进行硬件虚拟==。因此容器要比传统虚拟机**更为轻便**。
- 每个容器之间相互隔离，每个容器有自己的文件系统，容器之间进行不会相互影响，能区分计算资源。

#### Docker的基本组成

##### 镜像

- 一个只读模板，可以用来创建Docker容器，一个镜像可以创建很多容器 

##### 容器

- 容器是用镜像创建的运行实例

##### 仓库

- 放一堆镜像的地方

#### 命令

##### 启动

- `systemctl start docker` (stop ,restart,status,enable)

##### 镜像命令

- docker images
- docker search --limit
- docker pull 
- docker system df 查看镜像/容器/数据卷所占空间
- docker rmi  

###### docker的虚悬镜像是什么？

  仓库名，标签都是<none>的镜像，俗称虚悬镜像

##### 容器命令

- docker run -it ubuntu  /bin/bash(bash)

- docker ps

- 退出容器

  - exit 直接退出容器
  - ctrl+p+q 退出交互 不退容器

- docker logs 查看日志

- docker inspect 查看内部内部细节

- docker exec -it   *** /bin/bash 进入容器

  - docker attach 

  ```
  1.attach直接进入容器启动命令的终端，不会启动新的进程，用exit退出会导致容器的停止。
  2.exec是在容器中打开新的终端，并且可以启动新的进程，用exit退出，不会导致容器的停止。
  ```

- docker export 容器 > **.tar 导出

- cat **.tar | docker import -镜像用户/镜像名:版本号

#### 本地镜像

##### 联合文件系统

- 镜像分层最大的一个好处就是共享资源，方便复制迁移，就是为了复用。
- **Docker镜像层都是只读的，容器层是可写的**

##### docker commit

- docker commit 提交容器副本使之成为一个新的镜像
- docker commit  -m ="提交的描述信息"  -a="作者" 容器ID 要创建的目标镜像名:标签名

##### 镜像推送到私服

1. `docker pull registry`

2. `docker run -d -p 5000:5000 -v /zzyyuse/myregistry/:/tmp/registry --privileged=true registry`

3. 查看私服仓库 `curl -XGET  http://192.168.5.128:5000/v2/_catalog` 主机ip

4. 将新镜像修改成符合私服规范的tag

    `docker tag 镜像:tag  Host:Port/Repository:Tag`

   `docker tag ubuntu-net:1.0 192.168.5.128:5000/ubuntu-net:1.0`

5. 修改配置文件使之支持http `"insecure-registries":["192.168.5.128:5000"]` 

   `vim /etc/docker/daemon.json`  添加上述语句 ==不生效可重启docker==

6. `docker push 192.168.5.128:5000/ubuntu-net:1.0`

7. 查看私服仓库 `curl -XGET  http://192.168.5.128:5000/v2/_catalog`

8. 删除本地镜像 测试私服仓库`docker pull 192.168.5.128:5000/ubuntu-net:1.0`

#### 容器数据卷

#####  解决问题 

- 如果出现cannot open directory:Permission denied

  在挂载目录多加一个 `--privileged=true`

##### 规则

​	1. `docker run -it --privileged=true -v /宿主机绝对路径目录:/容器内目录:rw 镜像名`  默认就是`rw`

​	 只能读取不能写 `ro`  宿主机可读可写，容器内部被限制只能读取不能写

##### 继承

 --volumes-from 父类   继承容器卷映射关系

#### 安装简介

##### 总体步骤

1. 搜索镜像  `docker hub`  `docker search tomcat`
2. 拉取镜像
3. 查看镜像
4. 启动镜像
5. 停止镜像
6. 移除镜像

##### 安装tomcat

1. `docker pull tomcat`
2. `docker run -d -p 8080:8080 --name=t1 tomcat`
3. 查看`localhost:8080`  404
   1. 进入容器 docker exec -it ID /bin/bash
   2. 删除`webapps` `mv webapps.dist webapps`
   3. 新版(10.0)webapps中没有文件 
   4. 可以换用 tomcat8 

##### 安装mysql

```shell
docker run -d -p 3306:3306 --privileged=true -v /mydata/mysql/log:/var/log/mysql -v /mydata/mysql/data:/var/lib/mysql -v /mydata/mysql/conf:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 --name mysql mysql:5.7


conf文件夹创建my.cnf文件
写入
[client]
default_character_set=utf8


[mysqld]
collation_server=utf8_general_ci
character_set_server=utf8
```

##### 安装redis

```shell
docker run -p 6379:6379 --name redis --privileged=true -v /mydata/redis/conf/redis.conf:/etc/redis/redis.conf -v /mydata/redis/data:/data -d redis:6.0.8 redis-server /etc/redis/redis.conf

redis.conf
#注释 bind 127.0.0.1
#appendonly yes

```

#### 进阶

##### mysql主从复制

###### 新建主服务器容器实例3307

```shell
docker run -p 3307:3306 --name mysql-server --privileged=true -v /mydata/mysql-master/log:/var/log/mysql \
-v /mydata/mysql-master/data:/var/lib/mysql \
-v /mydata/mysql-master/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=123456 \
-d mysql:5.7
```

###### conf中新建my.cnf

```sql
[mysqld]
server_id=101
#自带数据库
binlog-ignore-db=mysql
log-bin=mall-mysql-bin
binlog_cache_size=1M
binlog_format=mixed
expire_logs_days=7
slave_skip_errors=1062
```

###### 重启master

###### 进入容器 

`docker exec -it mysql-server /bin/bash`

`mysql -uroot -p`

###### 创建数据同步用户

```sql
create user 'slave'@'%' identified by '123456';
grant replication slave,replication client on *.* to 'slave'@'%';
```

###### 新建从服务器 3308

```
docker run -p 3308:3306 --name mysql-slave --privileged=true -v /mydata/mysql-slave/log:/var/log/mysql \
-v /mydata/mysql-slave/data:/var/lib/mysql \
-v /mydata/mysql-slave/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=123456 \
-d mysql:5.7
```

###### conf中新建my.cnf

```sql
[mysqld]
server_id=102
binlog-ignore-db=mysql
log-bin=mall-mysql-slave1-bin
binlog_cache_size=1M
binlog_format=mixed
expire_logs_days=7
slave_skip_errors=1062
relay_log=mall-mysql-relay-bin
log_slave_updates=1
read_only=1
```

###### 重启slave

###### 在主数据库中查看同步状态

`show master status`

###### 进入从机

###### 在从数据库中配置主从复制

`change master to master_host='192.168.5.128', master_user='slave', master_password='123456', master_port=3307, master_log_file='mall-mysql-bin.000001', master_log_pos= 617, master_connect_retry=30;`

**注意同步文件名称**

######  在从数据库中查看同步状态

`show slave status \G`     \G 可加可不加

######  在从数据库中开启主从同步

  `start slave`
