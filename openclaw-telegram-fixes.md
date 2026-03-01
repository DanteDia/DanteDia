# OpenClaw Telegram Notification Fixes

## Issues Identified

1. **Message Truncation** - Telegram messages from agents getting cut off mid-sentence
2. **Verbose Review Notifications** - Receiving noisy internal metadata instead of clean output
3. **Deprecation Warning** - Pango's model or config flagging deprecated items

---

## Priority 0: Update OpenClaw First

Many of these are **known bugs** that were fixed in recent releases. Before anything
else, update on your VPS:

```bash
npm install -g openclaw@latest
openclaw restart
```

Key fixes in recent versions:
- **PR #15360**: Subagent announce was leaking internal prompt, performance metadata
  (tokens, runtime), and system identifiers to user chat. Fixed by moving stats to
  internal logging.
- **PR #24069**: Conversation metadata and cron announce payloads were rendered as
  visible messages. Fixed by sanitizing system framing at the delivery boundary.
- **Issue #10069**: `--post-mode full` was removed in v2026.2.3 — announce now only
  posts a **short summary**, not full output. This is likely why your messages appear
  truncated.

---

## Fix 1: Message Truncation

### Root Causes

There are multiple known causes:

**A. Telegram's hard 4096-character API limit**
OpenClaw's `textChunkLimit` (default 4000) controls chunking. Messages longer than
this get split into multiple Telegram messages.

