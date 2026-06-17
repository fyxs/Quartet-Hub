# New-API 深度解析 & 改造部署指导

> 基于对 Quartet-Hub (fork new-api) 的完整源码分析
> 更新日期：2026-06-17

---

## 目录

1. [项目全景概述](#1-项目全景概述)
2. [核心架构解析](#2-核心架构解析)
3. [请求处理全链路](#3-请求处理全链路)
4. [数据模型与核心概念](#4-数据模型与核心概念)
5. [计费系统深度解析](#5-计费系统深度解析)
6. [部署方案](#6-部署方案)
7. [改造指南](#7-改造指南)
8. [运维与监控](#8-运维与监控)

---

## 1. 项目全景概述

### 1.1 这是什么？

New-API 是一个 **AI API 网关 + 资产管理平台**，它把 40+ 个上游 AI 供应商（OpenAI、Claude、Gemini、Azure、AWS Bedrock、国内大模型等）聚合到一个统一的接口后面，对外提供 **OpenAI 兼容的 API 格式**。

**核心价值**：
- 你只需要对接一个 API，就能使用所有 AI 供应商的模型
- 自带用户管理、充值计费、额度控制、速率限制
- 支持渠道（供应商）的负载均衡、自动重试、故障切换
- 完整的 Web 管理后台 + 数据看板

### 1.2 技术栈

| 层 | 技术 |
|---|------|
| **后端** | Go 1.22+, Gin Web 框架, GORM v2 ORM |
| **前端（default）** | React 19, TypeScript, Rsbuild, Base UI, Tailwind CSS |
| **前端（classic）** | React 18, Vite, Semi Design |
| **数据库** | SQLite / MySQL >= 5.7.8 / PostgreSQL >= 9.6 |
| **缓存** | Redis + 内存缓存 |
| **认证** | JWT, WebAuthn/Passkeys, OAuth (GitHub, Discord, OIDC 等) |
| **包管理器** | Bun (前端) |

### 1.3 项目目录速览

```
Quartet-Hub/
├── main.go                 # 入口：初始化资源 → 启动 HTTP 服务
├── router/                  # 路由定义（API、Relay、Dashboard、Web）
│   ├── relay-router.go     # 核心：所有 AI API 转发路由
│   ├── api-router.go       # 管理 API 路由
│   ├── dashboard.go        # 管理后台页面路由
│   └── web-router.go       # 前端页面路由
├── controller/              # 请求处理器（100+ 文件）
│   ├── relay.go            # 核心：转发请求到上游供应商
│   ├── channel.go          # 渠道管理
│   ├── token.go            # Token 管理
│   └── billing.go          # 计费相关
├── middleware/              # 中间件
│   ├── distributor.go      # 核心：渠道选择与分发
│   ├── auth.go             # Token 鉴权
│   ├── rate-limit.go       # 速率限制
│   └── stats.go            # 统计中间件
├── model/                   # 数据模型（GORM）
│   ├── channel.go          # 渠道模型
│   ├── token.go            # Token 模型
│   ├── user.go             # 用户模型
│   └── pricing.go          # 定价模型
├── relay/                   # AI 供应商适配层（核心）
│   ├── channel/            # 38 个供应商适配器
│   │   ├── openai/         # OpenAI 适配器
│   │   ├── claude/         # Anthropic Claude 适配器
│   │   ├── gemini/         # Google Gemini 适配器
│   │   ├── deepseek/       # DeepSeek 适配器
│   │   ├── ollama/         # Ollama 适配器
│   │   └── ...             # 等 30+ 个
│   ├── common/             # 转发公共工具
│   ├── helper/             # 转发辅助函数
│   └── constant/           # 转发常量
├── service/                 # 业务逻辑层
├── dto/                     # 数据传输对象
├── common/                  # 通用工具（JSON、加密、Redis、限流）
├── setting/                 # 配置管理（计费、模型、性能等）
├── pkg/
│   └── billingexpr/        # 计费表达式引擎（核心）
├── web/
│   ├── default/            # 默认前端（React 19 + Rsbuild）
│   └── classic/            # 经典前端（React 18 + Vite）
└── docs/                    # 文档
```

---

## 2. 核心架构解析

### 2.1 分层架构

```
┌─────────────────────────────────────────────────┐
│                    前端 (React)                   │
│         web/default/  +  web/classic/            │
├─────────────────────────────────────────────────┤
│                   HTTP 路由层                     │
│    router/relay-router.go (AI API 转发)          │
│    router/api-router.go  (管理 API)              │
│    router/dashboard.go   (管理后台)              │
├─────────────────────────────────────────────────┤
│                   中间件层                        │
│  TokenAuth → Distribute → RateLimit → Stats      │
├─────────────────────────────────────────────────┤
│                   控制器层                        │
│         controller/relay.go (转发核心)            │
├─────────────────────────────────────────────────┤
│                   服务层                          │
│              service/ (业务逻辑)                  │
├─────────────────────────────────────────────────┤
│                   适配器层                        │
│     relay/channel/* (38+ 供应商适配器)           │
├─────────────────────────────────────────────────┤
│                   数据层                          │
│     model/ (GORM): User, Token, Channel, Log     │
├─────────────────────────────────────────────────┤
│               SQLite / MySQL / PostgreSQL         │
└─────────────────────────────────────────────────┘
```

### 2.2 核心概念关系图

```
用户 (User)
  │
  ├── 拥有多个 API Token
  │     ├── Token 有额度限制 (remain_quota)
  │     ├── Token 可以限制模型访问 (model_limits)
  │     ├── Token 可以限制 IP (allow_ips)
  │     └── Token 属于某个分组 (group)
  │
  ├── 渠道 (Channel)
  │     ├── 每个渠道对应一个上游 AI 供应商的 API Key
  │     ├── 渠道有类型 (Type): OpenAI=1, Claude=2, Gemini=3...
  │     ├── 渠道有权重 (Weight): 用于负载均衡
  │     ├── 渠道有模型列表 (Models): 声明该渠道支持的模型
  │     ├── 渠道有优先级 (Priority): 高优先级优先使用
  │     └── 渠道属于某个分组 (group)
  │
  └── 分组 (Group)
        ├── Token 的 group 和 Channel 的 group 匹配
        ├── "default" 分组为默认分组
        └── 可以实现供应商隔离（不同用户组使用不同渠道）
```

### 2.3 各种 Channel Type（渠道类型）

项目支持 **38+ 种渠道类型**，核心的包括：

| 类型常量 | 值 | 说明 |
|---------|---|------|
| `ChannelTypeOpenAI` | 1 | OpenAI 及兼容接口 |
| `ChannelTypeAnthropic` | 2 | Anthropic Claude |
| `ChannelTypeGemini` | 3 | Google Gemini |
| `ChannelTypeAzure` | 4 | Microsoft Azure |
| `ChannelTypeDeepSeek` | 12 | DeepSeek |
| `ChannelTypeOllama` | 14 | Ollama 本地模型 |
| `ChannelTypeAli` | 15 | 阿里 (通义千问) |
| `ChannelTypeZhipu` | 16 | 智谱 (GLM) |
| `ChannelTypeMoonshot` | 17 | 月之暗面 (Kimi) |
| `ChannelTypeBaidu` | 18 | 百度 (文心) |
| `ChannelTypeXunfei` | 20 | 讯飞 (星火) |
| ... | ... | 还有 30+ 种 |

---

## 3. 请求处理全链路

### 3.1 一次 AI API 调用的完整流程

以 `POST /v1/chat/completions` 为例：

```
客户端请求
  │  POST /v1/chat/completions
  │  Authorization: Bearer sk-xxxxxx
  │  Body: {"model": "gpt-4o", "messages": [...]}
  ▼
┌─ 1. CORS & 解压缩 ─────────────────────────────┐
│  middleware.CORS()                              │
│  middleware.DecompressRequestMiddleware()       │
└────────────────────────────────────────────────┘
  ▼
┌─ 2. Token 鉴权 ─────────────────────────────────┐
│  middleware.TokenAuth()                         │
│  • 从 Header 提取 sk-xxxxxx                    │
│  • 查询 token 缓存/数据库                      │
│  • 验证状态、有效期、IP 白名单                 │
│  • 将 token 信息注入 Context                    │
└────────────────────────────────────────────────┘
  ▼
┌─ 3. 渠道选择 ───────────────────────────────────┐
│  middleware.Distribute()                        │
│  • 解析请求中的 model 名称                      │
│  • 检查 token 的模型访问限制                    │
│  • 根据 model + group 从缓存中选择可用渠道       │
│  • 选择策略：优先级 > 权重随机 > 响应时间        │
│  • 支持自动重试和故障切换                        │
└────────────────────────────────────────────────┘
  ▼
┌─ 4. 格式转换 & 转发 ───────────────────────────┐
│  controller.Relay()                            │
│  • 根据渠道类型选择对应的适配器                  │
│  • 格式转换：OpenAI → Claude/Gemini/Azure 等     │
│  • 通过 HTTP 转发到上游供应商                    │
│  • 流式响应实时推送给客户端                      │
└────────────────────────────────────────────────┘
  ▼
┌─ 5. 响应处理 & 计费 ───────────────────────────┐
│  • 转换上游响应为 OpenAI 格式                    │
│  • 统计 token 使用量                            │
│  • 根据计费表达式计算费用                        │
│  • 扣除用户/token 额度                          │
│  • 记录使用日志                                  │
└────────────────────────────────────────────────┘
  ▼
客户端收到响应
```

### 3.2 关键中间件详解

**Token 鉴权** (`middleware/auth.go`):
- 从 `Authorization: Bearer sk-xxx` 提取 token
- 也支持 `x-api-key` header（兼容 Anthropic 格式）
- 验证后注入 Context：`token_id`, `token_group`, `token_model_limit` 等

**渠道分发** (`middleware/distributor.go`):
- 核心选择逻辑：`Service.SelectChannel(model, group)`
- 优先使用 `priority` 高的渠道
- 同优先级按 `weight` 权重随机选择
- 失败后自动换渠道重试（最多 3 次）

**速率限制** (`middleware/rate-limit.go`):
- 用户级别的模型请求频率限制
- 基于 Redis 或内存实现

---

## 4. 数据模型与核心概念

### 4.1 用户 (User)

```go
type User struct {
    Id          int     // 用户 ID
    Username    string  // 用户名（登录用）
    Password    string  // 密码（bcrypt 加密）
    Role        int     // 角色：1=普通用户, 10=管理员, 100=超级管理员
    Status      int     // 状态：1=启用, 2=禁用
    Quota       int     // 剩余额度（以 token 为单位）
    UsedQuota   int     // 已用额度
    Group       string  // 所属分组
    AccessToken string  // 系统管理 token（32位，管理员用）
}
```

### 4.2 Token（API 密钥）

```go
type Token struct {
    Id              int     // Token ID
    UserId          int     // 所属用户
    Key             string  // API Key (sk- 开头)
    Status          int     // 状态
    RemainQuota     int     // 剩余额度
    UnlimitedQuota  bool    // 无限额度
    ModelLimits     string  // 模型限制（JSON）
    AllowIps        string  // IP 白名单
    Group           string  // 分组（影响渠道选择）
    ExpiredTime     int64   // 过期时间（-1=永不过期）
}
```

### 4.3 渠道 (Channel)

```go
type Channel struct {
    Id           int      // 渠道 ID
    Type         int      // 渠道类型（OpenAI=1, Claude=2...）
    Key          string   // API Key（上游供应商的密钥）
    BaseURL      string   // 上游 API 地址
    Models       string   // 支持的模型列表（逗号分隔）
    Group        string   // 所属分组
    Weight       uint     // 权重（负载均衡）
    Priority     int64    // 优先级（越高越优先）
    Status       int      // 状态：1=启用, 2=禁用
    Balance      float64  // 余额（USD）
    AutoBan      int      // 是否自动禁用（0=否, 1=是）
}
```

### 4.4 分组机制

分组是实现**多租户隔离**的关键：
- Token 的 `group` 决定用户能看到和使用哪些渠道
- Channel 的 `group` 决定该渠道服务于哪些用户
- `"default"` 分组是默认分组
- 可以设置多个分组：`"vip,default"` 表示同时属于两个分组

---

## 5. 计费系统深度解析

### 5.1 计费表达式引擎

项目使用自研的表达式引擎 `pkg/billingexpr/`，基于 [expr-lang/expr](https://github.com/expr-lang/expr)。

**核心设计理念**：
- 一条表达式定义所有计费逻辑
- 价格为真实美元价格（$ / 1M tokens）
- 支持缓存命中、图片、音频的差异化定价

### 5.2 表达式变量

| 变量 | 含义 |
|------|------|
| `p` | prompt（输入）token 数 |
| `c` | completion（输出）token 数 |
| `cr` | 缓存命中（读取）token 数 |
| `cc` | 缓存创建 token 数 |
| `img` | 图片输入 token 数 |
| `ai` | 音频输入 token 数 |
| `ao` | 音频输出 token 数 |

### 5.3 表达式示例

```
# 简单定价（GPT-4o）
p * 2.5 + c * 10

# 含缓存定价（Claude）
p * 3 + c * 15 + cr * 0.3 + cc * 3.75

# 阶梯定价（达到一定量后折扣）
tier("standard", p * 2.5 + c * 10)
tier("high", p * 2.0 + c * 8)  # 高用量折扣
```

### 5.4 计费计算流程

```
请求到达 → Token 鉴权 → 渠道选择 → 转发到上游
    ↓
收到响应（含 token 使用量）
    ↓
计费表达式引擎计算费用
    ↓
扣除用户/Token 额度
    ↓
记录使用日志（含原始 token 数 + 计费倍数 + 实际费用）
```

---

## 6. 部署方案

### 6.1 方案一：Docker Compose（推荐）

这是最简单的部署方式，一键启动所有服务：

```bash
# 1. 克隆项目
git clone https://github.com/QuantumNous/new-api.git
cd new-api

# 2. 编辑 docker-compose.yml，修改密码
#    - PostgreSQL 密码
#    - Redis 密码
#    - SQL_DSN 中的密码

# 3. 启动
docker compose up -d

# 4. 访问 http://localhost:3000
```

此方案会启动：
- `new-api`：主应用（端口 3000）
- `postgres`：PostgreSQL 15 数据库
- `redis`：Redis 缓存

### 6.2 方案二：纯 Docker

```bash
# 使用内置 SQLite（单机部署）
docker run --name new-api -d --restart always \
  -p 3000:3000 \
  -e TZ=Asia/Shanghai \
  -v ./data:/data \
  calciumion/new-api:latest

# 使用外部 MySQL
docker run --name new-api -d --restart always \
  -p 3000:3000 \
  -e SQL_DSN="root:password@tcp(mysql-host:3306)/new-api?parseTime=true" \
  -e REDIS_CONN_STRING="redis://:password@redis-host:6379" \
  -e TZ=Asia/Shanghai \
  -v ./data:/data \
  calciumion/new-api:latest
```

### 6.3 方案三：源码构建

```bash
# 1. 构建前端
cd web && bun install
cd default && bun run build
cd ../classic && bun run build

# 2. 构建后端
cd ../..
go build -o new-api main.go

# 3. 配置环境变量
cp .env.example .env
# 编辑 .env 文件

# 4. 启动
./new-api
```

### 6.4 关键环境变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `PORT` | 服务端口 | `3000` |
| `SQL_DSN` | 数据库连接 | `postgresql://root:pwd@localhost:5432/new-api` |
| `REDIS_CONN_STRING` | Redis 连接 | `redis://:pwd@localhost:6379` |
| `SESSION_SECRET` | Session 密钥（多机必设） | 随机字符串 |
| `CRYPTO_SECRET` | 加密密钥（Redis 必设） | 随机字符串 |
| `TZ` | 时区 | `Asia/Shanghai` |
| `STREAMING_TIMEOUT` | 流式超时（秒） | `300` |
| `ERROR_LOG_ENABLED` | 错误日志 | `true` |
| `BATCH_UPDATE_ENABLED` | 批量更新 | `true` |
| `NODE_TYPE` | 节点类型 | `master` |

### 6.5 多机部署注意事项

- **必须设置** `SESSION_SECRET`（所有节点相同）—— 否则登录状态不一致
- **共享 Redis 必须设置** `CRYPTO_SECRET`（所有节点相同）—— 否则加密数据无法解密
- **数据库共享**：所有节点连接同一个数据库和 Redis

---

## 7. 改造指南

### 7.1 品牌/UI 改造

#### 修改项目名称和 Logo

```
修改位置：
├── web/default/public/logo.png          # Logo 图片
├── web/default/src/                     # 前端源码中的品牌名称
│   ├── components/                      # 搜索 "New API" 替换为你的品牌名
│   └── i18n/locales/*.json              # 多语言文本
├── README.md                            # 项目说明
└── main.go                              # 启动日志中的名称
```

#### 修改前端主题色

```
web/default/tailwind.config.ts          # Tailwind 主题色
web/default/src/styles/                  # 全局样式
```

### 7.2 添加新的 AI 供应商

在 `relay/channel/` 下创建新目录，实现适配器接口：

```
relay/channel/your-provider/
├── adapter.go        # 适配器实现（必须）
│   ├── GetChannelType()
│   ├── GetRequestURL(info)
│   ├── SetupRequestHeader(c, info)
│   ├── ConvertRequest(c, info)
│   └── ConvertResponse(c, info)
└── ...
```

参考最简单的适配器 `relay/channel/openai/adapter.go` 作为模板。

### 7.3 计费/定价改造

#### 修改默认定价

```
model/pricing_default.go   # 默认模型定价表
setting/ratio_setting/     # 汇率/倍率设置
```

#### 自定义计费表达式

管理后台 → 模型管理 → 编辑模型的计费表达式。

例如将 GPT-4o 从 `p * 2.5 + c * 10` 改为 `p * 1.5 + c * 8` 来调整价格。

### 7.4 支付集成改造

项目已支持：
- **Stripe** (`controller/topup_stripe.go`, `controller/subscription_payment_stripe.go`)
- **EPay** (`controller/topup.go` 中通过 `go-epay` 库)
- **Creem** (`controller/topup_creem.go`)

添加新支付方式：
1. 创建 `controller/topup_yourpayment.go`
2. 实现支付回调处理
3. 在管理后台配置支付参数

### 7.5 认证方式扩展

项目已支持：
- 用户名/密码
- OAuth: GitHub, Discord, LinuxDO, OIDC, Telegram, WeChat
- WebAuthn/Passkeys

添加新 OAuth 提供商：
1. 在 `oauth/` 下创建实现
2. 在 `model/user.go` 添加对应的 OAuth ID 字段
3. 在 `controller/` 创建对应的回调处理

### 7.6 数据库选型建议

| 场景 | 推荐数据库 |
|------|-----------|
| 单机/个人使用 | SQLite（零配置） |
| 小团队（< 100 用户） | PostgreSQL |
| 生产环境/多机部署 | PostgreSQL + Redis |
| 已有 MySQL 基础设施 | MySQL 8.0+ |

### 7.7 安全加固清单

部署到生产环境前必须完成：

- [ ] 修改 `docker-compose.yml` 中所有默认密码
- [ ] 设置 `SESSION_SECRET` 为强随机字符串
- [ ] 设置 `CRYPTO_SECRET` 为强随机字符串
- [ ] 配置 HTTPS（通过反向代理 Nginx/Caddy）
- [ ] 配置防火墙，仅暴露必要端口
- [ ] 定期备份数据库
- [ ] 修改默认管理员密码
- [ ] 审查渠道 API Key 的权限范围

---

## 8. 运维与监控

### 8.1 日常运维

```bash
# 查看日志
docker compose logs -f new-api

# 重启服务
docker compose restart new-api

# 数据库备份（PostgreSQL）
docker compose exec postgres pg_dump -U root new-api > backup.sql

# 数据库备份（SQLite）
cp data/one-api.db data/one-api.db.backup.$(date +%Y%m%d)
```

### 8.2 性能监控

项目内置：
- **Pyroscope** 持续性能分析（设置 `PYROSCOPE_URL` 环境变量）
- **pprof** 调试端点（设置 `ENABLE_PPROF=true`，端口 8005）
- **数据看板**：管理后台 → 数据看板

### 8.3 常见问题

| 问题 | 解决方案 |
|------|---------|
| 渠道测试失败 | 检查上游 API Key 和 Base URL，确认网络连通性 |
| 流式响应中断 | 增大 `STREAMING_TIMEOUT`（默认 300 秒） |
| Token 额度不更新 | 检查 `BATCH_UPDATE_ENABLED` 和 Redis 连接 |
| 多机登录不一致 | 确保所有节点 `SESSION_SECRET` 相同 |
| 数据库锁定（SQLite） | 考虑切换到 PostgreSQL |

### 8.4 官方资源

- 官方文档：https://docs.newapi.pro
- GitHub：https://github.com/QuantumNous/new-api
- Docker Hub：`calciumion/new-api:latest`

---

## 附录：快速上手路径

### 如果你是第一次接触这个项目：

1. **先用 Docker Compose 跑起来**（见 §6.1），访问 `http://localhost:3000`
2. **完成初始化设置**：创建管理员账户
3. **添加一个渠道**：在管理后台 → 渠道管理 → 添加渠道
   - 类型选择 OpenAI
   - 填入你的 OpenAI API Key
   - 模型填 `gpt-4o,gpt-4o-mini`
4. **创建一个 Token**：在管理后台 → Token 管理 → 添加 Token
5. **测试调用**：
   ```bash
   curl http://localhost:3000/v1/chat/completions \
     -H "Authorization: Bearer sk-your-token" \
     -H "Content-Type: application/json" \
     -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello"}]}'
   ```
6. **理解数据流**：再回来看 §3 的请求处理全链路
7. **开始改造**：参考 §7 的改造指南

---

> **重要提醒**：本项目使用 AGPL v3 许可证。如果你修改了代码并对外提供服务，需要开源你的修改。
> 具体条款见项目根目录的 `LICENSE` 文件。
