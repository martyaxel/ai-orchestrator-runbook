# RUNBOOK: Cursor + Claude Code + Codex setup (orchestrator + delegate)

> **For the human (the only part you read):** hand this file to your AI agent and tell it:
> **"Execute this runbook."** The agent does everything and only pings you for logins — and,
> if you don't have them yet, for buying the subscriptions. At the end you get a report that
> everything works.

---

## For the agent: your mission

You are configuring the "orchestrator + delegate" working setup: the user works from **Cursor**,
with **Claude Code** (the orchestrator: thinks, delegates, verifies) and **Codex/gpt-5.5** (the
cheap delegate that executes). Run the steps in order; each one has a VERIFY — do not move on if
it fails. At the **[HUMAN]** points, stop and ask the user for the action, then continue.

### Step 0 — Questions (all at once)

1. Do you have a Claude subscription (Pro/Max) and a ChatGPT subscription (Plus/Pro)? *(both required)*
2. Do you have Cursor installed? *(if they prefer VS Code, everything works identically)*
3. Do you already use an `AGENTS.md` for your agents? *(if yes, we import it at Step 5)*

**[HUMAN]** If a subscription is missing, ask them to buy it now and wait for confirmation.

### Step 1 — CLIs + extensions in Cursor

```bash
npm install -g @anthropic-ai/claude-code
npm install -g @openai/codex
```

Then in Cursor: install the **Claude Code** and **Codex** extensions from the marketplace
(Extensions → search by name). The extensions use the CLIs underneath — that's why we install both.

**VERIFY:** `claude --version` and `codex --version` work; both extensions show up in Cursor.

### Step 2 — Logins **[HUMAN]**

Run `claude`, ask the user to finish the login in the browser; same with `codex`.

**VERIFY:** `codex exec -s read-only "reply exactly: PONG"` returns PONG.

### Step 3 — Codex config (`~/.codex/config.toml`)

```toml
model = "gpt-5.5"
model_reasoning_effort = "high"
```

("high", not "xhigh" — xhigh is token-hungry, and max is a furnace with worse output. Source: practice.)

**VERIFY:** `grep model ~/.codex/config.toml` shows both lines.

### Step 4 — The Codex plugin for Claude Code (the bridge between the two)

The official plugin that gives Claude Code its delegation skills toward Codex
(`codex-implementation`, `codex-review`, `codex-computer-use`):

1. Read the install instructions in the README: **https://github.com/openai/codex-plugin-cc**
   and follow them exactly (installation happens from inside Claude Code, through its plugin
   system — the exact command is in the README; do not improvise).
2. **VERIFY:** in a Claude Code session, the codex-* skills show up as available (ask Claude
   "what codex skills do you have?" or check the installed plugin list).

### Step 5 — The global CLAUDE.md (`~/.claude/CLAUDE.md`)

Write the file with EXACTLY the content between the markers. Two adaptations allowed: (a) the
numbers in the `cost` column, based on the user's subscriptions (high cost score = cheap for
them); (b) if the user has an existing `AGENTS.md` (Step 0), add as the first line of the file:
`@<path-to-AGENTS.md>` (Claude Code imports the referenced file). If CLAUDE.md already exists,
show the diff and ask for OK.

<<<CLAUDE_MD_START>>>
## Picking the right models for workflows and subagents

Rankings, higher = better. Cost reflects what I actually pay, not list price. Intelligence is how
hard a problem you can hand the model unsupervised. Taste covers UI/UX, code quality, API design, copy.

| model    | cost | intelligence | taste |
|----------|------|--------------|-------|
| gpt-5.5  | 9    | 8            | 5     |
| sonnet-5 | 5    | 5            | 7     |
| opus-4.8 | 4    | 7            | 8     |
| fable-5  | 2    | 9            | 9     |

How to apply:
- These are defaults, not limits. You have standing permission to override them: if a cheaper
  model's output doesn't meet the bar, rerun or redo the work with a smarter model without asking.
  Judge the output, not the price tag. Escalating costs less than shipping mediocre work.
- Cost is a tie-breaker only; when axes conflict for anything that ships, intelligence > taste > cost.
- Bulk/mechanical work (clear-spec implementation, data analysis, migrations): gpt-5.5 — it's
  effectively free.
- Anything user-facing (UI, copy, API design) needs taste ≥ 7.
- Reviews of plans/implementations: fable-5 or opus-4.8, optionally gpt-5.5 as an extra
  independent perspective.
- Never use Haiku.
- Things that are unnecessarily token hungry (computer use, codebase analysis, big scans): do them
  with cheaper models or plain scripts and report results back — never burn main-session tokens.
  Mechanical grepping of logs/transcripts: no LLM at all, use Bash/grep/python.
- Mechanics: gpt-5.5 is reachable through the Codex CLI — use the codex-implementation,
  codex-review, and codex-computer-use skills (from the codex plugin); for work they don't cover
  (investigation, data analysis), run `codex exec -s read-only` directly with a self-contained prompt.
- Codex is notably better at computer use and UI/UX verification — prefer it there.
- Claude models run via the Agent/Workflow model parameter.

Using gpt-5.5 inside workflows and subagents (the model parameter only takes Claude models):
- Spawn a thin Claude wrapper agent with `model: 'sonnet', effort: 'low'` whose prompt instructs
  it to write a self-contained codex prompt, run `codex exec` via Bash, and return its output verbatim.
- Orchestrator discipline: the main session reviews, decides, and delegates. Hand implementation
  to subagents (Claude via Agent/Workflow, gpt-5.5 via Codex); don't implement substantial code
  changes inline in the main loop — review the delegate's diff instead. Tiny config/doc edits exempt.
- Delegates self-verify visually: when delegated work has visual/rendered output (documents, UI,
  PDFs), the delegate must render it, screenshot it, and LOOK at the images before declaring
  success. The user's manual test is the final gate only, never the debugging loop.
- Never trust a delegate's report without verification: review the diff against the spec AND
  empirically reproduce at least one claimed result. Delegates must honestly mark what they could
  NOT run in their sandbox (docker, network, browsers) as PENDING — the orchestrator runs those first.
- Committing to a side branch while keeping main's working tree: NEVER `git checkout <existing-branch>`
  in the main repo when files are gitignored-but-tracked-on-branch — git silently clobbers ignored
  working-tree files. Use `git worktree add /tmp/wt <branch>`, copy files in, commit there, remove.
- `pkill -f <pattern>`: the pattern must never match your own command line (use `[x]` bracket
  tricks), or you kill your own shell.
<<<CLAUDE_MD_END>>>

**VERIFY:** the file contains the model table + the bullets; if requested, the `@AGENTS.md`
import is on the first line.

### Step 6 — End-to-end test (do not skip it)

In a temporary folder with `git init`: ask Claude Code (from Cursor) to delegate to Codex the
creation of a `hello.py` that prints the sum 2+3, and to verify its work. Then **verify it
YOURSELF** (the agent executing this runbook): the file exists and `python3 hello.py` prints `5`.
Delete the folder.

This validates the whole chain: Cursor → Claude Code → plugin → Codex → orchestrator verification.

### Step 7 — Final report

Table of step → status (OK / OK with notes / FAILED); what was left undone and why; then tell
the user their first commands:
1. Open your project in Cursor, start Claude Code, tell it: *"Explore the repo and get a feel for it."*
2. For any big piece of work: *"Write a plan first, delegate the implementation to codex, verify
   its work before telling me it's done."*
3. Usage: `/usage-credits` in Claude Code. Main session on effort "high" (not xhigh/max).