**B. Announce delivery only posts short summaries (Issue #10069)**
Since v2026.2.3, `--post-mode full` was removed from isolated cron jobs. The announce
delivery now only posts a **short summary** to the main session — NOT the full agent
output. This is likely the main reason your health check results appear truncated.

**C. Announce delivery timeout (Issue #14540)**
Cron announce delivery has a hardcoded 60-second timeout. If an isolated cron job
takes >60s to generate output, the announce silently fails.

**D. Streaming mode truncation (Issue #30434)**
When `streaming: "partial"` is enabled, messages exceeding ~4000 chars cause the
stream to halt and remaining content trickles in line-by-line.

### Config Fix

Edit `~/.openclaw/openclaw.json` on your VPS:

```json5
{
  channels: {
    telegram: {
      textChunkLimit: 4000,    // Keep below Telegram's 4096 hard limit
      chunkMode: "newline",    // Split on paragraph boundaries, not mid-sentence
      streaming: "off"         // Avoid partial streaming truncation bug
    }
  }
}
```

### Workaround for Short Announce Summaries

Since `--post-mode full` was removed, have your agents use the **message tool**
directly to send results to Telegram instead of relying on announce delivery:

Add to your agent system prompts:
```
When completing a task, use the message tool to send your full results directly
to the Telegram chat. Do not rely on the announce system for delivering output.
```

---

## Fix 2: Verbose Review Notifications ("output looks substantive", metadata leak)

### Root Cause: Known Bugs

The messages like:
```
📋 Task submitted FOR REVIEW by pango (google/gemini-3.1-pro-preview):
Review task "Scheduled Health Check: ..."
✅ output looks substantive
Reason: Output contains work artifacts...
```

This is a **known announce information leak bug**:

- **Issue #6669**: Subagent announce leaked internal prompt, performance metadata
  (tokens, runtime), and system identifiers (session keys, transcript paths) to
  user chat. The "output looks substantive" text is part of the internal announce
  evaluation logic that should NOT be visible to you.
- **Issue #23971**: Conversation metadata and cron announce payloads rendered as
  visible messages in chat.

### Fix: Update OpenClaw (Primary)

```bash
npm install -g openclaw@latest
openclaw restart
```

PRs #15360 and #24069 fixed these metadata leaks. If you're on an older version,
updating should resolve most of the verbose noise.

### Fix: Update Agent System Prompts (Secondary)

Even after updating, refine pango's system prompt for cleaner review output.

In your agent config for pango (`~/.openclaw/agents/pango/` or `openclaw.json`):

```json5
{
  agents: {
    pango: {
      systemPrompt: "... [your existing prompt] ...\n\nIMPORTANT: When reviewing and approving tasks, be concise:\n- For approvals: '✅ Approved: [one-line summary]'\n- For rejections: '❌ Rejected: [one-line reason]'\n- Do NOT include internal evaluation steps or metadata\n- Do NOT echo the full task description back\n- Only surface actual findings or a brief summary"
    }
  }
}
```

For charles, add to system prompt:
```
When running scheduled health checks, output ONLY results:
- Healthy: '✅ Health Check OK - All systems nominal'
- Degraded: '⚠️ DEGRADED - [specific issue and recommended action]'
Do NOT include scheduling metadata or task descriptions.
```

### Fix: Use ANNOUNCE_SKIP for Routine Approvals

Add to pango's system prompt:
```
For routine approvals with no issues found, reply with ANNOUNCE_SKIP during
the announce step. Only announce to Telegram if there are critical findings.
```

### Fix: Customize HEARTBEAT.md

Create/edit `~/.openclaw/workspace/HEARTBEAT.md` (or per-agent workspace):
```markdown
## Heartbeat Instructions
- Send ONLY actual findings or actionable results
- Do NOT include model names, token counts, session keys, or internal metadata
- Do NOT include labels like "FOR REVIEW" or "output looks substantive"
- If nothing needs attention, reply HEARTBEAT_OK
```

---

## Fix 3: Deprecation Warning

### streamMode Config (Deprecated since v2026.2.21)

The legacy `streamMode` config key was deprecated. If you have this in your config:

```json5
// OLD — deprecated, causes warning:
channels: { telegram: { streamMode: "partial" } }
```

Replace with:
```json5
// NEW:
channels: { telegram: { streaming: "off" } }
// (use "off" to also avoid the streaming truncation bug #30434)
```

### Gemini Model Deprecation

Your pango agent uses `google/gemini-3.1-pro-preview`. Note:
- **Gemini 3 Pro Preview** (the older one) is deprecated by Google, shutting down
  March 9, 2026.
- **Gemini 3.1 Pro Preview** is the replacement but may not be in OpenClaw's
  built-in model catalog yet (Issues #21176, #22323).

Check available models:
```bash
openclaw models list | grep gemini
```

If `gemini-3.1-pro-preview` shows as `missing`, add it manually:
```json5
{
  models: {
    providers: {
      google: {
        models: {
          "gemini-3.1-pro-preview": {
            contextWindow: 1048576,
            maxTokens: 65536
          }
        }
      }
    }
  }
}
```

Or switch pango to a stable model like `google/gemini-2.5-pro` until the catalog
is updated.

---

## Quick Checklist for VPS

```bash
# 1. SSH into your VPS
ssh your-vps

# 2. Update OpenClaw (fixes announce metadata leaks)
npm install -g openclaw@latest

# 3. Edit config
nano ~/.openclaw/openclaw.json

# 4. Run diagnostics
openclaw doctor --fix

# 5. Check available models
openclaw models list | grep gemini

# 6. Restart gateway
openclaw gateway restart
```

### Minimal openclaw.json changes:

```json5
{
  channels: {
    telegram: {
      textChunkLimit: 4000,
      chunkMode: "newline",
      streaming: "off"           // Replace deprecated streamMode + avoid bug
    }
  }
}
```

Plus update pango + charles agent system prompts for concise output.

---

## Related GitHub Issues

- [#6669 - Subagent announce leaks internal prompt and stats](https://github.com/openclaw/openclaw/issues/6669)
- [#10069 - --post-mode full removed, announce only posts short summary](https://github.com/openclaw/openclaw/issues/10069)
- [#13911 - Feature: per-channel announce suppression](https://github.com/openclaw/openclaw/issues/13911)
- [#14540 - Announce delivery timeout hardcoded to 60s](https://github.com/openclaw/openclaw/issues/14540)
- [#21176 - Add gemini-3.1-pro-preview to google-gemini-cli catalog](https://github.com/openclaw/openclaw/issues/21176)
- [#23971 - Metadata and cron announce payloads visible in chat](https://github.com/openclaw/openclaw/issues/23971)
- [#30434 - Telegram partial streaming auto-split truncation](https://github.com/openclaw/openclaw/issues/30434)
