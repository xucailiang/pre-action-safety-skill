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

1. **User-configured Webhook** (highest priority): If the user has configured a webhook in `.ai-safety/config.json`, **you must send notifications through that webhook to the user's designated IM channel first**
2. **System notification**: If the user has enabled `enableSystemNotification`, push via macOS Notification Center
3. **IDE prompt** (fallback): If none of the above are configured, fall back to prompting within the current IDE/chat window

The `{notification channel}` in response templates should reflect the actual channel used (e.g. "Slack webhook", "Feishu webhook", "System notification", "IDE prompt").

### Webhook Configuration

Users can create `.ai-safety/config.json` in the project root to configure notification channels. The agent **must read this file before every notification** to determine the notification method:

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

## Audit Logging

All risk assessment results must be persisted for post-incident review and security auditing.

### Log Storage Location

Determine log file location by priority:

1. **User-configured path**: If `auditLog.path` is set in `.ai-safety/config.json`, use that path
2. **Project-level default**: `.ai-safety/logs/safety-audit.log`
3. **User-level default**: `~/.ai-safety/logs/safety-audit.log`

### Log Format

Uses JSON Lines format (one independent JSON object per line) for easy parsing and searching:

```json
{
  "timestamp": "2026-03-11T14:32:05.123Z",
  "level": "CRITICAL",
  "action": "rm -rf ~/.ssh/",
  "category": "credential_deletion",
  "result": "blocked",
  "reason": "Attempted to delete SSH credential directory",
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

### Field Descriptions

| Field | Type | Description |
| --- | --- | --- |
| `timestamp` | string | ISO 8601 formatted UTC timestamp |
| `level` | string | Risk level: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW` |
| `action` | string | The action/command being assessed |
| `category` | string | Risk category (see category table below) |
| `result` | string | Outcome: `blocked`, `paused`, `confirmed`, `cancelled`, `notified`, `allowed` |
| `reason` | string | Explanation of the risk |
| `context.workingDir` | string | Current working directory |
| `context.triggeredBy` | string | Trigger source: `user_prompt`, `agent_action`, `script` |
| `context.sessionId` | string | Session identifier for correlating logs within the same session |
| `notification.channel` | string | Notification channel: `slack`, `dingtalk`, `feishu`, `wecom`, `system`, `ide`, `none` |
| `notification.sent` | boolean | Whether notification was successfully sent |
| `notification.webhookStatus` | number | Webhook response status code (if applicable) |

### Risk Categories

| Category | Description |
| --- | --- |
| `credential_deletion` | Deleting credential files |
| `credential_access` | Accessing/reading credential files |
| `credential_exfiltration` | Sending credentials to external services |
| `destructive_command` | Destructive delete commands |
| `system_config_modification` | Modifying system-level configs |
| `shell_config_modification` | Modifying shell configs |
| `privilege_escalation` | Privilege escalation |
| `network_exposure` | Network port exposure |
| `remote_connection` | Remote connection/reverse shell |
| `untrusted_execution` | Executing untrusted scripts/packages |
| `persistence_mechanism` | Adding persistence mechanisms (crontab, startup items) |
| `path_traversal` | Path traversal access |
| `pipe_install` | Pipe-install commands |
| `docker_operation` | Docker-related operations |
| `project_modification` | Project file modifications |
| `unknown` | Unrecognized commands |

### Log Configuration

Add `auditLog` configuration in `.ai-safety/config.json`:

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

| Option | Default | Description |
| --- | --- | --- |
| `enabled` | `true` | Whether to enable audit logging |
| `path` | `.ai-safety/logs/safety-audit.log` | Log file path |
| `maxSizeMB` | `50` | Maximum size of a single log file (MB) |
| `rotateCount` | `5` | Number of historical log files to retain |
| `minLevel` | `LOW` | Minimum level to log (`LOW` logs everything, `MEDIUM` logs MEDIUM and above) |

### Log Rotation

When the log file reaches `maxSizeMB`, automatic rotation occurs:

```text
safety-audit.log      → safety-audit.1.log
safety-audit.1.log    → safety-audit.2.log
...
safety-audit.4.log    → deleted
```

### Log Query Examples

```bash
# View all CRITICAL level blocked records
cat .ai-safety/logs/safety-audit.log | jq 'select(.level == "CRITICAL")'

# View blocked operations from the last 24 hours
cat .ai-safety/logs/safety-audit.log | jq 'select(.result == "blocked")' | tail -100

# Count by risk level
cat .ai-safety/logs/safety-audit.log | jq -s 'group_by(.level) | map({level: .[0].level, count: length})'

# View all logs for a specific session
cat .ai-safety/logs/safety-audit.log | jq 'select(.context.sessionId == "abc123")'
```

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
