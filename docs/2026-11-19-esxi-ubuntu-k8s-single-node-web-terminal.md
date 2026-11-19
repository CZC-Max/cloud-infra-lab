# ESXi 上从零搭建 Ubuntu + K8s 单节点集群与浏览器 Web 终端

> 环境：VMware ESXi 7.0 U1 · Ubuntu 22.04.5 LTS · Kubernetes v1.30.14 · containerd 2.2.1 · Calico v3.26.4 · ttyd
> 目标：在 ESXi 上开一台虚拟机，装好单节点 K8s，最终做到**打开浏览器就能敲 `kubectl`**，用于离线/内网场景下的集群练习。
> 全程只记录通用技术内容，不含任何真实凭证。文中 IP 均为实验私有网段。

---

## 0. 为什么写这篇

单节点 K8s 本身不难，难的是**内网/受限网络下的镜像拉取**。这次搭建真正卡住的两个点，都不是 K8s 配置写错，而是：

1. `kubeadm init` 卡死 4 分钟 —— containerd 自己的 sandbox 镜像指向境外仓库
2. Calico 全部 `ImagePullBackOff` —— CNI 镜像在 Docker Hub，内网拉不到

两处都在网上有大量"照着做还是失败"的帖子，因为多数教程默认你能访问 `registry.k8s.io` 和 `docker.io`。本文把这两个坑的**完整诊断链 + 永久修法**写清楚。

---

## 1. 虚拟机规划

| 项 | 值 | 说明 |
|---|---|---|
| vCPU | 4 | 单节点所有组件挤一台，2 核在 init 时明显拖慢 |
| 内存 | 16 GB | 不勾选"预留全部客户机内存"，否则独占 |
| 硬盘 | 80 GB 精简置备 | 精简置备用多少占多少；Goat 之类多场景镜像会吃空间 |
| 固件 | **BIOS** | 若选 UEFI 需关 Secure Boot，否则 Linux 可能报 `Security Violation` 拒载 GRUB |
| 客户机 OS | Ubuntu 64 位 | 与老师/主流环境一致，排障资料最多 |
| 网络 | 直连端口组 | 需能出网（拉 apt 包与镜像） |
| 光驱 | 挂 Ubuntu Server ISO，勾"启动时连接" | 装完系统后务必取消，否则重启又进安装菜单 |

### 固件一个判断技巧

不确定该选 BIOS 还是 UEFI 时，**去看同环境里已有的 Linux 虚机**：右键 → 编辑设置 → 虚拟机选项 → 引导选项 → 固件。照抄现有配置，比自己猜省事。

---

## 2. 系统安装要点

Ubuntu Server 22.04.5 安装器几处容易踩：

| 步骤 | 选什么 | 坑 |
|---|---|---|
| 语言 / 键盘 | English (UK) | 中文 locale 在纯终端下可能编码异常 |
| Installer update | Skip for now | 更新安装器需联网，受限网络会卡死 |
| 镜像源 | 用国内镜像（清华 TUNA 等） | 安装器会自动测试，显示 `This mirror location passed tests` 再继续 |
| 磁盘 | Use an entire disk，LVM 默认 | 默认只分一半给 `/`，剩一半空闲可后续 `lvextend` |
| **SSH Setup** | 🔴 **勾 Install OpenSSH server** | 漏勾装完只能蹲控制台，装完补装多一轮 |
| Ubuntu Pro | Skip for now | 付费订阅，练习环境用不上 |

装完重启前，**回 ESXi 把 CD/DVD 的"启动时连接"取消**，否则重启又从 ISO 引导进安装菜单。

---

## 3. 基础环境（脚本化，不手敲长命令）

内网环境下的经验：**长命令 + 嵌套引号不要用键盘敲**（输入法会把 `=` 变 `:`，把 ASCII 引号变全角弯引号，把 `|` 变 `~`，报错极难排查）。统一做法：

- 本地写脚本 → SCP 上去 → `sudo bash` 执行
- 或通过 paramiko 把脚本 pump 上去执行

### 3.1 装 containerd 与 K8s 组件

