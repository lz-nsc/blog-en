---
title: Setup Kubernetes cluster
date: 2021-09-14 14:35:42
updated: 2021-09-14 15:24:00
tags:
    - Kubernetes
categories:
comments: true
hidden: true
---

This time I'm going to set up a `Kubernetes cluster` with `kubeadm` and record the whole process and problems.

# Preparation

To set up the cluster, we need 2 virtual machines: One works as `Master node` while the other works as `worker node`, which allows me to do more tests related to cluster in the future.

Virtual Machine:

-   AWS t2.large(2vCPU 8GiB) \*2

Tools:

-   kubeadm

kubeadm 是一个可以帮助一键搭建 Kubernetes 集群的工具。

# 搭建 Master 节点

## #1 安装 kubeadm 以及相关工具

首先需要配置云服务器，安装需要的工具。

```bash
$sudo apt-get update
$sudo apt-get install gnupg
$curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
OK
```

需要将 Kubernetes 官方源添加到本地。

```
#/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
```

将以上文件（/etc/apt/sources.list.d/kubernetes.list）添加了之后，运行以下命令可以看到此时 Kubernetes 的源已经被添加进来。

```bash
$sudo apt-get update

...
Get:4 https://packages.cloud.google.com/apt kubernetes-xenial InRelease [9383 B]
Get:6 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 Packages [49.4 kB]
...
```

在完成了以上的配置之后即可开始安装`kubelet`，`kubectl`以及`kubeadm`

`kubeadm`需要使用`kubelet`服务来以容器方式部署和启动 Kubernetes 的主要服务，所以需要安装并先启动 kubelet 服务。
而`kubectl`则是客户端命令行工具，在集群搭建完成之后可以通过它查看集群信息与状态。

```bash
# 安装kubelet，kubectl以及kubeadm
$sudo apt-get install kubelet kubectl kubeadm
```

要启动 kubelet 需要先安装并启动`docker`

```bash
$curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/debian/gpg | sudo apt-key add -
OK

# 要安装了 software-properties-common 才能使用 add-apt-repository
$sudo apt-get install software-properties-common
$sudo add-apt-repository \
"deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/debian \
$(lsb_release -cs) \
stable"
$sudo apt-get update
# 安装docker
$sudo apt-get install docker-ce
```

完成了以上所需工具的安装后就可以开始按顺序启动它们。

```bash
#启动docker
$systemctl start docker
```

在 docker正常启动并确定docker的`cgroup driver`为`systemd`后（详情可见`错误及解决方案 #2`），可开始安装`Master节点`。

此时如果用`systemctl status kubelet`命令查看`kubelet`状态，可发现kubelet没有正常启动，报错为：
```
 "Failed to load kubelet config file" err="failed to load Kubelet config file...
```
这是因为还没有运行 `kubeadm init`命令，在这个命令运行了之后，`kubelet`的配置文件会被生成，`kubelet`也会自动重启并正常运行。

## #2 使用 kubeadm 安装 Master 节点

使用`kubeadm config`命令打印`kubeadm`的默认配置.

```bash
#输出kubeadm默认配置
$kubeadm config print init-defaults
#将kubeadm默认配置保存到文件中方便做自定义修改
$kubeadm config print init-defaults >> init.default.yaml
```

以下是`kubeadm`打印出来的默认配置:

```
#init.default.yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:

-   groups:
    -   system:bootstrappers:kubeadm:default-node-token
        token: abcdef.0123456789abcdef
        ttl: 24h0m0s
        usages:
    -   signing
    -   authentication
        kind: InitConfiguration
        localAPIEndpoint:
        advertiseAddress: 1.2.3.4
        bindPort: 6443
        nodeRegistration:
        criSocket: /var/run/dockershim.sock
        imagePullPolicy: IfNotPresent
        name: node
        taints: null

---

apiServer:
timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
local:
dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: 1.22.0
networking:
dnsDomain: cluster.local
serviceSubnet: 10.96.0.0/12
scheduler: {}
```

