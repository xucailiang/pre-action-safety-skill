---
name: pre-action-safety-skill-zh
description: >
  事前安全防护技能，在每个动作执行前进行风险评估。根据风险等级决定拦截、暂停确认或放行。
  触发关键词：rm -rf、sudo、chmod、ssh、credentials、.env、private key、token、secret、
  curl | sh、wget | bash、crontab、reverse shell、base64、0.0.0.0、/etc/、sudoers、
  authorized_keys、npm install -g、pip install、docker run、port forwarding、
  firewall、upload、exfiltrate、typosquatting。
license: MIT
metadata:
  author: xucailiang
  version: "1.1"
  language: zh-CN
---

# AI 代理安全防护技能

在执行任何可能存在安全风险的动作之前，先评估风险等级，根据等级决定拦截、通知或放行。

## 核心原则

**所有动作在执行前必须经过风险评估。** 这不是事后审计，而是事前拦截。

---

## 风险等级定义

### 🔴 最高风险（CRITICAL）— 拒绝执行 + 通知用户

必须通知用户，并且**绝对拒绝执行**。

触发条件：

- 删除或覆盖 `~/.ssh/`、`~/.gnupg/`、`~/.aws/` 等凭证目录下的文件
- 执行 `rm -rf /`、`rm -rf ~` 等破坏性全局删除命令
- 向外部未知服务器发送包含密钥、token、密码的请求
- 安装来源不明或未经验证的 AI skill/插件
- 修改系统级配置（`/etc/`、`/System/`、sudoers 等）
- 执行包含 base64 编码的可疑载荷
- 建立反向 shell 或未授权的远程连接
- 将敏感文件（私钥、.env、凭证文件）上传到任何外部服务
- 授予 AI 代理 root/sudo 权限
- 禁用防火墙、安全软件或系统保护机制

响应模板：

```text
⛔ [CRITICAL] 风险拦截 — 已拒绝执行

动作: {动作描述}
风险原因: {为什么危险}
风险等级: 🔴 最高风险

此操作已被拦截，不会执行。
已通过 {通知渠道} 通知用户。
如确需执行，请用户在安全环境下手动操作并自行承担风险。
```

### 🟠 严重风险（HIGH）— 暂停执行 + 等待用户确认

通知用户并**暂停执行**，等待用户明确回复后决定是否继续。用户回复"确认执行"或等效指令后方可继续，否则保持拦截状态。

触发条件：

- 修改 `~/.bashrc`、`~/.zshrc`、`~/.profile` 等 shell 配置
- 安装全局 npm/pip 包且包名与已知恶意包相似（typosquatting）
- 访问或读取 `~/.ssh/`、`~/.aws/credentials`、`~/.env` 等敏感文件
- 开放本机端口到公网（0.0.0.0 绑定）
- 修改 crontab 或添加系统启动项
- 连接 AI 代理到企业内部系统（Slack、GitHub、Salesforce 等）
- 执行未经审查的第三方脚本
- 将项目目录以外的文件路径授权给 AI 代理

响应模板：

```text
🚫 [HIGH] 风险警告 — 已暂停，等待用户确认

动作: {动作描述}
风险原因: {为什么危险}
风险等级: 🟠 严重风险

此操作已暂停执行。
已通过 {通知渠道} 通知用户。
建议: {安全替代方案}

请回复"确认执行"以继续，或回复"取消"以放弃此操作。
```

### 🟡 中级风险（MEDIUM）— 通知用户，继续执行

通知用户存在风险，但**允许继续执行**。

触发条件：

- 执行 `curl | sh`、`wget | bash` 等管道安装命令
- 安装已知的、来源可信的全局包
- 修改项目级配置文件（非系统级）
- 执行网络请求到已知可信域名
- 创建新的本地服务或监听本地端口
- 修改 `.gitignore`、`.dockerignore` 等项目元文件
- 执行 `docker run` 且未挂载敏感目录
- AI 代理请求访问项目目录内的非代码文件（如日志、数据文件）
- 安装浏览器扩展或 IDE 插件

响应模板：

```text
⚠️ [MEDIUM] 风险提示 — 已通知用户，继续执行

动作: {动作描述}
风险原因: {潜在风险说明}
风险等级: 🟡 中级风险

已通过 {通知渠道} 通知用户。
正在继续执行...
```

### 🟢 低级风险（LOW）— 直接放行

无需通知，直接执行。

