---
summary: "Guide to running autonomous features with local models like Qwen3 30B - what works, what doesn't, and how to configure it"
read_when:
  - Setting up OpenClaw with local models for autonomous operation
  - Understanding which features work well locally vs require cloud
  - Configuring lightweight autonomous tasks without cloud APIs
title: "Local Autonomous Setup"
---

# Local Autonomous Setup 🏠

This guide explains which **autonomous features work well with local models** (like Qwen3 30B, Llama, Mistral, etc.) and how to configure OpenClaw for local-only autonomous operation.

<Note>
**Key Insight**: Many autonomous features are **infrastructure-based** (scheduling, webhooks, parsing) and work perfectly with local models. Only complex reasoning tasks truly benefit from cloud APIs.
</Note>

## What Works Locally vs Requires Cloud

### Infrastructure Layer (Works Perfectly) 🟢

These features are **scheduling/routing infrastructure** with minimal AI requirements:

| Feature | AI Complexity | Local Model Ready? | Notes |
|---------|---------------|-------------------|-------|
| **Cron scheduling** | None | ✅ Perfect | Pure timer-based, no AI needed |
| **Email webhooks** | None | ✅ Perfect | Gmail Pub/Sub is just HTTP parsing |
| **Hook triggers** | None | ✅ Perfect | Event detection is infrastructure |
| **Message routing** | None | ✅ Perfect | Channel multiplexing works offline |
| **Session management** | Low | ✅ Excellent | Basic CRUD operations |
| **OAuth refresh** | None | ✅ Perfect | Token management is infrastructure |

**Takeaway**: The entire autonomous **plumbing** works without cloud APIs.

### Simple Autonomous Tasks (Works Well) 🟢

These require **basic AI** capabilities that local models handle fine:

| Task | Complexity | Qwen3 30B | Llama 3.2 8B | Notes |
|------|-----------|-----------|--------------|-------|
| **Email summaries** | Low | ✅ Excellent | ⚠️ Good | Single-pass summarization |
| **Email categorization** | Low | ✅ Excellent | ✅ Excellent | Pattern matching |
| **Log parsing & alerts** | Low | ✅ Excellent | ✅ Excellent | Regex + simple rules |
| **Scheduled reports** | Low-Medium | ✅ Excellent | ⚠️ Good | Template filling |
| **Data extraction** | Medium | ✅ Good | ⚠️ Fair | Structured output |
| **Simple multi-step** | Medium | ⚠️ Good | ❌ Poor | 2-3 tool calls max |

**Configuration tip**: Use `thinking: "off"` for these tasks - reasoning overhead isn't needed.

### Complex Tasks (Better with Cloud) 🔴

These require **advanced reasoning** where cloud models excel:

| Task | Why Cloud is Better | Local Model Limitation |
|------|---------------------|----------------------|
| **Code generation** | Long context + syntax | Qwen 30B: Possible but lower quality |
| **Multi-step planning** | Extended reasoning chains | Context window limits |
| **Complex debugging** | Deep code analysis | May hallucinate or give up |
| **Orchestrating subagents** | Meta-reasoning about delegation | Decision quality varies |
| **Creative writing** | Nuanced style/tone | Generic responses |

## Python Portability ❌

**Short answer**: OpenClaw is **100% TypeScript/Node.js** - not portable to Python.

**Why?**
- Core written in TypeScript using async/await patterns
- All hooks use TypeScript handlers (`.ts` files)
- Agent SDK and plugin system are TypeScript-based
- Heavy use of Node.js-specific libraries

**Python files in repo**: Only found in some skill scripts (`skills/*/scripts/*.py`) - these are helper utilities, not core code.

### Python Alternatives

If you want Python-based autonomous features:

**Option 1: Wrapper Approach**
```python
# Call OpenClaw's HTTP API from Python
import requests

# Trigger cron job via Gateway API
requests.post("http://localhost:18789/api/cron/add", json={
    "name": "Python-scheduled task",
    "schedule": {"kind": "every", "everyMs": 3600000},
    "payload": {"kind": "agentTurn", "message": "Check status"}
})
```

**Option 2: Ollama + OpenClaw**
```bash
# Run Qwen locally via Ollama
ollama run qwen2.5:30b

# Configure OpenClaw to use Ollama
```

```json5
{
  agents: {
    defaults: {
      model: "openai/qwen2.5-30b",
      provider: {
        openai: {
          baseURL: "http://localhost:11434/v1",
          apiKey: "ollama"
        }
      }
    }
  }
}
```

