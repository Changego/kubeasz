# kubeasz GPU 支持使用指南

## 概述

kubeasz 的 GPU 支持扩展分为两个层次：

| 层次 | 职责 | 触发方式 |
|------|------|----------|
| **节点层**（`role: gpu-node`） | 安装 NVIDIA 驱动 + nvidia-container-toolkit，配置 containerd GPU runtime | `ezctl setup <cluster> 08` |
| **集群层**（`role: cluster-addon`） | 部署 device-plugin / HAMi vGPU 调度器 | `ezctl setup <cluster> 07` |

---

## 支持的 GPU/NPU 类型

| 设备 | 插件 | 说明 |
|------|------|------|
| NVIDIA GPU（独占模式） | `nvidia-device-plugin` | 1 Pod 独占 1 块 GPU |
| NVIDIA GPU（虚拟化共享） | **HAMi vGPU** | 多 Pod 共享 1 块 GPU，支持显存/算力隔离 |
| 华为昇腾 NPU | `npu-device-plugin` | 华为 Ascend 系列 NPU |

> ⚠️ `nvidia-device-plugin` 与 `HAMi` **互斥**，二者只能启用其一。

---

## 快速上手

### 第一步：配置 hosts 文件

在 `clusters/<cluster>/hosts` 中，将 GPU 物理节点同时加入 `[kube_node]` 和 `[gpu_node]`：

```ini
[kube_node]
192.168.1.4 k8s_nodename='gpu-worker-01'
192.168.1.5 k8s_nodename='gpu-worker-02'

[gpu_node]
192.168.1.4 k8s_nodename='gpu-worker-01'
192.168.1.5 k8s_nodename='gpu-worker-02'
```

### 第二步：配置 config.yml

#### 场景 A：NVIDIA GPU 独占（nvidia-device-plugin）

```yaml
# role:gpu-node
nvidia_driver_version: "550.54.15"
nvidia_driver_url: "https://example.com/NVIDIA-Linux-x86_64-550.54.15.run"
nvidia_toolkit_install_source: "online"

# role:cluster-addon
nvidia_device_plugin_install: "yes"
nvidia_device_plugin_image: "nvcr.io/nvidia/k8s-device-plugin:v0.17.0"
hami_install: "no"   # 与 HAMi 互斥，必须为 no
```

#### 场景 B：NVIDIA GPU 虚拟化共享（HAMi vGPU）推荐

```yaml
# role:gpu-node（驱动安装相同）
nvidia_driver_version: "550.54.15"
nvidia_driver_url: "https://example.com/NVIDIA-Linux-x86_64-550.54.15.run"
nvidia_toolkit_install_source: "online"

# role:cluster-addon
nvidia_device_plugin_install: "no"   # HAMi 内置 device-plugin，此处必须为 no
hami_install: "yes"
hami_ver: "v2.4.1"
hami_namespace: "kube-system"
hami_node_scheduler_policy: "binpack"   # 或 spread
hami_gpu_scheduler_policy: "spread"
```

#### 场景 C：离线部署

```yaml
nvidia_toolkit_install_source: "offline"
# 将 rpm/deb 包放到 roles/gpu-node/files/nvidia-container-toolkit/

hami_install_source: "offline"
# 将 hami-v2.4.1.tgz 放到 roles/cluster-addon/files/

hami_image_registry: "registry.example.com"   # 内网镜像仓库
```

### 第三步：安装 GPU 驱动（节点层）

```bash
# 方法1：通过 ezctl（推荐）
ezctl setup <cluster> 08
# 或
ezctl setup <cluster> gpu-node

# 方法2：手动
ansible-playbook -i clusters/<cluster>/hosts \
  -e @clusters/<cluster>/config.yml \
  playbooks/08.gpu-node.yml
```

### 第四步：部署 device-plugin 或 HAMi（集群层）

```bash
ezctl setup <cluster> 07
```

---

## 验证安装

### 验证 GPU 驱动

```bash
# 在 GPU 节点上
nvidia-smi

# 验证 containerd GPU runtime
nvidia-container-cli info
```

### 验证 Device Plugin / HAMi

```bash
# 查看 GPU 资源是否注册到节点
kubectl get nodes -o custom-columns=\
  'NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'

# HAMi 特有资源
kubectl get nodes -o custom-columns=\
  'NAME:.metadata.name,\
   GPU:.status.allocatable.nvidia\.com/gpu,\
   GPUMEM:.status.allocatable.nvidia\.com/gpumem'
```

---

## HAMi vGPU 使用示例

### 申请 vGPU（共享模式）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  containers:
    - name: gpu-container
      image: nvidia/cuda:12.0-base
      command: ["nvidia-smi", "-L"]
      resources:
        limits:
          nvidia.com/gpu: 1          # 申请 1 个 vGPU 槽位
          nvidia.com/gpumem: 4096    # 限制使用 4096 MiB 显存
          nvidia.com/gpucores: 50    # 限制使用 50% GPU 算力
```

### 申请完整 GPU（独占模式，HAMi 兼容）

```yaml
resources:
  limits:
    nvidia.com/gpu: 1   # 不指定 gpumem/gpucores 则独占整卡
```

---

## 调度策略说明

| 策略 | `hami_node_scheduler_policy` | 适用场景 |
|------|------------------------------|----------|
| `binpack` | 优先填满单节点的 GPU 再用下一节点 | 节省成本，提高 GPU 利用率 |
| `spread` | 将 GPU 工作负载分散到不同节点 | 高可用，避免单节点热点 |

| 策略 | `hami_gpu_scheduler_policy` | 适用场景 |
|------|------------------------------|----------|
| `binpack` | 优先将多个 vGPU 调度到同一块物理 GPU | 减少显存碎片 |
| `spread` | 将 vGPU 分散到不同物理 GPU | 减少显存竞争 |

---

## 目录结构

```
kubeasz/
├── playbooks/
│   └── 08.gpu-node.yml                          # GPU 节点安装 playbook
├── roles/
│   ├── gpu-node/                                # GPU 节点驱动安装 role
│   │   ├── defaults/main.yml                    # 默认变量
│   │   ├── tasks/
│   │   │   ├── main.yml                         # 主入口（GPU 类型检测）
│   │   │   └── nvidia.yml                       # NVIDIA 驱动安装任务
│   │   └── files/
│   │       ├── .gitkeep
│   │       └── nvidia-container-toolkit/        # 离线 rpm/deb 包目录
│   └── cluster-addon/
│       ├── tasks/
│       │   ├── nvidia-device-plugin.yml         # NVIDIA device plugin 安装
│       │   ├── npu-device-plugin.yml            # NPU device plugin 安装
│       │   └── hami.yml                         # HAMi vGPU 安装
│       └── templates/gpu/
│           ├── nvidia-device-plugin.yaml.j2     # NVIDIA plugin DaemonSet 模板
│           ├── npu-device-plugin.yaml.j2        # NPU plugin DaemonSet 模板
│           └── hami-values.yaml.j2              # HAMi Helm values 模板
└── example/
    ├── config.yml                               # 新增 GPU/HAMi 配置段
    ├── hosts.multi-node                         # 新增 [gpu_node] 分组
    └── hosts.allinone                           # 新增 [gpu_node] 分组
```
