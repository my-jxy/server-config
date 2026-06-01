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

*最后更新：2026-06-01*
