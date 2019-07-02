# k8s install

## 环境

三台 CentOS 7.4 服务器：

kube1 、kube2 、kube3 ，配置：2 核 4G

### 关闭、禁用防火墙：

```bash
systemctl stop firewalld
systemctl disable firewalld
```

### 禁用 SELINUX：

```bash
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

创建 /etc/sysctl.d/k8s.conf 文件，添加如下内容：

```bash
tee /etc/sysctl.d/k8s.conf <<end
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
end
```

执行如下命令使修改生效：

```bash
modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
```

## 关闭 swap

Kubernetes 从 1.8 开始要求关闭系统的 Swap ，如果不关闭，默认配置的 kubelet 将无法启动：

```bash
swapoff -a
```

### 安装 Docker-CE

```bash
# step 1: 安装必要的一些系统工具
yum install -y device-mapper-persistent-data lvm2

# Step 2: 添加软件源信息
curl -so /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# Step 3: 更新并安装 Docker-CE
yum install docker-ce.ce

# Step 4: 设置开机启动
systemctl enable docker

# Step 5: 开启Docker服务
sysctemctl restart docker
```

### 配置阿里云镜像加速器：

```bash
tee /etc/docker/daemon.json <<end
{
  "registry-mirrors": ["registry.aliyuncs.com"]
}
end
systemctl daemon-reload
systemctl restart docker
```

## 所需镜像

Master (kube1) 节点所需镜像：

```bash
export image=pause:3.1
docker pull registry.aliyuncs.com/google_containers/${image}
docker tag registry.aliyuncs.com/google_containers/${image} k8s.gcr.io/${image}
docker rmi registry.aliyuncs.com/google_containers/${image}

export image=kube-apiserver:v1.15.0
docker pull registry.aliyuncs.com/google_containers/${image}
docker tag registry.aliyuncs.com/google_containers/${image} k8s.gcr.io/${image}
docker rmi registry.aliyuncs.com/google_containers/${image}

export image=kube-scheduler:v1.15.0
docker pull registry.aliyuncs.com/google_containers/${image}
docker tag registry.aliyuncs.com/google_containers/${image} k8s.gcr.io/${image}
docker rmi registry.aliyuncs.com/google_containers/${image}

export image=kube-controller-manager:v1.15.0
docker pull registry.aliyuncs.com/google_containers/${image}
docker tag registry.aliyuncs.com/google_containers/${image} k8s.gcr.io/${image}
docker rmi registry.aliyuncs.com/google_containers/${image}

export image=kube-proxy:v1.15.0
docker pull registry.aliyuncs.com/google_containers/${image}
docker tag registry.aliyuncs.com/google_containers/${image} k8s.gcr.io/${image}
docker rmi registry.aliyuncs.com/google_containers/${image}

export image=etcd:3.3.10
docker pull registry.aliyuncs.com/google_containers/${image}
docker tag registry.aliyuncs.com/google_containers/${image} k8s.gcr.io/${image}
docker rmi registry.aliyuncs.com/google_containers/${image}

export image=coredns:1.3.1
docker pull registry.aliyuncs.com/google_containers/${image}
docker tag registry.aliyuncs.com/google_containers/${image} k8s.gcr.io/${image}
docker rmi registry.aliyuncs.com/google_containers/${image}
```

Node (kube2 / kube3) 节点所需镜像：

```bash
export image=pause:3.1
docker pull registry.aliyuncs.com/google_containers/${image}
docker tag registry.aliyuncs.com/google_containers/${image} k8s.gcr.io/${image}
docker rmi registry.aliyuncs.com/google_containers/${image}

export image=kube-proxy:v1.15.0
docker pull registry.aliyuncs.com/google_containers/${image}
docker tag registry.aliyuncs.com/google_containers/${image} k8s.gcr.io/${image}
docker rmi registry.aliyuncs.com/google_containers/${image}
```

## 安装脚手架

Master 和 Node 节点 安装 kubelet kubeadm kubectl

```bash
tee /etc/yum.repos.d/kubernetes.repo <<end
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
end

