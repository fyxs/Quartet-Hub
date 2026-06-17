# New-API Fork 改造清单

> 从 fork 到部署的逐步骤操作指南
> 更新日期：2026-06-17

---

## 目录

1. [安全加固（必须）](#1-安全加固)
2. [品牌替换（建议）](#2-品牌替换)
3. [前端定制（可选）](#3-前端定制)
4. [验证清单](#4-验证清单)

---

## 1. 安全加固

### 1.1 生成密钥

```bash
# 生成两个随机密钥
openssl rand -hex 32  # → 用作 SESSION_SECRET
openssl rand -hex 32  # → 用作 CRYPTO_SECRET
```

### 1.2 修改 docker-compose.yml

```yaml
# 文件: docker-compose.yml

environment:
  # 改密码（三处）
  - SQL_DSN=postgresql://root:<新密码>@postgres:5432/new-api
  - REDIS_CONN_STRING=redis://:<新密码>@redis:6379
  # 改密钥
  - SESSION_SECRET=<上一步生成的值>
# - CRYPTO_SECRET=<上一步生成的值>   # 取消注释并填入

# Redis 密码
redis:
  command: ["redis-server", "--requirepass", "<新密码>"]

# PostgreSQL 密码
postgres:
  environment:
    POSTGRES_PASSWORD: <新密码>
```

### 1.3 创建 .env 文件

```bash
cp .env.example .env
```

编辑 `.env`，至少取消以下注释并填入值：

```ini
SESSION_SECRET=<你的密钥>
# CRYPTO_SECRET=<你的密钥>   # 用 Redis 时取消注释
```

### 1.4 检查项

- [ ] 没有使用默认密码 `123456`
- [ ] `SESSION_SECRET` 已设置且足够随机
- [ ] 如果用 Redis，`CRYPTO_SECRET` 已设置
- [ ] `.env` 文件已加入 `.gitignore`（确认未提交）

---

## 2. 品牌替换

### 2.1 后端品牌名

**文件: `common/constants.go`**

```go
// 修改前
var SystemName = "New API"
var Footer = ""
var Logo = ""

// 修改后 — 替换为你的品牌
var SystemName = "<你的品牌名>"
var Footer = "<页脚 HTML，可留空>"
var Logo = "<Logo URL，可留空>"
```

**文件: `main.go`** 第 59 行

```go
// 修改前
common.SysLog("New API " + common.Version + " started")

// 修改后
common.SysLog("<你的品牌名> " + common.Version + " started")
```

### 2.2 前端多语言文本

**目录: `web/default/src/i18n/locales/`**

需要修改的文件：`zh.json`, `en.json`, `fr.json`, `ru.json`, `ja.json`, `vi.json`

每文件中搜索 `New API` 约 10-15 处，替换为你的品牌名。主要出现在：

| 场景 | 示例 |
|------|------|
| 系统名称展示 | `"New API"` → `"<你的品牌>"` |
| 邮件发件人 | `"New API <noreply@example.com>"` |
| 项目仓库链接文本 | `"New API Project Repository:"` |
| 欢迎语 | `"Welcome to our New API..."` |
| 文档中的项目自称 | 各处提到 `New API` 的说明文字 |

> 注意：附带 `One API`（前身项目名）的文本如 `"If connecting to upstream One API or New API relay projects..."` 建议保留，因为那是技术兼容性说明。

### 2.3 Logo 图片

```
web/default/public/logo.png    → 替换为你的 Logo（保持文件名）
```

### 2.4 前端构建配置

**文件: `web/default/package.json`**

```json
// 修改前
"name": "newapi-web"

// 修改后
"name": "<你的项目名>-web"
```

---

## 3. 前端定制

### 3.1 主题色

**文件: `web/default/src/styles/index.css`**

查找并替换主色调（默认是蓝色系）：

```css
/* 搜索以下 CSS 变量或颜色值，替换为你的品牌色 */
/* 常见位置：:root 或 theme 相关 CSS 文件 */
```

### 3.2 邮箱配置（通知邮件）

在管理后台或 `.env` 中配置：

```ini
# SMTP 配置（在管理后台 -> 系统设置中配置更方便）
# 或在 .env 中设置
SMTP_SERVER=smtp.example.com
SMTP_PORT=587
SMTP_ACCOUNT=noreply@example.com
SMTP_TOKEN=your-smtp-password
```

---

## 4. 验证清单

部署前逐项确认：

### 安全
- [ ] docker-compose.yml 无默认密码
- [ ] SESSION_SECRET 已设置
- [ ] CRYPTO_SECRET 已设置（使用 Redis 时）
- [ ] HTTPS 已配置（通过 Nginx/Caddy 反向代理）
- [ ] 防火墙仅暴露必要端口

### 品牌
- [ ] SystemName 已修改
- [ ] 前端 i18n 文本中的 "New API" 已替换
- [ ] Logo 已替换
- [ ] 启动日志中的名称已修改

### 功能
- [ ] 管理后台可正常登录
- [ ] 添加渠道后可成功测试
- [ ] Token 调用返回正常
- [ ] 流式响应正常

### 部署
- [ ] 数据库备份策略已建立
- [ ] 日志目录已挂载
- [ ] 健康检查端点正常（`/api/status`）

---

## 快速命令参考

```bash
# 构建前端
cd web && bun install
cd default && bun run build
cd ../classic && bun run build

# 构建后端
cd ../..
go build -o new-api main.go

# Docker 部署
docker compose up -d

# 查看日志
docker compose logs -f new-api

# 重启
docker compose restart new-api
```
