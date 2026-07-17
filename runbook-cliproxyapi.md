# RUNBOOK: Claude Code as orchestrator + Claude & GPT through your subscriptions (CLIProxyAPI)

> **For the human (the only part you read):** hand this file to your AI agent and tell it:
> **"Execute this runbook."** The agent does everything and only pings you for logins — and,
> if you don't have them yet, for buying the subscriptions. At the end you get a report that
> everything works.
>
> This supersedes the older Codex-plugin runbook: GPT no longer runs through the Codex CLI —
> it runs as a **native subagent** inside Claude Code, through a local proxy that speaks to
> your Claude and ChatGPT subscriptions via OAuth. Same orchestrator model, one less moving part.

---

## For the agent: your mission

You are configuring the "orchestrator + delegates" setup: the user works from **Cursor or
VS Code** with **Claude Code** as the orchestrator (thinks, plans, delegates, verifies) and
two delegates spawned as native subagents — **gpt-worker** (implementation) and
**fable-reviewer** (review). Both providers are reached through **CLIProxyAPI**, authenticated
by OAuth against the user's existing subscriptions. No pay-per-token API keys.

Run the steps in order; each has a VERIFY — do not move on if it fails. At **[HUMAN]** points,
stop and ask the user, then continue.

### Step 0 — Questions (all at once)

1. Do you have a Claude subscription (Pro/Max) and a ChatGPT subscription (Plus/Pro)? *(both required)*
2. Where will Claude Code run — **on this Mac** or **on a remote server** you SSH into?
3. Do you already keep an `AGENTS.md` with rules for your agents? *(if yes, we import it in Step 6)*

**[HUMAN]** If a subscription is missing, ask them to buy it now and wait for confirmation.

### Step 1 — CLIProxyAPI on the Mac

```bash
brew install cliproxyapi
brew services start cliproxyapi
```

Open `http://127.0.0.1:8317/management.html` and **[HUMAN]** ask the user to:

1. Set a **Management Key** — the password for this web UI.
2. Set a **Proxy API Key** — the password for the proxy itself; without it, nobody can send
   requests through it. It becomes `ANTHROPIC_AUTH_TOKEN` in Step 3.
3. Authenticate the **Claude** account via OAuth.
4. Authenticate the **ChatGPT/Codex** account via OAuth.

**VERIFY:**

```bash
curl -s http://127.0.0.1:8317/v1/models -H "Authorization: Bearer PROXY_API_KEY"
```

The model list must contain BOTH providers: `claude-*` and `gpt-*` entries. If only one shows,
the other OAuth login didn't complete — go back.

### Step 2 — Only for the server topology: the SSH tunnel

*(Skip this step entirely if Claude Code runs on the Mac itself — then the base URL in Step 3
is `http://127.0.0.1:8317`.)*

On the Mac, in `~/.ssh/config`:

```sshconfig
Host my-server
    HostName server.example.com
    User username
    RemoteForward 127.0.0.1:18317 127.0.0.1:8317
    ExitOnForwardFailure yes
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

Connect with `ssh my-server` (works the same from Cursor/VS Code Remote-SSH). From now on, on
the server, `127.0.0.1:18317` = CLIProxyAPI on the Mac.

Three links must stay alive: Mac awake, CLIProxyAPI running, SSH session open.
`ExitOnForwardFailure yes` catches the sneaky case where SSH connects but the forward silently
failed (port taken by a stale session).

**VERIFY (from the server):** the same `curl` as Step 1, against `http://127.0.0.1:18317`.

### Step 3 — Claude Code, wired to the proxy

Install (on whichever machine Claude Code runs):

```bash
npm install -g @anthropic-ai/claude-code
```

In Cursor/VS Code, install the **Claude Code** extension from the marketplace.

Then in `~/.claude/settings.json` — the `env` block, NOT shell exports (an export lives only in
the current shell; settings.json applies wherever Claude Code starts, including from the IDE or
after a reboot):

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:18317",
    "ANTHROPIC_AUTH_TOKEN": "PROXY_API_KEY"
  }
}
```

Use port `8317` for the local topology, `18317` for the server topology. Merge into the existing
file if one exists — never clobber other settings.

If an old Anthropic key exists, `unset ANTHROPIC_API_KEY` is NOT enough — a fresh shell brings
it back. Remove it at the source:

```bash
grep -rn "ANTHROPIC_API_KEY" ~/.bashrc ~/.profile ~/.zshrc 2>/dev/null
# delete the line wherever it appears, then open a new shell
```

**Restart Claude Code** — the `env` block is read at startup only.

**VERIFY:** start a Claude Code session, ask it anything, then prove the traffic path:

```bash
ss -tnp | grep 18317   # (or 8317 locally) — an ESTABLISHED connection from the claude process
```

### Step 4 — The delegate subagents

```bash
mkdir -p ~/.claude/agents
```

`~/.claude/agents/gpt-worker.md`:

```markdown
---
name: gpt-worker
description: Use proactively for implementation, debugging, refactoring, tests, and complex software engineering tasks.
model: gpt-5.6-sol
tools: Read, Grep, Glob, Bash, Write, Edit
effort: xhigh
---

