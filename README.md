# Server Configuration

ECS 服务器配置记录，方便随时查阅。

---

## Cloudflare WARP 安装配置指南（阿里云 ECS）

### 1. 安装 WARP 客户端

```bash
# 添加 Cloudflare WARP GPG 密钥
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | sudo gpg --dearmor --yes -o /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg

# 添加软件源（以 Debian/Ubuntu 为例）
echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflare-client.list

# 更新并安装
sudo apt update
sudo apt install cloudflare-warp -y
```

### 2. 注册与连接

```bash
# 注册（首次使用时需要，交互式操作）
warp-cli registration new

# 连接 WARP
warp-cli connect

# 断开连接
warp-cli disconnect
```

### 3. 切换为全局模式（重要！）

> ⚠️ **阿里云 ECS 注意事项**：Proxy 模式在阿里云 ECS 上**不可用**，必须使用全局模式（WARP 模式）。

```bash
# 切换到全局模式（WARP 模式）
warp-cli mode warp

# 确认当前模式
warp-cli settings
```

### 4. 常用命令速查

```bash
# 查看连接状态
warp-cli status

# 查看详细设置（当前模式等）
warp-cli settings

# 断开连接
warp-cli disconnect

# 重新连接
warp-cli connect

# 切换模式：全局模式（推荐 ECS 使用）
warp-cli mode warp

# 切换模式：Proxy 模式（仅代理，ECS 不可用）
warp-cli mode proxy

# 查看当前 Warp 状态与 IP
curl -4 https://www.cloudflare.com/cdn-cgi/trace

# 查看版本
warp-cli --version
```

### 5. 设置开机自启（可选）

WARP 客户端安装后默认以 systemd 服务运行，通常已自动开机自启。可手动确认：

```bash
sudo systemctl status warp-svc
sudo systemctl enable warp-svc
```

### 6. 注意事项

- **阿里云 ECS** 上 Proxy 模式不可用，请务必使用 `warp-cli mode warp` 切换到全局模式
- WARP 连接后会改变服务器出口 IP，可能影响需要固定 IP 的服务绑定（如域名白名单等）
- 如遇连接问题，可尝试：`warp-cli disconnect && warp-cli connect`
- 完全重置：`warp-cli registration delete` 然后重新注册
- 国内服务器连接 WARP 可能需要系统已配置 DNS（如 `8.8.8.8` 或 `1.1.1.1`）

---

## 中国 IP 分流配置（直连国内，国外走 WARP）

> ⚠️ 默认 WARP 全局模式会把**所有流量**（包括国内网站）都走 Cloudflare 出口，导致访问百度等国内服务变慢。以下配置实现 **国内 IP 直连、国外 IP 走 WARP**。

### 原理

```
                  ┌─ 中国 IP → eth0 直连（快）
出站流量 → iptables ─┤
                  └─ 国外 IP → CloudflareWARP（代理）
```

WARP 通过策略路由 `32765: not from all fwmark 0x100cf lookup 65743` 将所有流量路由到 WARP 接口。我们用 iptables 给**去往中国 IP 的包**打上 `0x100cf` 标记，使其跳过 WARP 路由表、走主路由表直连。

### 步骤

#### 1. 下载中国 IP 段并创建 ipset

```bash
# 安装 ipset
sudo apt install -y ipset

# 下载 APNIC 中国 IPv4 分配数据
curl -s http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest | \
  awk -F'|' '/CN\|ipv4/ {print $4 "/" 32-log($5)/log(2)}' > /tmp/china-ip.txt

# 创建 ipset 并批量导入
sudo ipset create china hash:net family inet hashsize 16384 maxelem 12000
awk '{print "add china " $0}' /tmp/china-ip.txt | sudo ipset restore -!
```

#### 2. 添加 iptables 标记规则

```bash
# 去往中国 IP 的流量打上标记 0x100cf（会被路由到主路由表直连）
sudo iptables -t mangle -A OUTPUT -m set --match-set china dst -j MARK --set-mark 0x100cf
```

#### 3. 验证分流效果

```bash
# 查看标记规则命中次数
sudo iptables -t mangle -L OUTPUT -n -v | grep china

# 带 mark 的包走 eth0（直连）
ip route get 110.242.68.66 mark 0x100cf
# 输出：110.242.68.66 via 172.17.63.253 dev eth0  ← 直连

# 不带 mark 的包走 WARP
ip route get 110.242.68.66
# 输出：110.242.68.66 dev CloudflareWARP table 65743  ← 走代理

# 实际速度对比
curl -4 -s -o /dev/null -w '%{time_total}s\n' https://www.baidu.com   # 国内 ~2s
curl -4 -s -o /dev/null -w '%{time_total}s\n' https://github.com      # 国外走 WARP
```

#### 4. 持久化（重启不丢）

```bash
# 保存 ipset
sudo ipset save | sudo tee /etc/ipset.conf > /dev/null

# 创建 systemd 服务
sudo tee /etc/systemd/system/china-split-route.service << 'EOF'
[Unit]
Description=Restore China IP split routing
After=network-online.target warp-svc.service
Wants=network-online.target warp-svc.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/sbin/ipset restore -! -f /etc/ipset.conf
ExecStart=/bin/sh -c '/sbin/iptables -t mangle -C OUTPUT -m set --match-set china dst -j MARK --set-mark 0x100cf 2>/dev/null || /sbin/iptables -t mangle -A OUTPUT -m set --match-set china dst -j MARK --set-mark 0x100cf'

[Install]
WantedBy=multi-user.target
EOF

# 启用服务
sudo systemctl daemon-reload
sudo systemctl enable china-split-route.service
sudo systemctl start china-split-route.service
```

### 注意事项

- APNIC 数据约 8700+ 条中国 IP 段，覆盖运营商分配的 IP，但不包含 CDN 边缘节点 IP（如 Cloudflare CDN 上的中国网站可能仍走 WARP）
- ipset 占用约 340KB 内存，可忽略不计
- 服务依赖 `warp-svc.service`，确保 WARP 先启动再恢复分流规则
- 如果 WARP 更新后路由规则变化，检查 `ip rule list` 确认标记逻辑是否仍然匹配

---

*最后更新：2026-06-13*
