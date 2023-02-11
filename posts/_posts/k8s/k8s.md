---
title: 搭建 Kubernetes 集群
date: 2022-08-08
comments:  true
---

记录搭建 kubernets 集群过程，以往都是从 Rancher 直接拉一个 RKE 集群，一键式傻瓜操作学不到什么真本事，今天尝试使用 kubeadm 进行搭建。

<!--more-->

如何容器化 kubelet：

kubelet 不仅要和容器运行时打交道，在给容器挂载持久化设备时还需要直接在宿主机上操作。如果将 kubelet 容器化，那么他的宿主机就是容器，容器之下才是真正的宿主机。



所以，在部署 kubelet 时，最好是直接部署在宿主机上，其他的组件可以进行容器化部署。



## 安装 kubeadm kubectl kubelet

- 禁用 swap

  `swapoff -a`

  ```shell
  vim /etc/sysctl.d/k8s.conf
  
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1    
  vm.swappiness=0
  
  sysctl --system
  ```

  

- 配置镜像源

  ```yaml
  
  # /etc/yum.repos.d/kubernetes.repo
  cd /etc/yum.repos.d/
  
  vim kubernetes.repo
  
  [kubernetes]
  name=kubernetes
  baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
  gpgcheck=0
  enable=1
  ```

  ```yaml
  # docker-ce 镜像源
  cd /etc/yum.repos.d/
  wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
  ```

- 刷新 yum 源

  ```shell
  yum clean all
  yum repolist
  ```

- 安装 `kubectl kubeadm kubelet`

  `yum install -y docker-ce kubeadm-1.23.1 kubectl-1.23.1 kubelet-1.23.1`

- 启动服务

  ```
  systemctl enable docker
  systemctl enable kubelet.service
  systemctl start docker
  systemctl start kubelet
  ```

- 配置 kubeadm 启动参数

  ```yaml
  # kubeadm.yaml
  apiVersion: kubeadm.k8s.io/v1beta3  #版本信息参考kubeadm config print init-defaults命令结果
  kind: ClusterConfiguration
  kubernetesVersion: 1.23.1  #版本信息参考kubeadm config print init-defaults命令结果
  imageRepository: registry.aliyuncs.com/google_containers #配置国内镜像
  
  apiServer:
    extraArgs:
      runtime-config: "api/all=true"
  ```

- 执行 `kubeadm init --config kubeadm.yaml`



kubeadm join 10.140.9.249:6443 --token as4an7.k3ht0rn31k18dkbt \
	--discovery-token-ca-cert-hash sha256:d1639704220f10f203fe606ef40464d5671e76db79e716ce3ef007eca659fa19 



init 执行完毕后会输出一个注册命令，可以将这个命令复制到worker节点中，使其加入到集群。



但是在这之前，我们的maste节点状态仍然显示的是 NotReady 状态，原因是我们还没有部署网络插件：

```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

执行上述命令安装网络插件，之后节点就是 Ready 状态了。



在 worker 节点上执行相同操作步骤，然后使用那个 Token 执行注册即可。



## kubeadm init 执行流程

- 一系列检查操作，检查是否可以用来部署 Kubernetes

- 生成 Kubernetes 对外提供服务所需的各种证书和对应的目录

  kubernetes 对外提供服务时，除非专门开启“不安全模式”，否则都要通过 HTTPS 才能访问 kube-apiserver，所以这就需要为集群配置好证书文件。



```shell
  Installing : kubelet-1.24.3-0.x86_64 
  Installing : kubernetes-cni-0.8.7-0.x86_64 
  Installing : kubectl-1.24.3-0.x86_64
  Installing : kubeadm-1.24.3-0.x86_64
```



## 问题记录

- failed to run Kubelet: misconfiguration: kubelet cgroup driver: \"systemd\" is different from docker cgroup driver: \"cgroupfs

  这个需要配置 `/etc/docker/daemon.json` 

  ```yaml
  {
          "exec-opts": ["native.cgroupdriver=systemd"],
          "log-driver": "json-file",
          "log-opts": {
                  "max-size": "100m"
          }
  }
  ```



- Error getting node" err="node \"k8smaster\" not found

  该问题暂时没找到解决办法，当时部署的是 1.24.0 的集群版本，降低到 1.23.1 版本后即可。



- FileAvailable--etc-kubernetes-manifests-kube-scheduler.yaml]: /etc/kubernetes/manifests/kube-scheduler.yaml already exists

  重新执行 kubeadm init 时报错，因为一些文件在上次执行时已经创建出来了，可以使用 kubeadm reset 命令恢复

  

- 如果 kubeadm 最后输出的token我们找不到了该怎么办？
  - 可以使用 kubeadm token list 查看token列表
  - 使用 kubeadm token create 命令创建 token