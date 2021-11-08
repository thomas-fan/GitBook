# Centos7.8 kubenetes 环境搭建

## 一. 关闭安全相关

1. 关闭防火墙

```bash
systemctl disable firewalld
systemctl stop firewalld
```

2. 关闭SELINUX

```bash
vim /etc/sysconfig/selinux
```

```bash
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
SELINUX=disabled
```



## 二. 安装 docker

```bash
# yum-utils
yum install -y yum-utils device-mapper-persistent-data lvm2
# 添加 docker 仓库
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
# 安装 docker
yum install -y docker-ce docker-ce-cli containerd.io
# 启动
systemctl start docker
systemctl enable docker
# 配置阿里云镜像,cgroup 驱动为 systemd
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://kivdavgp.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
# 重启
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 三. 使用 kubeadm init安装 master

1. 配置 yum 源（优先配置阿里源）

```bash
# google 源
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
# aliyun 源
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

setenforce 0

```

2. yum 安装

```bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

3. 启动 kubelet,并设为开机自启

```bash
systemctl start kubelet
systemctl enable kubelet
```

4. 关闭 linux swap 系统交换区

```bash
swapoff -a
```

5. 生成初始化配置文件

```bash
kubeadm config print init-defaults > init.default.yaml
#remove 
#localAPIEndpoint:
#  advertiseAddress: 1.2.3.4
```

6. 修改仓库

```bash
vim init.default.yaml
imageRespostory: registry.aliyuncs.com/google_containers
```

7. 拉取镜像

```bash
kubeadm config images pull --config=init-default.yaml
```

8. 预检查

```bash
kubeadm init --config=init-default.yaml phase preflight
```

9. 运行

```bash
kubeadm init --config=init-default.yaml
//记录 token
kubeadm join 172.17.242.64:6443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:2c5d73cbec65d7b2f4e35164d0a6d499d9dd1722451ef64c9b33037a438faa1a
```

10. 杂项

```bash
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```

```bash
[root@iZ2zehn6jf1v4c5p1bxh6pZ ~]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; disabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since 三 2021-08-11 13:48:00 CST; 2s ago
     Docs: https://kubernetes.io/docs/
  Process: 2221 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=1/FAILURE)
 Main PID: 2221 (code=exited, status=1/FAILURE)
```





## 四. 使用 kubeadm join安装 node 节点

1. 前置工作同上

2. 只安装 kubeadm 和 kubelet, node 上不安装 kubectl

```bash
yum install -y kubelet kubeadm --disableexcludes=kubernetes
```

3. 加入集群

```bash
kubeadm join 172.17.242.64:6443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:2c5d73cbec65d7b2f4e35164d0a6d499d9dd1722451ef64c9b33037a438faa1a
```

## 五. master 安装 calico CNI 插件

```bash
 kubectl apply -f "https://docs.projectcalico.org/manifests/calico.yaml"
```

```bash
NAME                      STATUS   ROLES                  AGE   VERSION
iz2zehn6jf1v4c5p1bxh6pz   Ready    <none>                 17m   v1.22.0
node                      Ready    control-plane,master   18m   v1.22.0
```

验证 kubernetes 集群是否工作正常

```bash
kubectl get pods --all-namespaces
```

```bash
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-58497c65d5-6snmd   1/1     Running   0          2m49s
kube-system   calico-node-d5bgp                          1/1     Running   0          2m49s
kube-system   calico-node-fmnlg                          1/1     Running   0          2m49s
kube-system   coredns-7f6cbbb7b8-525nm                   1/1     Running   0          20m
kube-system   coredns-7f6cbbb7b8-hvmv4                   1/1     Running   0          20m
kube-system   etcd-node                                  1/1     Running   3          20m
kube-system   kube-apiserver-node                        1/1     Running   3          20m
kube-system   kube-controller-manager-node               1/1     Running   7          20m
kube-system   kube-proxy-qmn4x                           1/1     Running   0          20m
kube-system   kube-proxy-qw4xp                           1/1     Running   0          18m
kube-system   kube-scheduler-node                        1/1     Running   7          20m
```

若有未完成 pod, 使用指令查看原因

```bash
kubectl --namespace=kube-system describe pod pod_name
```