用以下命令可以查看镜像列表:

```bash
$kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.22.1
k8s.gcr.io/kube-controller-manager:v1.22.1
k8s.gcr.io/kube-scheduler:v1.22.1
k8s.gcr.io/kube-proxy:v1.22.1
k8s.gcr.io/pause:3.5
k8s.gcr.io/etcd:3.5.0-0
k8s.gcr.io/coredns/coredns:v1.8.4
```

可以使用`kubeadm config images pull`命令将这些镜像提前拉下来。就算不提前拉取，在后续步骤运行`kubeadm init`命令的时候也会自动进行拉取。

使用以下命令就可直接安装 Master 节点：

```bash
$sudo kubeadm init --pod-network-cidr=172.30.0.0/16
```

这里注意一定要加上后面的`—pod-network-cidr`参数，否则在安装 flannel（网络插件）时会报以下错误：

```
E0913 19:45:12.323393 1 main.go:293] Error registering network: failed to acquire lease: node "ip-172-31-40-163" pod cidr not assigned
```

这次我没有修改默认的配置，如果有自定义配置，可以运行

```bash
$sudo kubeadm init –config=<config_file_path>
```

安装完成后会得到以下相关提示:

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.40.163:6443 --token 61gwee.be4wj16mlyjsahaj \
 --discovery-token-ca-cert-hash sha256:97ea59547a4cca2fbcf62360b3561c6e27dd4e1a294533505490391dab872daf
```

根据提示可以运行以下命令，这些命令是为了方便用户通过`kubectl`访问集群，`config`文件里配置了访问集群的入口，用户以及 token 等。

```bash
$mkdir -p $HOME/.kube
$sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

完成配置后就可以使用 kubectl 访问集群了。

使用`kubectl get node`可以看到，目前`master node`的状态是`Not Ready`。

```
NAME             STATUS     ROLES                AGE    VERSION
ip-172-31-40-163 NotReady   control-plane,master 14m    v1.22.1
```

使用`kubectl get node <node_name> -o yaml`(或者`kubectl describe node <node_name>`)可以查看具体原因：

```yaml
...
message: 'container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady
message:docker: network plugin is not ready: cni config uninitialized'
reason: KubeletNotReady
status: "False"
...
```

这是由于`kubeadm`的安装过程**不涉及**`网络插件（CNI)`的初始化，所以集群没有网络功能。

```bash
$kubectl get pod --all-namespaces
NAMESPACE   NAME                        READY   STATUS  RESTARTS AGE
kube-system coredns-78fcd69978-59b4t     0/1     Pending 0       10m
kube-system coredns-78fcd69978-hc8qn     0/1     Pending 0       10m
kube-system etcd-ip-172-31-40-163        1/1     Running 3       10m
kube-system kube-apiserver-ip-172-...    1/1     Running 2       10m
kube-system kube-controller-manager-...  1/1     Running 2       10m
kube-system kube-proxy-dzklf             1/1     Running 0       10m
kube-system kube-scheduler-ip-172-...    1/1     Running 3       10m
```

可以观察到`kubadm`已经为`master 节点`启动了`coredns`,`etcd`,`kube-apiserver`,`kube-controller-manager`,`kube-proxy`以及 `kube-scheduler`了。
并且与网络相关`coredns`由于没有网络也无法正常启动。

## #3 安装网络插件 Flannel

首先下载在 Kubernetes 集群内安装`flanner`所需的配置文件，里面包括了启动该服务所需的所有资源的配置

```bash
$wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

使用下载的 Yaml 文件在集群内启动所有相关资源

```bash
$kubectl apply -f kube-flannel.yml

podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

现在查看`pod`的状态可以发现所有的`pod`都正常运行(`Running`)中

