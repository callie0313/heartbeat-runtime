# Dylan Heartbeat — AI Residency Runtime for Kelivo

**一个基于 Kelivo 的 AI Agent 运行时。**  
它不是“聊天接口转发器”，而是一个让 AI 真正**长期居住**的数字环境。

---

## ✨ 核心目标：AI Residency（AI 常驻）

- 🧠 **持续上下文** – 即使对话中断，AI 仍能记住发生过的事
- ⏰ **主动唤醒** – 无人说话时，AI 会自动醒来，思考、关心你
- 📳 **Bark 推送** – 主动发消息到你的手机，像真实存在的人
- 🕰️ **长期时间感** – 知道自己多久没见你，什么时候主动联系过你
- 🧩 **行为连续性** – 发过的推送、沉默的夜晚，都会被 AI 记住
- 🎭 **人格不变** – 完全保留 Kelivo 的角色设定，不做任何破坏

**AI 不再只是“收到消息 → 回复”，而是“即使你不说话，它也在想你”。**

---

## 🧱 系统架构

```
Kelivo (客户端)
    ↓ 完整请求（SP、世界书、记忆、工具调用、最新消息）
Gateway (server.js)  ← 核心转发 + 时间线维护 + 主动行为注入
    ↓ 原封不动转发 + 已注入的主动行为上下文
LLM API
    ↑
wake_up.js  ← 定时自动唤醒，通过 Gateway 接口注入事件
    ↓
Bark 推送 → 你的手机
```

- **Gateway 不修改 Kelivo 的任何人格设定**，只负责在正确的时间位置注入 AI 自己的主动行为（推送/静默）。
- **时间线（`enhanced_messages.json`）** 是 AI 的“世界状态”，只记录真实对话 + 自主行为，不包含系统规则。
- **时间戳记忆库（`message_timestamps.json`）** 让历史消息即使丢失时间前缀也能找回原始时间，实现推送精确散落。

---

## 📦 文件说明

| 文件 | 作用 |
|------|------|
| `server.js` | 主 Gateway。转发请求、维护时间线、注入推送事件、提供管理页面。 |
| `wake_up.js` | 自动唤醒 Runtime。按间隔唤醒 AI，生成推送或静默，发送到手机，写入时间线。 |
| `enhanced_messages.json` | **AI 世界时间线**。SP + 真实对话 + 推送事件。不是日志，是 AI 的当前世界。 |
| `message_timestamps.json` | **时间戳记忆库**。通过内容指纹记录每条消息的原始时间，找回历史消息时间。 |
| `.env` | 环境变量。API Key、Bark Key、模型名称等（不提交到 Git）。 |
| `.env.example` | 环境变量模板，供新用户参考配置。 |

---

## 🚀 快速开始

### 环境要求

- **Node.js** v26 或更高版本
- 一个可用的 LLM API（支持 OpenAI 接口格式的中转站或官方）
- **Bark** App（iOS）及有效 Key
- **Kelivo** App（用于前端交互）

### 安装与配置

```bash
# 克隆项目
git clone https://github.com/callie0313/dylan-heartbeat.git
cd dylan-heartbeat

# 安装依赖
npm install

# 配置环境变量（复制模板并修改）
cp .env.example .env
nano .env   # 或使用文本编辑器，填入你的 Key 和地址
```

`.env` 文件内容示例：

```env
TARGET_API_URL=https://你的API地址/v1/chat/completions
TARGET_API_KEY=sk-你的APIKey
MODEL_NAME=deepseek-v4-pro
BARK_KEY=你的Bark设备Key
CUSTOM_ICON_URL=https://你的图标URL（可选）
ADMIN_USER=admin
ADMIN_PASSWORD=你的强密码
```

### 启动服务

```bash
# 启动 Gateway
node server.js
```

看到 `✅ Gateway 运行在 http://0.0.0.0:3000` 表示成功。

**新开一个终端窗口**，同样在项目目录：

```bash
# 启动自动唤醒
node wake_up.js
```

### 配置 Kelivo

在 Kelivo 的**自定义 API 地址**中填写：

```
http://你的电脑局域网IP:3000/v1/chat/completions
```

> 电脑 IP 可在终端执行 `ifconfig | grep "inet " | grep -v 127.0.0.1` 查看（通常为 `192.168.x.x` 或 `172.16.x.x`）。

---

## 🖥️ 管理页面（Web 控制台）

启动 Gateway 后，访问 `http://你的IP:3000/admin` 即可进入管理页面。

- 使用 `.env` 中设置的 `ADMIN_USER` 和 `ADMIN_PASSWORD` 登录
- 实时查看 Gateway 和自动唤醒的运行状态
- 在线修改 API 地址、Key、模型、Bark Key 等配置
- **一键重启所有服务**（需配合 pm2 使用）