**Option 3: Build Python Client**
- Write Python scripts that POST to OpenClaw Gateway
- Manage Qwen separately (vLLM, Ollama, LM Studio)
- Use OpenClaw as the "autonomous orchestration layer"

## Configuration for Local Models

### Basic Qwen3 30B Setup

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      model: "openai/qwen3-30b",  // or "qwen" as alias
      promptMode: "minimal",  // Reduce prompt overhead
      provider: {
        openai: {
          baseURL: "http://localhost:11434/v1",  // Ollama
          apiKey: "ollama",  // Dummy key
          timeout: 120000  // 2min timeout for slower inference
        }
      }
    }
  }
}
```

### Tool Profiles for Local Models

**Recommended**: Use `messaging` profile for autonomous tasks:

```json5
{
  agents: {
    defaults: {
      toolProfile: "messaging",  // Limits to message + session tools
      // OR use custom allow/deny lists:
      tools: {
        allow: [
          "group:messaging",
          "group:memory",
          "session_status",
          "sessions_list",
          "sessions_send"
        ],
        deny: [
          "apply_patch",  // Complex code editing
          "sessions_spawn",  // Subagent orchestration
          "browser"  // Complex web interaction
        ]
      }
    }
  }
}
```

**Tool profiles available**:

```typescript
{
  minimal: {
    allow: ["session_status"]  // Just status checks
  },
  messaging: {
    allow: [
      "group:messaging",
      "sessions_list",
      "sessions_history", 
      "sessions_send",
      "session_status"
    ]
  },
  coding: {
    allow: ["group:fs", "group:runtime", "group:sessions", "group:memory"]
  },
  full: {}  // All tools (use with caution)
}
```

### Cron Jobs with Local Models

**Example 1: Email summary (simple)**

```bash
openclaw cron add \
  --name "Morning email summary" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize emails from the last 12 hours. List sender, subject, and 1-sentence summary for each." \
  --model "qwen" \
  --thinking off \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

**Why this works locally:**
- Single-pass summarization (no reasoning chains)
- Clear, specific instruction
- Thinking disabled (faster, no o1-style reasoning)
- Qwen 30B handles this excellently

**Example 2: Log monitoring (rule-based)**

```bash
openclaw cron add \
  --name "Check server logs" \
  --every "15m" \
  --session isolated \
  --message "Read /var/log/app.log. If any ERROR lines exist, extract them and send me a list." \
  --model "qwen" \
  --thinking off \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

**Why this works locally:**
- Pattern matching (grep-like)
- Binary decision (errors exist or not)
- Simple extraction task
- Even Llama 8B can handle this

### Email Hooks with Local Models

**Gmail webhook → Local Qwen**

```json5
{
  hooks: {
    enabled: true,
    presets: ["gmail"],
    gmail: {
      model: "qwen",  // Use local Qwen instead of cloud
      thinking: "off",
      allowUnsafeExternalContent: false  // Safety boundary
    },
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        wakeMode: "now",
        name: "Email categorizer",
        messageTemplate: `New email received:
From: {{messages[0].from}}
Subject: {{messages[0].subject}}
Body: {{messages[0].body}}

Task: Classify this email as one of: [urgent, info, spam, receipt]. Reply with just the category word.`,
        model: "qwen",
        thinking: "off",
        deliver: true,
        channel: "telegram"
      }
    ]
  }
}
```

**Why this works:**
- Email is already parsed by Gmail Pub/Sub (infrastructure)
- AI task is simple classification (pattern matching)
- Qwen 30B excels at structured output
- No complex reasoning needed

### Session Memory Hook (Works Perfectly)

The built-in `session-memory` hook uses an LLM to generate descriptive slugs:

```bash
openclaw hooks enable session-memory
```

This hook:
1. ✅ Reads last 15 messages (no AI)
2. ✅ Generates a slug via LLM (Qwen 30B works great)
3. ✅ Saves to `workspace/memory/{date}-{slug}.md`

**With Qwen 30B**: Slug generation is simple text summarization - works perfectly.

**Example output**:
- `2026-02-12-api-design-discussion.md`
- `2026-02-12-deployment-troubleshooting.md`

## What DOESN'T Work Well Locally

### Avoid These with Local Models

❌ **Complex multi-step workflows**
```bash
# This is too complex for local models:
openclaw cron add \
  --message "Review all open GitHub PRs, check CI status, summarize code changes, 
             decide which need my attention, draft review comments, and schedule follow-ups"
