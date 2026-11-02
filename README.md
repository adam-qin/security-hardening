# CVE-2026-98432 Ubuntu 加固 Playbook

本目录提供一套面向 Ubuntu 16.04～24.04 的 CVE-2026-98432 临时缓解方案，基于 Ansible Playbook 2.9.27 设计。

> **重要：本 Playbook 是临时缓解措施，不替代 Linux 内核或发行版供应商的正式安全更新。**
>
> 当前生产配置采用 **B 方案**：使用现有 iptables 的 `filter/INPUT` 路径，仅保护送达宿主机本地服务的 IPv4 TCP 流量；不修改 Docker、Kubernetes、CNI 及其他转发链。

## 1. 目录结构

```text
playbook/
├── cve-2026-98432-mitigation.yml       # 加固/回滚入口
├── cve-2026-98432-check.yml            # 只读检查入口
└── roles/
    ├── cve-2026-98432-mitigation/
    │   ├── defaults/main.yml           # 加固默认参数
    │   ├── tasks/main.yml              # enforce/rollback 分流
    │   ├── tasks/apply.yml             # 加固动作
    │   ├── tasks/rollback.yml          # 回滚动作
    │   ├── handlers/main.yml           # systemd unit 重启处理
    │   └── templates/
    │       ├── cve-2026-98432-iptables.sh.j2
    │       ├── cve-2026-98432-mitigation.service.j2
    │       └── cve-2026-98432.nft.j2   # 历史模板，当前生产路径不使用
    └── cve-2026-98432-check/
        ├── defaults/main.yml           # 检查参数
        ├── tasks/main.yml              # 状态和业务健康检查
        ├── tasks/pod_network.yml      # 可选 Pod 网络检查
        └── files/                      # 检查脚本
```

## 2. 功能和动作总览

### 2.1 加固入口

```text
playbook/cve-2026-98432-mitigation.yml
```

通过 `cve_98432_action` 控制动作：

| 动作 | 变量 | 说明 |
|---|---|---|
| 加固 | `cve_98432_action=enforce` | 执行备份、sysctl 设置、iptables 规则和 systemd 持久化 |
| 回滚 | `cve_98432_action=rollback` | 校验可信备份后，仅移除本方案对象并恢复本方案修改 |
| 兼容回滚 | `cve_98432_rollback=true` | 兼容旧调用方式，优先执行回滚 |

`tasks/main.yml` 先校验动作，只允许 `enforce` 或 `rollback`，然后分流到 `apply.yml` 或 `rollback.yml`。

### 2.2 当前 B 方案的网络范围

规则路径：

```text
iptables
└── filter
    └── INPUT
        └── CVE-2026-98432
```

专用链中的核心规则：

```sh
-p tcp -m u32 --u32 '0>>24&0x0f=6:15' -j DROP
```

含义：丢弃 IPv4 header 存在 options 的 TCP 报文，即 IPv4 IHL 大于 5；不匹配 TCP header options（例如 MSS、窗口扩大等）。

覆盖：

- 送达宿主机本地 `INPUT` 的 IPv4 TCP 流量；
- 宿主机上监听的本地 TCP 服务；
- 外部客户端访问宿主机本地服务的路径。

不覆盖：

- Docker/Kubernetes 的 `FORWARD` 流量；
- 发往 Pod 或容器的转发流量；
- CNI、`DOCKER-*`、`KUBE-*` 链；
- 宿主机作为路由器转发的流量。

Playbook 不执行以下操作：

- 不清空 iptables 表；
- 不修改默认策略；
- 不修改 `DOCKER-*`、`KUBE-*`、`CNI-*` 链；
- 不使用整表 `iptables-restore` 回滚。

### 2.3 MPTCP 和 TCP request migration

如果内核提供以下 sysctl，Playbook 会检测并设置为 0：

```text
net.mptcp.enabled=0
net.ipv4.tcp_migrate_req=0
```

同时写入：

```text
/etc/sysctl.d/99-cve-2026-98432.conf
```

注意：关闭 MPTCP 只影响后续创建的 socket；已有 MPTCP listener 不会自动消失。如需清理已有 listener，必须显式配置服务重启列表并经过业务确认。

### 2.4 备份和持久化

首次加固前会创建：

```text
/var/lib/cve-2026-98432/backup/
├── manifest
├── manifest.pending
├── runtime-state
└── iptables-save
```

备份内容包括：

- 原 sysctl 文件是否存在及其内容；
- `/etc/modules-load.d/cve-2026-98432.conf`；
- systemd unit 文件；
- iptables 脚本；
- 加固前 MPTCP、TCP request migration 值；
- systemd unit 的 enabled/active 状态；
- `iptables-save` 快照。

`manifest` 使用 `complete=1` 作为完成标志。发现未完成备份时，加固和回滚都会停止，避免使用不可信快照。