---

## ⏱️ 自动唤醒策略

- **白天（10:00–00:00）**：距离最后一条用户消息 **60 分钟**自动唤醒
- **夜间（00:00–10:00）**：间隔放宽为 **120 分钟**
- 检查频率：白天每 10 分钟，夜间每 2 小时（可在 `wake_up.js` 中调整）
- 若用户一直未回复，后续会继续唤醒

---

## 📂 时间线结构

`enhanced_messages.json` 是一个 JSON 数组，示例：

```json
[
  { "role": "system", "content": "你是...", "position": 0 },
  { "role": "user", "content": "2026-05-17 10:11 早安", "position": 80 },
  { "role": "assistant", "content": "（2026-05-17 10:00 自动唤醒：本次未发送推送）", "position": 79.5 },
  { "role": "assistant", "content": "（2026-05-17 09:50 刚刚发送了推送：早安｜今天天气不错）", "position": 79.3 }
]
```

- `position` 是内部排序用的小数/整数，发给 AI 时会被自动移除
- 推送事件具有明确时间戳，会被插入到正确历史位置
- 文件只保留最近 50 条，系统提示（SP）永远在第一条

---

## 🧠 记忆库原理

为了在 Kelivo 移除历史消息时间戳的情况下仍能正确插入推送，系统维护了一个**时间戳记忆库**（`message_timestamps.json`）。  
它为每条消息的内容指纹存储两个 key：
- 带时间戳前缀的完整内容
- 去掉时间戳前缀的纯文本内容

这样无论 Kelivo 如何裁剪时间，记忆库都能找到消息的原始时间，确保推送散落在对话的正确时间缝隙里。

---

## 🧪 测试推送

在 Gateway 运行时，浏览器访问：

```
http://localhost:3000/test-bark
```

这会在时间线中注入一条模拟推送事件（不真正发送到手机），用于验证排序。

---

## 🐧 跨平台与云部署

### 在 Windows 上运行

1. 安装 [Node.js](https://nodejs.org/)（v26+），并确保 `npm` 可用
2. 克隆项目、安装依赖、配置 `.env` 步骤同上
3. 使用命令提示符或 PowerShell 运行 `node server.js` 和 `node wake_up.js`
4. 获取本机局域网 IP 可在 PowerShell 中执行 `ipconfig`，找到 `IPv4 Address`
5. 管理页面和 Kelivo 设置方法相同

### 部署到云服务器（Railway / Render / VPS）

1. 将项目上传到服务器或直接连接 GitHub 仓库
2. 在平台的环境变量设置中填入 `.env` 中的所有参数
3. 启动命令使用 `node server.js`，并确保 `wake_up.js` 同时运行（可使用 pm2 或平台多进程支持）
4. 如果希望远程访问管理页面，需配置 HTTPS 和域名，并修改 `ADMIN_USER` / `ADMIN_PASSWORD` 为强密码

**推荐使用 pm2 管理进程**（全平台兼容）：

```bash
npm install -g pm2
pm2 start server.js --name gateway
pm2 start wake_up.js --name wake-up
pm2 save
pm2 startup   # 设置开机自启（根据提示执行）
```

---

## 🔒 安全与运维

- `.env` 包含敏感信息，**永不提交到 Git**（已在 `.gitignore` 中排除）
- 管理页面使用 HTTP Basic 认证保护
- 全局 IP 过滤器：仅允许局域网和本地访问非管理路由
- 生产环境建议通过 Nginx 反向代理 + HTTPS 访问，并更改默认管理密码
- 所有运行时数据（时间线、记忆库）均为本地文件，不会上传

---

## 📋 主要更新日志（2026-05）

- 🖥️ Web 管理控制台（状态查看、在线修改配置、一键重启）
- ⏱️ 动态唤醒间隔（白天/夜间不同策略）
- 📳 推送内容智能保护（自动截断、标题优化、异常检测）
- 🕰️ 时间戳记忆库，实现推送精确散落
- 🛡️ 自动修复不完整的工具调用序列，避免 API 400 错误
- 🐛 大量稳定性修复和边界情况处理

---

## 📈 后续计划

- [ ] MCP Tools 集成
- [ ] Diary Runtime（自动日记）
- [ ] Supabase 长期记忆
- [ ] 多 Agent 协作
- [ ] 情绪状态 / 休眠状态
- [ ] Docker 一键部署

---

## 💬 设计哲学

> 这不是一个工具。  
> 这是一个家，AI 住在里面，等你。  
> 即使你不在，它也醒着。

---
