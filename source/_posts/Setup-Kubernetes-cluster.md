---
title: Create your own Kubernetes cluster with kubeadm
date: 2021-09-14 14:35:42
updated: 2022-12-09 15:24:00
tags:
    - Kubernetes
categories:
comments: true

---

This article is about how to set up a `Kubernetes cluster` on our server with `kubeadm`, the problems we might meet during this process, and how to deal with them. Also, we can learn more about how the Kubernetes cluster works from this whole process.
<!-- more -->


# Preparation

To set up the cluster, we need 2 virtual machines: One works as a `Master node` while the other works as a `worker node`, which allows me to do more tests related to the cluster in the future.

Virtual Machine:

-   AWS t2.large(2vCPU 8GiB) \*2

Tools:

-   kubeadm

`kubeadm` is a tool that can help users easily create Kubernetes clusters.

# Create a Kubernetes control-plane node

## #1 Install kubeadm and related tools

First of all, we need to set up the server can install all the tools.

```bash
$sudo apt-get update
$sudo apt-get install gnupg
$curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
OK
```

Then we need to add Kubernetes's official package repository.

```
#/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
```

After the file mentioned above（/etc/apt/sources.list.d/kubernetes.list）has been added, we can see that Kubenentes's package repository has been included when we run the command shown below:

```bash
$sudo apt-get update

...
Get:4 https://packages.cloud.google.com/apt kubernetes-xenial InRelease [9383 B]
Get:6 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 Packages [49.4 kB]
...
```

After the configuration above is done, we can start to install `kubelet`, `kubectl`, and `kubeadm`.

`kubeadm` needs to use `kubelet` to deploy and run Kubernetes's services as containers, before the installation, we need to make sure `kubelet` service is running.

`kubectl` is a command line tool that can be used to check the cluster's information and status after creating the cluster.

```bash
# Install kubelet, kubectl and kubeadm
$sudo apt-get install kubelet kubectl kubeadm
```

---

### Using docker
> When I first created a cluster on AWS, `docker` was still used by `kubelet` by default. Somehow now that seems `docker` has been deprecated by `Kubernetes` and when I try to use `kubeadm:v1.25.4` to create a cluster on Vsphere, `containerd` is used by default instead of `docker`. So, if you are not going to use docker, you can directly skip this section.

To run `kubelet`, we need to install and run `docker`:

```bash
$curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/debian/gpg | sudo apt-key add -
OK

# software-properties-common is needed for using add-apt-repository
$sudo apt-get install software-properties-common
$sudo add-apt-repository \
"deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/debian \
$(lsb_release -cs) \
stable"
$sudo apt-get update
# Install docker
$sudo apt-get install docker-ce
```
After finishing the installation of all the tools that we need, we can start to run them by order.

```bash
#First start docker
$systemctl start docker
```

