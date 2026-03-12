# Freemail（D1-only）全流程记录（含浏览器 Relay 端口）

> 本文把本次从 0 到可用的所有关键步骤串起来（部署、端口、变量、路由、常见故障）。
> **注意：不要在文档里写明文口令/Token**；统一在 Cloudflare Variables（建议 Secret）里保存。

---

## A. 安全处置（必须）

1) **GitHub Token 泄露**
- 到 GitHub → Settings → Developer settings → Personal access tokens
- 立即 Revoke/Delete 泄露的 token

---

## B. 代码与一键部署

### B1) 仓库
- 源代码仓库：`cycn104/freemail-d1-only`
- 一键部署生成仓库：`cycn104/mailfree-zidie`

### B2) Cloudflare 一键部署（Deploy to Workers）
1) 打开：`Deploy to Workers` 按钮或 deploy 链接（指向 `freemail-d1-only`）
2) Project/Worker 名：可用 `mailfree-zidie`
3) D1：创建或选择一个 D1 数据库（名称随意）
4) 构建/部署：保持默认 deploy 命令 `npx wrangler deploy`

### B3) 常见失败：`src/server.js not found`
现象：构建日志报：
- `The entry-point file at "src/server.js" was not found.`

原因：Cloudflare 的 Build settings 里把 **Root/Working directory** 指到了子目录（例如 `public` / `src`），导致 entry-point 不在该目录。

修复：Workers → `mailfree-zidie` → Settings → Build configuration：
- **Root/Working directory**：留空或 `.`
- **Build command**：留空
- **Deploy command**：`npx wrangler deploy`
保存后 Retry/Redeploy。

### B4) 部署成功的标志
构建日志出现：
- `Your Worker has access to the following bindings: env.TEMP_MAIL_DB ...`（D1 绑定正常）
- 输出 `https://xxxxx.workers.dev`

---

## C. Worker 运行所需配置（Variables / Secrets）

Workers → `mailfree-zidie` → Settings → Variables（**Production**）

### 必需变量
- `MAIL_DOMAIN`：例如 `zidie.eu.org`
- `ADMIN_PASSWORD`：管理员密码（建议 **Secret**）
- `JWT_TOKEN`：JWT 签名密钥（也就是你说的 `FREEMAIL_TOKEN`，建议 **Secret**）

### 可选变量
- `ADMIN_NAME`：默认 `admin`

### 重要提醒
- 必须确认你改的是 **Production**（不是 Preview），否则会出现“刚能登录、又突然不行”。
- 修改 `JWT_TOKEN` 会使旧登录 Cookie 全部失效；建议用无痕窗口重新登录。

---

## D. Email Routing（收信必做）

域名 `zidie.eu.org` → Email Routing

1) 启用 Email Routing（如有开关）
2) 创建路由规则（Catch-all）：
- 匹配：Catch-all（或 `*@zidie.eu.org`）
- 动作：Deliver/Send to Worker
- Worker：选择 `mailfree-zidie`
3) 保存后回到 Rules 列表刷新确认规则仍存在

---

## E. 端到端自测（收信验证）

1) 打开 Worker 域名（`WORKER_DOMAIN`）
2) 用管理员账号登录：
- username：`admin`（或 `ADMIN_NAME`）
- password：`ADMIN_PASSWORD`
3) 选一个新前缀（避免旧缓存/旧数据），例如：
- `cfok1@zidie.eu.org`
4) 用外部邮箱发一封测试邮件到该地址
5) 页面里应出现新邮件（预览/验证码等）

---

## F. 已知投递问题（Outlook/Hotmail/Gmail）

Cloudflare Email Routing → Activity 中观察到：
- `S3150`（550 5.7.1）：对 Cloudflare 某些 IP 段 **block list**（永久拒收）
- `S775`（451 4.7.650）：对 Cloudflare 某些 IP **reputation rate limit**（临时限流）

结论：这类属于 **上游（Microsoft/Outlook 的 Protocol Filter Agent）信誉/策略问题**，通常不是 Worker 代码或路由规则配置错误。

参考：
- Cloudflare Postmaster：https://developers.cloudflare.com/email-routing/postmaster/
- Microsoft Postmaster：https://aka.ms/postmaster

---

## G. 浏览器接管（OpenClaw Chrome Relay）端口设置（可复用）

> 目标：让 Chrome 扩展能把 Cloudflare 标签页 attach 给自动化端，并保证局域网可访问 control/cdp。

### G1) 固定端口
- Gateway/UI：18797（对外 LAN）
- Browser control：18798（本机监听 127.0.0.1）
- CDP/Extension relay：18799（本机监听 127.0.0.1）

### G2) Chrome 扩展（Clawdbot Browser Relay）设置
扩展 Options 里：
- **Port**：`18799`
- **Gateway token**：与 OpenClaw 的 `gateway.auth.token` 一致

使用方式：
- 在目标网页标签页点击扩展工具栏按钮，角标显示 **ON** = 已 attach。

### G3) 为什么要做 iptables 转发
Browser control / extension relay 服务强制 loopback（127.0.0.1）。
若希望在局域网通过 `LAN_IP:18798/18799` 访问，需要做端口放行 + NAT 重定向。

### G4) iptables 规则（LAN_IP 示例：192.168.191.128）
```bash
# 放行端口
sudo iptables -I INPUT -p tcp --dport 18798 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 18799 -j ACCEPT

# 外部访问 LAN_IP:18798/18799 → 重定向到本机 loopback 同端口
sudo iptables -t nat -I PREROUTING -p tcp -d 192.168.191.128 --dport 18798 -j REDIRECT --to-ports 18798
sudo iptables -t nat -I PREROUTING -p tcp -d 192.168.191.128 --dport 18799 -j REDIRECT --to-ports 18799

# 本机访问 LAN_IP 也能通
sudo iptables -t nat -I OUTPUT -p tcp -d 192.168.191.128 --dport 18798 -j REDIRECT --to-ports 18798
sudo iptables -t nat -I OUTPUT -p tcp -d 192.168.191.128 --dport 18799 -j REDIRECT --to-ports 18799
```

### G5) 联通性自检
```bash
# 18798 不带 token 应 401（说明端口通了）
curl -sS -o /dev/null -w '%{http_code}\n' http://192.168.191.128:18798/

# 18798 带 Bearer token 应 200
curl -sS -o /dev/null -w '%{http_code}\n' -H 'Authorization: Bearer <GATEWAY_TOKEN>' http://192.168.191.128:18798/

# 18799 根路径应 200
curl -sS -o /dev/null -w '%{http_code}\n' http://192.168.191.128:18799/

# 18799 /json/version 需 relay token（通常同 gateway token）
curl -sS -o /dev/null -w '%{http_code}\n' -H 'x-openclaw-relay-token: <GATEWAY_TOKEN>' http://192.168.191.128:18799/json/version
```

### G6) iptables 规则持久化
已保存规则示例路径：`/etc/iptables/rules.v4`

保存：
```bash
sudo mkdir -p /etc/iptables
sudo iptables-save | sudo tee /etc/iptables/rules.v4 >/dev/null
sudo chmod 600 /etc/iptables/rules.v4
```

并可用 systemd 在开机恢复（见：`docs/relay-ports-persistence.md`）。

---

## H. 文档索引

- 部署 checklist：`DEPLOY_NOTES.md`
- 端口/持久化说明：`docs/relay-ports-persistence.md`
- 本全流程：`ALL_STEPS.md`
