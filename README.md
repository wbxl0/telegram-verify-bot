# telegram-verify-bot

一个基于 Cloudflare Workers 的 Telegram 消息转发机器人，集成了数学验证、反欺诈和用户管理功能。支持 KV 版本和 D1 版本。

> 💡 本项目参考 [NFD](https://github.com/LloydAsp/nfd)，在其基础上增加了多重验证和管理功能。

---

## ✨ 功能特性

- 🔐 **数学验证** - 用户必须通过验证才能使用机器人
  - 支持 加减乘除 四种运算
  - 6个选项按钮，2x3 布局
  - 5分钟内必须回答，超期自动失效
  - 最多尝试 10 次，超过则永久屏蔽
- ✅ **验证有效期** - 验证成功后 3 天内 无需重复验证
- 🚫 **反欺诈** - 内置欺诈用户数据库，自动检测和阻止
- 👤 **用户管理** - 支持屏蔽/解除屏蔽用户、白名单管理
- 📱 **消息转发** - 支持文本、图片、视频等多种媒体转发
- 📋 **白名单功能** - 白名单用户直接跳过验证和屏蔽检查
- ⏰ **通知提醒** - 可配置的定时提醒功能（默认一天一次）
- 🔒 **Webhook 加密** - 使用密钥验证确保消息来源
- 💾 **数据持久化** - 支持 KV 和 D1 两种存储方案

---

## 🚀 快速开始

### 前置条件

- Cloudflare 账户
- Telegram 账户

### 部署步骤

#### 1️⃣ 获取 Telegram 配置

- 从 @BotFather 获取 Bot Token，并执行 `/setjoingroups` 禁止 Bot 被添加到群组
- 从 @username_to_id_bot 获取你的用户 ID

#### 2️⃣ 生成 Webhook 密钥

- 访问 [UUID 生成器](https://www.uuidgenerator.net/) 生成一个随机 UUID 作为 `SECRET`

#### 3️⃣ 在 Cloudflare 创建 Worker

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com)
2. 进入 Workers & Pages → Create application → Start with Hello World!
3. 给 Worker 命名（如 telegram-verify-bot）
4. 点击 Deploy

#### 4️⃣ 配置环境变量

在 Worker 设置中，进入 Settings → Variables，添加以下环境变量：

| 变量名 | 说明 | 示例 |
|------|------|------|
| BOT_TOKEN | Telegram Bot Token | 123456:ABCDEFxyz... |
| BOT_SECRET | Webhook 密钥 | 550e8400-e29b-41d4-a716-446655440000 |
| ADMIN_UID | 你的 Telegram 用户 ID | 123456789 |

#### 5️⃣ 选择数据库方案

本项目提供两个版本，请根据需要选择：

##### 📦 方案 A：KV 版本（worker-kv.js）

**适合：** 小型应用，数据量不大（<10MB）

**配置步骤：**

1. 进入 Workers KV
2. 创建新的 KV 命名空间：`nfd`
3. 在 Worker 设置中，进入 Settings → Bindings → Add binding
   - Variable name： `nfd`
   - KV namespace： 选择刚创建的 `nfd`
4. 部署 worker-kv.js 代码

**KV 绑定配置：**

// 在代码中访问
let nfd = env.nfd;


**存储结构：**

whitelist-{userId} # 白名单标记
verify-{userId} # 当前验证码答案
verify-attempts-{userId} # 验证尝试次数
verified-{userId} # 验证成功标记（3天过期）
isblocked-{userId} # 屏蔽标记
msg-map-{messageId} # 消息映射关系
lastmsg-{userId} # 上次消息时间戳
whitelist-data # 白名单数据集合


##### 💾 方案 B：D1 版本（worker-d1.js）

**适合：** 中大型应用，需要结构化查询，数据持久化

**配置步骤：**

1. 创建 D1 数据库：

wrangler d1 create telegram-verify-bot


2. 获取数据库 ID，复制输出中的 `database_id`

3. 在 wrangler.toml 中配置：

[[d1_databases]]
binding = "LAN"
database_name = "telegram-verify-bot"
database_id = "你的数据库ID"


4. 部署 worker-d1.js 代码

5. 初始化数据库表：

https://你的worker.workers.dev/initDatabase


**D1 数据库表结构：**

-- 白名单用户
CREATE TABLE whitelist (
user_id TEXT PRIMARY KEY,
created_at INTEGER
);

-- 验证码状态
CREATE TABLE verification (
user_id TEXT PRIMARY KEY,
answer TEXT,
attempts INTEGER DEFAULT 0,
created_at INTEGER
);

-- 验证成功用户
CREATE TABLE verified_users (
user_id TEXT PRIMARY KEY,
expiry_time INTEGER
);

-- 屏蔽用户
CREATE TABLE blocked_users (
user_id TEXT PRIMARY KEY,
blocked_at INTEGER
);

-- 消息映射
CREATE TABLE message_mappings (
mapping_key TEXT PRIMARY KEY,
mapped_value TEXT,
created_at INTEGER
);

-- 消息频率限制
CREATE TABLE message_rates (
user_id TEXT PRIMARY KEY,
last_message_time INTEGER
);


**D1 优势：**

- ✅ 支持 SQL 查询，灵活度高
- ✅ 自动过期时间管理
- ✅ 数据结构清晰，易于维护
- ✅ 支持批量操作和事务
- ✅ 免费额度更高（读/写操作更多）

#### 6️⃣ 部署代码

1. 进入 Worker Edit code
2. 选择对应版本的代码：
   - KV 版本： 复制 worker-kv.js
   - D1 版本： 复制 worker-d1.js
3. 点击 Deploy

#### 7️⃣ 注册 Webhook

访问以下 URL 注册 webhook（替换 `xxx.workers.dev` 为你的 Worker 域名）：

https://xxx.workers.dev/registerWebhook


成功后将看到 `Ok` 响应。

---

## 📖 使用指南

### 普通用户

**初次使用流程：**

1. 给机器人发送 `/start` 查看欢迎消息
2. 回答数学验证题：
   - 看题目：12 + 34 = ?
   - 点击 6 个选项中的任意一个
   - 支持加减乘除四种运算
3. ✅ 验证成功后可正常使用
4. 发送的消息会被转发给机器人创建者

**⏱️ 时间限制：**

- 验证码有效期：5 分钟
- 验证成功有效期：3 天
- 验证失败上限：10 次（超过则永久屏蔽）

### 机器人创建者（管理员）

#### 基础操作

**回复用户消息流程：**

1. 用户发送消息 → 机器人转发给你
2. 长按转发的消息
3. 选择"回复"
4. 输入消息内容
5. 消息自动回复给原用户

#### 管理命令

**屏蔽/解除屏蔽用户：**

| 命令 | 功能 | 使用方式 |
|-----|------|--------|
| `/block` | 屏蔽用户 | 回复用户消息后发送 |
| `/unblock` | 解除屏蔽 | 回复用户消息后发送，或 `/unblock [UID]` |
| `/checkblock` | 查看屏蔽列表 | 直接发送（私聊）或话题内发送 |

**白名单管理：**

| 命令 | 功能 | 使用方式 |
|-----|------|--------|
| `/addwhite [UID]` | 添加白名单 | 直接指定 UID 或回复消息 |
| `/removewhite [UID]` | 移除白名单 | 直接指定 UID 或回复消息 |
| `/checkwhite [UID]` | 检查白名单状态 | 直接指定 UID 或回复消息 |
| `/listwhite` | 列出所有白名单 | 直接发送 |

**白名单用户特殊权限：**

- ✅ 直接跳过数学验证
- ✅ 无需再次验证（永久有效）
- ✅ 屏蔽列表检查被跳过
- ✅ 消息直接转发

#### 使用示例

**场景 1：屏蔽用户**

1. 长按用户转发的消息
2. 选择"回复"
3. 输入 `/block`
4. 结果：UID: 123456789 屏蔽成功

**场景 2：添加白名单**

1. 使用命令：`/addwhite 123456789`
2. 结果：UID: 123456789 已添加到白名单

**场景 3：查看白名单**

1. 使用命令：`/listwhite`
2. 结果：显示所有白名单用户列表

---

## ⚙️ 配置说明

### 时间参数详解

代码中的所有时间参数都可以自定义修改：

| 参数名 | 默认值 | 单位 | 含义 | 位置 |
|------|-------|------|------|------|
| VERIFICATION_TTL | 300 | 秒 | 验证码过期时间 | 代码顶部 |
| VERIFIED_TTL | 259200 | 秒 | 验证成功有效期 | 代码顶部 |
| MAX_VERIFY_ATTEMPTS | 10 | 次数 | 最大验证尝试次数 | 代码顶部 |
| NOTIFY_INTERVAL | 86400000 | 毫秒 | 通知间隔（24小时） | handleNotify() |

### 时间换算对照表

1 分钟 = 60 秒 = 60000 毫秒
5 分钟 = 300 秒 = 300000 毫秒
1 小时 = 3600 秒 = 3600000 毫秒
1 天 = 86400 秒 = 86400000 毫秒
3 天 = 259200 秒 = 259200000 毫秒
7 天 = 604800 秒 = 604800000 毫秒
30 天 = 2592000 秒 = 2592000000 毫秒


### 修改验证码过期时间

编辑代码顶部的常量：

// 改为 10 分钟过期
const VERIFICATION_TTL = 600;

// 改为 2 小时过期
const VERIFICATION_TTL = 7200;


### 修改验证成功有效期

编辑代码顶部的常量：

// 改为 7 天有效期
const VERIFIED_TTL = 604800;

// 改为 1 天有效期
const VERIFIED_TTL = 86400;


### 修改最大验证尝试次数

编辑代码顶部的常量：

// 改为 5 次尝试
const MAX_VERIFY_ATTEMPTS = 5;

// 改为 20 次尝试
const MAX_VERIFY_ATTEMPTS = 20;


### 启用通知功能

默认关闭，要启用请改为：

const enable_notification = true;


启用后，每次用户发送消息超过 24 小时后会触发一次通知。

---

## 🔍 反欺诈数据库

### 数据源

- **文件路径：** `fraud.db`
- **格式：** 每行一个 UID，如：

123456789
987654321
111111111


- **更新方式：** 通过 PR 或 Issue 补充

### 工作原理

- 当已验证用户在欺诈数据库中时，消息不会被转发
- 管理员会收到警告消息：⚠️ 检测到诈骗人员 UID: xxxxx
- 有助于防止骗子使用机器人

---

## 🛠️ 常见问题

**Q: 验证码过期了怎么办？**

A: 系统会自动生成新的验证码。只需在5分钟内点击答案即可。超过5分钟会自动清除验证码，下次发消息时需要重新验证。

**Q: 为什么出现 10 个数学题而不是 4 个？**

A: 本版本改进为 6 个选项（2行×3列布局），提高了安全性和用户体验。用户从中选择正确答案。

**Q: 屏蔽用户后他能做什么？**

A: 被屏蔽用户给机器人发消息时会收到 `You are blocked` 提示，无法继续使用。

**Q: 白名单用户有什么优势？**

A:
- ✅ 无需进行数学验证
- ✅ 消息永久有效（无需重新验证）
- ✅ 直接转发消息，无任何限制

**Q: 消息转发支持哪些类型？**

A: 支持文本、图片、视频、文件等 Telegram 支持的所有媒体类型。

**Q: 如何自定义欢迎消息？**

A: 编辑代码中 `/start` 命令的 text 字段即可：

if (message.text === '/start') {
return sendMessage({
chat_id: message.chat.id,
text: '你的自定义欢迎消息内容' // ← 改这里
});
}


**Q: KV 和 D1 哪个更好？**

A:

| 对比项 | KV | D1 |
|------|----|----|
| 学习难度 | 简单 | 中等 |
| 查询能力 | 简单键值 | SQL查询 |
| 数据量 | <10MB | 无限制 |
| 性能 | 快速 | 快速 |
| 价格 | 免费额度100K请求/月 | 免费额度更高 |
| 推荐用途 | 小型应用 | 中大型应用 |

**Q: 如何修改验证有效期？**

A: 编辑代码顶部的常量：

// D1 版本或 KV 版本
const VERIFIED_TTL = 259200; // 改为你需要的秒数


然后重新部署。

**Q: 如何查看存储的数据？**

A:
- **KV 版本：** 进入 Cloudflare Dashboard → Workers KV → 查看值
- **D1 版本：** 进入 Cloudflare Dashboard → D1 Databases → 使用查询工具

**Q: 性能如何？**

A:
- 响应时间：<100ms
- 支持用户数：KV 版本 100+ 用户，D1 版本 1000+ 用户
- 消息延迟：<1 秒

---

## 📝 项目结构

telegram-verify-bot/
├── README.md # 项目说明文档
├── worker-kv.js # KV 版本（使用 Cloudflare KV）
├── worker-d1.js # D1 版本（使用 Cloudflare D1 数据库）
├── wrangler.toml # Wrangler 配置文件
└── fraud.db # 欺诈数据库（行分隔的 UID 列表）


### 版本对比

| 文件 | 适用场景 | 特点 |
|-----|--------|------|
| worker-kv.js | 小型应用、快速部署 | 无需初始化，开箱即用 |
| worker-d1.js | 中大型应用、需要查询 | 需要初始化表，功能完整 |

---

## 🔄 版本迭代记录

### v2.0.0（当前版本）

**✅ 新增功能：**

- 新增 6 选项数学验证（2x3 布局）
- 新增 D1 数据库版本，支持 SQL 查询
- 新增 白名单管理命令（`/addwhite`, `/removewhite`, `/checkwhite`, `/listwhite`）
- 新增 验证码 5 分钟自动过期
- 新增 验证成功 3 天有效期
- 新增 验证尝试次数限制（最多 10 次）

**📊 改进：**

- 优化数据存储结构（支持 KV 和 D1）
- 增强用户隐私保护
- 改进错误提示信息
- 完善文档说明

---

## 🤝 贡献

欢迎通过以下方式贡献：

- 补充欺诈数据： 提交 PR 更新 `fraud.db`
- 功能改进： 提交 Issue 讨论功能建议
- Bug 反馈： 报告发现的问题，带上错误日志

### 提交欺诈信息时的注意事项

- 请提供可靠的消息出处
- 多个 UID 请分行提交
- 确认 UID 无误后再提交

---

## 📜 许可证

本项目参考 [NFD](https://github.com/LloydAsp/nfd)，致谢原项目作者。

---

## 🔗 相关资源

- [Cloudflare Workers 文档](https://developers.cloudflare.com/workers/)
- [Cloudflare D1 数据库文档](https://developers.cloudflare.com/d1/)
- [Cloudflare KV 文档](https://developers.cloudflare.com/workers/runtime-apis/kv/)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [NFD 原项目](https://github.com/LloydAsp/nfd)

---

## 💬 支持

有问题或建议？

- 📮 提交 Issue
- 💭 开启讨论区
- 🔗 提交 Pull Request

⭐ 如果对你有帮助，请给个 Star！
