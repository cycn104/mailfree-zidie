# Chrome Relay 端口（18797/18798/18799）固化方案（可复用）

## 目标

- Gateway/UI: `http://<LAN_IP>:18797`
- Browser control: `http://<LAN_IP>:18798`（实际服务监听在 127.0.0.1，使用 iptables 转发到 LAN）
- CDP/Extension relay: `http://<LAN_IP>:18799`（同上）

> 说明：OpenClaw 的 browser control / extension relay 组件要求 loopback（127.0.0.1）启动。
> 若希望在局域网可用，需要做端口转发/放行。

## 当前环境 LAN_IP

- `192.168.191.128`

## iptables 规则（一次性执行）

```bash
# 放行端口
sudo iptables -I INPUT -p tcp --dport 18798 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 18799 -j ACCEPT

# 外部访问 LAN_IP:18798/18799 → 重定向到本机 loopback 同端口
sudo iptables -t nat -I PREROUTING -p tcp -d 192.168.191.128 --dport 18798 -j REDIRECT --to-ports 18798
sudo iptables -t nat -I PREROUTING -p tcp -d 192.168.191.128 --dport 18799 -j REDIRECT --to-ports 18799

# 本机访问 LAN_IP:18798/18799 也能通（避免本机 curl 走 LAN_IP 时失败）
sudo iptables -t nat -I OUTPUT -p tcp -d 192.168.191.128 --dport 18798 -j REDIRECT --to-ports 18798
sudo iptables -t nat -I OUTPUT -p tcp -d 192.168.191.128 --dport 18799 -j REDIRECT --to-ports 18799
```

## 联通性自检

```bash
# 18798 不带 token 应 401（说明通了）
curl -sS -o /dev/null -w '%{http_code}\n' http://192.168.191.128:18798/

# 18798 带 Bearer token 应 200
curl -sS -o /dev/null -w '%{http_code}\n' -H 'Authorization: Bearer <GATEWAY_TOKEN>' http://192.168.191.128:18798/

# 18799 根路径应 200
curl -sS -o /dev/null -w '%{http_code}\n' http://192.168.191.128:18799/

# 18799 /json/version 需 relay token（同 gateway token）应 200
curl -sS -o /dev/null -w '%{http_code}\n' -H 'x-openclaw-relay-token: <GATEWAY_TOKEN>' http://192.168.191.128:18799/json/version
```

## 持久化（重启后仍生效）

当前机器没有 `netfilter-persistent`，可用 `iptables-save` 保存到：

- `/etc/iptables/rules.v4`

保存命令：

```bash
sudo mkdir -p /etc/iptables
sudo iptables-save | sudo tee /etc/iptables/rules.v4 >/dev/null
sudo chmod 600 /etc/iptables/rules.v4
```

要在系统启动时自动恢复，可以用 systemd 单元（示例）：

`/etc/systemd/system/iptables-restore.service`

```ini
[Unit]
Description=Restore iptables rules
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/sbin/iptables-restore /etc/iptables/rules.v4
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

启用：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now iptables-restore.service
```

## Chrome 扩展配置（Clawdbot Browser Relay）

扩展 Options：
- Port：`18799`
- Gateway token：与 `gateway.auth.token` 一致

附加 tab：在目标网页标签页点工具栏按钮，角标显示 `ON`。