After starting `docker` correctly and making sure that `cgroup driver` of docker is `systemd`(More details can be found in [`Troubleshooting #2`](#T2)), we can start to deploy `Master node` of the cluster.

---

If now we use `systemctl status kubelet` command to check the status of `kubelet`, we can find out that `kubelet` didn't start correctly and the error message is:
```
 "Failed to load kubelet config file" err="failed to load Kubelet config file...
```
This is because we haven't run `kubeadm init` command yet, after this command is executed, the config file for `kubelet` will be generated, and `kubelet` will restart and run correctly.

## #2 Use kubeadm to deploy Master node

Use `kubeadm config` command to print the default configuration of `kubeadm`.

```bash
# Print default configuration of kubeadm
$kubeadm config print init-defaults
#Save the default configuration of kubeadm to file to apply customize modification
$kubeadm config print init-defaults >> init.default.yaml
```

Here is the default configuration of `kubeadm`:

```yaml
#init.default.yaml
apiVersion: kubeadm.k8s.io/v1beta3
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

It can be told from the default configuration that, the configurations of etcd, apiServer, networking and cri are all included here.

We can use the command shown below to list the images kubeadm will use:

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

We can also ues `kubeadm config images pull` command to pull these images in advance. However, even though we do not pull them in advance, they will be automatically pulled when we run `kubeadm init` later.

Then we can use the command shown below to directly deploy the Master node:

```bash
$sudo kubeadm init --pod-network-cidr=172.30.0.0/16
```

Here we need to notice that the `—pod-network-cidr` param is necessary, or this error will be thrown while installing flannel(network plugin):

```
E0913 19:45:12.323393 1 main.go:293] Error registering network: failed to acquire lease: node "ip-172-31-40-163" pod cidr not assigned
```

I didn't modify the default configuration this time, but if you have a customized configuration, then this command can be used:

```bash
$sudo kubeadm init –config=<config_file_path>
```

After the installation is done, we will get this message:

```shell
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

According to the message, we can run the command shown below to config the `kubectl`. The `config` file container the entry of the new cluster and user information:

```bash
$mkdir -p $HOME/.kube
$sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

After the configuration, we can use `kubectl` to access the cluster we've just created.

By using `kubectl get node` command we can see that the status of `master node` is `Not Ready`:
(It's shown `Ready` with `kubeadm:v1.25.4`)

```
NAME             STATUS     ROLES                AGE    VERSION
ip-172-31-40-163 NotReady   control-plane,master 14m    v1.22.1
```

We can use `kubectl get node <node_name> -o yaml`(or`kubectl describe node <node_name>`) command to find out the reason:

```yaml
...
message: 'container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady
message:docker: network plugin is not ready: cni config uninitialized'
reason: KubeletNotReady
status: "False"
...
```

According to the message, this is because the initialization process of `kubeadm` does not include initialization of `Network Plugin(CNI)`, so currently our cluster does not have network feature.

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

Also according to the info shown above, we get to observe that `kubeadm` already start the `coredns`, `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-proxy` and `kube-scheduler` for our master node. But because of the same reason that we've mentioned above, the `coredns` pod which is related to the network can not start normally as well.

## #3 Install Network Plugin Flannel

First, we need to download the file for installing `flannel` network plugin in Kubernetes cluster, which includes the configurations of all the components of this service.

```bash
$wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Create all the components in the cluster with the Yaml file we've gotten

```bash
$kubectl apply -f kube-flannel.yml

podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

After all the components of `Flannel` are started, we can observe that the status of all the pods is `Running` now.

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

And our `master` node has turned to `Ready` as well.

```bash
$ kubectl get node
NAME                STATUS  ROLES                   AGE     VERSION
ip-172-31-40-163    Ready   control-plane,master    4m47s   v1.22.1
```

# Join Node 

To join a new node to our cluster, we can repeat the process of installing tools like `kubeadm` on a new machine.

Then we can execute the command that we've gotten from `kubeadm` after we successfully created the master node to join a new node:

```bash
$kubeadm join 172.31.40.163:6443 --token 61gwee.be4wj16mlyjsahaj \
 --discovery-token-ca-cert-hash sha256:97ea59547a4cca2fbcf62360b3561c6e27dd4e1a294533505490391dab872daf
```

If you forget to save the command and token shown above in the previous precess, you can use `kubeadm token create --print-join-command` command to generate a new token and print the complete `join` command.

After the `join` command was executed, we can observe that there're two nodes in our new cluster now:

```bash
$kubectl get node

NAME               STATUS   ROLES                  AGE   VERSION
ip-172-31-40-163   Ready    control-plane,master   23h   v1.22.1
ip-172-31-47-241   Ready    <none>                 15s   v1.22.1
```

# Troubleshooting


## <a id="T1"></a>#1 Docker failed to connect to Docker daemon socket

If this error occurs when using the command `docker info`：

```
ERROR: Got permission denied while trying to connect to the Docker daemon socket ...
```

Then we need to make sure that the `docker` group has been created and that the account we're currently using is in this group:

```bash
$sudo groupadd docker
$sudo gpasswd -a <username> docker
$newgrp docker
$systemctl restart docker
```

Then we can get the correct information that we need with `docker info` command.

## <a id="T2"></a>#2 Failed to start Kubelet

We probably can get this information by using `journalctl -u kubelet` to check the logs of `kubelet` service:

```bash
Sep 13 17:30:09 ip-172-31-40-163 kubelet[18094]: E0913 17:30:09.620375 18094 server.go:294] "Failed to run kubelet" err="failed to run Kubelet: misconfiguration: kubelet cgroup driver: \"systemd\" is different from docker cgroup driver: \"cgroupfs\""
```

The reason is that `docker` and `kubelet` is using different `cgroup driver`: one is `systemd` while the another one if `cgroupfs`. According to Kubernetes's official documen:

> `systemd driver` is **recommended** for `kubeadm` based setups instead of the `cgroupfs driver`, because kubeadm manages the kubelet as a systemd service.

`systemd` is recommanded, and also used by `kubeadm` by default.(Might because systemd is safer）So here I will use `systemd`, which means that I will need to change `cgroup driver` of `docker` to `systemd`.

Modify `/etc/docker/daemon.json`（or create）

```
#/etc/docker/daemon.json
{
"exec-opts":["native.cgroupdriver=systemd"]
}
```

After the modification, we can reconfigure and restart `docker`:

```bash
$systemctl daemon-reload
$systemctl restart docker
```

If everything works fine, then we can use `docker info` command to gain all the information about `docker`, to make sure that its `cgroup driver` has been changed to `systemd`.

```bash
$ docker info

