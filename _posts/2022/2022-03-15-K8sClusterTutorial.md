---
title: Kubernetes 集群搭建
date: 2022-03-15 17:57:19 +0800
toc: true
comments: true
tags:
  - Linux
  - Kubernetes
---

## <font color=DodgerBlue>k8s 搭建流程</font>

### <font color=DodgerBlue>基础工具及 k8s 安装</font>

#### <font color=DodgerBlue>CentOS</font>

```shell
yum install git -y
git clone https://github.com/SeptemberHX/scripts.git
cd scripts
./basic_utils.sh
git clone https://github.com/SeptemberHX/scripts.git # 注意版本号格式
./k8s_install.sh 1.13.1-0
```

#### <font color=DodgerBlue>Ubuntu</font>

```shell
sh -c "$(curl -fsSL https://raw.githubusercontent.com/SeptemberHX/scripts/master/one-key-k8s-ubuntu.sh)"
```

### <font color=DodgerBlue>初始化 master 节点</font>

`kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v1.13.1 --apiserver-advertise-address=IP`

gcr.io 无法访问时，拉取 k8s 镜像的替代方案

```shell
./k8s_gxrcio.sh v1.13.1   # 注意版本号格式
```

### <font color=DodgerBlue>安装 flannel</font>

`kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`

### <font color=DodgerBlue>开放 http api 以允许外部操作集群</font>

官网文档推荐使用 `kubectl proxy`，从安全因素考量

```shell
kubectl proxy --port=8082 --address='0.0.0.0' --accept-hosts='^*$' &
```
注意 `--accept-hosts` ，否则可能出现 `forbidden` 错误。

------

## <font color=DodgerBlue>问题</font>

### <font color=DodgerBlue>kubelet isn't running or healthy</font>

原问题链接：https://stackoverflow.com/questions/52119985/kubeadm-init-shows-kubelet-isnt-running-or-healthy

解决：`/etc/docker/daemon.json` 文件中添加一下内容：
```json
{
    "exec-opts": ["native.cgroupdriver=systemd"]
}
```
之后执行命令：
```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl restart kubelet
```

### <font color=DodgerBlue>network plugin is not ready: cni config uninitialized</font>

具体影响：每个节点状态永远是 NotReady，无法进行容器调度

解决：`kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`

原问题链接：[github issue](https://github.com/kubernetes/kubeadm/issues/1031#issuecomment-410253279)

或者试一试：`mkdir /etc/cni/net.d`

### <font color=DodgerBlue>swap</font>

kubeadm 添加参数：`--ignore-preflight-errors Swap`
kubelet 参数配置：`/etc/sysconfig/kubelet` ： `--fail-swap-on=false`
有时 `/etc/sysconfig/kubelet` 不好使时，可以将参数加在 `/var/lib/kubelet/kubeadm-flags.env` 
```shell
KUBELET_EXTRA_ARGS="--runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice --fail-swap-on=false"
```

### <font color=DodgerBlue>输出 pod node status 列表：</font>

`kubectl get pod -o=custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName --all-namespaces`

### <font color=DodgerBlue>kubeadm CPU 核心需求</font>

kubeadm the number of available cpus 1 is less than the required 2

它有配置要求：
* 2 CPUs or more
* 2 GB or more of RAM per machine (any less will leave little room for your apps)
* Swap disabled. You MUST disable swap in order for the kubelet to work properly.

### <font color=DodgerBlue>节点加入后 NotReady</font>

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

或者
```shell
mkdir /etc/cni/net.d -p
systemctl restart kubelet
```

有时 `/etc/sysconfig/kubelet` 不好使时，可以将参数加在 `/var/lib/kubelet/kubeadm-flags.env` 

问题 issue：[github issue](<https://github.com/kubernetes/kubeadm/issues/1031#issuecomment-410253279>)

### <font color=DodgerBlue>Flannel</font>

需要在 `kubeadm init` 时指定 `--pod-network-cidr=10.244.0.0/16` 

### <font color=DodgerBlue>容器一直处于创建状态</font>

查看 `systemctl status kubelet` ，发现：` Failed to get system container stats for "/system.slice/docker.service": failed to get cgroup stats for "/system.slice/docker.service": failed to`

在 `/etc/sysconfig/kubelet` 中添加：`KUBELET_EXTRA_ARGS="--runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice"`

Issue 链接：[github issue](https://github.com/kubernetes/kops/issues/4049)

### <font color=DodgerBlue>阿里云没有公网ip，master 上只看见它的局域网ip：无解</font>

创建公网ip：

`ifconfig eth0:0 IP`

然后在 `/etc/sysconfig/kubelet` 中的 `KUBELET_EXTRA_ARGS` 添加 `--node-ip=IP` 即可

------

## <font color=DodgerBlue>常用命令</font>

### <font color=DodgerBlue>初始化</font>

`kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v1.13.1 --apiserver-advertise-address=IP`

### <font color=DodgerBlue>安装 flannel</font>

`kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`

### <font color=DodgerBlue>安装 weave scope</font>

`kubectl apply -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"`

### <font color=DodgerBlue>weave scope 开放端口，及 ssh 隧道</font>

`kubectl port-forward -n weave "$(kubectl get -n weave pod --selector=weave-scope-component=app -o jsonpath='{.items..metadata.name}')" 4040`

`ssh -N -L 0.0.0.0:4041:localhost:4040 root@IP -p 22`

### <font color=DodgerBlue>运行 master 节点运行 pod</font>

```shell
kubectl describe node t0 | grep Taints  # 查看 taints
kubectl taint node t0 node-role.kubernetes.io/master:NoSchedule-  # 取消 NoSchedule
```

### <font color=DodgerBlue>给节点打标签</font>

```shell
kubectl label nodes t0 node=node0  # 给节点 t0 打上标签：node=node0
kubectl get nodes --show-labels    # 查看节点标签
```

### <font color=DodgerBlue>清理 cni：</font>

```shell
ifconfig cni0 down
ip link delete flannel.1
brctl delbr cni0
```

------

### <font color=DodgerBlue>注意</font>

- `kubeadm reset` 后还不够，还需要清理 network interface: cni0 和 flannel.1，否则可能出现无法解析地址

### <font color=DodgerBlue>端口</font>

防火墙方面需要放行，具体参考官方文档：[Ports and Protocols Kubernetes](https://kubernetes.io/docs/reference/ports-and-protocols/)