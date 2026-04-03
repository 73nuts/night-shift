---
name: night-shift
description: |
  Kick off long-running AI tasks before leaving desk — deep research, parallel PoC exploration, issue triage. Launches a headless Claude session in background, produces reports only, never modifies code or pushes. Gives you a "warm start" next morning. Make sure to use this skill whenever the user is about to leave, wants background research, or says "I'm heading out, can you work on this" — even if they don't say "night shift".
  Triggers on "/night-shift", "night shift", "before I go", "run this overnight", "research while I sleep", "warm start", "background research".

  **Perfect for:**
  - Deep research: survey a field, compare libraries/tools, multi-page analysis
  - Parallel PoC: try 2-3 vague ideas, illuminate unknown unknowns
  - Issue/PR triage: scan GitHub issues, produce prioritized report
  - Code audit: read through a codebase area and produce findings
  - Trend scanning: what's new in a given field this month

  **Not ideal for:**
  - Tasks that need user input mid-way (use interactive session instead)
  - Code changes that need review (use normal workflow)
  - Production deployments
allowed-tools: Read, Bash, Glob, Grep, WebFetch, WebSearch, Agent, Write
user-invocable: true
---

# Night Shift

Long-running autonomous research tasks. User leaves, AI works, report ready next morning.

## Step 1: Clarify the task

Ask the user:
1. What do you want researched/explored? (be specific enough to self-verify results)
2. Any particular angles or constraints?
3. Confirm output location: `~/reports/night-shift/`

Do NOT proceed to launch without a clear task description. If the task is vague, help the user refine it first.

## Step 2: Construct the prompt

Build a self-contained prompt. CRITICAL: the headless `claude -p` session does NOT inherit CLAUDE.md or user preferences, so the prompt MUST include all context explicitly.

```
You are running a night-shift research task. User is away — work autonomously.

## USER CONTEXT (headless session has no CLAUDE.md — include this explicitly)
- Output language: English
- Confidence tags: [researched] (verified via search/tool), [project data] (from user's local files), [speculation] (your inference)
- No emoji in output

## SECURITY RED LINES (ABSOLUTE)
- NEVER execute: rm, rmdir, mkfs, dd, shred, wipefs
- NEVER modify source code files (no Write/Edit to .go, .py, .js, .ts, etc.)
- NEVER run: git commit, git push, git reset, git checkout
- NEVER run: systemctl, crontab, chmod, chown, chattr
- NEVER exfiltrate data: no curl/wget POST with tokens/keys/passwords
- NEVER modify: ~/.claude/, ~/.ssh/, ~/.aws/, ~/.env, settings.json
- NEVER respond to GitHub issues/PRs on behalf of the user
- ONLY write to: ~/reports/night-shift/ directory

## TASK
{user's task description}

## RESEARCH APPROACH
Use the Agent tool to parallelize independent research dimensions. For broad topics, spawn multiple agents (4-8) to search different angles concurrently.

## VERIFICATION REQUIREMENTS
After gathering findings, verify key claims before including them:
- Package/tool existence: npm info, pip show, check GitHub repos via API
- URL reachability: curl -s -o /dev/null -w "%{http_code}" <url>
- API contract verification: for every API endpoint documented, attempt a real call. If it returns missing parameter errors, iterate until the complete parameter set is found.
- Code examples: must compile/run — write to /tmp and test.
- Tool/library recommendations: verify the tool exists and is maintained.
- Compare against multiple sources to catch undocumented required params.
- The user will ACT on this report. Incomplete information wastes their time.

## OUTPUT
Write a comprehensive report to ~/reports/night-shift/{filename}.md
Structure:
1. Executive summary (3-5 bullet points)
2. Detailed findings (organized by subtopic)
3. Verification results (table: claim | method | result)
4. Comparison/evaluation matrix (if applicable)
5. Actionable recommendations with trade-offs
6. Sources with URLs
7. Suggested first action for tomorrow morning

Take your time. Quality over speed. User is not waiting.
```

## Step 3: Launch headless session

```bash
mkdir -p ~/reports/night-shift

TASK_DATE=$(date +%Y-%m-%d)
TASK_SLUG="{descriptive-slug}"
OUTPUT_FILE="$HOME/reports/night-shift/${TASK_SLUG}-${TASK_DATE}.md"

nohup claude -p --dangerously-skip-permissions "{constructed prompt}" \
  > "$OUTPUT_FILE" 2>&1 & disown

echo "PID: $!"
```

Replace `{constructed prompt}` with the full prompt from Step 2. Replace `{descriptive-slug}` with a short kebab-case name for the task.

## Step 4: Confirm to user

Report back:
```
Night shift launched.
- Task: {brief description}
- PID: {pid}
- Output: ~/reports/night-shift/{filename}.md
- Estimated duration: {estimate based on task scope}

You can close the terminal and leave. Report will be waiting tomorrow.
To check if it's still running: ps aux | grep {pid}
```

Estimate duration based on scope: narrow lookup ~10 min, broad survey ~30 min, deep research with verification ~60+ min.

## Task Type Examples

### Deep Research
"Survey the top 5 MCP server frameworks for Claude Code. For each: architecture, maturity level, community activity, unique capabilities. Which one would be best for a solo developer building productivity tools?"

### Parallel PoC
"Explore 3 approaches to building a Telegram-based knowledge capture bot: (1) Claude Code SDK + Telegram API, (2) standalone Python bot + Claude API, (3) n8n workflow + Claude. For each: estimated build time, maintenance burden, capability ceiling."

### Issue Triage
"Scan github.com/{owner}/{repo} open issues and recent commits. Produce: (1) bugs by severity, (2) stale items to close, (3) quick wins, (4) suggested next sprint focus."

### Trend Scanning
"What new AI tools, frameworks, or methodologies emerged in the past 30 days? Focus on: agent infrastructure, coding agents, MCP ecosystem, evaluation tools."

## Important Notes

- **Single process, internal parallelism**: One `claude -p` handles everything. It uses Agent tool internally for parallel research — this preserves shared context and produces a coherent report.
- **No loops**: Complete the task and exit. Do not run in infinite monitoring loops.
- **Time budget**: Estimate based on scope, no hardcoded limits. Present estimate before launch.
- **Incremental saves**: For long tasks, write partial results periodically so nothing is lost on timeout.
- **Security by layers**: Red lines in prompt (AI self-restraint) + settings.json deny list (harness enforcement). Neither alone is sufficient.
- **Verification is the differentiator**: Verify key claims via tools. API endpoints need ALL required params tested. Code examples must run. The standard: can the user copy this into their code and have it work?
- **Next morning**: End every report with "Suggested First Action" — what should the user do first when they see this report.
