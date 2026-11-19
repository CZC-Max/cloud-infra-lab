# cloud-infra-lab

> 云基础设施实操笔记 · Terraform IaC / Kubernetes 从零搭建 / 私有云环境排错
> 只收录**通用技术内容**，不含任何竞赛卷面、题号、分值或判分细节。

## 这是什么

一套边做边记的实验笔记，记录从一台空虚拟机到可用 Kubernetes 集群、再到 Terraform 全生命周期闭环的全过程。
写作目的不是"教程"，而是**留下可复现的过程证据**：每一步的现象、根因、修法、以及下次怎么不再踩。

### 内容亮点

| 主题 | 文件 | 能说明什么 |
|---|---|---|
| Terraform 全生命周期 | [`terraform/week01/`](./terraform/week01/) · [`W1 第 1 轮闭环`](./docs/2026-09-23-w1-round1-full-cycle.md) | init / plan / apply / destroy 完整链路、幂等性验证、销毁后三看 |
| Kubernetes 从零搭建 | [`k8s/README.md`](./k8s/README.md) · [`实验记录`](./docs/2026-09-25-k8s-cluster-from-scratch.md) | kubeadm init、CNI、节点加入、control-plane-endpoint、HA 扩容考量 |
| 排错方法论 | [**26 个坑复盘**](./docs/2026-09-26-pitfall-review-0922-0926.md) | 引号/不可见字符、锁文件、分层排错、变量防空、配置不落盘等通用规律 |
| 练熟曲线 | [`w1-rounds.md`](./docs/w1-rounds.md) | 同一任务从 40 分钟压到 2.1 分钟的过程与计时方法 |
| ESXi 单节点集群 + Web 终端 | [`搭建实录`](./docs/2026-11-19-esxi-ubuntu-k8s-single-node-web-terminal.md) | 受限网络下 kubeadm init 卡死、Calico ImagePullBackOff 的根因与永久修法；sandbox 镜像、镜像源超时、ttyd |
| 环境准备 | [`Git 安装与首次推送`](./docs/2026-09-22-git-install-and-first-push.md) · [`实验记录模板`](./docs/lab-record-template.md) | Windows 国内网络环境下的工具链搭建、记录规范 |

## 目录结构

```
.
├── terraform/week01/   # VPC + 交换机 + 安全组 + ECS + 数据盘 + 双站点服务
├── k8s/                # Kubernetes 集群搭建与网络配置
├── scripts/            # 加固 / 巡检 / 故障复现脚本（占位，持续补充）
├── docs/               # 实验记录：现象 → 思路 → 步骤 → 经验
└── README.md
```

## 快速开始

```bash
git clone <this-repo>
cd cloud-infra-lab/terraform/week01

# 凭证走环境变量，绝不写进代码
export ALICLOUD_ACCESS_KEY="<your-ak>"
export ALICLOUD_SECRET_KEY="<your-sk>"
export ALICLOUD_REGION="cn-hangzhou"

cp terraform.tfvars.example terraform.tfvars   # tfvars 已被 gitignore
terraform init
terraform plan
terraform apply
terraform destroy                              # 用完立刻销毁
```

## 销毁后三看（血泪教训）

`terraform destroy` 报 `Resources: 0 destroyed` **不等于真删干净了**。每次销毁后核对：

1. **销毁前** `terraform plan -destroy` —— 显示数量必须等于建的数量，不是 0
2. **销毁后** `terraform state list` —— 必须返回空
3. **控制台复核** —— 实例 / 安全组真的消失

## 安全声明

- 本仓库**不含任何真实 AK/SK、私钥**；`.tfvars`、`.tfstate` 已被 `.gitignore` 排除
- 文中 IP 均为本地实验私有网段，非公网资产；所有演练在自有或明确授权的环境中进行
- 涉及安全的脚本仅用于靶场 / 实验账号 / 授权测试

## 许可

[MIT](./LICENSE)
