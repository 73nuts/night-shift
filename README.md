<div align="center">

# Night Shift

**The missing async mode for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).**

Give Claude a research task. Close the terminal. Go to sleep.<br>
Wake up to a verified, structured report.

[![Claude Code Skill](https://img.shields.io/badge/Claude_Code-Skill-5A45FF?style=for-the-badge)](https://docs.anthropic.com/en/docs/claude-code)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)

</div>

---

## The Problem

Claude Code is synchronous. You ask, it answers, you follow up. Great for implementation. Terrible for the 11 PM question that needs two hours of research before you can act on it.

You shouldn't have to babysit an AI while it reads documentation.

## The Solution

Night Shift detaches Claude from your terminal and lets it run autonomously. One skill invocation. One background process. Zero supervision.

```
/night-shift
```

```
You, 11:47 PM:    "Survey the top Claude Code configuration systems — gstack, claw-code,
                   oh-my-codex. Architecture, maturity, what we can steal."

Claude:           Night shift launched.
                  - PID: 48291
                  - Output: ~/reports/night-shift/harness-ecosystem-2026-04-02.md
                  - Est. duration: ~45 min

                  You can close the terminal and leave.

You:              *closes laptop*

Next morning:     49KB report. 13-point executive summary. Three repositories dissected.
                  Blog claims fact-checked against actual source code.
                  Comparison matrix. Prioritized action items.
                  "Suggested first action: read the oh-my-codex planning gate."
```

That's a [real example](examples/sample-report.md). Not a demo. Not a mock. An actual report produced while the user was asleep.

---

> [!CAUTION]
> ## This Will Eat Your Rate Limits
>
> Night Shift is powerful but **token-hungry**. A single run spawns a headless Claude process that:
> - Runs for **10–60+ minutes** without stopping
> - Spawns **4–8 parallel research sub-agents** internally
> - Each sub-agent performs web searches, page fetches, file reads, and code verification
>
> Here's what that means for your subscription:
>
> | Task Type | Duration | Rate Limit Impact (Max $100) | Rate Limit Impact (Max $200) |
> |-----------|----------|------------------------------|------------------------------|
> | Quick lookup | ~10 min | ~5–10% of 5h window | Negligible |
> | Broad survey | ~30 min | ~15–20% of 5h window | ~8–10% of 5h window |
> | Deep research | ~60+ min | **~30%+ of 5h window** | ~15% of 5h window |
>
> A deep research run on the **$100 Max plan** will consume roughly **a third of your 5-hour rate limit window**. On the **$200 Max plan**, it's more comfortable. On **Pro ($20)**, a deep run may hit the limit entirely — stick to quick lookups or use Sonnet.
>
> **On API billing:** deep research costs roughly $3–8 per run at Opus pricing, $0.50–1.50 with Sonnet.
>
> **Tip:** Time your Night Shift runs before bed or during breaks — let the rate limit recover while you're away. That's the whole point.

---

## Why I Built This

I run a trading system. Every few days I need deep research — new frameworks, API changes, competitor analysis, methodology surveys. The kind of work where you open 30 browser tabs, read for two hours, and synthesize.

I kept wishing I could just *tell Claude what I need* and walk away. So I built that.

Night Shift has produced **12 reports** over two weeks. Topics ranged from anti-rug-pull detection algorithms to autonomous dev sprint architecture to harness engineering ecosystem surveys. The longest was 49KB. The shortest saved me a weekend of reading.

The insight that made it work: **a headless `claude -p` session with the right prompt is a surprisingly capable autonomous researcher** — if you give it verification requirements and security boundaries. Without those, it hallucinates and drifts. With them, it produces work I'd be comfortable acting on.

---

## How It Works

Night Shift is a [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) — a structured prompt that Claude follows when you invoke `/night-shift`. Here's what happens:

### 1. Clarify

Claude asks what you want researched. If your request is vague ("look into MCP stuff"), it helps you sharpen it into something a researcher can act on independently ("compare the top 5 MCP server frameworks by architecture, maturity, and community activity").

### 2. Construct

Claude builds a **self-contained prompt** — this is the key engineering decision. The headless session has no access to your CLAUDE.md, your memory files, or your conversation history. Everything the background process needs must be in the prompt:

- Your task description
- Output language and format preferences
- Security red lines (what the AI absolutely cannot do)
- Verification requirements (claims must be tested, not assumed)
- Report structure template

### 3. Launch

```bash
nohup claude -p --dangerously-skip-permissions "{prompt}" \
  > ~/reports/night-shift/your-report.md 2>&1 & disown
```

One detached process. Your terminal is free. Close it, shut the laptop, go to bed. The process runs independently.

### 4. Research

This is where the token budget goes. The headless Claude session:

- **Spawns 4–8 parallel research agents** using Claude Code's Agent tool — each tackling a different dimension of your question simultaneously
- **Searches the web** across multiple sources for each subtopic
- **Fetches actual pages and documentation** — not just search snippets
- **Cross-references findings** between sources to catch inconsistencies
- **Verifies claims**: checks if packages exist (`npm info`, `pip show`), tests if URLs are reachable, compiles code examples, makes real API calls to verify documented parameters
- **Writes incrementally** — if the process crashes, you still have partial results

### 5. Report

A structured markdown file waiting at `~/reports/night-shift/`:

```
your-topic-2026-04-03.md
├── Executive Summary (3–13 bullet points depending on scope)
├── Detailed Findings (organized by subtopic, with [researched]/[speculation] tags)
├── Verification Table (claim | method | result)
├── Comparison Matrix (if applicable)
├── Actionable Recommendations (with trade-offs)
├── Sources (with URLs)
└── Suggested First Action (what to do tomorrow morning)
```

---

## What Makes It Actually Work

Three design decisions separate Night Shift from "just paste a long prompt into Claude":

**Verification requirements.** The prompt explicitly tells Claude to verify before including. Package exists? `npm info` it. URL works? `curl` it. API endpoint documented? Make a real call. Code example? Write it to `/tmp` and compile it. This catches the hallucination problem at the source. Reports include a verification table so you can see what was checked and how.

**Parallel agent architecture.** A single Claude process spawning multiple sub-agents means broad coverage without losing coherence. Agent 1 researches the architecture, Agent 2 checks community reception, Agent 3 reads the actual source code — and the parent process synthesizes everything into one report. This is why a "broad survey" takes 30 minutes, not 3 hours.

**Security boundaries.** Running `--dangerously-skip-permissions` is necessary (no human to click "approve" at 3 AM) but scary. Night Shift constrains what the AI can do:

```
NEVER execute: rm, rmdir, mkfs, dd, shred, wipefs
NEVER modify: source code, git history, system config
NEVER exfiltrate: no POST requests with tokens/keys/passwords
ONLY write to: ~/reports/night-shift/
```

These are prompt-level constraints (belt) — pair them with Claude Code's `settings.json` deny list (suspenders) for defense in depth.

---

## What It's Good At

**Deep research** — Survey a field, compare tools, produce analysis with sources. The kind of thing that takes a human an entire weekend of tab-switching. Night Shift does it in under an hour.

**Parallel PoC exploration** — "Try 3 approaches, tell me the trade-offs." Night Shift spawns agents for each approach and synthesizes.

**Issue triage** — Point it at a GitHub repo. Get back: bugs by severity, stale items to close, quick wins.

**Code audits** — "Read through the auth module and find problems." Fresh-eyes review without scheduling a meeting.

**Trend scanning** — "What's new in AI tooling this month?" Wake up to a curated briefing with verified links.

## What It's Not For

- Tasks that need your input halfway through
- Writing or modifying code
- Production deployments
- Anything you wouldn't trust a smart intern to do alone at midnight

---

## Example Output

See [`examples/sample-report.md`](examples/sample-report.md) — a real Night Shift report (truncated from the original 49KB).

That run researched three Claude Code configuration systems (gstack, claw-code, oh-my-codex), verified star counts and file structures against actual GitHub repos, caught systematic exaggerations in popular blog posts, produced a cross-comparison matrix, and ended with prioritized action items. All while the user was away from the computer.

Here's just the verification table from that report — this is the part that makes Night Shift reports trustworthy:

| Claim | Verification Method | Verdict |
|-------|-------------------|---------|
| gstack has 7,501 lines of prompts | GitHub API file size stats | **Misleading** — actually 22K–27K+ |
| gstack has 14 skill modules | File tree traversal | **Inaccurate** — actually 34 |
| claw-code has 142K stars | GitHub API | Confirmed |
| oh-my-codex has 10.4K stars | GitHub API | Confirmed |
| "Boil the Lake" is a global principle | Source text citation | Accurate |
| "Error messages for machines" is global | ARCHITECTURE.md | **Exaggerated** — local to browse daemon only |
| CLAUDE.md best practice: 200–300 lines | HumanLayer + Anthropic docs | Confirmed |

Every claim checked. Every exaggeration caught. That's the difference between "AI research" and research you can act on.

---

## How It Compares

| | Night Shift | Normal Claude Code | ChatGPT Deep Research | Gemini Deep Research |
|---|:---:|:---:|:---:|:---:|
| Runs after terminal close | Yes | No | N/A | N/A |
| Accesses local tools (grep, curl, git) | Yes | Yes | No | No |
| Parallel sub-agent research | Yes | Yes | No | No |
| Verifies claims with real calls | Yes | Sometimes | No | No |
| Writes to your filesystem | Yes | Yes | No | No |
| Can read your codebase for context | Yes | Yes | No | No |
| Rate limit impact (deep run) | ~30% of 5h window | Interactive | Included | Included |
| Requires babysitting | No | Yes | Sort of | Sort of |
| Output quality ceiling | Very high | Very high | Medium | Medium |

Night Shift uses more of your rate limit. It also produces higher-quality, locally-verified research with zero supervision. ChatGPT and Gemini Deep Research are convenient and don't eat into your limits, but they can't run your local tools, test your code, or read your repos.

---

## Installation

```bash
# Clone
git clone https://github.com/73nuts/night-shift.git

# Copy the skill into your Claude Code skills directory
cp -r night-shift/night-shift/ ~/.claude/skills/night-shift/

# Create the output directory
mkdir -p ~/reports/night-shift
```

Done. Open Claude Code and type `/night-shift`.

### Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed and authenticated
- Claude API key or Pro/Max subscription
- Enough rate limit headroom for autonomous research (see [rate limit warning](#this-will-eat-your-rate-limits))

---

## Customization

The skill is a single markdown file. Open `~/.claude/skills/night-shift/SKILL.md` and edit.

| What to change | Where in SKILL.md | Default |
|---------------|-------------------|---------|
| Output language | `Output language:` in the prompt template | English |
| How Claude addresses you | Add `Address the user as:` to the prompt | *(none)* |
| Report output path | `~/reports/night-shift/` references | `~/reports/night-shift/` |
| Report structure | `## OUTPUT` section in the prompt | 7-section format |
| Security boundaries | `## SECURITY RED LINES` section | Strict (recommended) |
| Trigger phrases | `description:` in YAML frontmatter | English phrases |
| Task examples | `## Task Type Examples` section | 4 generic examples |

> [!WARNING]
> Think twice before loosening the security red lines. Night Shift runs **unattended with full permissions**. If you let it modify code or run git commands, you're trusting an AI to make unsupervised changes while you sleep. The defaults are strict for a reason.

---

## FAQ

<details>
<summary><b>Can I run multiple Night Shifts at once?</b></summary>

Yes. Each is an independent process. Two deep research runs in parallel will burn through rate limits fast, but they work.
</details>

<details>
<summary><b>What if it crashes mid-research?</b></summary>

Night Shift writes incrementally. If the process dies, you get a partial report with whatever completed. Not ideal, but you don't lose everything.
</details>

<details>
<summary><b>Can I check progress?</b></summary>

```bash
# Still running?
ps aux | grep claude

# Peek at partial results
tail -50 ~/reports/night-shift/your-report.md
```
</details>

<details>
<summary><b>What model does it use?</b></summary>

Whatever model your Claude Code is configured with. Sonnet for most research. Opus for deep cross-domain synthesis. The skill doesn't specify a model — it inherits yours.
</details>

<details>
<summary><b>Is <code>--dangerously-skip-permissions</code> safe?</b></summary>

It bypasses Claude Code's interactive permission prompts — necessary because there's no human at 3 AM to click "approve." The prompt-level security constraints and (optionally) `settings.json` deny lists are the actual safety layers. See [Security](#what-makes-it-actually-work).
</details>

<details>
<summary><b>How is this different from claude -p with a long prompt?</b></summary>

Night Shift IS `claude -p` with a carefully engineered prompt — that's the point. The value is in what that prompt contains: verification requirements that prevent hallucination, security boundaries for unattended operation, a report structure that produces actionable output, and parallel agent spawning instructions for research breadth. You could build this yourself. This saves you the iteration.
</details>

---

## License

MIT — do whatever you want with it.

---

<div align="center">

**Built for people who'd rather sleep than watch a loading spinner.**

*Night Shift is a [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills). It requires Claude Code CLI.*

</div>