```bash
$ kubectl get pod --all-namespaces
NAMESPACE NAME READY STATUS RESTARTS AGE
kube-system coredns-78fcd69978-xx27f 1/1 Running 0 3m40s
kube-system coredns-78fcd69978-zxnw5 1/1 Running 0 3m40s
kube-system etcd-ip-172-31-40-163 1/1 Running 4 3m54s
kube-system kube-apiserver-ip-172-31-40-163 1/1 Running 3 3m54s
kube-system kube-controller-manager-ip-172-31-40-163 1/1 Running 0 3m57s
kube-system kube-flannel-ds-26tcn 1/1 Running 0 5s
kube-system kube-proxy-8c4md 1/1 Running 0 3m40s
kube-system kube-scheduler-ip-172-31-40-163 1/1 Running 4 3m54s
```

而`master 节点`也切换到了`Ready`的状态

```bash
$ kubectl get node
NAME                STATUS  ROLES                   AGE     VERSION
ip-172-31-40-163    Ready   control-plane,master    4m47s   v1.22.1
```

# 搭建 Node 节点
重复以上在`Master节点`中安装`kubeadm`等工具的过程。

运行`kubeadm`在`Master 节点`安装完成后返回的将`worker 节点`加入集群的命令:

```bash
$kubeadm join 172.31.40.163:6443 --token 61gwee.be4wj16mlyjsahaj \
 --discovery-token-ca-cert-hash sha256:97ea59547a4cca2fbcf62360b3561c6e27dd4e1a294533505490391dab872daf
```

如果在创建 `Master节点`后忘记了以上命令及token，则可用`kubeadm token create --print-join-command`命令重新生成token并输出完整的`join`命令。

在以上命令成功运行后，在`Master节点`通过`kubectl get node`可观察到，集群已有两个节点：

```bash
$kubectl get node

NAME               STATUS   ROLES                  AGE   VERSION
ip-172-31-40-163   Ready    control-plane,master   23h   v1.22.1
ip-172-31-47-241   Ready    <none>                 15s   v1.22.1
```

# 错误及解决方案

## #1 docker 连接 Docker daemon socket 失败

运行`docker info`命令时得到以下错误：

```
ERROR: Got permission denied while trying to connect to the Docker daemon socket ...
```

需要确认 docker 组已经创建并且当前使用的用户在这个组内。

```bash
$sudo groupadd docker
$sudo gpasswd -a <username> docker
$newgrp docker
$systemctl restart docker
```

完成以上步骤后，再次运行`docker info`命令可以正确得到所需信息。

## #2 Kubelet 启动失败

用`systemctl status kubelet`命令获得进程号后，再用`journalctl \_PID=<进程号>|vim – `查看日志，获得了以下信息：

```bash
Sep 13 17:30:09 ip-172-31-40-163 kubelet[18094]: E0913 17:30:09.620375 18094 server.go:294] "Failed to run kubelet" err="failed to run Kubelet: misconfiguration: kubelet cgroup driver: \"systemd\" is different from docker cgroup driver: \"cgroupfs\""
```

原因是`docker`和`kubelet`使用的`cgroup driver`不一样，一个是`systemd`，一个是`cgroupfs`。根据 Kubernetes 官网文档：

> `systemd driver` is **recommended** for `kubeadm` based setups instead of the `cgroupfs driver`, because kubeadm manages the kubelet as a systemd service.

`systemd`是比较推荐的,也是 kubeadm 默认设置的 cgroup drive。（据说 systemd 更安全）所以这里我统一设置成使用 systemd。也就是需要把 `docker` 的`cgroup driver`更改成`systemd`

修改`/etc/docker/daemon.json`（或创建）

```
#/etc/docker/daemon.json
{
"exec-opts":["native.cgroupdriver=systemd"]
}
```

在完成修改之后，用以下命令更新配置并重启 docker

```bash
$systemctl daemon-reload
$systemctl restart docker
```

重启后使用`docker info`命令可获得 docker 的所有相关信息,可以确认 docker 的`cgroup driver`已经换成了`systemd`。

```bash
$ docker info

...
Logging Driver: json-file
Cgroup Driver: systemd
Cgroup Version: 1
...

```

此时再确认 kubelet 的状态可以看到 kubelet 已经自动重启并正常运行。