# CallPilot — Design Document

## What It Is

CallPilot is a shareable Claude Code skill that automatically processes meeting transcripts after a client or stakeholder call. It extracts action items, classifies them, routes them to the right team, and takes action — opening draft PRs, pinging developers on Slack, or creating backlog issues.

Targeted at early-to-mid stage startups. Zero infrastructure required.

---

## Problem

Signal from client calls evaporates before anyone acts on it. Engineers don't attend every call. PMs forget to file tickets. Bugs and feature requests get lost in meeting notes.

---

## Architecture

```
callpilot.yaml          callpilot.state.json
(startup config)        (last-run timestamp)
      ↓                       ↓
  ┌─────────────────────────────────────┐
  │         CallPilot Skill             │
  │  1. Poll transcript API             │
  │  2. Extract + classify items        │
  │  3. Route via callpilot.yaml        │
  │  4. Execute actions                 │
  └───────────┬──────────┬─────────────┘
              ↓          ↓
        GitHub MCP    Slack MCP
     (PRs, Issues)   (team pings)
```

No servers. No infrastructure. Four files: the skill, the config, the state file, and env vars.

---

## How It Works

Each time the routine fires (on a schedule):

1. Read `callpilot.yaml` and `callpilot.state.json`
2. Poll transcript API for meetings completed since `last_run`
3. Skip meetings already in `processed_meeting_ids`
4. For each new meeting, call Claude to extract items — each item gets a type, complexity score, and verbatim quote from the client
5. Route each item to a team by semantically matching against team `description` fields in config; if confidence < threshold, use fallback strategy
6. Execute action based on routing matrix

---

## Routing Matrix

| Type                 | Complexity | Action                                              |
|----------------------|------------|-----------------------------------------------------|
| `bug`                | simple     | Claude reads repo → writes fix → opens draft PR    |
| `bug`                | complex    | Slack ping to team oncall with full context         |
| `feature_enhancement`| simple     | Claude reads repo → writes change → opens draft PR |
| `feature_enhancement`| complex    | Slack ping to team oncall with full context         |
| `new_feature`        | any        | GitHub Issue created in `backlog_repo`              |

---

## callpilot.yaml Schema

```yaml
transcript_source:
  type: fireflies          # fireflies | otter | zoom | manual
  api_key: ${FIREFLIES_API_KEY}

teams:
  - name: auth
    description: "Authentication, login, sessions, OAuth, passwords, tokens, user accounts"
    repos: [org/auth-service, org/user-service]
    slack_channel: "#team-auth"
    oncall: "@priya"

  - name: data
    description: "Data pipelines, analytics, dashboards, exports, reporting, warehouse"
    repos: [org/data-pipeline, org/warehouse]
    slack_channel: "#team-data"
    oncall: "@raj"

  - name: frontend
    description: "UI, web app, design, forms, pages, navigation, layout"
    repos: [org/web-app]
    slack_channel: "#team-frontend"
    oncall: "@sam"

backlog_repo: org/product-roadmap

fallback:
  strategy: slack_channel       # slack_channel | github_issue | ping_both
  slack_channel: "#engineering"
  mention: "@oncall-lead"

routing:
  confidence_threshold: 0.75    # below this → fallback

pr:
  draft: true                   # open PRs as draft by default
  label: "callpilot"
```

The `description` field under each team is the key for semantic routing. Richer descriptions → better routing accuracy.

---

## callpilot.state.json (auto-managed)

```json
{
  "last_run": "2026-06-02T14:30:00Z",
  "processed_meeting_ids": ["abc123", "def456"]
}
```

Updated automatically after each run. Never edit manually.

---

## Transcript Sources

| Source     | Type      | Notes                                    |
|------------|-----------|------------------------------------------|
| Fireflies  | API poll  | `/v1/transcripts` endpoint               |
| Otter      | API poll  | `/v2/speeches` endpoint                  |
| Zoom       | Webhook / API | Meetings API                         |
| Manual     | File drop | Drop a `.txt` file into a watched folder |

Adding a new source = adding one adapter. Core logic is identical across all sources.

---

## What "Create a PR" Actually Means

For bugs and simple enhancements, the skill:

1. Reads the relevant files in the repo (guided by the team's `repos` list)
2. Understands the issue from the client quote + meeting context
3. Writes a code fix using Claude's understanding of the codebase
4. Opens a draft PR with: the fix, description of the issue, client quote, and meeting metadata

The PR is always a draft by default. Developer reviews and merges.

---

## Distribution

```bash
# Install
curl -o ~/.claude/skills/callpilot.md \
  https://raw.githubusercontent.com/you/callpilot/main/skill.md

# Configure
cp callpilot.example.yaml callpilot.yaml
# edit callpilot.yaml with your teams, repos, Slack channels

# Set env vars
export FIREFLIES_API_KEY=...
# GitHub and Slack tokens are already configured via Claude Code MCP

# Schedule
/schedule callpilot every 30 minutes
```

Entire onboarding under 5 minutes. Open source on GitHub.

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Skill (not server) | Zero infrastructure for startups to manage |
| Config-as-code (YAML) | Version-controlled, dev-friendly, generalizable |
| Reuse GitHub + Slack MCP | No auth to build or manage |
| State file (not DB) | Simple, portable, no dependencies |
| Draft PRs by default | Human always reviews AI-generated code changes |
| Confidence threshold in config | Each startup defines their own routing tolerance |
| Fallback strategy in config | Each startup owns their escalation policy |

---

## Open Questions / V2

- Web dashboard for non-technical PMs to configure routing (requires OAuth + multi-tenant infra)
- Human confirmation gate: "Found 3 items — react ✅ to execute or ✏️ to edit" before actions fire
- GitHub App auth for cleaner org-level installation
- Support for Linear, Jira, Notion as backlog targets (not just GitHub Issues)
