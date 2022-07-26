> Kubernetes Version 1.24.1

### 集群准备

:::tip
至少2台服务器或虚拟机完成集群部署
:::

  IP | 主机名 | 用途
---- | ----- | ----
192.168.1.100| k8s-master-01 | 主节点，控制平面
192.168.1.111| k8s-node-01   | 工作节点 
192.168.1.112| k8s-node-02   | 工作节点 

设置主机名
```shell
$ hostnamectl set-hostname k8s-node-* # 最好对应ip或其他容易识别得标识
```

### 运行时
安装Containerd

```shell
$ yum install -y yum-utils
$ yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    
# install
$ yum install containerd.io
```

配置containerd

```shell
$ mkdir -p /etc/containerd

$ containerd config default | sudo tee /etc/containerd/config.toml

# 重启
$ systemctl restart containerd
```
:::tip 可选
修改containerd配置文件中gcr.io地址为国内仓库地址
> `registry.cn-hangzhou.aliyuncs.com/google_containers`
:::


> Runc & cni

* 下载 runc: https://github.com/containerd/containerd/releases
* 下载 cni: https://github.com/containernetworking/plugins/releases

安装：
```shell
$ install -m 755 runc.amd64 /usr/local/sbin/runc

$ mkdir -p /opt/cni/bin
$ tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

* 配置crictl配置文件

#### 使用 systemd cgroup 驱动程序

修改配置文件`/etc/containerd/config.toml`:
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

重启`containerd`
```shell
$ systemctl restart containerd
```

---

### 安装kubeadm

#### 设置iptables
1. 加载`br_netfilter`模块
```shell
modprobe br_netfilter

# 检查是否加载
lsmod | grep br_netfilter
```

2. 设置`sysctl`配置中的`net.bridge.bridge-nf-call-iptables`为1：
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```



### 安装kubeadm, kubelet, kubectl

```shell
# 官方源
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 阿里源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF


# 关闭SELinux
setenforce 0

sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config


# Installing
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes --nogpgcheck

# Enable kubelet
systemctl enable --now kubelet

```

### 初始化
- [ ] kubeadm init on master
- [ ] kubeadm join on each node

### K8S Dashboard
:::tip
新版本创建token与以往不同
```
kubectl -n kubernetes-dashboard create token admin-userkubectl -n kubernetes-dashboard create token admin-user
```
:::

配置：
```shell
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.1/aio/deploy/recommended.yaml
```

修改端口为`nodePort`

### Warning

* 使用阿里云k8s镜像需要修改containerd中sandbox_image地址为对应的镜像
* 修改containerd配置文件grc.io地址为aliyun
