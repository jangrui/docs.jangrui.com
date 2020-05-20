# Kubernetes 高可用集群部署

## 环境准备

### 服务器说明

五台 CentOS 虚拟机:

|IP             |节点    |CPU    |Memory   |Hostname |
|-              |-      |-      |-        |-        |
|192.168.11.141 |master |>=2    |>=2G     |m1       |
|192.168.11.142 |master |>=2    |>=2G     |m2       |
|192.168.11.143 |master |>=2    |>=2G     |m3       |
|192.168.11.151 |worker |>=2    |>=2G     |w1       |
|192.168.11.152 |worker |>=2    |>=2G     |w2       |

- keepalived 提供 kube-apiserver 对外服务的 VIP;
- haproxy 监听 VIP，后端连接所有 kube-apiserver 实例，提供健康检查和负载均衡功能；
- 由于 keepalived 是一主多备运行模式，故至少两个节点安装 keepalived 和 haporxy。
- 注意：如果是云服务器（需要申请虚拟IP并绑定到服务器上，公有云不支持 keepalived 虚拟VIP）

### 主机名

每个节点主机名必须不一样,并且保证所有节点之间可以通过 hostname 互相访问。

```bash
hostnamectl set-hostname <节点名> --static
```

所有节点添加 hosts 记录

```bash
cat >> /etc/hosts <<end
192.168.11.141 m1
192.168.11.142 m2
192.168.11.143 m3
192.168.11.151 w1
192.168.11.152 w2
end
scp /etc/hosts root@m2:/etc/hosts
scp /etc/hosts root@m3:/etc/hosts
scp /etc/hosts root@w1:/etc/hosts
scp /etc/hosts root@w2:/etc/hosts
```

### 关闭并清理防火墙

```bash
ssh root@m1 "systemctl stop firewalld && systemctl disable firewalld && iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT"

ssh root@m2 "systemctl stop firewalld && systemctl disable firewalld && iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT"

ssh root@m3 "systemctl stop firewalld && systemctl disable firewalld && iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT"

ssh root@w1 "systemctl stop firewalld && systemctl disable firewalld && iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT"

ssh root@w1 "systemctl stop firewalld && systemctl disable firewalld && iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT"
```

### 关闭交换分区

```bash
ssh root@m1 "swapoff -a && sed -i '/swap/s/^\(.*\)$/#\1/' /etc/fstab"
ssh root@m2 "swapoff -a && sed -i '/swap/s/^\(.*\)$/#\1/' /etc/fstab"
ssh root@m3 "swapoff -a && sed -i '/swap/s/^\(.*\)$/#\1/' /etc/fstab"
ssh root@21 "swapoff -a && sed -i '/swap/s/^\(.*\)$/#\1/' /etc/fstab"
ssh root@w2 "swapoff -a && sed -i '/swap/s/^\(.*\)$/#\1/' /etc/fstab"
```

### 关闭SELINUX

```bash
ssh root@m1 "setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config"
ssh root@m2 "setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config"
ssh root@m3 "setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config"
ssh root@w1 "setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config"
ssh root@w2 "setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config"
```

### 添加系统路由

```bash
tee /etc/sysctl.d/kubernetes.conf <<end
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
end

scp /etc/sysctl.d/kubernetes.conf root@m2:/etc/sysctl.d/kubernetes.conf
scp /etc/sysctl.d/kubernetes.conf root@m3:/etc/sysctl.d/kubernetes.conf
scp /etc/sysctl.d/kubernetes.conf root@w1:/etc/sysctl.d/kubernetes.conf
scp /etc/sysctl.d/kubernetes.conf root@w2:/etc/sysctl.d/kubernetes.conf

ssh root@m1 "modprobe br_netfilter && sysctl -p /etc/sysctl.d/kubernetes.conf"
ssh root@m2 "modprobe br_netfilter && sysctl -p /etc/sysctl.d/kubernetes.conf"
ssh root@m3 "modprobe br_netfilter && sysctl -p /etc/sysctl.d/kubernetes.conf"
ssh root@w1 "modprobe br_netfilter && sysctl -p /etc/sysctl.d/kubernetes.conf"
ssh root@w2 "modprobe br_netfilter && sysctl -p /etc/sysctl.d/kubernetes.conf"
```

