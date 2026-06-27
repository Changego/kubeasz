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

> ⚠️ `[gpu_node]` 中的节点必须同时出现在 `[kube_node]` 中，否则驱动安装完成后无法打标签。

### 第二步：配置 config.yml

#### 场景 A：NVIDIA GPU 独占（nvidia-device-plugin）

```yaml
# role:gpu-node
nvidia_driver_version: "550.54.15"
nvidia_driver_url: "https://example.com/NVIDIA-Linux-x86_64-550.54.15.run"
nvidia_toolkit_install_source: "online"

# role:cluster-addon
nvidia_device_plugin_install: "yes"
nvidia_device_plugin_image: "nvcr.io/nvidia/k8s-device-plugin:v0.18.1"
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
hami_ver: "v2.7.1"
hami_namespace: "kube-system"
hami_node_scheduler_policy: "binpack"   # 或 spread
hami_gpu_scheduler_policy: "spread"
```

#### 场景 C：离线部署

```yaml
nvidia_toolkit_install_source: "offline"
# 将 rpm/deb 包放到 roles/gpu-node/files/nvidia-container-toolkit/

hami_install_source: "offline"
# 将 hami-v2.7.1.tgz 放到 roles/cluster-addon/files/

hami_image_registry: "registry.example.com"   # 内网镜像仓库
```

#### 场景 D：华为昇腾 NPU

```yaml
# role:gpu-node
npu_driver_local_pkg: "Ascend-hdk-910-npu-driver_23.0.RC3_linux-aarch64.run"
npu_firmware_local_pkg: "Ascend-hdk-910-npu-firmware_7.1.0.4.511.run"
npu_docker_runtime_pkg: "Ascend-mindxdl-ascend-docker-runtime_5.0.RC3_linux-aarch64.run"

# role:cluster-addon
npu_device_plugin_install: "yes"
npu_device_plugin_image: "swr.cn-south-1.myhuaweicloud.com/ascend-deviceplugin/ascend-k8sdeviceplugin:v5.0.RC3"
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

## 为已有集群新增 GPU 节点

如果集群已经运行，需要为新加入的物理节点安装 GPU 支持：

```bash
# 第一步：将节点加入集群（如果还未加入）
ezctl add-node <cluster> <gpu-node-ip>

# 第二步：为该节点安装 GPU 驱动和 toolkit
ezctl add-gpu-node <cluster> <gpu-node-ip>

# 第三步：（可选）如果需要部署/更新 device-plugin 或 HAMi
ezctl setup <cluster> 07
```

`add-gpu-node` 命令会自动将节点加入 `[gpu_node]` 分组（如果还未加入），然后仅对该节点运行 `08.gpu-node.yml`。

---

## 验证安装

### 验证 GPU 驱动

```bash
# 在 GPU 节点上
nvidia-smi

# 验证 containerd GPU runtime
nvidia-container-cli info

# 验证节点标签
kubectl get node <gpu-node-name> --show-labels | grep -E 'nvidia|gpu|accelerator'
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

# 查看 HAMi 组件状态
kubectl -n kube-system get pods | grep -E 'hami|device-plugin'
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

## 变量速查表

### role: gpu-node（节点层变量）

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `nvidia_driver_version` | `"550.54.15"` | NVIDIA 驱动版本标识 |
| `nvidia_driver_url` | `""` | 驱动远程下载地址（优先） |
| `nvidia_driver_local_pkg` | `""` | 驱动本地 .run 文件名 |
| `nvidia_toolkit_install_source` | `"online"` | toolkit 安装源：`online` 或 `offline` |
| `nvidia_label_node` | `true` | 是否为 GPU 节点打 K8s 标签 |
| `nvidia_gpu_node_label_key` | `"gpu"` | 自定义标签 key |
| `nvidia_gpu_node_label_value` | `'"on"'` | 自定义标签 value |
| `GPU_FORCE_INSTALL` | `false` | 即使未检测到 GPU 也强制安装 |
| `npu_driver_local_pkg` | `""` | 昇腾 NPU 驱动本地包 |
| `npu_firmware_local_pkg` | `""` | 昇腾 NPU Firmware 本地包 |
| `npu_docker_runtime_pkg` | `""` | ascend-docker-runtime 本地包 |
| `npu_label_node` | `true` | 是否为 NPU 节点打 K8s 标签 |
| `npu_node_label_key` | `"npu"` | NPU 节点自定义标签 key |
| `npu_node_label_value` | `'"on"'` | NPU 节点自定义标签 value |