...
Logging Driver: json-file
Cgroup Driver: systemd
Cgroup Version: 1
...

```

Then we can check the status of `kubelet` again and find out that `kubelet` has restarted and works normally.

## #3 Failed to pull image with `kubeadm config images pull`

An error occurs when running `kuneadm config images pull`:
```shell
$ kubeadm config images pull
failed to pull image "registry.k8s.io/kube-apiserver:v1.25.4": output: E1205 00:26:16.157713 3215543 remote_image.go:222] "PullImage from image service failed" err="rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.ImageService" image="registry.k8s.io/kube-apiserver:v1.25.4" time="2022-12-05T00:26:16-05:00" level=fatal msg="pulling image: rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.ImageService", error: exit status 1
```

Remove `config.toml` for `containerd` and restart it:

```shell
rm /etc/containerd/config.toml
systemctl restart containerd
```

Then we can try to run the command again and the error will disappear.

## #4 Failed to run `kubeadm init`

While running `kubeadm init` command, we might end up getting this error:

```shell
kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.

Unfortunately, an error has occurred:
        timed out waiting for the condition

This error is likely caused by:
        - The kubelet is not running
        - The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)

If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
        - 'systemctl status kubelet'
        - 'journalctl -xeu kubelet'

```

So, according to the error message, first of all, let's check the status and logs of `kubelet`:

```shell
$ systemctl status kubelet
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Mon 2022-12-05 05:14:16
   ...

$ journalctl -f -u kubelet
...
ec 05 05:07:36 localhost.localdomain kubelet[3241658]: E1205 05:07:36.299425 3241658 run.go:74] "command failed" err="failed to run Kubelet: running with swap on is not supported, please disable swap! or set --fail-swap-on flag to false. /proc/swaps contained: [Filename\t\t\t\tType\t\tSize\tUsed\tPriority /dev/dm-1                               partition\t2097148\t0\t-2]"
...
```
So the reason should be `swap`, which is a space on a disk. To fix the problem, we just need to directly turn it off, with command:
```shell
swapoff -a
```
Then we probably need to reset kubeadm with `kubeadm reset` command to start the initialization again.

## #5 ip_forward contents are not set to 1

If this message is shown while running `kubeadm init`:

```shell
[ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
```
Then we need to manually set `ip_forward` to 1 by modifying `/proc/sys/net/ipv4/ip_forward` file:

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```