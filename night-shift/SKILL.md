---
name: night-shift
description: |
  Kick off long-running AI tasks before leaving desk — deep research, parallel PoC exploration, issue triage. Launches a headless Claude session under launchd as a system-level daemon, so it survives terminal and Claude Code REPL exits. Auto-resumes after 5-hour rate-limit windows via `claude --resume`. Telegram notifications on start / pause / resume / done / fail. Produces reports only, never modifies code or pushes. Gives you a "warm start" next morning. Use this skill whenever the user is about to leave, wants background research, or says "I'm heading out, can you work on this" — even if they don't say "night shift".
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

## COMPLETION PROTOCOL
After all deliverables are written and verified, execute:
  touch ~/reports/night-shift/state/{TASK_SLUG}.done
The launchd daemon polls this file. Without it, the daemon will attempt another resume round.

Take your time. Quality over speed. User is not waiting.
```

## Execution Mode

Two modes. **Default is launchd daemon** (Step 3 below). Only fall back to the legacy `nohup` mode (appendix) for quick tasks estimated under 15 minutes that do NOT require resume-after-rate-limit.

### Why launchd is default

The legacy `nohup + disown` approach runs `claude -p` as a child of the user's terminal. Two failure modes hit real work:
1. **Terminal close or Claude Code REPL exit kills the child** via SIGHUP (despite `nohup`, some shell setups still propagate). Reports at `~/reports/night-shift/` show empty logs when this happens.
2. **5-hour rate-limit windows**: long research blows through the quota mid-run. `claude -p` exits with an error, work is lost, no auto-resume.

The launchd daemon mode solves both: the daemon is a system-level process independent of any terminal/REPL, and it auto-resumes via `claude --resume <session-id>` after sleeping through the rate-limit window.

macOS only. On Linux, port to `systemd --user` with `OnCalendar=` (structure identical, replace launchctl with systemctl --user).

## Step 3: Launch headless session (launchd daemon — default)

Prerequisite (one-time, skip if `~/bin/night-shift-daemon.sh` already exists): copy the daemon template to `~/bin/`.

```bash
[ -f ~/bin/night-shift-daemon.sh ] || {
  mkdir -p ~/bin
  cp <skills-dir>/night-shift/daemon.sh.template ~/bin/night-shift-daemon.sh
  chmod +x ~/bin/night-shift-daemon.sh
  # Edit TG_CHAT_ID inside the script (or export TG_CHAT_ID env) if Telegram notifications are wanted.
  # Telegram bot token is read from ~/.claude/channels/telegram/.env — adjust the path if your setup differs.
}
```

Per-task setup:

```bash
mkdir -p ~/reports/night-shift/prompts ~/reports/night-shift/logs ~/reports/night-shift/state

TASK_DATE=$(date +%Y-%m-%d)
TASK_SLUG="{descriptive-slug}"   # kebab-case, becomes part of filenames and launchd label
PROMPT_FILE="$HOME/reports/night-shift/prompts/${TASK_DATE}_${TASK_SLUG}.md"

# Write the full prompt (from Step 2) to a file — NEVER pass long prompts on the command line
cat > "$PROMPT_FILE" << 'NIGHT_SHIFT_EOF'
{constructed prompt}
NIGHT_SHIFT_EOF

# Generate plist from template (see plist.template in this skill dir), or write inline:
PLIST=~/Library/LaunchAgents/com.$(whoami).night-shift-${TASK_SLUG}.plist
cat > "$PLIST" <<PLIST_EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key><string>com.$(whoami).night-shift-${TASK_SLUG}</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>${HOME}/bin/night-shift-daemon.sh</string>
        <string>${PROMPT_FILE}</string>
        <string>${TASK_SLUG}</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Month</key><integer>{MONTH}</integer>
        <key>Day</key><integer>{DAY}</integer>
        <key>Hour</key><integer>{HOUR}</integer>
        <key>Minute</key><integer>{MINUTE}</integer>
    </dict>
    <key>StandardOutPath</key><string>${HOME}/reports/night-shift/logs/launchd-${TASK_SLUG}-stdout.log</string>
    <key>StandardErrorPath</key><string>${HOME}/reports/night-shift/logs/launchd-${TASK_SLUG}-stderr.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key><string>${HOME}/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>HOME</key><string>${HOME}</string>
        <key>MAX_BUDGET_USD</key><string>8</string>
    </dict>
    <key>RunAtLoad</key><false/>
    <key>KeepAlive</key><false/>