### 开启ipvs

```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF

scp /etc/sysconfig/modules/ipvs.modules root@m2:/etc/sysconfig/modules
scp /etc/sysconfig/modules/ipvs.modules root@m3:/etc/sysconfig/modules
scp /etc/sysconfig/modules/ipvs.modules root@w1:/etc/sysconfig/modules
scp /etc/sysconfig/modules/ipvs.modules root@w2:/etc/sysconfig/modules

ssh root@m1 "chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack"
ssh root@m2 "chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack"
ssh root@m3 "chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack"
ssh root@w1 "chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack"
ssh root@w2 "chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack"
```

### 安装依赖

```bash
# 下载 docker-ce 源
ssh root@m1 "curl -so /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo && yum makecache fast"
ssh root@m2 "curl -so /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo && yum makecache fast"
ssh root@m3 "curl -so /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo && yum makecache fast"
ssh root@w1 "curl -so /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo && yum makecache fast"
ssh root@w2 "curl -so /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo && yum makecache fast"

# 安装依赖
ssh root@m1 "yum update -y && yum install -y epel-release conntrack ipvsadm ipset jq sysstat curl iptables libseccomp device-mapper-persistent-data lvm2 docker-ce"
ssh root@m2 "yum update -y && yum install -y epel-release conntrack ipvsadm ipset jq sysstat curl iptables libseccomp device-mapper-persistent-data lvm2 docker-ce"
ssh root@m3 "yum update -y && yum install -y epel-release conntrack ipvsadm ipset jq sysstat curl iptables libseccomp device-mapper-persistent-data lvm2 docker-ce"
ssh root@w1 "yum update -y && yum install -y epel-release conntrack ipvsadm ipset jq sysstat curl iptables libseccomp device-mapper-persistent-data lvm2 docker-ce"
ssh root@w2 "yum update -y && yum install -y epel-release conntrack ipvsadm ipset jq sysstat curl iptables libseccomp device-mapper-persistent-data lvm2 docker-ce"

# 启动 docker
ssh root@m1 "systemctl daemon-reload && systemctl enable docker && systemctl restart docker"
ssh root@m2 "systemctl daemon-reload && systemctl enable docker && systemctl restart docker"
ssh root@m3 "systemctl daemon-reload && systemctl enable docker && systemctl restart docker"
ssh root@w1 "systemctl daemon-reload && systemctl enable docker && systemctl restart docker"
ssh root@w2 "systemctl daemon-reload && systemctl enable docker && systemctl restart docker"
```

> docker-ce 官方 repo：<https://download.docker.com/linux/centos/docker-ce.repo>

### 为Docker配置Cgroup