systemd unit：

```text
/etc/systemd/system/cve-2026-98432-mitigation.service
```

主机重启后由 systemd 调用：

```sh
/usr/local/sbin/cve-2026-98432-iptables.sh apply
```

## 3. 前置条件

### 3.1 控制端

建议使用：

```text
ansible-playbook 2.9.27
```

Ansible 2.9 已停止上游维护，Ubuntu 24.04 常见 Python 版本需要在实际环境中验证。正式执行前必须完成 syntax-check。

### 3.2 受管主机

需要满足：

- Ubuntu 16.04～24.04，或具备等效 iptables 环境的 Linux 主机；
- root 或具备 sudo 权限的 Ansible 用户；
- 已安装 `iptables`、`iptables-save`、`modprobe`、`systemctl`；
- `filter/INPUT` 链存在；
- 内核能够加载：
  ```text
  xt_u32
  xt_comment
  ```
- 生产机器使用 iptables，firewalld 未启用；
- 建议提前准备业务健康检查和远端探针。

### 3.3 Inventory 示例

```ini
[canary]
app-01 ansible_host=192.0.2.11

[production]
app-01 ansible_host=192.0.2.11
app-02 ansible_host=192.0.2.12

[all:vars]
ansible_user=ansible
ansible_become=true
```

## 4. 第一步：语法检查

语法检查不连接业务主机，不执行加固动作：

```bash
ansible-playbook \
  --syntax-check \
  -i inventory.ini \
  playbook/cve-2026-98432-mitigation.yml
```

```bash
ansible-playbook \
  --syntax-check \
  -i inventory.ini \
  playbook/cve-2026-98432-check.yml
```

建议同时检查 YAML/Jinja 文件没有 Tab 和未闭合模板。

## 5. 第二步：执行只读检查

### 5.1 普通检查

```bash
ansible-playbook \
  -i inventory.ini \
  --become \
  playbook/cve-2026-98432-check.yml
```

检查内容包括：

- 内核版本；
- `net.mptcp.enabled`；
- `net.ipv4.tcp_migrate_req`；
- systemd unit 状态；
- `filter/INPUT` 专用 hook 和 `u32` 规则；
- sysctl 配置文件；
- MPTCP listener 数量；
- 本地普通 TCP 连接；
- 本地带 IPv4 options 的 TCP 连接；
- 可选 Kubernetes、DNS、外部 URL 和远端 TCP 探针。

### 5.2 严格检查

```bash
ansible-playbook \
  -i inventory.ini \
  --become \
  playbook/cve-2026-98432-check.yml \
  -e cve_98432_check_strict=true
```

严格模式要求：

- 规则必须存在；
- 普通 TCP 本地连接成功；
- IPv4 options 探针连接失败；
- Kubernetes `/readyz` 正常时必须返回 `ok`；
- 如果启用远端探针，远端 TCP 必须可达。

### 5.3 远端业务探针

本地探针只能验证宿主机内部路径，不能证明外部业务访问正常。建议配置独立探针主机：

```bash
ansible-playbook \
  -i inventory.ini \
  --become \
  playbook/cve-2026-98432-check.yml \
  -e cve_98432_check_strict=true \
  -e cve_98432_remote_probe_host=192.0.2.100 \
  -e cve_98432_remote_probe_port=443 \
  -e cve_98432_require_remote_probe=true
```

## 6. 第三步：单节点灰度执行

生产首次执行必须使用单节点：

```bash
ansible-playbook \
  -i inventory.ini \
  --become \
  --limit app-01 \
  playbook/cve-2026-98432-mitigation.yml \
  -e cve_98432_action=enforce \
  -e cve_98432_serial=1
```

执行顺序：

1. 校验动作和生产后端；
2. 检测旧版 raw/nftables 对象和孤立 hook；
3. 检查 iptables 命令、模块和 `filter/INPUT`；
4. 创建可信备份；
5. 检测可用 sysctl；
6. 持久化并设置 MPTCP/TCP migration sysctl；
7. 持久化 `xt_u32` 模块加载；
8. 安装专用 iptables 脚本；
9. 安装并启动 systemd unit；
10. 由脚本创建专用链和 INPUT 跳转；
11. 验证规则、unit 和 sysctl；
12. 默认不重启业务服务。

执行后立即检查：

```bash
ansible-playbook \
  -i inventory.ini \
  --become \
  --limit app-01 \
  playbook/cve-2026-98432-check.yml \
  -e cve_98432_check_strict=true \
  -e cve_98432_serial=1
```

灰度观察项：

- SSH 和管理面连接；
- 业务端口外部访问；
- 负载均衡健康状态；
- Docker/Kubernetes Pod 网络；
- iptables 计数器；
- MPTCP listener 数量；
- 应用错误率、连接失败率和延迟。

## 7. 第四步：批量执行

