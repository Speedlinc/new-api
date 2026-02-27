# new-api 新人入门技术文档

## 一、项目简介

new-api 是一个 **AI API 网关/代理系统**，核心功能是将 40+ 家 AI 提供商（OpenAI、Claude、Gemini、AWS Bedrock 等）统一在一套 API 接口后面，同时提供用户管理、计费、限流和管理后台。

> **一句话理解：** 用户只需要一个 API Key，就能访问所有主流 AI 模型，系统自动路由到对应的提供商。

---

## 二、技术栈

| 层级 | 技术 |
|------|------|
| 后端语言 | Go 1.22+ |
| Web 框架 | Gin |
| ORM | GORM v2 |
| 数据库 | SQLite / MySQL / PostgreSQL（三者同时支持） |
| 缓存 | Redis + 内存缓存 |
| 认证 | JWT、WebAuthn/Passkeys、OAuth |
| 前端框架 | React 18 + Vite |
| UI 组件库 | Semi Design (@douyinfe/semi-ui) |
| 前端包管理 | **Bun**（必须用 Bun，不用 npm/yarn） |
| 国际化 | 后端 go-i18n，前端 i18next |

---

## 三、目录结构总览

```
new-api/
├── main.go              # 程序入口
├── router/              # 路由层：定义 URL 路径
├── controller/          # 控制器层：处理 HTTP 请求
├── service/             # 业务逻辑层
├── model/               # 数据模型层：数据库操作
├── middleware/          # 中间件：认证、限流、分发
├── relay/               # AI 中继层（核心）
│   └── channel/         # 各提供商适配器（40+个）
├── common/              # 通用工具库
├── dto/                 # 数据传输对象（请求/响应结构体）
├── constant/            # 常量定义
├── types/               # 类型定义
├── setting/             # 配置管理
├── i18n/                # 后端国际化
├── oauth/               # OAuth 提供商实现
└── web/                 # 前端 React 项目
```

---

## 四、架构设计

### 4.1 分层架构

```
HTTP 请求
    ↓
Router（路由匹配）
    ↓
Middleware 链（认证 → 限流 → 渠道分发）
    ↓
Controller（请求处理）
    ↓
Service（业务逻辑）
    ↓
Model（数据库操作）
```

### 4.2 AI 请求完整流程

```
客户端 POST /v1/chat/completions
    ↓
1. TokenAuth()       — 验证 API Key 是否有效
2. RateLimit()       — 检查是否超过速率限制
3. Distribute()      — 根据模型名选择合适的渠道（负载均衡）
    ↓
4. controller/relay.go → Relay()
    ↓
5. relay/ 层根据渠道类型获取对应 Adaptor
    ↓
6. Adaptor 将请求转换为目标提供商格式
    ↓
7. 调用上游 API（OpenAI / Claude / Gemini 等）
    ↓
8. 响应转换为统一格式返回给客户端
    ↓
9. 记录日志 + 扣除配额
```

---

## 五、核心概念

### 5.1 渠道（Channel）

渠道是连接上游 AI 提供商的配置单元，包含：
- 提供商类型（OpenAI、Claude 等）
- API Key
- 基础 URL
- 支持的模型列表
- 优先级和权重（用于负载均衡）

**文件位置：** `model/channel.go`

### 5.2 令牌（Token）

用户创建的 API Key，用于调用本系统。每个令牌可以：
- 限制可用模型
- 设置配额上限
- 设置过期时间

**文件位置：** `model/token.go`

### 5.3 适配器（Adaptor）

每个 AI 提供商都有一个适配器，负责：
- 将统一格式的请求转换为提供商特定格式
- 将提供商响应转换回统一格式
- 处理流式响应（SSE）

**文件位置：** `relay/channel/{provider}/adaptor.go`

### 5.4 中继模式（RelayMode）

| 模式 | 说明 |
|------|------|
| ChatCompletions | 对话补全（最常用） |
| Embeddings | 向量嵌入 |
| ImagesGenerations | 图像生成 |
| AudioTranscription | 音频转文字 |
| AudioSpeech | 文字转语音 |
| Rerank | 重排序 |

---

## 六、关键规则（必读）

### 规则 1：JSON 操作必须用 common 包

```go
// ❌ 错误：直接用标准库
import "encoding/json"
json.Marshal(v)

// ✅ 正确：用项目封装
import "new-api/common"
common.Marshal(v)
common.Unmarshal(data, &v)
common.DecodeJson(reader, &v)
```