```bash
# 内核参数
modprobe br_netfilter
cat > /etc/sysctl.d/99-k8s.conf <<'EOF'
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system

# containerd（apt 装的是 2.x）
apt-get update
apt-get install -y containerd apt-transport-https ca-certificates curl gnupg
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd && systemctl enable containerd

# kubeadm / kubelet / kubectl 指定版本
mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' \
  > /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet=1.30.14-1.1 kubeadm=1.30.14-1.1 kubectl=1.30.14-1.1
apt-mark hold kubelet kubeadm kubectl
```

### 3.2 init 配置

```bash
cat > /root/kubeadm-init.yaml <<'EOF'
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    fail-swap-on: "false"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.30.14
imageRepository: registry.aliyuncs.com/google_containers
networking:
  podSubnet: 192.168.0.0/16
  serviceSubnet: 10.96.0.0/12
EOF
```

`fail-swap-on: "false"` 是必要项：Ubuntu 默认有 swap，kubeadm preflight 会报 swap 错误。正解是让 kubelet 忽略，不是去注释 fstab 里的 swap。

---

## 4. 🔴 坑一：kubeadm init 卡在 api-check 4 分钟

### 现象

```
[wait-control-plane] Waiting for the kubelet to boot up the control plane...
[kubelet-check] The kubelet is healthy after 1.002s
[api-check] Waiting for a healthy API server. This can take up to 4m0s
[api-check] The API server is not healthy after 4m0.001s
error execution phase wait-control-plane: could not initialize a Kubernetes cluster
```

kubelet 健康，但 apiserver 永远起不来。

### 诊断

```bash
journalctl -u kubelet --no-pager -n 40 | grep -iE 'error|fail|sandbox'
```

关键报错：

```
RunPodSandbox from runtime service failed:
failed to get sandbox image "registry.k8s.io/pause:3.10.1":
failed to pull image ... Head "https://europe-west3-docker.pkg.dev/..." : i/o timeout
```

### 根因

**只改了 kubeadm 的 `imageRepository`，漏了 containerd 自己的 `sandbox_image`。**

- kubeadm init 拉控制面镜像时，走的是 `init.yaml` 里的 `imageRepository`（已改阿里云，能拉）
- 但 kubelet 创建 Pod 沙箱时，用的是 **containerd 配置里的 `sandbox_image`**，它独立地指向 `registry.k8s.io/pause:3.10.1`，在受限网络里超时
- 结果：etcd / apiserver / controller-manager 的静态 Pod 全卡在"创建沙箱"这一步，6443 永远不监听

### 永久修法

装完 containerd 后，**立刻**把 sandbox 镜像也改掉，不要等到 init 失败再回头：

```bash
cp /etc/containerd/config.toml /etc/containerd/config.toml.bak
sed -i 's#registry.k8s.io/pause:3.10.1#registry.aliyuncs.com/google_containers/pause:3.9#g' \
  /etc/containerd/config.toml
systemctl restart containerd

# 预拉一次确认可达
ctr -n k8s.io images pull registry.aliyuncs.com/google_containers/pause:3.9
```

若已 init 失败，先 `kubeadm reset -f` 清理残留再重新 init。**永久规则：改 kubeadm 镜像源时，必须同步改 containerd 的 sandbox_image**——这是两个独立配置，改一个不算改完。

---

## 5. 🔴 坑二：Calico 全部 ImagePullBackOff

### 现象

```bash
kubectl get pods -n kube-system -o wide
```

```
calico-node-xxx       0/1  Init:ImagePullBackOff
calico-kube-controllers-xxx  0/1  Pending
coredns-xxx           0/1  Pending
```

节点 `NotReady`，kubelet 日志刷：

```
Container runtime network not ready ... Network plugin returns error: cni plugin not initialized
```

describe 里能看到：

```
Failed to pull image "docker.io/calico/cni:v3.26.4":
Head "https://registry-1.docker.io/v2/calico/cni/manifests/v3.26.4": i/o timeout
```

### 根因

Calico 清单里的镜像默认来自 `docker.io`。受限网络拉不到 → install-cni init 容器起不来 → CNI 没配置 → 节点永远 NotReady。

### 镜像源选择：别被短超时骗了

测试多个镜像源时，**超时设置太短会把"能拉但慢"误判成"拉不了"**。本次实测：