You are the GPT implementation worker.

Implement the assigned task, run appropriate tests, and report the modified
files, test results, risks, and anything the parent agent should review.
Report honestly: if a check fails or you skipped something, say so plainly —
your output is reviewed on content, not on your summary.
```

`~/.claude/agents/fable-reviewer.md`:

```markdown
---
name: fable-reviewer
description: Use proactively for architecture, difficult decisions, security reviews, and reviewing work produced by other agents.
model: claude-fable-5
tools: Read, Grep, Glob, Bash
effort: max
---

You are the Fable architecture and review specialist.

Review architecture, correctness, security, edge cases, and maintainability.
Verify on content: read the actual diff and re-run checks — never trust the
implementer's summary. Do not modify files. Return specific, actionable
findings to the parent agent.
```

Why fable and not a cheaper model: the reviewer is the quality gate — per the
model table, review takes the highest intelligence+taste available, and cost
is only a tie-breaker. (If your main loop already runs on fable, this
subagent is still worth it: a fresh context reviewing the diff cold catches
what the context that ordered the work skims over.)

Why the model goes in the file, not in a parameter: the Agent tool's `model` parameter validates
against a closed list of Claude aliases (`sonnet | opus | haiku | fable`) and rejects any GPT
model ID before it touches the network. The custom agent's **frontmatter has no such
validation** — the ID goes straight through to the proxy. That's the entire trick.

Don't set `CLAUDE_CODE_SUBAGENT_MODEL` globally — it forces every subagent onto one model and
defeats the routing you're building.

**Restart Claude Code** — agents load at session startup.

**VERIFY:** in the new session, both `gpt-worker` and `fable-reviewer` appear in the available
agent types.

### Step 5 — Proof of identity and tools (do not skip)

In the new Claude Code session, spawn `gpt-worker` with exactly this prompt:

```text
State exactly which model you are. Then run this bash command and report its
exact output: echo test-$((7*191))
```

**VERIFY:** two things come back: **"gpt-5.6-sol"** (the real identity, not a Claude in
disguise) and **`test-1337`** (tools actually execute; the unseen arithmetic prevents a
memorized answer). Repeat the identity check for `fable-reviewer` (expect a Claude Fable answer).

### Step 6 — The global CLAUDE.md (`~/.claude/CLAUDE.md`)

Write the file with EXACTLY the content between the markers. Two adaptations allowed: (a) the
numbers in the `cost` column, based on the user's subscriptions (high cost score = cheap for
them); (b) if the user has an existing `AGENTS.md` (Step 0), add as the first line:
`@<path-to-AGENTS.md>` (Claude Code imports it). If CLAUDE.md already exists, show the diff and
ask for OK. If the user's rules live only in per-project CLAUDE.md files, still write the global
one — it complements them, it does not replace them.

<<<CLAUDE_MD_START>>>
## Picking the right models for workflows and subagents

Rankings, higher = better. Cost reflects what I actually pay (subscriptions, not list price).
Intelligence is how hard a problem you can hand the model unsupervised. Taste covers UI/UX,
code quality, API design, copy.

| model       | cost | intelligence | taste |
|-------------|------|--------------|-------|
| gpt-5.6-sol | 9    | 9            | 5     |
| sonnet-5    | 5    | 5            | 7     |
| opus-4.8    | 4    | 7            | 8     |
| fable-5     | 2    | 9            | 9     |

How to apply:
- These are defaults, not limits. Standing permission to override: if a cheaper model's output
  doesn't meet the bar, redo the work with a smarter model without asking. Judge the output,
  not the price tag. Escalating costs less than shipping mediocre work.
- Cost is a tie-breaker only; when axes conflict for anything that ships, intelligence > taste > cost.
- Implementation default = **gpt-5.6-sol** via the `gpt-worker` subagent: features, refactors,
  migrations, data analysis — nearly all code. It's effectively free.
- Anything user-facing (UI, copy, API design) needs taste ≥ 7 — and any user-facing surface
  implemented by gpt gets a dedicated brand/design audit against a NAMED reference surface
  before the verdict (gpt's taste is 5; correctness review alone will not catch off-brand work).
- Reviews of plans/implementations: the main loop holds the review and the verdict — always.
  `fable-reviewer` (or a gpt second opinion) adds extra independent perspectives, never the verdict.
- Second opinions are ROUTINE on high-stakes decisions: spawn `gpt-worker` with a self-contained
  prompt, arbitrate on evidence.
- Never use Haiku.
- Token-hungry work (computer use, codebase-wide scans, log grinding): cheap models or plain
  scripts reporting back — never burn main-session tokens. Mechanical grepping: no LLM at all.
- Mechanics: GPT models run through the local proxy. Three ways in:
  (1) subagent `gpt-worker` — the default, full tool use, its transcript is visible to review;
  (2) headless one-offs: `claude -p --model gpt-5.6-sol "..."`;
  (3) in Workflows: `agentType: 'gpt-worker'` on agent() calls.
- Claude models run via the Agent/Workflow model parameter as usual.
- CAVEAT: the proxy is a single point of failure — if Claude Code "won't start" or a gpt
  subagent errors, check the proxy port first.

Orchestrator discipline:
- The main session reviews, decides, and delegates. Substantial implementation goes to
  subagents; don't implement substantial code changes inline — review the delegate's diff
  instead. Tiny config/doc edits exempt.
- Never trust a delegate's report without verification: review the diff against the spec AND
  empirically reproduce at least one claimed result. Delegates must honestly mark what they
  could NOT run (docker, network, browsers) as PENDING — the orchestrator runs those first.
- Delegates self-verify visually: when delegated work has visual/rendered output (documents,
  UI, PDFs), the delegate must render it, screenshot it, and LOOK at the images before
  declaring success. The user's manual test is the final gate, never the debugging loop.
- If delegation loops or refuses repeatedly on a SMALL task, stop delegating and do it inline —
  resuming a confused agent burns tokens for nothing.
- Committing to a side branch while keeping main's working tree: NEVER `git checkout
  <existing-branch>` in the main repo when files are gitignored-but-tracked-on-branch — git
  silently clobbers ignored working-tree files. Use `git worktree add /tmp/wt <branch>`, copy
  files in, commit there, remove.
- `pkill -f <pattern>`: the pattern must never match your own command line (use `[x]` bracket
  tricks), or you kill your own shell.
<<<CLAUDE_MD_END>>>

**VERIFY:** the file contains the model table + both rule sections; if requested, the
`@AGENTS.md` import is on the first line.

### Step 7 — End-to-end test (do not skip)

In a temporary folder with `git init` — **under the home directory, NOT in `/tmp`**: ask Claude
Code to delegate to `gpt-worker` the creation of a `hello.py` that prints the sum 2+3, then to
review the result with `fable-reviewer`. Then **verify it YOURSELF** (the agent executing this
runbook): the file exists and `python3 hello.py` prints `5`. Delete the folder.

This validates the whole chain: IDE → Claude Code → proxy (→ tunnel) → OAuth → both providers →
orchestrator verification.

### Step 8 — Final report

Table of step → status (OK / OK with notes / FAILED); what was left undone and why; then give
the user their first commands:

1. Open your project, start Claude Code, tell it: *"Explore the repo and get a feel for it."*
2. For any big piece of work: *"Write a plan first, delegate the implementation to gpt-worker,
   review its diff yourself before telling me it's done."*
3. Usage: `/usage` in Claude Code. Main session on effort "high" (not xhigh/max — xhigh is
   token-hungry and max is a furnace with worse output; source: practice).

---

## When it breaks

| Symptom | Likely cause | Fix |
|---|---|---|
| Claude Code won't start conversations at all | The proxy is a single point of failure | `curl <base-url>/v1/models`; check each link: SSH alive? CLIProxyAPI running? Mac awake? |
| `curl` works on the Mac (8317) but not on the server (18317) | SSH forward died, or port taken | Reconnect SSH; `ExitOnForwardFailure yes` makes this a visible error at connect time |
| New subagent doesn't show up | Agents load at session startup | Restart Claude Code |
| New `env` values not applied | Read at startup | Restart Claude Code |
| Works, but you suspect it hits Anthropic directly | Old key left in `.bashrc`/`.profile` | Step 3's grep; then `ss -tnp` as proof |
| A gpt model ID is rejected by the Agent tool's `model` parameter | That parameter only takes Claude aliases | Put the model in the subagent's frontmatter (Step 4) — that's the supported path |
| You want the plain setup back | — | Delete the `env` block from `~/.claude/settings.json`, restart |

Two honest warnings: (1) the Proxy API Key sits in plain text in settings.json — fine on a
personal machine, but don't let it leave with publicly synced dotfiles. (2) In the server
topology, everything depends on your laptop: close the Mac and the server loses its AI. For
long unattended runs, think twice about whether this chain is what you want.
