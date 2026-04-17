# Self-Learn Plugin for OpenClaw

An OpenClaw plugin that enables autonomous skill creation and evolution. The agent learns from complex tasks, saves reusable procedures as skills, and patches them when they become stale — replicating the self-learning loop from Hermes Agent.

## How It Works

1. **Guidance prompt** is injected into every system prompt via a `before_prompt_build` hook, telling the agent when and how to create skills
2. **`skill_manage` tool** gives the agent CRUD operations for skills (create, edit, patch, delete, plus supporting file management)
3. **Reference skill** provides the agentskills.io format spec on demand when the agent is writing a skill
4. **Security guard** scans all skill content for threat patterns before writing
5. **Atomic writes** ensure skills are never left in a partially-written state

Skills are stored in the workspace `skills/` directory with a `self-learned-` prefix (e.g., `skills/self-learned-deep-research/SKILL.md`), so they're automatically discovered by OpenClaw's skill loading pipeline. The prefix ensures the plugin only manages its own skills and never touches others.

## Installation

Install using the `openclaw plugins install` CLI. The plugin supports all standard installation methods.

### From a local path

```bash
openclaw plugins install /path/to/self-learn
```

Or link it instead of copying (useful during development):

```bash
openclaw plugins install -l /path/to/self-learn
```

### From an archive

```bash
openclaw plugins install self-learn.tar.gz
```

Supported formats: `.zip`, `.tgz`, `.tar.gz`, `.tar`.

### From npm (once published)

```bash
openclaw plugins install @openclaw/self-learn-plugin
```

Pin to an exact version:

```bash
openclaw plugins install @openclaw/self-learn-plugin@0.1.0 --pin
```

### Verify installation

```bash
openclaw plugins list --enabled
```

You should see `self-learn` in the output. You can also inspect it:

```bash
openclaw plugins inspect self-learn
```

Once loaded:
- The `skill_manage` tool appears in the agent's available tools
- Self-learning guidance is injected into the system prompt
- A `self-learn-review` heartbeat task is added to `HEARTBEAT.md` for periodic skill review

### Manage the plugin

```bash
openclaw plugins disable self-learn   # Temporarily disable
openclaw plugins enable self-learn    # Re-enable
openclaw plugins uninstall self-learn # Remove entirely
openclaw plugins doctor              # Diagnose load issues
```

## Configuration

Configure via your OpenClaw config under `plugins.entries.self-learn.config`:

```json
{
  "plugins": {
    "entries": {
      "self-learn": {
        "config": {
          "maxSkills": 100,
          "enableHeartbeat": true
        }
      }
    }
  }
}
```

| Option | Default | Description |
|--------|---------|-------------|
| `maxSkills` | `100` | Maximum number of self-learned skills |
| `enableHeartbeat` | `true` | Add a periodic heartbeat task to HEARTBEAT.md for skill hygiene review |

## Heartbeat Integration

The plugin automatically adds a `self-learn-review` task to `HEARTBEAT.md` on first activation. This task runs every 30 minutes and prompts the agent to:

1. Check if recent complex work should become a new skill
2. Patch any existing skills that were found stale during use
3. Move on if nothing needs attention

This is the periodic safety net that catches complex workflows the agent didn't capture in-the-moment. The heartbeat task works alongside the system prompt guidance — the guidance nudges the agent during the task, the heartbeat catches anything that slipped through.

The task is idempotent — re-enabling the plugin won't duplicate it. Uninstalling the plugin leaves the task in HEARTBEAT.md (remove it manually if desired).

## Tool Reference

The `skill_manage` tool supports 8 actions:

| Action | Required Params | Description |
|--------|----------------|-------------|
| `list` | — | List all self-learned skills (metadata only) |
| `view` | `name` | Read a skill's SKILL.md (optionally a supporting file via `file_path`) |
| `create` | `name`, `content` | Create a new skill with validation and security scan |
| `edit` | `name`, `content` | Full SKILL.md rewrite (major overhauls) |
| `patch` | `name`, `find`, `replace` | Targeted find-and-replace within SKILL.md |
| `delete` | `name` | Remove a self-learned skill |
| `write_file` | `name`, `file_path`, `content` | Add/overwrite a supporting file |
| `remove_file` | `name`, `file_path` | Remove a supporting file |

## Skill Format

Skills follow the [agentskills.io specification](https://agentskills.io/specification). Minimal example:

```yaml
---
name: deploy-k8s-helm
description: >
  Deploy services to Kubernetes using Helm charts. Use when deploying,
  upgrading, or rolling back Helm releases in any k8s cluster.
author: self-learn-plugin
---

# Deploy with Helm

## Quick Reference
| Action   | Command                              |
|----------|--------------------------------------|
| Deploy   | `helm upgrade --install release chart/` |
| Rollback | `helm rollback release`              |

## Gotchas
- Always run `helm diff` before upgrading to preview changes.
- The staging cluster uses a different kubeconfig context than production.
```