单节点验证通过后，再按小批次扩大范围：

```bash
ansible-playbook \
  -i inventory.ini \
  --become \
  playbook/cve-2026-98432-mitigation.yml \
  -e cve_98432_action=enforce \
  -e cve_98432_serial=20%
```

推荐顺序：

```text
单节点 → 同可用区小批次 → 单集群小批次 → 全量
```

不要在未完成业务验证前直接对全部生产主机执行。

## 8. 可选：显式重启业务服务

默认不重启业务服务：

```yaml
cve_98432_restart_services: false
cve_98432_services_to_restart: []
```

如果确认已有 MPTCP listener 必须清理，应在变量文件中明确指定：

```yaml
cve_98432_restart_services: true
cve_98432_services_to_restart:
  - nginx
  - example-api.service
```

然后仍然建议先使用单节点执行，并在业务窗口内观察。不要通过命令行临时传入未经审核的服务列表。

## 9. 回滚操作

### 9.1 回滚前检查

回滚前先确认：

- 当前主机使用过本 Playbook；
- `/var/lib/cve-2026-98432/backup/manifest` 存在；
- manifest 包含 `complete=1`；
- `runtime-state` 存在；
- 没有同时运行其他规则变更任务；
- 已准备业务影响确认和回滚窗口。

### 9.2 执行回滚

```bash
ansible-playbook \
  -i inventory.ini \
  --become \
  --limit app-01 \
  playbook/cve-2026-98432-mitigation.yml \
  -e cve_98432_action=rollback \
  -e cve_98432_serial=1
```

兼容旧开关：

```bash
ansible-playbook \
  -i inventory.ini \
  --become \
  --limit app-01 \
  playbook/cve-2026-98432-mitigation.yml \
  -e cve_98432_rollback=true \
  -e cve_98432_serial=1
```

回滚动作：

1. 校验可信 manifest；
2. 停止并禁用本方案 systemd unit；
3. 仅删除带有本方案 comment 的 `filter/INPUT` 跳转；
4. 清空并删除本方案专用链；
5. 按 manifest 恢复原文件或删除原本不存在的文件；
6. reload systemd；
7. 恢复原始 sysctl 运行值；
8. 验证本方案对象已经移除。

回滚明确不会执行：

```bash
iptables-restore < /var/lib/cve-2026-98432/backup/iptables-save
```

原因是整表恢复可能覆盖 Docker/Kubernetes 在运行期间动态产生的规则。

### 9.3 回滚后验证

```bash
ansible-playbook \
  -i inventory.ini \
  --become \
  --limit app-01 \
  playbook/cve-2026-98432-check.yml \
  -e cve_98432_serial=1
```

同时确认：

```bash
iptables -t filter -S INPUT
iptables -t filter -S DOCKER 2>/dev/null || true
iptables -t filter -S KUBE-SERVICES 2>/dev/null || true
systemctl is-enabled cve-2026-98432-mitigation.service 2>/dev/null || true
```

重点确认 Docker/Kubernetes 既有链仍存在，且没有被整表替换。

## 10. 失败和中止策略

以下情况应立即停止批量执行并人工检查：

- Ansible syntax-check 失败；
- `xt_u32` 或 `xt_comment` 无法加载；
- 发现旧 raw/nftables 方案对象；
- 发现孤立的本方案 INPUT hook；
- 备份 manifest 不完整；
- 规则安装后 `--check` 失败；
- SSH、业务端口或健康检查异常；
- 发现 Docker/Kubernetes 链被改变；
- 回滚前没有可信备份。

不要使用以下高风险操作处理失败：

```bash
iptables -F
iptables -t nat -F
iptables-restore < 全量快照
systemctl restart docker
systemctl restart kubelet
```

除非已经完成影响评估并由业务负责人批准。

## 11. 当前已知限制

1. 这是临时缓解措施，正式修复仍应优先升级到发行版提供的安全内核。
2. B 方案只覆盖宿主机 `filter/INPUT`，不覆盖 Pod/容器转发流量。
3. 关闭 MPTCP 不会自动关闭已有 listener。
4. `cve_98432_fixed_patch` 当前用于记录上游修复版本参考，发行版 backport 情况必须以 Ubuntu 安全公告和实际内核包为准。
5. 当前仓库版本尚未在真实 Ubuntu 16.04～24.04 矩阵中完成实机验证。
6. 当前控制端需安装并使用 Ansible 2.9.27 完成 syntax-check 和灰度验证。

## 12. 版本与仓库

当前版本：

```text
v1.0
```

远端仓库：

```text
https://github.com/adam-qin/security-hardening
```

本 README 的后续修改应继续遵循：

- 不提交 `.workbuddy/`；
- 不提交临时压缩包和密钥；
- 生产变更前先单节点灰度；
- commit、tag、push 前核对入库清单。
