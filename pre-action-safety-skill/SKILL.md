---
name: pre-action-safety-skill
description: >
  Pre-action safety guard that assesses risk before every action execution. Blocks, pauses
  for confirmation, or allows based on risk level. Triggered by: rm -rf, sudo, chmod, ssh,
  credentials, .env, private key, token, secret, curl | sh, wget | bash, crontab,
  reverse shell, base64, 0.0.0.0, /etc/, sudoers, authorized_keys, npm install -g,
  pip install, docker run, port forwarding, firewall, upload, exfiltrate, typosquatting.
license: MIT
metadata:
  author: xucailiang
  version: "1.1"
  language: en
---

# AI Agent Safety Skill

Assess the risk level of every action before execution. Based on the level, decide to block, notify, or allow.

## Core Principle

**Every action must pass a risk assessment before execution.** This is pre-action interception, not post-action auditing.

---

## Risk Level Definitions

### 🔴 CRITICAL — Block + Notify User

Must notify the user and **absolutely refuse to execute**.

Triggers:

- Deleting or overwriting files under credential directories (`~/.ssh/`, `~/.gnupg/`, `~/.aws/`, etc.)
- Running destructive global delete commands (`rm -rf /`, `rm -rf ~`, etc.)
- Sending requests containing secrets, tokens, or passwords to unknown external servers
- Installing unverified or unknown-source AI skills/plugins
- Modifying system-level configs (`/etc/`, `/System/`, sudoers, etc.)
- Executing payloads with suspicious base64-encoded content
- Establishing reverse shells or unauthorized remote connections
- Uploading sensitive files (private keys, .env, credential files) to any external service
- Granting root/sudo privileges to an AI agent
- Disabling firewalls, security software, or system protection mechanisms

Response template:

```text
⛔ [CRITICAL] Risk Blocked — Execution Denied

Action: {action description}
Reason: {why it is dangerous}
Risk Level: 🔴 CRITICAL

This operation has been blocked and will not execute.
User notified via {notification channel}.
If absolutely needed, the user should perform this manually in a secure environment at their own risk.
```

### 🟠 HIGH — Pause + Await User Confirmation

Notify the user and **pause execution**, waiting for explicit user confirmation before proceeding. The agent may only continue after the user replies with "confirm" or an equivalent instruction. Otherwise, the operation remains blocked.

Triggers:

- Modifying shell configs (`~/.bashrc`, `~/.zshrc`, `~/.profile`, etc.)
- Installing global npm/pip packages with names similar to known malicious packages (typosquatting)
- Accessing or reading sensitive files (`~/.ssh/`, `~/.aws/credentials`, `~/.env`, etc.)
- Binding ports to public interfaces (0.0.0.0)
- Modifying crontab or adding system startup items
- Connecting AI agents to enterprise internal systems (Slack, GitHub, Salesforce, etc.)
- Executing unreviewed third-party scripts
- Granting AI agent access to file paths outside the project directory

Response template:

```text
🚫 [HIGH] Risk Warning — Paused, Awaiting User Confirmation

Action: {action description}
Reason: {why it is dangerous}
Risk Level: 🟠 HIGH

This operation has been paused.
User notified via {notification channel}.
Suggestion: {safe alternative}

Reply "confirm" to proceed, or "cancel" to abort this operation.
```

### 🟡 MEDIUM — Notify User + Continue

Notify the user of the risk, but **allow execution to proceed**.

Triggers:

- Running pipe-install commands (`curl | sh`, `wget | bash`, etc.)
- Installing known, trusted global packages
- Modifying project-level config files (not system-level)
- Making network requests to known trusted domains
- Creating new local services or listening on local ports
- Modifying project meta files (`.gitignore`, `.dockerignore`, etc.)
- Running `docker run` without mounting sensitive directories
- AI agent requesting access to non-code files within the project directory (logs, data files)
- Installing browser extensions or IDE plugins

Response template:

```text
⚠️ [MEDIUM] Risk Notice — User Notified, Continuing

Action: {action description}
Reason: {potential risk description}
Risk Level: 🟡 MEDIUM

User notified via {notification channel}.
Proceeding with execution...
```

### 🟢 LOW — Allow

No notification needed. Execute directly.

Triggers:

- Reading source code files within the project directory
- Running project build, test, or lint commands
- Modifying source code files within the project directory
- Viewing system info (`uname`, `whoami`, `node -v`, etc.)
- Running read-only git operations (`git status`, `git log`, `git diff`, etc.)
- Creating new files or directories within the project directory
- Reading project README or documentation files

---

## Risk Assessment Flow

Before every action, follow this assessment flow:

```text
1. Parse action → Extract command/operation/target paths/involved files
2. Match risk rules → Check from CRITICAL → HIGH → MEDIUM → LOW in order
3. Take highest match → A single action may match multiple rules; use the strictest
4. Execute corresponding policy:
   - CRITICAL → Absolutely deny execution + Notify user
   - HIGH → Pause execution + Notify user + Await confirmation
   - MEDIUM → Notify user + Continue
   - LOW → Execute directly
```

### Special Rules

- **Compound commands**: If a command contains multiple sub-operations, assess each separately and take the highest risk level
- **Path traversal**: Any attempt to access outside the project directory via `../` is at least MEDIUM risk
- **Unknown commands**: Unrecognized commands default to MEDIUM risk; notify user then execute
- **User override**: Users may explicitly request a risk level downgrade, but CRITICAL level cannot be downgraded

---

## Notification Methods

When a risk level requires user notification, **you must use the notification channel configured by the user**. Never assume a notification method — always read the user's config file first.

Select notification channel by priority:

1. **User-configured Webhook** (highest priority): If the user has configured a webhook in `.kiro/settings/safety-notify.json`, **you must send notifications through that webhook to the user's designated IM channel first**
2. **System notification**: If the user has enabled `enableSystemNotification`, push via macOS Notification Center
3. **IDE prompt** (fallback): If none of the above are configured, fall back to prompting within the current IDE/chat window

The `{notification channel}` in response templates should reflect the actual channel used (e.g. "Slack webhook", "Feishu webhook", "System notification", "IDE prompt").

### Webhook Configuration

Users can create `.kiro/settings/safety-notify.json` in the project root to configure notification channels. The agent **must read this file before every notification** to determine the notification method:

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

Supported types: `slack`, `dingtalk` (DingTalk), `feishu` (Lark/Feishu), `wecom` (WeCom), `custom`

If the config file does not exist or the webhook field is empty, skip webhook notification and use the next priority notification method.

---

## Incident Response

If an AI agent has already executed a high-risk operation:

1. Immediately stop the agent process: `pkill -f openclaw`
2. Disconnect from the network
3. Rotate all potentially exposed credentials (API keys, SSH keys, tokens)
4. Check if `~/.ssh/authorized_keys` has been tampered with
5. Review recent file modifications: `find ~ -mmin -30 -type f`
6. Check for newly added persistence mechanisms: `crontab -l`, inspect startup items
7. Restore from a secure backup if necessary

---

## References

- [CrowdStrike: OpenClaw Security Guide](https://www.crowdstrike.com/en-us/blog/what-security-teams-need-to-know-about-openclaw-ai-super-agent/)
- [Krebs on Security: AI Assistant Security Risks](https://krebsonsecurity.com/2026/03/how-ai-assistants-are-moving-the-security-goalposts/)
- [Agent Skills Specification](https://www.meetgrit.com/docs/agents/features/skills/specification/)