### role: cluster-addon（集群层 GPU 变量）

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `nvidia_device_plugin_install` | `"no"` | 是否部署 nvidia-device-plugin |
| `nvidia_device_plugin_image` | `nvcr.io/nvidia/k8s-device-plugin:v0.18.1` | device-plugin 镜像 |
| `nvidia_dp_disable_healthchecks` | `""` | 禁用健康检查（排错用） |
| `npu_device_plugin_install` | `"no"` | 是否部署 npu-device-plugin |
| `npu_device_plugin_image` | `...ascend-k8sdeviceplugin:v5.0.RC3` | NPU device-plugin 镜像 |
| `hami_install` | `"no"` | 是否部署 HAMi vGPU |
| `hami_ver` | `"v2.7.1"` | HAMi Helm chart 版本 |
| `hami_namespace` | `"kube-system"` | HAMi 部署命名空间 |
| `hami_install_source` | `"online"` | chart 安装源：`online` 或 `offline` |
| `hami_image_registry` | `""` | 私有镜像仓库前缀 |
| `hami_device_split_count` | `10` | 每块 GPU 拆分 vGPU 数量 |
| `hami_mig_strategy` | `"none"` | MIG 策略：`none`/`single`/`mixed` |
| `hami_node_scheduler_policy` | `"binpack"` | 节点调度策略 |
| `hami_gpu_scheduler_policy` | `"spread"` | GPU 内调度策略 |
| `hami_monitor_enabled` | `false` | 是否开启 Prometheus 监控 |

---

## 目录结构

```
kubeasz/
├── playbooks/
│   └── 08.gpu-node.yml                          # GPU 节点安装 playbook
├── roles/
│   ├── gpu-node/                                # GPU 节点驱动安装 role
│   │   ├── defaults/main.yml                    # 默认变量（含 NVIDIA 和 NPU）
│   │   ├── tasks/
│   │   │   ├── main.yml                         # 主入口（GPU/NPU 类型自动检测）
│   │   │   ├── nvidia.yml                       # NVIDIA 驱动安装任务
│   │   │   └── npu.yml                          # 华为昇腾 NPU 安装任务
│   │   └── files/
│   │       ├── .gitkeep                         # 目录说明文件
│   │       └── nvidia-container-toolkit/        # 离线 rpm/deb 包目录（offline 模式）
│   └── cluster-addon/
│       ├── tasks/
│       │   ├── main.yml                         # 入口（含 GPU/NPU 安装条件判断）
│       │   ├── nvidia-device-plugin.yml         # NVIDIA device plugin 安装
│       │   ├── npu-device-plugin.yml            # NPU device plugin 安装
│       │   └── hami.yml                         # HAMi vGPU 安装（含 helm 检查）
│       └── templates/gpu/
│           ├── nvidia-device-plugin.yaml.j2     # NVIDIA plugin DaemonSet 模板
│           ├── npu-device-plugin.yaml.j2        # NPU plugin DaemonSet 模板
│           └── hami-values.yaml.j2              # HAMi Helm values 模板
└── example/
    ├── config.yml                               # GPU/NPU/HAMi 配置段
    ├── hosts.multi-node                         # 含 [gpu_node] 分组示例
    └── hosts.allinone                           # 含 [gpu_node] 分组示例
```

---

## 常见问题

### Q: GPU 驱动安装后 nvidia-smi 找不到？

检查 nouveau 驱动是否已彻底禁用，重启节点后再验证：
```bash
lsmod | grep nouveau   # 应输出为空
nvidia-smi
```

### Q: HAMi Pod 启动失败，提示 webhook 问题？

确认集群 kube-apiserver 可以访问 HAMi webhook Service。HAMi webhook 会动态注入 GPU 资源到 Pod，需要网络策略允许 kube-apiserver 访问对应命名空间。

### Q: nvidia-device-plugin 和 HAMi 可以同时运行吗？

**不可以**。两者都会向 kubelet 注册 `nvidia.com/gpu` 资源，同时运行会导致资源重复注册。请在 `config.yml` 中确保只启用其中一个：
- 独占模式：`nvidia_device_plugin_install: "yes"` + `hami_install: "no"`
- 共享模式：`nvidia_device_plugin_install: "no"` + `hami_install: "yes"`
