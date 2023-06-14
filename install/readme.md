



# 1.准备工作

## 关闭swap
```
sudo swapoff -a
sudo rm /swapfile
sudo vim /etc/fstab
sudo swapon --show
```

## 安装kubeadm， kubelet kubectl 1.23.10版本
```
sudo vim /etc/apt/sources.list.d/kubernetes.list  # 设置为中科大ubuntu镜像源 deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
cd ~/Downloads

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg 
sudo apt-key add apt-key.gpg 

sudo apt remove kubelet kubeadm kubectl
sudo apt update
sudo apt install -y kubelet=1.23.0-00 kubeadm=1.23.0-00 kubectl=1.23.0-00
sudo apt autoremove 
```

## 检查docker和kubernetes的cgroup是否一致
```
docker info | grep Cgroup
sudo cat  /var/lib/kubelet/config.yaml | grep cgroup
sudo vim /etc/docker/daemon.json
systemctl daemon-reload
systemctl restart docker
```

/etc/docker/daemon.json
```
{
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    },
    "registry-mirrors": [
        "https://dhx52mku.mirror.aliyuncs.com"
    ],
    "exec-opts": [
        "native.cgroupdriver=systemd"
    ]
}
```


## host

```
192.168.99.83    dx-System-Product-Name
```


# 部署主节点

## 编写kubeadm init config
mkdir -p ~/data/kubeadm
cd ~/data/kubeadm
vim kubeadm.yaml
```
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
controllerManager:
  extraArgs:
    horizontal-pod-autoscaler-use-rest-clients: "true"
    horizontal-pod-autoscaler-sync-period: "10s"
    node-monitor-grace-period: "10s"
apiServer:
  extraArgs:
    runtime-config: "api/all=true"
kubernetesVersion: "v1.23.10"
imageRepository: "registry.aliyuncs.com/google_containers"  # 使用阿里云镜像源
```

## init
```
sudo kubeadm reset
sudo kubeadm init --config kubeadm.yaml

kubectl get nodes -o wide
kubectl get pod -n kube-system -o wide

# 检查集群状态
kubectl get componentstatus


```

## cni插件

需要kube-controller-manager的网段和kube-flannel的network一致
sudo vim /etc/kubernetes/manifests/kube-controller-manager.yaml
```
# 增加以下参数
--allocate-node-cidrs=true
--cluster-cidr=10.244.0.0/16
```

sudo vim /etc/cni/net.d/10-flannel.conflist
```
{
  "name": "cbr0",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
} 
```

```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
sed -i 's/quay.io/quay-mirror.qiniu.com/g' kube-flannel.yml
kubectl apply -f kube-flannel.yml
```


## 安装完成后,第一次使用 Kubernetes 集群所需要的配置命令
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


## 设置单节点
```
kubectl describe node dx-system-product-name

kubectl taint nodes --all node-role.kubernetes.io/master-
```

## 安装dashboard插件

```
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
kubectl apply -f recommended.yaml
kubectl get svc,pods  -n kubernetes-dashboard

# 创建账号
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
# 授权
kubectl create clusterrolebinding dashboard-admin-rb --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin
# 获取账号token
kubectl get secrets -n kubernetes-dashboard | grep dashboard-admin
# 提取token
kubectl describe secrets dashboard-admin-token-99mrt -n kubernetes-dashboard


```

访问地址： http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/.


## 安装持久化插件-rook

## k8s镜像下载
主要会遇到gfw的问题，kubeadm init可以通过设置国内源还好，但是安装插件时，通过kubectl来，就会被强，参考[解决 gcr.io 镜像下载失败的 4 种方法](https://huaweicloud.csdn.net/63311832d3efff3090b51d64.html)，使用其中的github仓库来解决，使用还方便。

```
wget https://raw.githubusercontent.com/anjia0532/gcr.io_mirror/master/pull-k8s-image.sh
chmod +x pull-k8s-image.sh

./pull-k8s-image.sh k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.3.0
./pull-k8s-image.sh k8s.gcr.io/sig-storage/csi-snapshotter:v4.2.0
./pull-k8s-image.sh k8s.gcr.io/sig-storage/csi-provisioner:v3.0.0
./pull-k8s-image.sh k8s.gcr.io/sig-storage/csi-attacher:v3.3.0
./pull-k8s-image.sh k8s.gcr.io/sig-storage/csi-resizer:v1.3.0

```


```
git clone --single-branch --branch v1.10.10 https://github.com/rook/rook.git
cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
kubectl create -f cluster.yaml

kubectl get pods -n rook-ceph
```


```
kubeadm version
kubelet --version

# 验证集群状态
kubectl get cs

systemctl enable kubelet
systemctl status kubelet
systemctl restart kubelet
systemctl stop kubelet
```


# 参考资料
- [github-kubeadm](https://github.com/kubernetes/kubeadm)
- [安装 kubeadm](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [github-dashboard](https://github.com/kubernetes/dashboard)
- [github-rook](https://github.com/rook)
- [github-gcr.io_mirror](https://github.com/anjia0532/gcr.io_mirror)
- [github-kubeasz](https://github.com/easzlab/kubeasz)
- [rook quick start](https://rook.github.io/docs/rook/v1.10/Getting-Started/quickstart/#prerequisites)
- [K8s部署部署遇到问题及解决方法](https://blog.csdn.net/weixin_42037525/article/details/124909969)
- [使用官方社区工具kubeadm部署k8s](https://zhuanlan.zhihu.com/p/64661540)
- [解决k8s Error registering network: failed to acquire lease: node “master“ pod cidr not assigne](https://blog.51cto.com/u_4820306/5425052)
- [k8s “Unable to update cni config: No networks found in /etc/cni/net.d“解决方案](https://blog.csdn.net/qq_26545503/article/details/123183184)
