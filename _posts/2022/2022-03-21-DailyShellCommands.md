---
title: 常用 shell 命令
date: 2022-03-21 21:17:17 +0800
toc: true
comments: true
tags:
  - Linux
  - shell
categories: [手册, Linux]
---

## <font color=DodgerBlue>Kubernetes</font>

```shell
# 0. Ubuntu 20.04 一键安装 kubectl kubeadm etcd 等 kubernetes 相关内容
sh -c "$(curl -fsSL https://raw.githubusercontent.com/SeptemberHX/scripts/master/one-key-k8s-ubuntu.sh)"

# 1. 清理命名空间 hx-test 下所有名称含有 service 的 pod
kubectl get pods -n hx-test | grep service | awk -F ' ' '{print $1}' | xargs kubectl delete pods -n hx-test

# 2. 允许 master 节点运行 pod
kubectl describe node t0 | grep Taints  # 查看 taints
kubectl taint node t0 node-role.kubernetes.io/master:NoSchedule-  # 取消 NoSchedule

# 3. 给 node 打标签
kubectl label nodes t0 node=node0  # 给节点 t0 打上标签：node=node0
kubectl get nodes --show-labels    # 查看节点标签

# 4. token 过期后还需要添加新节点
kubeadm token create --print-join-command
```

## <font color=DodgerBlue>Docker</font>

```shell
# 1. 给用户 user 操作 docker 权限
sudo usermod -aG docker user

# 2. 执行一个运行中容器 0e45245944d4 的 bash
docker exec -it 0e45245944d4 /bin/bash

# 3. 删除包含 service 的镜像
docker image list | grep service | awk -F ' ' '{print $1}' | xargs docker image rm

# 4. 删除所有未使用的 volume，image、container 类似
docker volume prune
```

## <font color=DodgerBlue>Nvidia</font>

```shell
# 1. 列出所有正在使用 NVIDIA GPU 的进程信息
nvidia-smi | grep -A 100 "PID" | tail -n +3 | sed '$d' | grep -v 'No' | grep -v 'N/A' | awk -F ' ' '{print $3}' | xargs --no-run-if-empty ps -ux
```

## <font color=DodgerBlue>日志</font>

```shell
# 1. 获取指定 service 的日志
sudo journalctl -u emby-server.service
```