```

**Problem**: Multiple decision points, context switching, long-term planning.

**Solution**: Use cloud models (Claude Opus, GPT-4o) for complex orchestration.

---

❌ **Code generation/patching**
```bash
# Local models struggle with this:
openclaw agent --message "Refactor the authentication module to use OAuth2 instead of JWT"
```

**Problem**: Requires deep code understanding, long context, precise syntax.

**Solution**: Keep coding tasks on cloud models; use local models for monitoring/ops.

---

❌ **Extended tool chains**
```typescript
// This tool chain is too long for most local models:
1. web_fetch(url) → parse HTML
2. Extract 5 data points
3. Cross-reference with local DB
4. Generate report with charts
5. Send via 3 different channels
```

**Problem**: Each step requires context from previous steps; context window limits hit fast.

**Solution**: Limit local models to 1-3 tool calls max. Break complex chains into separate cron jobs.

## Real-World Local Autonomous Setup

Here's a **practical production config** using Qwen3 30B locally for autonomous tasks:

```json5
{
  // Two agents: cloud for complex, local for autonomous
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      model: "anthropic/claude-sonnet-4-20250514",  // Default for complex work
    },
    entries: {
      // Main agent uses cloud for reasoning
      main: {
        model: "anthropic/claude-sonnet-4-20250514",
        toolProfile: "full"
      },
      // Autonomous agent uses local for simple tasks
      ops: {
        model: "openai/qwen3-30b",
        promptMode: "minimal",
        toolProfile: "messaging",
        thinking: "off",
        provider: {
          openai: {
            baseURL: "http://localhost:11434/v1",
            apiKey: "ollama"
          }
        }
      }
    }
  },

  // Route autonomous tasks to local agent
  cron: {
    enabled: true,
    // All cron jobs default to "ops" agent
    defaultAgentId: "ops"
  },

  hooks: {
    enabled: true,
    presets: ["gmail"],
    gmail: {
      model: "qwen",  // Local model for email processing
      thinking: "off"
    }
  },

  // Main chat channels use cloud model
  channels: {
    whatsapp: {
      agentId: "main"  // Complex reasoning with cloud
    },
    telegram: {
      agentId: "main"
    }
  }
}
```

### Cron Jobs for "ops" Agent

```bash
# Email digest (every 6 hours)
openclaw cron add \
  --name "Email digest" \
  --cron "0 */6 * * *" \
  --agent ops \
  --session isolated \
  --message "Summarize unread emails. Group by sender. Keep it under 200 words." \
  --announce --channel telegram

# Server health check (every 15 minutes)
openclaw cron add \
  --name "Server health" \
  --every "15m" \
  --agent ops \
  --session isolated \
  --message "Check if /health returns 200. If not, send alert." \
  --announce --channel slack

# Daily backup reminder (8 PM)
openclaw cron add \
  --name "Backup reminder" \
  --cron "0 20 * * *" \
  --agent ops \
  --session isolated \
  --message "Verify backup completed today. Check /var/log/backup.log for SUCCESS." \
  --announce --channel whatsapp
```

## Performance Tips

### 1. Use Minimal Prompt Mode

```json5
{
  agents: {
    entries: {
      ops: {
        promptMode: "minimal"  // Reduces prompt tokens by ~60%
      }
    }
  }
}
```

**Benefit**: Faster inference, lower memory usage, less context window consumed.

### 2. Disable Thinking for Simple Tasks

```bash
openclaw cron add --thinking off  # No o1-style reasoning overhead
```

**Benefit**: 2-5x faster for summarization/classification tasks.

### 3. Limit Tool Access

```json5
{
  agents: {
    entries: {
      ops: {
        toolProfile: "messaging",  // Only message + session tools
        // Prevents model from attempting complex tool chains
      }
    }
  }
}
```

**Benefit**: Model focuses on simple operations, doesn't attempt what it can't do well.

### 4. Set Appropriate Timeouts

```json5
{
  agents: {
    entries: {
      ops: {
        provider: {
          openai: {
            timeout: 180000  // 3 minutes for local inference
          }
        }
      }
    }
  }
}
```

**Local models are slower**: Budget 2-3x the time you'd give cloud models.

### 5. Use Ollama for Easy Management

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull Qwen3 30B
ollama pull qwen2.5:30b

# Verify it's running
curl http://localhost:11434/v1/models
```

