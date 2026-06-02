# CallPilot

**Your client calls write their own tickets.**

Every client call contains bugs to fix, features to build, and improvements your team needs to know about. Most of that signal evaporates before anyone opens a ticket.

CallPilot reads the transcript after every call, extracts every actionable item, and handles it — opening draft PRs for simple bugs, pinging engineers on Slack for complex ones, and filing everything else to your backlog.

Automatically. No servers. One YAML file. Works with Fireflies, Otter, and Zoom.

---

## What it looks like

Your call ends. 30 minutes later, your team sees this in Slack:

```
📞 CallPilot — "Q2 Review with Acme Corp"  ·  Jun 2, 2026

Found 3 items:

  ✅ "Fix random session expiry on Safari"
     → Draft PR #47  github.com/org/auth-service/pull/47  [🔴 HIGH]

  💬 "Add CSV export to reports"
     → Pinged @raj in #team-data  [🟡 MEDIUM]

  📋 "Mobile app for field team"
     → Backlog issue #23  github.com/org/product-roadmap/issues/23  [🟢 LOW]

3 items from 1 call. 0 minutes of manual triage.
```

And in your repo, a draft PR is already waiting:

```
[CallPilot] Fix random session expiry on Safari

## What & Why
Users are being logged out unexpectedly on Safari approximately twice per week
mid-session. The pattern suggests a cookie or session token issue specific to
Safari's storage behavior.

## Client Said
> "users are getting logged out randomly, especially on Safari. It happens
>  maybe twice a week"
— Alice Chen · Q2 Review with Acme Corp · Jun 2, 2026

## Changes
- auth/session.ts: extend SameSite cookie attribute handling for Safari ITP
```

---

## Install in 60 seconds

```bash
# 1. Install the skill
mkdir -p ~/.claude/skills/callpilot && curl -o ~/.claude/skills/callpilot/SKILL.md \
  https://raw.githubusercontent.com/Manpreet-2002/callpilot/main/skill/callpilot.md

# 2. Set your transcript API key
export FIREFLIES_API_KEY=your_key_here

# 3. Run guided setup (writes your config file)
/callpilot init

# 4. Schedule it
/schedule callpilot every 30 minutes
```

That's it. CallPilot runs every 30 minutes, checks for new transcripts, and takes action.

---

## First run is always safe

CallPilot starts in **confirm mode**. For your first 5 runs, it shows you exactly what it found and what it's about to do — and waits for your approval:

```
┌─ CallPilot Preview ───────────────────────────────────────────┐
│ Meeting: Q2 Review with Acme Corp                             │
│ Items found: 3                                                │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  1. [🔴 BUG · SIMPLE → DRAFT PR] auth-service                │
│     "Fix random session expiry on Safari"                     │
│     > "users are getting logged out randomly..."              │
│                                                               │
│  2. [🟡 ENHANCEMENT · COMPLEX → SLACK #team-data] @raj       │
│     "Add CSV export to reports"                               │
│     > "it would be nice if we could export as CSV"            │
│                                                               │
│  3. [🟢 NEW FEATURE → BACKLOG]                               │
│     "Mobile app for field team"                               │
│     > "we'd love a mobile app at some point"                  │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│ Proceed? (yes / no / edit)                                    │
└───────────────────────────────────────────────────────────────┘
```

After 5 confirmed runs with no corrections, it switches to fully autonomous mode automatically.

---

## Works with or without Slack

CallPilot doesn't require Slack. If it's not connected, everything routes to GitHub Issues instead. Slack is an upgrade, not a requirement.

| You have | CallPilot does |
|----------|----------------|
| GitHub MCP only | PRs + GitHub Issues for everything |
| GitHub + Slack MCP | PRs + Slack pings + Issues |
| Neither | Structured terminal report |

---

## Supported transcript sources

| Tool | Config | Required env vars |
|------|--------|-------------------|
| [Fireflies](https://fireflies.ai) | `type: fireflies` | `FIREFLIES_API_KEY` |
| [Otter](https://otter.ai) | `type: otter` | `OTTER_CLIENT_ID`, `OTTER_CLIENT_SECRET` |
| [Zoom](https://zoom.us) | `type: zoom` | `ZOOM_ACCOUNT_ID`, `ZOOM_CLIENT_ID`, `ZOOM_CLIENT_SECRET` |
| Manual / paste | `type: manual` | none |

**Manual mode** also works for: Slack threads, email chains, support tickets — drop any customer feedback into `callpilot_transcript.txt` and run `/callpilot`.

---

## Supported backlog targets

| Target | Config |
|--------|--------|
| GitHub Issues | `backlog.type: github_issues` (default) |
| Linear | `backlog.type: linear` (requires Linear MCP) |

---

## Routing matrix

| Type | Complexity | Action |
|------|------------|--------|
| Bug | Simple | Draft PR opened automatically |
| Bug | Complex | Slack ping to team oncall (or GitHub Issue if no Slack) |
| Feature enhancement | Simple | Draft PR opened automatically |
| Feature enhancement | Complex | Slack ping (or GitHub Issue if no Slack) |
| New feature | Any | Backlog: GitHub Issue or Linear |
| Ambiguous routing | Any | Fallback channel or triage issue |

Routing is driven by the `description` field on each team in your config. The more specific you make it, the more accurately CallPilot routes.

---

## Config in 30 seconds

```yaml
# callpilot.yaml
transcript_source:
  type: fireflies
  api_key: ${FIREFLIES_API_KEY}

teams:
  - name: backend
    description: "APIs, auth, sessions, database, performance, errors"
    repos: [your-org/api-service]
    slack_channel: "#team-backend"   # optional
    oncall: "@your-lead"             # optional

backlog:
  type: github_issues
  repo: your-org/product-roadmap

summary:
  slack_channel: "#callpilot"   # optional: post summaries here after each run
```

Full annotated config: [`examples/callpilot.example.yaml`](examples/callpilot.example.yaml)

---

## Requirements

- [Claude Code](https://claude.ai/code)
- [GitHub MCP](https://github.com/anthropics/anthropic-tools/tree/main/mcp-servers/github) connected
- A Fireflies, Otter, or Zoom account (or use manual mode)
- Slack MCP — optional

---

## How it works

```
Call ends
  ↓
Fireflies/Otter/Zoom generates transcript
  ↓
CallPilot wakes up (scheduled every 30 min)
  ↓
Polls transcript API for new meetings since last run
  ↓
Claude extracts items: type · complexity · urgency · client quote
  ↓
Routes each to the right team via your callpilot.yaml
  ↓
Executes actions using GitHub MCP + Slack MCP (no extra auth)
  ↓
Posts run summary to your #callpilot Slack channel
  ↓
Saves state — each meeting processed exactly once
```

No servers to host. No webhooks to configure. No auth to build. It's a Claude Code skill that runs on a schedule.

---

## Commands

```
/callpilot             — run now
/callpilot init        — guided setup (first time)
/callpilot --dry-run   — preview without acting
/callpilot --confirm   — force confirmation for this run
```

---

## License

MIT — fork it, extend it, share it.

---

*Built for early-stage startups that can't afford signal from client calls to disappear into meeting notes.*
