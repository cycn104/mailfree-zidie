# Freemail (D1-only) 部署记录 / Checklist

> 目标：在 Cloudflare Workers 上部署 `mailfree-zidie`（临时邮箱），仅使用 **D1**（不使用 R2）。

## 0. 仓库

- 种子仓库（源代码）：https://github.com/cycn104/freemail-d1-only
- 一键部署生成仓库（你的账号下）：https://github.com/cycn104/mailfree-zidie

## 1. 一键部署（Deploy to Workers）

部署命令：`npx wrangler deploy`（保持默认即可）

### 1.1 常见失败：entry-point 找不到

如果构建日志出现：

- `✘ [ERROR] The entry-point file at "src/server.js" was not found.`

几乎总是因为 **Root / Working directory** 选错（进入了子目录），导致 `src/server.js` 在该目录下不存在。

修复：
- Root / Working directory：留空 或 `.`
- 不要填 `public` / `src` / `docs`

然后重新 Deploy。

## 2. D1 绑定（必需）

本项目 Worker 代码通过绑定名访问 D1：
- D1 binding（变量/绑定名）：`TEMP_MAIL_DB`

说明：
- 部署表单中不一定会出现“Binding name”输入框（可能直接读取 `wrangler.toml`）。
- 如果部署后出现数据库相关报错（如 TEMP_MAIL_DB 未定义/绑定缺失），需要在：
  - Cloudflare Dashboard → Workers & Pages → 你的 Worker → Settings → Bindings → D1 Database bindings
  - Add binding：`TEMP_MAIL_DB` → 选择你的 D1 数据库 → Save

> D1 数据库“名字”可随意；关键是绑定名 `TEMP_MAIL_DB` 必须匹配代码。

## 3. 环境变量（必需）

在 Worker → Settings → Variables 中设置（不要写进仓库、不要提交到 Git）：

- `MAIL_DOMAIN`：你的收件域名，例如 `zidie.eu.org`
- `ADMIN_NAME`：默认 `admin`（可不填）
- `ADMIN_PASSWORD`：强密码（建议随机生成）
- `JWT_TOKEN`：强随机密钥（JWT 签名密钥；也就是 FREEMAIL_TOKEN）

> ⚠️ 安全：`ADMIN_PASSWORD` / `JWT_TOKEN` 属于密钥/口令，只应存放在 Cloudflare Variables 或密码管理器中。

## 4. R2（D1-only 版本不需要）

- 不需要配置 `MAIL_EML`（R2 绑定）。
- D1-only 模式下：邮件仍可写入 D1（验证码/预览），但不支持完整 EML 下载/附件。

## 5. 部署成功后的产物

部署成功后会得到：
- `WORKER_DOMAIN`：形如 `https://xxxxx.workers.dev`

## 6. 收信必做：Email Routing 绑定 Worker

若不绑定，永远收不到邮件。

Cloudflare → 域名（如 `zidie.eu.org`）→ Email Routing：
- 配置 Catch-all 或路由规则
- 动作：Deliver to Worker → 选择此 Worker

## 7. 自测（建议）

1) 打开 `WORKER_DOMAIN`，用管理员账号登录：
- 用户名：`admin`（或你设置的 `ADMIN_NAME`）
- 密码：`ADMIN_PASSWORD`

2) 从外部邮箱发一封测试邮件到：
- `<任意前缀>@MAIL_DOMAIN`（例如 `test@zidie.eu.org`）

3) 页面应出现邮件预览/验证码（D1-only 不提供完整 EML 下载属正常）。

## 8. 本次实际操作记录（2026-03-12）

### 已完成
- ✅ GitHub 泄露的 Personal Access Token 已撤销（Revoke/Delete）。
- ✅ 通过 Deploy to Workers 流程生成并部署项目 `mailfree-zidie`。
- ✅ 修复构建报错 `src/server.js not found`：将 Cloudflare Build settings 中 `Root/Working directory` 清空（或设为 `.`），`Build command` 留空，`Deploy command` 使用 `npx wrangler deploy`。
- ✅ 成功部署并确认绑定：
  - `env.TEMP_MAIL_DB`（D1：`freemail_db`）
  - `env.ASSETS`（静态资源）
- ✅ 部署成功输出：
  - `WORKER_DOMAIN`：`https://mailfree-zidie.si-ling.workers.dev`
  - `FREEMAIL_TOKEN`：等同于 Cloudflare Variables 中配置的 `JWT_TOKEN`

### 待完成
- ⏳ （可选）进一步完善：为 Outlook/Hotmail/Gmail 等部分发件侧的投递失败做兜底方案（见“已知问题”）。

## 9. 已知问题（投递侧 / 上游）

- ✅ 已确认：Email Routing → Worker → D1 入库链路正常（使用可送达的外部邮箱发信已能在页面看到新邮件）。
- ⚠️ Outlook/Hotmail（以及部分 Gmail 测试邮件）可能出现投递失败，Cloudflare Activity 显示为上游拒绝/限流：
  - `S3150`：`550 5.7.1 ... part of their network is on our block list`（示例 IP：`104.30.10.222` / `104.30.10.101`）
  - `S775`：`451 4.7.650 ... temporarily rate limited due to IP reputation`（示例 IP：`104.30.8.68`）

这类问题发生在 Microsoft 上游过滤（Protocol Filter Agent）对 Cloudflare 相关 IP 的信誉/策略判断，通常**不是 Worker 代码或路由规则配置错误**。

参考：
- Cloudflare Email Routing Postmaster：https://developers.cloudflare.com/email-routing/postmaster/
- Microsoft Postmaster：https://aka.ms/postmaster