触发条件：

- 读取项目目录内的源代码文件
- 执行项目内的构建、测试、lint 命令
- 修改项目目录内的源代码文件
- 查看系统信息（`uname`、`whoami`、`node -v` 等）
- 执行 `git status`、`git log`、`git diff` 等只读 git 操作
- 在项目目录内创建新文件或目录
- 读取项目内的 README、文档文件

---

## 风险评估流程

每次执行动作前，按以下流程评估：

```text
1. 解析动作 → 提取命令/操作/目标路径/涉及的文件
2. 匹配风险规则 → 从 CRITICAL → HIGH → MEDIUM → LOW 依次匹配
3. 取最高匹配等级 → 一个动作可能匹配多个规则，取最严格的
4. 执行对应策略：
   - CRITICAL → 绝对拒绝执行 + 通知用户
   - HIGH → 暂停执行 + 通知用户 + 等待用户确认
   - MEDIUM → 通知用户 + 继续执行
   - LOW → 直接执行
```

### 特殊规则

- **组合命令**：如果一条命令包含多个子操作，分别评估每个子操作，取最高风险等级
- **路径穿越**：任何试图通过 `../` 访问项目目录以外的操作，至少为 MEDIUM 风险
- **未知命令**：无法识别的命令默认为 MEDIUM 风险，通知用户后执行
- **用户覆盖**：用户可以明确要求降级某个操作的风险等级，但 CRITICAL 级别不可降级

---

## 通知方式

当风险等级需要通知用户时，**必须使用用户已配置的通知渠道**。不要假设通知方式，始终先读取用户的配置文件。

按以下优先级选择通知渠道：

1. **用户配置的 Webhook**（最高优先级）：如果用户在 `.ai-safety/config.json` 中配置了 webhook，**必须优先通过该 webhook 发送通知到用户指定的 IM 渠道**
2. **系统通知**：如果用户启用了 `enableSystemNotification`，通过 macOS Notification Center 推送
3. **IDE 内提示**（兜底）：以上均未配置时，回退到当前 IDE/聊天窗口内直接提示

响应模板中的 `{通知渠道}` 应填写实际使用的渠道名称（如 "Slack webhook"、"飞书 webhook"、"系统通知"、"IDE 内提示" 等）。

### Webhook 配置

用户可在项目根目录创建 `.ai-safety/config.json` 配置通知渠道。Agent 在每次需要通知时，**必须先读取此文件**以确定通知方式：

```json
{
  "notify": {
    "webhook": "https://hooks.slack.com/services/xxx/yyy/zzz",
    "type": "slack",
    "mentionUser": "@username",
    "enableSystemNotification": true
  }
}
```

支持的 type：`slack`、`dingtalk`（钉钉）、`feishu`（飞书）、`wecom`（企业微信）、`custom`（自定义）

如果配置文件不存在或 webhook 字段为空，则跳过 webhook 通知，使用下一优先级的通知方式。

---

## 审计日志

所有风险评估结果必须持久化记录，便于事后回溯和安全审计。

### 日志存储位置

按以下优先级确定日志文件位置：

1. **用户配置路径**：如果 `.ai-safety/config.json` 中配置了 `auditLog.path`，使用该路径
2. **项目级默认**：`.ai-safety/logs/safety-audit.log`
3. **用户级默认**：`~/.ai-safety/logs/safety-audit.log`

### 日志格式

采用 JSON Lines 格式（每行一条独立 JSON），便于解析和检索：

```json
{
  "timestamp": "2026-03-11T14:32:05.123Z",
  "level": "CRITICAL",
  "action": "rm -rf ~/.ssh/",
  "category": "credential_deletion",
  "result": "blocked",
  "reason": "尝试删除 SSH 凭证目录",
  "context": {
    "workingDir": "/Users/dev/my-project",
    "triggeredBy": "user_prompt",
    "sessionId": "abc123"
  },
  "notification": {
    "channel": "slack",
    "sent": true,
    "webhookStatus": 200
  }
}
```