</dict>
</plist>
PLIST_EOF

plutil -lint "$PLIST"
launchctl unload "$PLIST" 2>/dev/null
launchctl load -w "$PLIST"
launchctl list | grep "night-shift-${TASK_SLUG}"
```

Replace `{MONTH}/{DAY}/{HOUR}/{MINUTE}` with the target fire time in local 24h format. `StartCalendarInterval` with pinned month/day makes this a one-shot trigger.

**Daemon behavior** (see `daemon.sh.template` in this skill directory for the full script):
- Generates a stable UUID session-id on first run (stored at `~/reports/night-shift/state/<slug>.session`) — all subsequent resumes reuse the same session.
- First run uses `claude -p --session-id <uuid> <prompt>`; resumes use `claude -p --resume <uuid> <nudge>`.
- After each round, checks for a done marker at `~/reports/night-shift/state/<slug>.done` — the prompt must instruct Claude to `touch` this file when finished (see Completion Protocol in Step 2).
- On non-zero exit: greps log for rate-limit keywords (`rate limit`, `usage limit`, `429`, `try again in`, `5-hour`). Match → sleep 5h5min → resume. Max 5 resume attempts.
- Non-rate-limit errors abort immediately with a Telegram failure notification.
- Telegram events: `start` / `rate-limited, pausing` / `done` / `failed` / `ambiguous`.

**Manual test before waiting for the scheduled time** (optional):
```bash
~/bin/night-shift-daemon.sh ~/reports/night-shift/prompts/{DATE}_{slug}.md {slug}-test
```

## Step 4: Confirm to user

Report back:
```
Night shift scheduled.
- Task: {brief description}
- Fires at: {HH:MM} local via launchd
- Daemon: ~/bin/night-shift-daemon.sh
- Plist: ~/Library/LaunchAgents/com.<user>.night-shift-{slug}.plist (loaded)
- Prompt: ~/reports/night-shift/prompts/{date}_{slug}.md
- Logs: ~/reports/night-shift/logs/{slug}-*.log
- State: ~/reports/night-shift/state/{slug}.{session,done,attempts}
- Report destination: ~/reports/night-shift/{filename}.md
- Rate-limit behavior: auto-sleep 5h5min and resume, up to 5 rounds
- Notifications: Telegram (start / pause / resume / done / fail)
- Estimated duration: {estimate}

You can close the terminal, quit Claude Code, close the laptop lid (but not power off).
The daemon will fire at the scheduled time independent of this session.
```

Estimate duration based on scope: narrow lookup ~10 min, broad survey ~30 min, deep research with verification ~60+ min. Note that rate-limit pauses add 5h per resume round.

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

## Appendix: Legacy nohup mode (quick tasks only)

For tasks estimated under 15 minutes where rate-limit pause is not a concern and terminal-death risk is acceptable (user stays at the terminal and waits):

```bash
mkdir -p ~/reports/night-shift/logs
TASK_DATE=$(date +%Y-%m-%d)
TASK_SLUG="{slug}"
PROMPT_FILE="/tmp/night-shift-${TASK_SLUG}-${TASK_DATE}.txt"
LOG_FILE="$HOME/reports/night-shift/logs/${TASK_SLUG}-${TASK_DATE}.log"

cat > "$PROMPT_FILE" << 'NIGHT_SHIFT_EOF'
{constructed prompt}
NIGHT_SHIFT_EOF

nohup claude -p --dangerously-skip-permissions \
  --system-prompt-file "$PROMPT_FILE" \
  "Execute the research task described in the system prompt." \
  > "$LOG_FILE" 2>&1 &
disown
echo "PID: $!"
```

Why this is a fallback, not default:
- `nohup` does not reliably protect the child across all shell configurations; terminal close or Claude Code REPL exit can still SIGHUP the child in some setups.
- No rate-limit detection or auto-resume — hitting the 5-hour quota aborts the run with no recovery.
- No Telegram notifications unless bolted on manually.
- `nohup` closes stdin (fd 0); pass the trigger as a positional argument, NOT via `< file` (silent EOF).
- Use `--system-prompt-file` to avoid ARG_MAX on long prompts.
- Quoted heredoc `<< 'NIGHT_SHIFT_EOF'` preserves prompt bytes verbatim (no variable expansion).