yum install -y kubelet kubeadm kubectl

systemctl enable kubelet && systemctl start kubelet && systemctl daemon-reload
```

## 构建 Kubernetes 集群

如果初始化集群报错了说明版本下载的没有对应，重新下载镜像并且打tag(镜像都在国外的，所以必须aliyun下载下来打上tag才能拉下来)

查看1.15.0版本的k8s可以使用命令获取需要的镜像版本如下（版本一定要对应）:

```bash
kubeadm config images list

k8s.gcr.io/kube-apiserver:v1.15.0
k8s.gcr.io/kube-controller-manager:v1.15.0
k8s.gcr.io/kube-scheduler:v1.15.0
k8s.gcr.io/kube-proxy:v1.15.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```

1、初始化 Master 节点 kube1

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=172.17.58.201 --kubernetes-version=v1.15.0
```

- –pod-network-cidr ：后续安装 flannel 的前提条件，且值为 10.244.0.0/16， 参考资料
- –apiserver-advertise-address ：Master 节点的 IP 地址

输出日志：

```bash
.....
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 172.17.58.201:6443 --token 831rfg.dw0vyb1h3beab5as --discovery-token-ca-cert-hash sha256:623681fde5b2bf564a8631942f31797f9bef75f40b14a86ef75e1d31b43709f1
```

从日志中，可以看出，要使用集群，需要执行如下命令：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

还需要部署一个 Pod Network 到集群中，此处选择 flannel ：

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
```

至此，Master 节点初始化完毕，查看集群相关信息：

```bash
# 查看集群信息
$ kubectl cluster-info
Kubernetes master is running at https://172.17.58.201:6443
KubeDNS is running at https://172.17.58.201:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
# 查看节点信息
$ kubectl get nodes
NAME           STATUS    ROLES     AGE       VERSION
lab-backend1   Ready     master    1m        v1.10.4
# 查看 Pods 信息
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                   READY     STATUS    RESTARTS   AGE
kube-system   etcd-lab-backend1                      1/1       Running   0          2m
kube-system   kube-apiserver-lab-backend1            1/1       Running   0          1m
kube-system   kube-controller-manager-lab-backend1   1/1       Running   0          2m
kube-system   kube-dns-86f4d74b45-zbnz6              3/3       Running   0          3m
kube-system   kube-flannel-ds-n67zn                  1/1       Running   0          1m
kube-system   kube-proxy-qdmqq                       1/1       Running   0          3m
kube-system   kube-scheduler-lab-backend1            1/1       Running   0          2m
```

如果初始化过程出现问题，使用如下命令重置：

```bash
kubeadm reset

rm -rf /var/lib/cni/

rm -f $HOME/.kube/config
```

2、添加 Worker 节点

方式 ① 使用 kubeadm init 时返回的信息kube3

```bash
kubeadm join 172.17.58.201:6443 --token 831831rf83188831rfg.dw0vyb1h3beab5as --discovery-token-ca-cert-hash sha256:623681fde5b2bf564a8631942f31797f9bef75f40b14a86ef75e1d31b43709f1
```

方式 ② 重新生成 token kube1

```bash
kubeadm token generate

kubeadm token create <generated-token> --print-join-command --ttl=24h
```

> –ttl=24h 代表这个Token 的有效期为 24 小时，初始化默认生成的 token 有效期也为 24 小时

加入集群 kube2 / kube3

```bash
kubeadm join 172.17.58.201:6443 --token 41ts3r.n2vw06xbniouo6u5 --discovery-token-ca-cert-hash sha256:f958e234e8554c2352127f356a7eb7dad422c10df9a749156df36e5972cba38b
```

再次查看集群节点 kube1

```bash
$ kubectl get nodes
NAME           STATUS    ROLES     AGE       VERSION
lab-backend1   Ready     master    6m        v1.11.2
lab-backend2   Ready     <none>    56s       v1.11.2
lab-backend3   Ready     <none>    14s       v1.11.2
```

至此，1 Master + 2 Worker 的 kubernetes 集群就创建成功了