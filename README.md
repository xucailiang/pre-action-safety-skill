# Pre-Action Safety Skill

[English](#english) | [中文](#中文)

---

## English

A pre-action safety guard skill for AI coding agents. It intercepts and evaluates every action before execution, classifying risk into four levels and deciding whether to block, notify, or allow.

### Risk Levels

| Level | Action | Example |
| --- | --- | --- |
| 🔴 CRITICAL | Block (absolute) + Notify | Deleting credential files, uploading secrets, granting root access |
| 🟠 HIGH | Pause + Await Confirmation | Modifying shell configs, reading sensitive files, opening ports to public |
| 🟡 MEDIUM | Notify + Continue | Pipe-install commands, installing trusted global packages, modifying project configs |
| 🟢 LOW | Allow | Reading/writing project source code, running build/test/lint, read-only git ops |

### Notification

The skill uses the notification channel configured by the user in `.ai-safety/config.json`. Supports Slack, DingTalk, Feishu, WeCom, and custom webhooks. Falls back to system notification or IDE prompt if not configured.

### Installation

Copy the skill folder to your AI agent's configuration directory. The exact path depends on your agent platform:

```bash
# Example: project-level
cp -r pre-action-safety-skill .ai-safety/skills/

# Example: user-level
cp -r pre-action-safety-skill ~/.ai-safety/skills/
```

For the Chinese version:

```bash
cp -r pre-action-safety-skill-zh .ai-safety/skills/
```

### Structure

```text
pre-action-safety-skill/
└── SKILL.md              # English version

pre-action-safety-skill-zh/
└── SKILL.md              # 中文版本
```

---

## 中文

一个面向 AI 编程代理的事前安全防护技能。在每个动作执行前进行风险评估，按四个等级决定拦截、通知或放行。

### 风险等级

| 等级 | 处理方式 | 示例 |
| --- | --- | --- |
| 🔴 最高风险 | 绝对拒绝 + 通知 | 删除凭证文件、上传密钥、授予 root 权限 |
| 🟠 严重风险 | 暂停 + 等待确认 | 修改 shell 配置、读取敏感文件、开放公网端口 |
| 🟡 中级风险 | 通知 + 继续 | 管道安装命令、安装可信全局包、修改项目配置 |
| 🟢 低级风险 | 直接放行 | 读写项目源码、运行构建/测试/lint、只读 git 操作 |

### 通知方式

技能会使用用户在 `.ai-safety/config.json` 中配置的通知渠道，支持 Slack、钉钉、飞书、企业微信和自定义 webhook。未配置时回退到系统通知或 IDE 内提示。

### 安装

将技能文件夹复制到你的 AI agent 配置目录。具体路径取决于你使用的 agent 平台：

```bash
# 示例：项目级别
cp -r pre-action-safety-skill-zh .ai-safety/skills/

# 示例：用户级别
cp -r pre-action-safety-skill-zh ~/.ai-safety/skills/
```

### 许可证

MIT
