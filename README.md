# heartbead-agents

OpenCode agent definitions for teams using [heartbeads](https://github.com/gainforest/heartbeads-cli) (`hb`) issue tracking.

Three roles, one shared DAG, no lateral communication between agents.

## Roles

| Agent | Mode | Model | Does | Doesn't |
|-------|------|-------|------|---------|
| **Lead** | primary | Opus | Decomposes work into specs with acceptance criteria | Write implementation code |
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

## Dispatch Modes

Workers operate in two modes depending on how the lead dispatches them:

### Solo Mode

One worker at a time. Full autonomy — the worker can run git, build, and hb commands freely.

```
@worker Claim and complete hb issue <id>. Run `hb show <id>` to read the full spec.
```

### Parallel Mode (Edit-Only)

Multiple workers simultaneously. Workers share the filesystem, so they're restricted to edit-only operations. The lead manages git, builds, and issue lifecycle between batches.

**Why this matters:** Multiple `npm run build` commands fight over `.next/` (OOM/corruption). Multiple `git commit` commands cause merge conflicts. `git stash` in one worker hides another's changes. Two workers editing the same file means one overwrites the other.

**Lead's responsibilities in parallel mode:**
1. Build a file-ownership table — zero file overlap between workers in the same batch
2. Dispatch workers with explicit edit-only rules and file lists
3. After all workers finish: verify `git status`, revert rogue changes, run single build, commit, close issues
4. If build fails: dispatch a fix worker (lead has no edit tools)

**Worker's responsibilities in parallel mode:**
1. Read the spec with `hb show`
2. Modify ONLY the listed files using Read/Write/Edit/Glob/Grep tools
3. Report which files changed and a summary — nothing else

```
@worker Implement hb issue <id>. Run `hb show <id>` to read the full spec.

CRITICAL RULES — YOU MUST FOLLOW THESE:
1. DO NOT run `npm run build`, `npx tsc --noEmit`, or ANY build/compile command
2. DO NOT run `git add`, `git commit`, `git push`, `git stash`, `git diff`, or ANY git command
3. DO NOT run `hb update`, `hb close`, or ANY hb command except `hb show` to read the spec
4. ONLY use Read, Glob, Grep, Edit, and Write tools to make your changes
5. ONLY modify the files listed in the spec — do NOT touch any other files
6. When done, report exactly which files you changed and a brief summary

You are modifying ONLY: <file1>, <file2>
```

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

**Lead** writes hyper-specific issue specs with acceptance criteria. Epics get `-d` descriptions with goals and context. Tasks get structured specs in `-d`, machine-readable acceptance criteria in `--acceptance`, a time budget via `-e 60`, and scope labels via `-l`. The spec quality determines system throughput — a good spec means the worker finishes in under an hour with zero questions.

**Worker** claims one issue (`hb update <id> --claim`), reads the spec and `acceptance_criteria` field, implements in only the listed files, verifies the acceptance criteria, commits, closes with the commit hash (`hb close <id> --reason "<hash> <msg>"`), and stops. In parallel mode, the worker only edits files — the lead handles git, builds, and issue lifecycle. Blockers are filed as new issues, not worked around.

**Reviewer** checks recently closed work against specs retroactively. Problems become new issues linked to the original via `discovered-from` dependency. Labels like `needs-redecomp`, `integration-risk`, and `test-suspect` are set via `--add-label`. The reviewer never reopens or rewrites — it files forward.

## Issue Lifecycle

```
Open → In Progress (claimed) → Closed (committed)
   ↘ Blocked/Deferred (blocker filed, Lead re-decomposes)
```

## Key Conventions

**Commit format:** `<summary> (<issue-id>)`

**Close requires commit hash:** `hb close <id> --reason "<hash> <message>"`

**Sync after every graph change:** `hb sync && git add .beads/ && git commit -m "beads: <action> <id>" && git push`

**Key flags:**

| Flag | Command | Purpose |
|------|---------|---------|
| `--acceptance` | create, update | Machine-readable acceptance criteria |
| `-e, --estimate` | create, update | Time budget in minutes |
| `-l, --labels` | create | Set labels at creation |
| `--add-label` | update | Add label after creation |
| `--claim` | update | Atomically set assignee + status=in_progress |
| `--silent` | create | Output only issue ID (for scripting) |
| `--deps` | create | Inline dependencies at creation |

**Labels:**

| Label | Set by | Meaning | Command |
|-------|--------|---------|---------|
| `scope:trivial/small/medium` | Lead | Estimated change size | `-l scope:small` on create |
| `needs-redecomp` | Reviewer | Spec failed twice, Lead must rewrite | `--add-label needs-redecomp` on update |
| `integration-risk` | Reviewer | Merged work may conflict | `--add-label integration-risk` on update |
| `test-suspect` | Reviewer | Acceptance criteria may be wrong | `--add-label test-suspect` on update |
| `needs-integration-review` | Lead | Epic ready for full-scope review | `--add-label needs-integration-review` on update |

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