### 规则 2：数据库必须兼容三种数据库

```go
// 判断数据库类型
if common.UsingPostgreSQL {
    // PostgreSQL 特定逻辑（布尔值用 true/false，列名用双引号）
} else if common.UsingSQLite {
    // SQLite 特定逻辑
}

// 保留字列名使用变量，不要硬编码
// model/main.go 中定义了 commonGroupCol、commonKeyCol
```

### 规则 3：前端必须用 Bun

```bash
# ✅ 正确
bun install
bun run dev
bun run build

# ❌ 错误
npm install
yarn dev
```

---

## 七、本地开发启动

### 后端

```bash
# 1. 复制环境变量
cp .env.example .env

# 2. 编辑 .env，至少配置数据库（留空则使用 SQLite）
# SQL_DSN=

# 3. 启动
go run main.go
# 默认监听 :3000
```

### 前端

```bash
cd web

# 安装依赖
bun install

# 开发模式（代理到后端 :3000）
bun run dev
# 默认监听 :5173
```

---

## 八、重要环境变量

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `PORT` | 服务端口 | 3000 |
| `SQL_DSN` | 数据库连接串（空=SQLite） | - |
| `REDIS_CONN_STRING` | Redis 连接串 | - |
| `SESSION_SECRET` | 会话密钥 | - |
| `RELAY_TIMEOUT` | 请求超时（秒） | - |
| `STREAMING_TIMEOUT` | 流式响应超时（秒） | - |
| `DEBUG` | 调试模式 | false |
| `NODE_TYPE` | 节点类型 master/slave | master |
| `MEMORY_CACHE_ENABLED` | 启用内存缓存 | false |
| `SYNC_FREQUENCY` | 配置同步频率（秒） | - |

---

## 九、新增 AI 提供商（快速参考）

1. 在 `constant/channel.go` 添加新的渠道类型常量
2. 在 `relay/channel/` 下新建目录，实现 `Adaptor` 接口
3. 在 `relay/relay_adaptor.go` 工厂函数中注册
4. 如果支持 StreamOptions，在 `streamSupportedChannels` 中添加
5. 在前端 `web/src/constants/` 中添加渠道名称显示

---

## 十、前端国际化

翻译文件位于 `web/src/i18n/locales/{lang}.json`，key 是中文原文：

```json
{
  "渠道管理": "Channel Management",
  "添加渠道": "Add Channel"
}
```

支持语言：`zh`（默认）、`en`、`fr`、`ru`、`ja`、`vi`

```bash
bun run i18n:extract  # 从代码中提取新的翻译 key
bun run i18n:sync     # 同步各语言文件
bun run i18n:lint     # 检查翻译完整性
```

---

## 十一、代码阅读建议路径

新人建议按以下顺序阅读代码，由浅入深：

| 顺序 | 文件 | 目的 |
|------|------|------|
| 1 | `main.go` | 了解启动流程和组件初始化 |
| 2 | `router/main.go` + `router/relay-router.go` | 了解路由结构 |
| 3 | `middleware/auth.go` | 了解认证机制 |
| 4 | `middleware/distributor.go` | 了解渠道分发和负载均衡 |
| 5 | `controller/relay.go` | 了解请求处理入口 |
| 6 | `relay/channel/openai/adaptor.go` | 了解适配器模式 |
| 7 | `model/channel.go` + `model/user.go` + `model/token.go` | 了解核心数据模型 |

---

## 十二、常见问题

**Q: 为什么不能直接用 `encoding/json`？**
A: 项目封装了 `common/json.go`，便于统一替换底层 JSON 库（如切换到更快的实现），所有业务代码必须通过 `common.*` 调用。

**Q: 为什么要同时支持三种数据库？**
A: 项目面向不同规模的用户，小型部署用 SQLite，生产环境用 MySQL 或 PostgreSQL，代码必须保证三者都能正常运行。

**Q: 渠道选择的逻辑是什么？**
A: `middleware/distributor.go` 中的 `Distribute()` 函数根据模型名称过滤出支持该模型的渠道，再按优先级和权重进行负载均衡选择。

**Q: 如何调试某个提供商的请求/响应？**
A: 设置 `DEBUG=true` 环境变量，日志会输出详细的请求转换信息。