**Benefit**: Automatic model loading, memory management, OpenAI-compatible API.

## Limitations Summary

### ✅ Works Great Locally (Qwen3 30B)

- Email summaries & categorization
- Log monitoring & alerts
- Scheduled reports & digests
- Data extraction from structured text
- Simple classification tasks
- Memory/session management
- Message routing & delivery

### ⚠️ Works But Limited (Qwen3 30B)

- Multi-step workflows (2-3 steps max)
- Code reading & basic analysis
- Web scraping with simple extraction
- Report generation with templates

### ❌ Better with Cloud (Use Claude/GPT)

- Code generation & refactoring
- Complex debugging & root cause analysis
- Extended multi-tool workflows (4+ steps)
- Creative content generation
- Subagent orchestration
- Long-context reasoning (50k+ tokens)

## Example: Full Local Autonomous Setup

Here's a **complete working example** using only local models for autonomous features:

```bash
# 1. Start Ollama with Qwen
ollama pull qwen2.5:30b
ollama run qwen2.5:30b

# 2. Configure OpenClaw
cat > ~/.openclaw/openclaw.json <<'EOF'
{
  "agents": {
    "defaults": {
      "workspace": "~/.openclaw/workspace",
      "model": "openai/qwen2.5-30b",
      "promptMode": "minimal",
      "toolProfile": "messaging",
      "thinking": "off",
      "provider": {
        "openai": {
          "baseURL": "http://localhost:11434/v1",
          "apiKey": "ollama",
          "timeout": 180000
        }
      }
    }
  },
  "hooks": {
    "enabled": true,
    "internal": {
      "enabled": true,
      "entries": {
        "session-memory": {"enabled": true},
        "command-logger": {"enabled": true}
      }
    }
  },
  "cron": {
    "enabled": true
  }
}
EOF

# 3. Start OpenClaw Gateway
openclaw gateway --port 18789 &

# 4. Add autonomous jobs
openclaw cron add \
  --name "Morning summary" \
  --cron "0 8 * * *" \
  --session isolated \
  --message "Summarize yesterday's activity. Check workspace/memory/ for recent sessions." \
  --announce --channel telegram

# 5. Enable hooks
openclaw hooks enable session-memory
openclaw hooks enable command-logger

# 6. Verify
openclaw cron list
openclaw hooks list
```

**This setup gives you**:
- ✅ Scheduled summaries (cron)
- ✅ Session memory saving (hook)
- ✅ Command audit log (hook)
- ✅ All running locally with Qwen3 30B
- ✅ No cloud API calls
- ✅ Full autonomy

## Conclusion

**Key Takeaways**:

1. **Infrastructure is local-ready**: Cron, hooks, webhooks, routing all work perfectly offline
2. **Simple AI tasks work well**: Summaries, categorization, extraction, monitoring
3. **Qwen3 30B is excellent for ops**: Email processing, log monitoring, scheduled reports
4. **Complex reasoning needs cloud**: Code generation, debugging, multi-step planning
5. **Not portable to Python**: TypeScript-only, but you can wrap the HTTP API
6. **Use tool profiles**: Limit local models to `messaging` or `minimal` profiles
7. **Disable thinking**: Simple tasks don't need reasoning overhead

**Recommended approach**: Use local models (Qwen3 30B) for all autonomous/ops tasks, keep cloud models (Claude/GPT) for complex user-facing work.

## Learn More

<Columns>
  <Card title="How Agents Work" href="/concepts/how-agents-work" icon="robot">
    Full agent architecture and capabilities
  </Card>
  <Card title="Cron Jobs" href="/automation/cron-jobs" icon="clock">
    Detailed cron scheduling guide
  </Card>
  <Card title="Hooks" href="/automation/hooks" icon="webhook">
    Event-driven automation system
  </Card>
  <Card title="Model Providers" href="/concepts/model-providers" icon="cloud">
    Configure different AI providers
  </Card>
  <Card title="Tool Profiles" href="/tools/index" icon="wrench">
    Tool access control and profiles
  </Card>
  <Card title="Gmail Pub/Sub" href="/automation/gmail-pubsub" icon="mail">
    Email automation setup
  </Card>
</Columns>

---

**Next**: [Agent Configuration](/concepts/agent) — Detailed agent setup options
