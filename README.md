# heartbead-agents

OpenCode agent definitions for teams using [heartbeads](https://github.com/gainforest/heartbeads-cli) (`hb`) issue tracking.

Three roles, one shared DAG, no lateral communication between agents.

## Roles

| Agent | Mode | Model | Does | Doesn't |
|-------|------|-------|------|---------|
| **Lead** | primary | Opus | Decomposes work into specs with acceptance tests | Write implementation code |
| **Reviewer** | subagent | Sonnet | Reviews closed work, files issues for problems | Gate merges or rewrite code |
| **Worker** | subagent | Sonnet | Claims one issue, implements it, closes it | Refactor outside scope or add dependencies |

## Flow

```
Lead decomposes → Workers execute in parallel → Workers close on commit → Reviewer reviews async
                                                                              ↓
                                                                    Files new issues if problems found
                                                                              ↓
                                                                    Workers pick those up via hb ready
```

All coordination happens through the hb issue graph. No messages, no emails, no chat between agents.

## Install

Copy the agent files into your OpenCode agents directory:

```bash
# Global (all projects)
cp agents/*.md ~/.config/opencode/agents/

# Or project-local
mkdir -p .opencode/agents
cp agents/*.md .opencode/agents/
```

## Prerequisites

- [OpenCode](https://opencode.ai) installed
- [hb](https://github.com/gainforest/heartbeads-cli) installed and authenticated (`hb account login`)
- A git repository with `hb init` run

## Usage

In OpenCode, the Lead is a primary agent (switch with Tab). Reviewer and Worker are subagents invoked with `@reviewer` or `@worker`, or delegated automatically by the Lead.

```
# As the Lead (Tab to select), decompose work:
Break this feature into tasks...

# Invoke a worker:
@worker pick up the top ready issue

# Invoke reviewer after a batch of work:
@reviewer review the last 5 closed issues
```

## How It Works

**Lead** writes hyper-specific issue specs with acceptance tests. The spec quality determines system throughput — a good spec means the worker finishes in under an hour with zero questions. The graph should be readable cold by anyone, including the Lead after total context loss.

**Worker** claims one issue (`hb update <id> --claim`), implements it in only the listed files, runs the acceptance test, commits, closes with the commit hash (`hb close <id> --reason "<hash> <msg>"`), and stops. Blockers are filed as new issues, not worked around.

**Reviewer** checks recently closed work against specs retroactively. Problems become new issues linked to the original via `discovered-from` dependency. The reviewer never reopens or rewrites — it files forward.

## Issue Lifecycle

```
Open → In Progress (claimed) → Closed (committed)
   ↘ Blocked/Deferred (blocker filed, Lead re-decomposes)
```

## Key Conventions

**Commit format:** `<summary> (<issue-id>)`

**Close requires commit hash:** `hb close <id> --reason "<hash> <message>"`

**Sync after every graph change:** `hb sync && git add .beads/ && git commit -m "beads: <action> <id>" && git push`

**Labels:**

| Label | Set by | Meaning |
|-------|--------|---------|
| `scope:trivial/small/medium` | Lead | Estimated change size |
| `needs-redecomp` | Reviewer | Spec failed twice, Lead must rewrite |
| `integration-risk` | Reviewer | Merged work may conflict |
| `test-suspect` | Reviewer | Acceptance test may be wrong |
| `needs-integration-review` | Lead | Epic ready for full-scope review |

## Customization

Edit the `.md` files directly. The frontmatter controls model, temperature, and tool access. The body is the system prompt.

To use a different model for workers:

```yaml
model: anthropic/claude-sonnet-4-20250514  # change this line
```

To restrict the reviewer from running bash (fully read-only):

```yaml
tools:
  bash: false
```