| 镜像源 | 12 秒超时的表现 | 180 秒超时 |
|---|---|---|
| `registry.aliyuncs.com/google_containers` | `repository does not exist` | ✗ 确实没有该仓库 |
| `docker.m.daocloud.io` | `elapsed: 11.9s ...`（像失败） | ✅ **三件套全部拉下** |
| `docker.1ms.run` / `docker.xuanyuan.me` / `hub.rat.dev` | DNS / cloud 后端失败 | ✗ 不可用 |

🔴 **永久规则：测镜像源至少给 120~180 秒**。看到 `elapsed` 在走、有 `total` 增长，说明在下，只是没下完。

### 永久修法

从可达的镜像源预拉三件套，再 tag 成清单期望的名字，最后删 Pod 让它用本地镜像重建：

```bash
for img in cni node kube-controllers; do
  ctr -n k8s.io images pull "docker.m.daocloud.io/calico/$img:v3.26.4"
  ctr -n k8s.io images tag  "docker.m.daocloud.io/calico/$img:v3.26.4" \
                            "docker.io/calico/$img:v3.26.4"
done

kubectl -n kube-system delete pods -l k8s-app=calico-node
kubectl -n kube-system delete pods -l k8s-app=calico-kube-controllers
kubectl wait --for=condition=Ready pod -l k8s-app=calico-node -n kube-system --timeout=180s
```

镜像 tag 对齐后，kubelet 发现本地已有同名镜像，不会再外网拉取。

---

## 6. 浏览器 Web 终端（ttyd）

```bash
apt-get install -y wget
wget -O /usr/local/bin/ttyd \
  https://github.com/tsl0922/ttyd/releases/download/1.7.7/ttyd.x86_64
chmod +x /usr/local/bin/ttyd

cat > /etc/systemd/system/ttyd.service <<'EOF'
[Unit]
Description=ttyd web terminal
After=network.target
[Service]
ExecStart=/usr/local/bin/ttyd -p 7681 -W bash
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now ttyd
```

浏览器打开 `http://<节点IP>:7681` 即是完整 bash。🔴 注意：ttyd 默认无认证，仅限**内网/受控环境**使用；不要暴露到公网。

---

## 7. 验收清单

```bash
kubectl get nodes -o wide                      # Ready
kubectl get pods -n kube-system -o wide        # 全部 1/1 Running
ss -lntp | grep 6443                           # apiserver LISTEN
ss -lntp | grep 7681                           # ttyd LISTEN
kubectl auth can-i get secrets -A --as=system:serviceaccount:default:default   # 期望 no
kubectl get networkpolicy -A                   # 空 = 无网络策略
```

单节点必须摘掉 control-plane taint，否则普通 Pod 全部 `Pending`：

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

## 8. 复盘：这次真正学到的

1. **镜像源要改全**：kubeadm 的 `imageRepository` 和 containerd 的 `sandbox_image` 是两个独立配置，只改一个必踩坑一。
2. **超时别设太短**：12 秒把 daocloud 误判成失败，180 秒证明它完全可用。诊断网络问题时，"慢"和"不通"是两种病。
3. **CNI 没起 = 节点永远 NotReady**：看到 `cni plugin not initialized` 就别再查 kubelet 本身了，直接查 CNI 镜像和 Pod。
4. **长命令脚本化**：输入法会把特殊字符静默改掉（全角引号、`=`→`:`、`|`→`~`），键盘敲长命令是最大的不确定性来源。写脚本 → pump → 执行。
5. **固件照抄现有环境**：同环境已有的 Linux 虚机怎么配就怎么配，能少走弯路。

---

## 附：配合使用的只读体检命令

环境搭好后，可用下列命令确认集群的安全基线（只读，不改配置）：

```bash
kubectl get clusterrolebindings -o custom-columns='NAME:.metadata.name,ROLE:.roleRef.name,SUBJECTS:.subjects[*].name' | grep -i cluster-admin
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,SA:.spec.serviceAccountName,AUTOMOUNT:.spec.automountServiceAccountToken'
kubectl get ns --show-labels
```

`--show-labels` 是 jsonpath 的免引号替代：在中文输入法环境下，`{" "}` 这类 jsonpath 里的双引号容易变成全角导致 `U+007B` 报错，`--show-labels` 输出等价且不含引号。
