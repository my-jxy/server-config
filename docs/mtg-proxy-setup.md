# MTProxy (MTG) 搭建笔记 - 阿里云轻量服务器

## 概述

在阿里云轻量应用服务器上搭建 MTProto 代理，用于 Telegram 翻墙。

- **服务器**: 阿里云轻量应用服务器 (SWAS), cn-beijing
- **系统**: Ubuntu 24.04
- **内网IP**: 172.17.41.204
- **公网IP**: 39.106.141.216 (阿里云超管 1:1 NAT)
- **代理工具**: [nineseconds/mtg](https://github.com/9seconds/mtg)
- **科学上网**: Cloudflare WARP (用于服务器访问 Telegram 服务器)
- **最终端口**: 8443 (443 被墙)

---

## 架构说明

```
用户 (中国) → GFW → 阿里云轻量服务器:8443 → mtg → WARP → Telegram DC
```

阿里云轻量服务器的网络架构比较特殊：
- 公网 IP `39.106.141.216` 并非直接配在网卡上
- eth0 实际 IP 是 `172.17.41.204`（内网地址）
- 阿里云超管做了一层 **1:1 NAT**：`39.106.141.216 ↔ 172.17.41.204`
- 所有出入站流量都经过这层 NAT

---

## 搭建步骤

### 1. 安装 Docker

```bash
curl -fsSL https://get.docker.com | bash
```

### 2. 安装并连接 WARP

```bash
# 安装 Cloudflare WARP
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | gpg --dearmor -o /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/cloudflare-warp.list
apt-get update && apt-get install -y cloudflare-warp

# 注册并连接
warp-cli --accept-tos register
warp-cli --accept-tos connect
warp-cli --accept-tos status  # 确认 Connected, Network: healthy
```

> WARP 连接后会自动创建 nftables 表 `inet cloudflare-warp`，默认 INPUT 策略为 DROP。

### 3. 生成密钥并启动 mtg

```bash
# 生成密钥（TLS 伪装 cloudflare.com）
SECRET=$(docker run --rm nineseconds/mtg:latest generate-secret --hex cloudflare.com)
echo "Secret: $SECRET"

# 启动容器（host 网络模式！）
docker run -d --name mtg-proxy --restart unless-stopped --network host \
  nineseconds/mtg:latest simple-run 0.0.0.0:8443 "$SECRET"
```

> **为什么用 host 模式？** bridge 模式下 mtg 走 Docker 的 MASQUERADE，WARP 策略路由对来自 Docker bridge 的包不生效，导致 mtg 无法出访 Telegram。host 模式直接使用宿主机网络栈，WARP 的路由规则正常生效。

### 4. 配置防火墙

#### 阿里云轻量服务器防火墙

在控制台 → 轻量应用服务器 → 防火墙 添加规则：
- 协议: TCP
- 端口: 8443
- 来源: 0.0.0.0/0

或用 CLI：

```bash
aliyun swas-open create-firewall-rule \
  --instance-id <实例ID> \
  --biz-region-id cn-beijing \
  --rule-protocol TCP \
  --port 8443
```

> 实例ID 可以通过 `aliyun swas-open list-instances --biz-region-id cn-beijing` 查询。

#### WARP nftables 放行

WARP 的 nftables 表 `inet cloudflare-warp` 的 input chain 默认 `policy drop`，需要放行 8443：

```bash
nft add rule inet cloudflare-warp input tcp dport 8443 accept
```

> ⚠️ WARP 重连后会重建这个表，规则会丢失，需要重新添加！下面有持久化方案。

### 5. 策略路由

WARP 连接后会自动添加一条策略路由规则（优先级 479），将所有出站流量劫持到 WARP 的虚拟网卡：

```
479: not from all fwmark 0x100cf lookup 65743
```

这导致 mtg 回复客户端的 SYN-ACK 包走 WARP 接口出去，客户端收不到。

解决方案：添加更高优先级的规则，让回复包走 eth0：

```bash
ip rule add from all to <用户IP> lookup main priority 478
```

> 如果不知道用户 IP，可以用 iptables CONNMARK 方案做通用匹配（见下文避坑）。

验证路由：

```bash
ip route get <用户IP>
# 期望结果: via 172.17.63.253 dev eth0 src 172.17.41.204
```

### 6. 持久化

创建自启脚本：

```bash
cat > /root/hermes-proxy-setup.sh << 'EOF'
#!/bin/bash
conntrack -D -p tcp --dport 8443 2>/dev/null
nft add rule inet cloudflare-warp input tcp dport 8443 accept 2>/dev/null
ip rule add from all to <用户IP> lookup main priority 478 2>/dev/null
EOF
chmod +x /root/hermes-proxy-setup.sh
```

创建 systemd 服务：

```bash
cat > /etc/systemd/system/hermes-proxy-setup.service << 'EOF'
[Unit]
Description=MTProxy setup - routing and firewall rules
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/root/hermes-proxy-setup.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable hermes-proxy-setup.service
```

---

## 避坑指南 🕳️

### 1. 443 端口被墙

阿里云轻量服务器默认开放 22/80/443/3389，但 **443 端口在大规模 GFW 扫描下很快被封**。SYN 包能到服务器但不回 SYN-ACK，就是被墙的典型表现。

**解决**：换用非标端口，如 8443。

### 2. WARP 重置 nftables 规则

WARP 每次重连（服务重启、网络波动等）都会重建 `inet cloudflare-warp` 表，之前手动添加的放行规则全部丢失。

**解决**：使用 systemd oneshot 或 cron 定期检查并重新添加规则。

### 3. Docker bridge 模式下 mtg 走不通

bridge 模式下 mtg 的流量经过 MASQUERADE 后才出宿主机，WARP 的策略路由（ip rule 479）对这种包的匹配不稳定。表现为：客户端能连上 mtg，但 mtg 连不上 Telegram 服务器，日志报 `cannot dial to telegram: dial tcp4 149.154.171.5:443: i/o timeout`。

**解决**：使用 `--network host` 模式。

### 4. conntrack 表卡死

mtg 的旧连接如果在 SYN_RECV 状态卡住（客户端发了 SYN，服务器回了 SYN-ACK，但客户端没回 ACK），conntrack 表会积累无用条目。新连接可能被这些残留条目干扰（取决于 conntrack 状态机配置）。

**解决**：定期刷新：
```bash
conntrack -D -p tcp --dport 8443
```

### 5. 阿里云 CLI 的参数坑

SWAS-OPEN API 的参数名跟 ECS 不同：
- 实例ID 用 `--instance-id`（不是 `--InstanceId`）
- 地域用 `--biz-region-id`（不是 `--region` 或 `--RegionId`）
- 实例ID不是 hostname 里的 `i-2ze9c76r...` 格式，而是 32 位 hex 字符串

```bash
# 正确用法
aliyun swas-open list-firewall-rules \
  --instance-id 46e32156783d4eeda3a97571409bd5b5 \
  --biz-region-id cn-beijing

# 查实例ID
aliyun swas-open list-instances --biz-region-id cn-beijing
```

### 6. 轻量服务器没有 ECS 安全组

阿里云轻量应用服务器（SWAS）的防火墙在控制台叫"防火墙"，不是 ECS 的"安全组"。API 也完全不同（swas-open 而非 ecs）。

### 7. TCP 握手完成但 Telegram 不可用

如果 `tcpdump` 看到完整的 TCP 三握（SYN → SYN-ACK → ACK），数据也在双向传输，但 Telegram 不可用，通常是：
- mtg 出访 Telegram 失败（检查 WARP 状态）
- 密钥不匹配（检查 secret）

---

## 最终配置

### 代理链接

```
tg://proxy?server=39.106.141.216&port=8443&secret=ee...
```

### 容器

```bash
docker run -d --name mtg-proxy --restart unless-stopped --network host \
  nineseconds/mtg:latest simple-run 0.0.0.0:8443 "ee..."
```

### nftables

```bash
nft add rule inet cloudflare-warp input tcp dport 8443 accept
```

### 策略路由

```bash
ip rule add from all to <用户IP> lookup main priority 478
```

### 启动脚本

```bash
# /root/hermes-proxy-setup.sh（由 hermes-proxy-setup.service 触发）
```