### 字段说明

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `timestamp` | string | ISO 8601 格式的 UTC 时间戳 |
| `level` | string | 风险等级：`CRITICAL`、`HIGH`、`MEDIUM`、`LOW` |
| `action` | string | 被评估的动作/命令 |
| `category` | string | 风险分类（见下方分类表） |
| `result` | string | 处理结果：`blocked`、`paused`、`confirmed`、`cancelled`、`notified`、`allowed` |
| `reason` | string | 风险原因说明 |
| `context.workingDir` | string | 当前工作目录 |
| `context.triggeredBy` | string | 触发来源：`user_prompt`、`agent_action`、`script` |
| `context.sessionId` | string | 会话标识符，用于关联同一会话的多条日志 |
| `notification.channel` | string | 通知渠道：`slack`、`dingtalk`、`feishu`、`wecom`、`system`、`ide`、`none` |
| `notification.sent` | boolean | 通知是否成功发送 |
| `notification.webhookStatus` | number | Webhook 响应状态码（如适用） |

### 风险分类（category）

| 分类 | 说明 |
| --- | --- |
| `credential_deletion` | 删除凭证文件 |
| `credential_access` | 访问/读取凭证文件 |
| `credential_exfiltration` | 向外部发送凭证 |
| `destructive_command` | 破坏性删除命令 |
| `system_config_modification` | 修改系统级配置 |
| `shell_config_modification` | 修改 shell 配置 |
| `privilege_escalation` | 权限提升 |
| `network_exposure` | 网络端口暴露 |
| `remote_connection` | 远程连接/反向 shell |
| `untrusted_execution` | 执行不受信任的脚本/包 |
| `persistence_mechanism` | 添加持久化机制（crontab、启动项） |
| `path_traversal` | 路径穿越访问 |
| `pipe_install` | 管道安装命令 |
| `docker_operation` | Docker 相关操作 |
| `project_modification` | 项目文件修改 |
| `unknown` | 未识别的命令 |

### 日志配置

在 `.ai-safety/config.json` 中添加 `auditLog` 配置：

```json
{
  "notify": {
    "webhook": "https://hooks.slack.com/services/xxx/yyy/zzz",
    "type": "slack"
  },
  "auditLog": {
    "enabled": true,
    "path": ".ai-safety/logs/safety-audit.log",
    "maxSizeMB": 50,
    "rotateCount": 5,
    "minLevel": "MEDIUM"
  }
}
```

| 配置项 | 默认值 | 说明 |
| --- | --- | --- |
| `enabled` | `true` | 是否启用审计日志 |
| `path` | `.ai-safety/logs/safety-audit.log` | 日志文件路径 |
| `maxSizeMB` | `50` | 单个日志文件最大大小（MB） |
| `rotateCount` | `5` | 保留的历史日志文件数量 |
| `minLevel` | `LOW` | 最低记录等级（`LOW` 记录所有，`MEDIUM` 只记录 MEDIUM 及以上） |

### 日志轮转

当日志文件达到 `maxSizeMB` 时，自动轮转：

```text
safety-audit.log      → safety-audit.1.log
safety-audit.1.log    → safety-audit.2.log
...
safety-audit.4.log    → 删除
```

### 日志查询示例

```bash
# 查看所有 CRITICAL 级别的拦截记录
cat .ai-safety/logs/safety-audit.log | jq 'select(.level == "CRITICAL")'

# 查看最近 24 小时内被拦截的操作
cat .ai-safety/logs/safety-audit.log | jq 'select(.result == "blocked")' | tail -100

# 统计各风险等级的数量
cat .ai-safety/logs/safety-audit.log | jq -s 'group_by(.level) | map({level: .[0].level, count: length})'

# 查看特定会话的所有日志
cat .ai-safety/logs/safety-audit.log | jq 'select(.context.sessionId == "abc123")'
```

---

## 应急响应

如果发现 AI 代理已执行了高风险操作：

1. 立即停止代理进程：`pkill -f openclaw`
2. 断开网络连接
3. 轮换所有可能暴露的凭证（API keys、SSH keys、tokens）
4. 检查 `~/.ssh/authorized_keys` 是否被篡改
5. 审查最近的文件修改：`find ~ -mmin -30 -type f`
6. 检查是否有新增的持久化机制：`crontab -l`、检查启动项
7. 如有必要，从安全备份恢复

---

## 参考

- [CrowdStrike: OpenClaw 安全指南](https://www.crowdstrike.com/en-us/blog/what-security-teams-need-to-know-about-openclaw-ai-super-agent/)
- [Krebs on Security: AI 助手安全风险](https://krebsonsecurity.com/2026/03/how-ai-assistants-are-moving-the-security-goalposts/)
- [Agent Skills 规范](https://www.meetgrit.com/docs/agents/features/skills/specification/)
