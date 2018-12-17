# Docker 下实现 PXC 集群负载均衡

## 1. 创建 PXC 集群

### 1.1 拉取 pxc 镜像 

```bash
docker pull docker.io/percona/percona-xtradb-cluster
docker tag docker.io/percona/percona-xtradb-cluster pxc # 重命名一个简短名字
docker rmi docker.io/percona/percona-xtradb-cluster
```

### 1.2 给 pxc 集群创建 docker 内部网络

```bash
dcoker network create net1 # 创建网络段 net1
docker network inspect net1 # 查看 net1 网络段信息
docker network rm net1 # 删除网络段 net1
```

### 1.3 给 pxc 集群创建 docker 卷 

```bash
docker volume create --name v1 # 创建 docker 卷 并命名为 v1
docker inspect v1 # 查看 v1 卷
docker volume rm v1 # 删除 v1 卷
```

### 1.4 创建 pxc 容器

```bash
docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -v v1:/var/lib/mysql --privileged --name=node1 --net=net1 --ip 172.18.0.2 pxc
```

### 1.5 创建 pxc 容器集群

以 node1 为基础

```bash
docker run -d -p 3307:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=node1 -v v2:/var/lib/mysql --privileged --name=node2 --net=net1 --ip 172.18.0.3 pxc
docker run -d -p 3308:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=node1 -v v3:/var/lib/mysql --privileged --name=node3 --net=net1 --ip 172.18.0.4 pxc
docker run -d -p 3309:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=node1 -v v4:/var/lib/mysql --privileged --name=node4 --net=net1 --ip 172.18.0.5 pxc
docker run -d -p 3310:3306 -e MYSQL_ROOT_PASSWORD=abc123456 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=abc123456 -e CLUSTER_JOIN=node1 -v v5:/var/lib/mysql --privileged --name=node5 --net=net1 --ip 172.18.0.6 pxc
```

> - CLUSTER_JOIN=node1 : 加入 node1节点.
> - 创建节点时需等前一个节点mysql 初始化完成.


## 2. 负载均衡

### 2.1 拉取haproxy镜像

```bash
docker pull haproxy
```

### 2.2 创建 haproxy 配置文件

```bash
mkdir /home/soft/haproxy
vim /home/soft/haproxy/haproxy.cfg

global
        #工作目录
        chroot /usr/local/etc/haproxy
        #日志文件，使用rsyslog服务中local5日志设备（/var/log/local5），等级info
        log 127.0.0.1 local5 info
        #守护进程运行
        daemon

defaults
        log     global
        mode    http
        #日志格式
        option  httplog
        #日志中不记录负载均衡的心跳检测记录
        option  dontlognull
        #连接超时（毫秒）
        timeout connect 5000
        #客户端超时（毫秒）
        timeout client  50000
        #服务器超时（毫秒）
        timeout server  50000

#监控界面   
listen  admin_stats
        #监控界面的访问的IP和端口
        bind  0.0.0.0:8888
        #访问协议
        mode        http
        #URI相对地址
        stats uri   /dbs
        #统计报告格式
        stats realm     Global\ statistics
        #登陆帐户信息
        stats auth  admin:abc123456
#数据库负载均衡
listen  proxy-mysql
        #访问的IP和端口
        bind  0.0.0.0:3306  
        #网络协议
        mode  tcp
        #负载均衡算法（轮询算法）
        #轮询算法：roundrobin
        #权重算法：static-rr
        #最少连接算法：leastconn
        #请求源IP算法：source 
        balance  roundrobin
        #日志格式
        option  tcplog
        #在MySQL中创建一个没有权限的haproxy用户，密码为空。Haproxy使用这个账户对MySQL数据库心跳检测
        option  mysql-check user haproxy
        server  MySQL_1 172.18.0.2:3306 check weight 1 maxconn 2000  
        server  MySQL_2 172.18.0.3:3306 check weight 1 maxconn 2000  
        server  MySQL_3 172.18.0.4:3306 check weight 1 maxconn 2000 
        server  MySQL_4 172.18.0.5:3306 check weight 1 maxconn 2000
        server  MySQL_5 172.18.0.6:3306 check weight 1 maxconn 2000
        #使用keepalive检测死链
        option  tcpka  
```

### 2.3 创建 haproxy 容器

```bash
docker run -it -d -p 4001:8888 -p 4002:3306 -v /home/soft/haproxy:/usr/local/etc/haproxy --name haproxy1 --privileged --net=net1 --ip 172.18.0.7 haproxy
```

进入 haproxy1 容器,指定haproxy 启动配置指令

```bash
docker exec -it haproxy1 bash
haproxy -f /usr/local/etc/haproxy/haproxy.cfg
```

创建haproxy 的 mysql 用户

```bash
mysql -h 172.18.0.2 -uroot -p
```

```mysql
create user 'haproxy'@'%' identified by '';
```

### 2.4 haproxy 监控:

[http://ip:4001/dbs](https://ip:4001/dbs)
