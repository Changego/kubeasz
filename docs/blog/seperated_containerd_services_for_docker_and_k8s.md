# 为docker和k8s创建独立的containerd进程

## 背景

公司有一台带GPU显卡的服务器，为了通过vllm推理镜像运行像“qwen3-32b”这样的大语言模型，已经配置了带`nvidia-container-runtime`运行时的docker服务；现在需要该服务器加入到k8s集群；因为docker和k8s都需要containerd服务；为了避免冲突，需要两个完全独立的 containerd 实例。

## 方案

必须隔离的 6 个关键点


| 项 | Docker containerd | K8s containerd |
| :--- | :--- | :--- |
| binary | Docker 自带 | 系统安装|
| systemd unit | docker.service	| containerd-k8s.service |
| socket | /var/run/docker/containerd.sock | /run/containerd-k8s/containerd.sock |
| root | /var/lib/docker/containerd | /var/lib/containerd-k8s |
| state | /run/docker/containerd | /run/containerd-k8s |
| config | Docker 管理 | /etc/containerd-k8s/config.toml |

## 基于kubeasz 安装步骤

`kubeasz` 3.6.9 版本以上支持快速配置自定义的`containerd`服务; 在正常安装之前，首先修改 example/config.yml 配置文件参考如下：

```
# [containerd] root 存储目录，默认：/var/lib/containerd
CONTAINERD_ROOT_DIR: "/var/lib/k8scontainerd"

# [containerd] state 存储目录，默认：/run/containerd
CONTAINERD_STATE_DIR: "/run/k8scontainerd"

# [containerd] config 目录，默认：/etc/containerd
CONTAINERD_CONFIG_DIR: "/etc/k8scontainerd"

# [containerd] systemd service 名称，默认：containerd.service
CONTAINERD_SERVICE_NAME: "k8scontainerd.service"
```

然后按照正常的安装流程即可。

## 验证

```
ps -ef | grep containerd

# 可以看到两个不同的进程

/opt/kube/bin/containerd-bin/containerd --log-level warn --config /etc/k8scontainerd/config.toml
/usr/bin/containerd --config /var/run/docker/containerd/containerd.toml
```

That's it. Have Fun!