根据文档 [CRI installation](https://kubernetes.io/docs/setup/cri/) 中的内容，对于使用 systemd 作为 init system 的 Linux 的发行版，使用 systemd 作为 docker 的 cgroup driver 可以确保服务器节点在资源紧张的情况更加稳定，因此这里修改各个节点上 docker 的 cgroup driver 为 systemd。

```bash
cat > /etc/docker/daemon.json <<end
{
  "registry-mirrors": ["https://registry.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
end

scp /etc/docker/daemon.json root@m2:/etc/docker/daemon.json
scp /etc/docker/daemon.json root@m3:/etc/docker/daemon.json
scp /etc/docker/daemon.json root@w1:/etc/docker/daemon.json
scp /etc/docker/daemon.json root@w2:/etc/docker/daemon.json

ssh root@m1 "systemctl daemon-reload && systemctl restart docker"
ssh root@m2 "systemctl daemon-reload && systemctl restart docker"
ssh root@m3 "systemctl daemon-reload && systemctl restart docker"
ssh root@w1 "systemctl daemon-reload && systemctl restart docker"
ssh root@w2 "systemctl daemon-reload && systemctl restart docker"
```

- registry-mirrors: 设置 docker 镜像地址，可配置国内镜像加速
- exec-opts: 设置 cgroup driver (默认是 cgroupfs，推荐设置 systemd)
- graph: 设置 docker 数据目录地址

### 安装Kubernetes脚手架

- kubeadm: 部署集群用的命令
- kubelet: 在集群中每台机器上都要运行的组件，负责管理 pod、容器的生命周期
- kubectl: 集群管理工具（可选，只要在控制集群的节点上安装即可）

```bash
cat > /etc/yum.repos.d/kubernetes.repo <<end
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
end

# 分发 kubernetes.repo
scp /etc/yum.repos.d/kubernetes.repo root@m2:/etc/yum.repos.d/kubernetes.repo
scp /etc/yum.repos.d/kubernetes.repo root@m3:/etc/yum.repos.d/kubernetes.repo
scp /etc/yum.repos.d/kubernetes.repo root@w1:/etc/yum.repos.d/kubernetes.repo
scp /etc/yum.repos.d/kubernetes.repo root@w2:/etc/yum.repos.d/kubernetes.repo

# 安装 kubeadm kubelet kubectl 并重启 kubelet
ssh root@m1 "yum makecache fast && yum install -y kubectl kubeadm kubelet && systemctl daemon-reload && systemctl enable kubelet && systemctl restart kubelet"
ssh root@m2 "yum makecache fast && yum install -y kubectl kubeadm kubelet && systemctl daemon-reload && systemctl enable kubelet && systemctl restart kubelet"
ssh root@m3 "yum makecache fast && yum install -y kubectl kubeadm kubelet && systemctl daemon-reload && systemctl enable kubelet && systemctl restart kubelet"
ssh root@w1 "yum makecache fast && yum install -y kubectl kubeadm kubelet && systemctl daemon-reload && systemctl enable kubelet && systemctl restart kubelet"
ssh root@w2 "yum makecache fast && yum install -y kubectl kubeadm kubelet && systemctl daemon-reload && systemctl enable kubelet && systemctl restart kubelet"
```

> google 官方：<https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64>
>
> ### 配置kubelet
>
> kubelet 的 cgroupdriver 默认为 systemd，如果上面没有设置 docker 的 exec-opts 为 systemd，这里就需要将 kubelet 的设置为 cgroupfs。
>
> 由于各自的系统配置不同，配置位置和内容都不相同
>
> 1、 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf(如果此配置存在的情况执行下面命令：)
>
> ```bash
> sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
> systemctl enable kubelet && systemctl start kubelet
> ```
>
> 2、 /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf(如果 1 中的配置不存在，则此配置应该存在，不需要做任何操)

## 部署高可用集群

### 安装Keepalived

> 至少两台 master 节点安装 keepalived，这里选择所有 master 节点都部署 keepalived。

```bash
ssh root@m1 "yum install -y keepalived && systemctl daemon-reload && systemctl enable keepalived && systemctl start keepalived"
ssh root@m2 "yum install -y keepalived && systemctl daemon-reload && systemctl enable keepalived && systemctl start keepalived"
ssh root@m3 "yum install -y keepalived && systemctl daemon-reload && systemctl enable keepalived && systemctl start keepalived"
```

修改 keepalived 配置文件：

```bash
# m1 节点
ssh root@m1 cat > /etc/keepalived/keepalived.conf <<end
! Configuration File for keepalived
global_defs {
  router_id keepalive-1  # 主机名,每个节点不同
}

vrrp_script check_apiserver { # 检测脚本。
  script "/etc/keepalived/check-apiserver.sh"
  interval 3
  weight -2
}

vrrp_instance VI-kube {
  state MASTER                # 主服务器
  interface ens32             # VIP 漂移到的网卡
  virtual_router_id 68        # 多个几点必须相同
  priority 100                # 优先级，备用服务器比主服务器低
  dont_track_primary
  advert_int 3
  virtual_ipaddress {
    192.168.11.188         # vip 虚拟 ip (同网段)
  }
  track_script {
      check_apiserver
  }
}
end

# m2 节点
ssh root@m2 cat > /etc/keepalived/keepalived.conf <<end
! Configuration File for keepalived
global_defs {
  router_id keepalive-2  # 主机名,每个节点不同
}

vrrp_script check_apiserver { # 检测脚本。
  script "/etc/keepalived/check-apiserver.sh"
  interval 3
  weight -2
}

vrrp_instance VI-kube {
  state BACKUP                # 主服务器
  interface ens32             # VIP 漂移到的网卡
  virtual_router_id 68        # 多个几点必须相同
  priority 90                # 优先级，备用服务器比主服务器低
  dont_track_primary
  advert_int 3
  virtual_ipaddress {
    192.168.11.188         # vip 虚拟 ip (同网段)
  }
  track_script {
      check_apiserver
  }
}
end

# m3 节点
ssh root@m3 cat > /etc/keepalived/keepalived.conf <<end
! Configuration File for keepalived
global_defs {
  router_id keepalive-3  # 主机名,每个节点不同
}

vrrp_script check_apiserver { # 检测脚本。
  script "/etc/keepalived/check-apiserver.sh"
  interval 3
  weight -2
}

vrrp_instance VI-kube {
  state BACKUP                # 主服务器
  interface ens32             # VIP 漂移到的网卡
  virtual_router_id 68        # 多个几点必须相同
  priority 80                # 优先级，备用服务器比主服务器低
  dont_track_primary
  advert_int 3
  virtual_ipaddress {
    192.168.11.188         # vip 虚拟 ip (同网段)
  }
  track_script {
      check_apiserver
  }
}
end


ssh root@m1 cat > /etc/keepalived/check_apiserver.sh <<end
#!/bin/sh

netstat -ntlp|grep 6443 || exit 1

end

# 分发检测脚本
scp /etc/keepalived/check_apiserver.sh root@m2:/etc/keepalived/check_apiserver.sh
scp /etc/keepalived/check_apiserver.sh root@m3:/etc/keepalived/check_apiserver.sh

# 重启 keepalived
ssh root@m1 systemctl restart keepalived
ssh root@m2 systemctl restart keepalived
ssh root@m3 systemctl restart keepalived
```

查看虚拟 ip 是否添加

```bash
# 这里的网卡是 ens32
ssh root@m1 ip a show ens32
ssh root@m2 ip a show ens32
ssh root@m3 ip a show ens32
```

### 安装HAproxy

所有 master 节点安装 haproxy

```bash
ssh root@m1 "yum install -y haproxy && systemctl daemon-reload && systemctl enable haproxy && systemctl start haproxy"

ssh root@m2 "yum install -y haproxy && systemctl daemon-reload && systemctl enable haproxy && systemctl start haproxy"

ssh root@m3 "yum install -y haproxy && systemctl daemon-reload && systemctl enable haproxy && systemctl start haproxy"
```

修改 haproxy 配置文件：

```bash
cat > /etc/haproxy/haproxy.cfg <<end
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend  kubernetes-apiserver
    mode tcp
    bind *:16443
    option tcplog
    default_backend             kubernetes-apiserver

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend kubernetes-apiserver
    mode tcp
    balance     roundrobin
    server  m1 192.168.11.141:6443 check
    server  m2 192.168.11.142:6443 check
    server  m3 192.168.11.143:6443 check

# 监控界面
listen  admin
    bind        *:8888                #监控界面的访问的IP和端口
    mode        http
    stats uri   /admin                #URI相对地址
    stats realm Global\ statistics    #统计报告格式
    stats auth  admin:abc123456       #登陆帐户信息
end
scp /etc/haproxy/haproxy.cfg root@m2:/etc/haproxy/haproxy.cfg
scp /etc/haproxy/haproxy.cfg root@m3:/etc/haproxy/haproxy.cfg

# 重启 haproxy
ssh root@m1 systemctl restart haproxy
ssh root@m2 systemctl restart haproxy
ssh root@m3 systemctl restart haproxy
```

查看端口是否正常：

```bash
ssh root@m1 "ss -lnt | grep -E '16443|8888'"
ssh root@m2 "ss -lnt | grep -E '16443|8888'"
ssh root@m3 "ss -lnt | grep -E '16443|8888'"
```

### 部署第一个Master

官方推荐我们使用–config指定配置文件，并在配置文件中指定原来这些flag所配置的内容。具体内容可以查看这里 [Set Kubelet parameters via a config file](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)。这也是Kubernetes为了支持动态Kubelet配置（Dynamic Kubelet Configuration）才这么做的，参考 [Reconfigure a Node’s Kubelet in a Live Cluster](https://kubernetes.io/docs/tasks/administer-cluster/reconfigure-kubelet/)。

使用 `kubeadm config print init-defaults` 可以打印集群初始化默认的使用的配置:

```bash
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: kube-master-a
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.14.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

基于默认配置定制出本次使用kubeadm初始化集群所需的配置文件kubeadm-config.yaml:

```bash
cat > ~/kubeadm-config.yml <<end
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
# 指定 kubernetes 的版本
kubernetesVersion: v1.15.0
# apiServer 的访问地址
controlPlaneEndpoint: "192.168.11.188:6443"
networking:
    podSubnet: "192.168.0.0/16"
# 指定 aliyun 仓库
imageRepository: registry.aliyuncs.com/google_containers
end

kubeadm init --config=kubeadm-config.yml --experimental-upload-certs |tee k8s-init.log
mkdir $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get pods --all-namespaces
```

- --experimental-upload-certs: 表示可以在后续执行加入节点时自动分发证书文件。

### 部署网络插件Flannel

```bash
mkdir -p /etc/kubernetes/addons
curl -o /etc/kubernetes/addons/kube-flannel.yml https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
sed -ie "s?10.244.0.0?192.168.0.0?g" /etc/kubernetes/addons/kube-flannel.yml
kubectl apply -f /etc/kubernetes/addons/kube-flannel.yml
```

> Flannet 默认指定 podSubnet 为 10.244.0.0/16 网段。

### 加入其它Master节点

1、 查看初始化日志。上一步初始化时指定了输出日志，里面有 master 和 worker 节点的 token 信息，默认有效期为 24h。

2、 初始化默认生成的 token 有效期为 24h，如果 token 还未过期且又没有保留初始化日志，可以利用还未过期的 token 添加节点。

查看 token 命令：

```bash
kubeadm token list
```

例如：

```bash
[root@m1 ~]# kubeadm token list|awk 'NR!=1{print $1,$2}'
iasnf5.zlav24b7q28ekoxy 20h
v89wfj.rpm44adpaujvwe25 22h
```

加入 master 节点并查看当前所有节点信息：

```bash
ssh root@m2 "kubeadm join --token iasnf5.zlav24b7q28ekoxy --discovery-token-unsafe-skip-ca-verification --experimental-control-plane && mkdir -p ~/.kube && cp -i /etc/kubernetes/admin.conf ~/.kube/config && chown $(id -u):$(id -g) ~/.kube/config && kubectl get nodes"
ssh root@m3 "kubeadm join --token iasnf5.zlav24b7q28ekoxy --discovery-token-unsafe-skip-ca-verification --experimental-control-plane && mkdir -p ~/.kube && cp -i /etc/kubernetes/admin.conf ~/.kube/config && chown $(id -u):$(id -g) ~/.kube/config && kubectl get nodes"
```

- --discovery-token-unsafe-skip-ca-verification: 忽略 ca 校验
- --experimental-control-plane: 添加 master 节点

3、 重新生成 token

```bash
kubeadm token create --print-join-command --ttl=24h
```

生成的 token：

```bash
kubeadm join 192.168.11.188:6443 --token iasnf5.zlav24b7q28ekoxy     --discovery-token-ca-cert-hash sha256:cb503a6e95702a8ba9ebc35861301126cd03da3d8666d21bebf640cc661545eb
```

- --ttl=24: 表示这个 token 的有效期为 24 小时，初始化默认生成的 token 有效期也是 24 小时

加入 master 节点：

```bash
ssh root@m2 "kubeadm join 192.168.11.188:6443 --token iasnf5.zlav24b7q28ekoxy --discovery-token-ca-cert-hash sha256:cb503a6e95702a8ba9ebc35861301126cd03da3d8666d21bebf640cc661545eb --experimental-control-plane && cp -i /etc/kubernetes/admin.conf ~/.kube/config && chown $(id -u):$(id -g) ~/.kube/config && kubectl get nodes"

ssh root@m3 "kubeadm join 192.168.11.188:6443 --token iasnf5.zlav24b7q28ekoxy --discovery-token-ca-cert-hash sha256:cb503a6e95702a8ba9ebc35861301126cd03da3d8666d21bebf640cc661545eb --experimental-control-plane && cp -i /etc/kubernetes/admin.conf ~/.kube/config && chown $(id -u):$(id -g) ~/.kube/config && kubectl get nodes"
```

### 加入worker节点

> worker 节点的 join 命令没有 `--experimental-control-plane` 参数。

```bash
ssh root@w1 kubeadm join --token iasnf5.zlav24b7q28ekoxy --discovery-token-unsafe-skip-ca-verification

ssh root@w2 kubeadm join --token iasnf5.zlav24b7q28ekoxy --discovery-token-unsafe-skip-ca-verification

kubectl get nodes
```

> ### 移除node节点
>
> 查看所有节点：
>
> ```bash
> kubectl get nodes
> ```
>
> 移除指定节点：
>
> ```bash
> kubectl drain w2 --delete-local-data --force --ignore-daemonsets
> kubectl delete node w2
> ```
>
> 在 w2 节点执行：
>
> ```bash
> kubeadm reset
> ```

## 部署Dashboard

```bash
curl -so /etc/kubernetes/addons/kubernetes-dashboard.yaml https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
kubectl apply -f /etc/kubernetes/addons/kubernetes-dashboard.yaml
kubectl get svc/kubernetes-dashboard  -n kube-system  -o yaml | sed 's/ClusterIP/NodePort/g' | kubectl apply -f -
kubectl get svc/kubernetes-dashboard  -n kube-system  -o yaml | sed 's/nodePort:.*/nodePort: 30443/g' | kubectl apply -f -
```

查看服务运行情况

```bash
kubectl get svc -A
netstat -ntlp|grep 30005
```

从 1.7 开始，dashboard 只允许通过 https 访问，我们使用nodeport的方式暴露服务，可以使用 <https://NodeIP:NodePort> 地址访问 关于自定义证书 默认dashboard的证书是自动生成的，肯定是非安全的证书，如果大家有域名和对应的安全证书可以自己替换掉。使用安全的域名方式访问dashboard。 在dashboard-all.yaml中增加dashboard启动参数，可以指定证书文件，其中证书文件是通过secret注进来的。

- –tls-cert-file
- dashboard.cer
- –tls-key-file
- dashboard.key

### 登录Dashboard

Dashboard 默认只支持 token 认证，所以如果使用 KubeConfig 文件，需要在该文件中指定 token，我们这里使用token的方式登录

```bash
# 创建service account
kubectl create sa dashboard-admin -n kube-system

# 创建角色绑定关系
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
```

或者创建 admin-token.yaml :

```bash
tee <<end> /etc/kubernetes/addons/admin-token.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: adm
  annotations:
    rbac.authorization.kubernetes.io/autoupdateee: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
end

kubectl apply -f /etc/kubernetes/addons/admin-token.yaml
```

### 打印Dashboard的登录token

```bash
cat >> ~/.bashrc <<end
export KDT=$(kubectl describe secret -n kube-system `kubectl get secrets -n kube-system | grep admin-token | awk '{print $1}'` | grep -E '^token' | awk '{print $2}')
alias kdt='echo $KDT'
end

source ~/.bashrc
kdt
```
