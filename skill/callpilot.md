# CallPilot

Your client calls write their own tickets.

After every meeting, CallPilot reads the transcript, extracts every bug report, feature request, and enhancement — then acts: opening draft PRs for simple bugs, pinging engineers for complex ones, filing everything else to your backlog. Zero servers. One YAML file. Works with whatever tools you already use.

**Announce at start:** "Running CallPilot — processing recent transcripts."

---

## Invocation Modes

| Command | What it does |
|---------|-------------|
| `/callpilot` | Normal run — process new transcripts |
| `/callpilot init` | First-time guided setup (writes callpilot.yaml for you) |
| `/callpilot --dry-run` | Preview what would happen — no actions taken |
| `/callpilot --confirm` | Force confirmation mode for this run |

---

## Operating Mode

CallPilot has two execution modes:

**`confirm`** (default for runs 1–5): After extracting items, shows a full preview and waits for your approval before opening any PRs, sending any Slack messages, or filing any issues. Teaches you what it found and how it's routing before it acts. Builds trust before going autonomous.

**`auto`** (default after 5 confirmed runs): Executes immediately. No confirmation needed.

CallPilot automatically promotes itself from `confirm` → `auto` after 5 successful runs where you approved its output without corrections. You can override this at any time:
- `mode: auto` in `callpilot.yaml` — skip confirm phase permanently
- `mode: confirm` in `callpilot.yaml` — always confirm, never go auto
- `/callpilot --confirm` — force confirm for a single run

---

## Slack is Optional

CallPilot works without Slack. If Slack MCP is not connected, or if no `slack_channel` is set on a team, it routes all items to GitHub Issues instead with appropriate labels. Slack is an upgrade — not a requirement.

| Connected | Behavior |
|-----------|----------|
| GitHub MCP only | Everything goes to GitHub Issues and PRs |
| GitHub MCP + Slack MCP | Full routing: PRs + Slack pings + Issues |
| Neither | Outputs a structured report to terminal only |

---

## Prerequisites

Check what's available:
- **GitHub MCP** (`mcp__github__*`) — for PRs, Issues, reading code
- **Slack MCP** (`mcp__slack__*` or `mcp__claude_ai_Slack__*`) — optional, for team pings
- **Linear MCP** (`mcp__linear__*`) — optional, if `backlog.type: linear`

If GitHub MCP is missing: output the terminal report only, note what actions would have been taken, and suggest running `/mcp` to connect.

---

## Workflow

Follow these steps exactly, in order:

1. **Detect invocation mode** — check if called with `init`, `--dry-run`, or `--confirm` (see respective sections)
2. **Read config** — read `callpilot.yaml` from CWD; if missing and this is not an `init` run, output: "No callpilot.yaml found. Run `/callpilot init` to set up in 2 minutes."
3. **Read/initialize state** — read `callpilot.state.json` or initialize it
4. **Check available MCP tools** — determine which actions are available based on what's connected
5. **Fetch transcripts** — use adapter for `transcript_source.type`; filter already-processed meetings
6. **Report discovery** — "Found <N> new meeting(s) to process."
7. **For each new meeting:**
   - Run extraction to get items list
   - Route each item to a team
   - If in `confirm` mode: show full preview and wait for approval (see Confirmation Mode)
   - If `--dry-run`: show what would happen, stop
   - Otherwise: execute each action
   - Add meeting ID to `processed_meeting_ids`
8. **Update state**
9. **Post run summary** (see Run Summary section)

---

## Init Mode

When invoked as `/callpilot init`:

Walk the user through setup interactively. One question at a time.

```
Welcome to CallPilot setup. I'll ask a few questions and write your config file.
This takes about 2 minutes.
```

**Questions to ask (in order):**

1. "What meeting tool do you use? (fireflies / otter / zoom / other)"
   - If other: "For now, use `type: manual` — drop a callpilot_transcript.txt file to process any transcript."

2. "How many engineering teams should CallPilot route to? (just say a number)"

3. For each team (ask in sequence):
   - "Team <N> name? (e.g. backend, frontend, data)"
   - "What does this team own? Give me 5-10 keywords (e.g. 'auth login sessions OAuth passwords')"
   - "GitHub repos for this team? (e.g. your-org/api-service — press enter to add more, empty to move on)"
   - "Slack channel for this team? (e.g. #team-backend — or press enter to skip)"

4. "Where should new feature requests go?
   a) GitHub Issues (in which repo?)
   b) Linear (I'll need your Linear team ID)"

5. "One Slack channel for CallPilot summaries after each run? (or press enter to skip)"

After collecting answers: write `callpilot.yaml` to CWD and show it to the user.

```
✅ callpilot.yaml written. Here's what I configured:

[show the generated YAML]

Ready to run? I'll check your MCP connections and process your most recent meeting.
Proceed? (yes / no)
```

If yes: run the normal workflow immediately in `--dry-run` mode to show what it found, then ask if they want to execute for real.

---

## State Management

State is stored in `callpilot.state.json` in CWD.

**Schema:**
```json
{
  "last_run": "2026-06-02T14:30:00Z",
  "processed_meeting_ids": ["meet_abc123"],
  "run_count": 3,
  "confirmed_runs": 3,
  "mode": "confirm"
}
```

**Fields:**
- `last_run`: ISO8601 timestamp of last run
- `processed_meeting_ids`: meeting IDs already processed (max 500, drop oldest)
- `run_count`: total number of runs ever executed
- `confirmed_runs`: runs where user reviewed and approved output without corrections
- `mode`: current operating mode (`confirm` or `auto`)

**Mode promotion logic:**
- If `confirmed_runs >= 5` AND config does not have explicit `mode:` set: set `mode: auto` in state
- If config has explicit `mode:` set: always use config value, ignore state

**On initialization (no state file):**
```json
{
  "last_run": "1970-01-01T00:00:00Z",
  "processed_meeting_ids": [],
  "run_count": 0,
  "confirmed_runs": 0,
  "mode": "confirm"
}
```

---

## Transcript Adapters

### Fireflies

When `transcript_source.type: fireflies`:

```bash
curl -s -X POST https://api.fireflies.ai/graphql \
  -H "Authorization: Bearer $FIREFLIES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query":"query{transcripts(fromDate:\"<last_run>\"){id title date duration participants{displayName email}summary{overview}sentences{index speaker_name text start_time}transcript_url}}"}'
```

Response path: `data.transcripts`. Filter by `id` not in `processed_meeting_ids`.

Assemble text: `<speaker_name>: <text>\n` for each sentence in order.

---

### Otter

When `transcript_source.type: otter`:

**Get token:**
```bash
curl -s -X POST https://otter.ai/api/v1/auth/token \
  -H "Content-Type: application/json" \
  -d "{\"client_id\":\"$OTTER_CLIENT_ID\",\"client_secret\":\"$OTTER_CLIENT_SECRET\",\"grant_type\":\"client_credentials\"}"
```

**List speeches since last_run (convert ISO8601 to Unix timestamp):**
```bash
curl -s "https://otter.ai/api/v1/speeches?created_after=<unix_ts>" \
  -H "Authorization: Bearer <token>"
```

Assemble from `transcripts` array sorted by `start_offset`: `<speaker>: <text>\n`.

---

### Zoom

When `transcript_source.type: zoom`:

**Get token:**
```bash
curl -s -X POST "https://zoom.us/oauth/token?grant_type=account_credentials&account_id=$ZOOM_ACCOUNT_ID" \
  -H "Authorization: Basic $(echo -n "$ZOOM_CLIENT_ID:$ZOOM_CLIENT_SECRET" | base64)"
```

**List recordings:**
```bash
curl -s "https://api.zoom.us/v2/accounts/me/recordings?from=<YYYY-MM-DD>&to=<YYYY-MM-DD>" \
  -H "Authorization: Bearer <token>"
```

For each meeting with `type: audio_transcript` file: fetch the VTT URL and parse `Speaker: text` from each cue block.

---

### Manual / Paste Mode

When `transcript_source.type: manual`:

Read `callpilot_transcript.txt` from CWD. If missing, output:
```
No transcript found. Drop your transcript into callpilot_transcript.txt and run again.

Supported formats:
  Speaker Name: text they said
  [Speaker Name]: text they said
  Or plain paragraphs (speaker detection will be attempted)
```

This mode also accepts pasted content from:
- **Slack threads**: paste the full thread including timestamps and names
- **Email chains**: paste the full email chain, names and content
- **Support tickets**: paste the ticket body and any customer replies
- **Any text**: classification works on any customer/stakeholder feedback

For all paste inputs, generate a synthetic meeting ID from content hash:
```bash
echo -n "$(head -c 200 callpilot_transcript.txt)" | sha256sum | cut -c1-16
```

Use the first line of the file as the meeting title if it looks like a title, otherwise use `Manual Input <date>`.

---

## Extraction and Classification

For each meeting, assemble the full transcript text and internally reason through this prompt:

---

**EXTRACTION PROMPT:**

```
You are analyzing a communication between a startup team member and a client, customer, or stakeholder.
The communication may be a meeting transcript, a Slack thread, an email chain, or a support ticket.

Your job:
1. Extract every distinct item that implies engineering work needs to be done.
   IGNORE: pleasantries, praise, scheduling, vague wishes without specificity ("it'd be cool if someday...")
   INCLUDE: specific bugs, broken workflows, frustrating limitations, concrete feature asks, data issues, performance complaints

2. For each item, output:
   - type: [bug | feature_enhancement | new_feature]
   - complexity: [simple | complex]
   - title: 5–10 words, imperative tense ("Fix session expiry on Safari", not "Session expiry issue")
   - description: 2–3 sentences — what the problem is, why it matters to the client
   - client_quote: verbatim sentence(s) that best capture the item
   - speaker: who raised it (name or role)
   - routing_hint: 8–12 domain keywords for team matching
   - urgency: [low | medium | high] — based on tone, frequency of mention, and business impact implied

Classification rules:
  bug              → broken, not working, crashing, showing wrong data, slow to the point of unusable
  feature_enhancement → existing feature needs extension, more flexibility, or improved UX
  new_feature      → does not exist yet, would need to be built from scratch

Complexity rules:
  simple  → clearly scoped, 1–3 files, no cross-service coordination, could ship in a day
  complex → ambiguous scope, multiple services, design decisions required, or needs clarification first

Return ONLY valid JSON. No prose, no markdown fences.

[
  {
    "type": "bug",
    "complexity": "simple",
    "urgency": "high",
    "title": "Fix random session expiry on Safari",
    "description": "Users are being logged out unexpectedly on Safari approximately twice per week mid-session. The pattern suggests a cookie or session token issue specific to Safari's storage behavior.",
    "client_quote": "users are getting logged out randomly, especially on Safari. It happens maybe twice a week",
    "speaker": "Alice Chen",
    "routing_hint": "login logout sessions Safari authentication cookie session expiry token"
  }
]
```

---

**After extraction:**
- Empty array: log `No actionable items found in: <title>` and skip
- Invalid JSON: log parse error, do NOT mark as processed (retry next run)
- Sort items by urgency: high → medium → low before routing

---

## Routing

For each item:

1. Load team `name` + `description` pairs from config
2. Semantically match item's `routing_hint` + `description` against each team's description
3. Assign confidence score 0.0–1.0 to each team
4. If top score ≥ `routing.confidence_threshold`: route to that team
5. If top score < threshold: use fallback strategy

Log routing decision before acting:
```
→ "Fix session expiry on Safari" → auth (confidence: 0.91) [HIGH urgency]
→ "Export reports as CSV" → data (confidence: 0.84) [MEDIUM urgency]
→ "Mobile app for field team" → new_feature → backlog (confidence: 0.79)
```

---

## Action Matrix

| type | complexity | slack available? | action |
|------|------------|-----------------|--------|
| bug | simple | yes or no | Draft PR (with code) |
| bug | complex | yes | Slack ping |
| bug | complex | no | GitHub Issue, label: `needs-human` |
| feature_enhancement | simple | yes or no | Draft PR (with code) |
| feature_enhancement | complex | yes | Slack ping |
| feature_enhancement | complex | no | GitHub Issue, label: `needs-human` |
| new_feature | any | yes or no | Backlog (GitHub Issue or Linear) |
| routing confidence < threshold | any | any | Fallback |

---

## Actions

### Create Draft PR

For `bug` or `feature_enhancement` with `complexity: simple`.

**Step 1 — Understand the codebase:**
Use GitHub MCP to read the first repo in the routed team's `repos` list:
- List top-level directory
- Read README, package.json / requirements.txt / go.mod / Cargo.toml (whichever exists)
- Identify 2–4 most likely files to change based on `routing_hint` + `description`; read them fully

**Step 2 — Write the minimal fix:**
Reason step by step. What is the root cause given the client's description? What is the smallest correct change? Do not refactor, rename, or clean up adjacent code.

**Step 3 — Create branch and PR via GitHub MCP:**
- Branch: `callpilot/<meeting-id-8chars>-<kebab-title-40chars>`
- Apply changes on that branch
- PR:
  - **title:** `[CallPilot] <item title>`
  - **draft:** true (always)
  - **labels:** `pr.label` from config (default: `callpilot`)
  - **body:**
    ```
    ## What & Why
    <description>

    ## Client Said
    > "<client_quote>"
    — <speaker> · <meeting title> · <meeting date>

    ## Changes
    - <file: what changed and why>

    ## Urgency
    <urgency> — <rationale based on client tone>

    ---
    > Auto-generated by [CallPilot](https://github.com/Manpreet-2002/callpilot) from client call transcript.
    > Review carefully before merging.
    > Meeting: <transcript_url>
    ```

Output: `✅ Draft PR: <pr_url>`

---

### Slack Ping

For `bug` or `feature_enhancement` with `complexity: complex` AND Slack MCP connected.

Send to team's `slack_channel`:

```
:rotating_light: *CallPilot — <urgency_emoji> <urgency> priority*

*<item title>*
From: <meeting title> · <date>

*Client said:*
> "<client_quote>"
— <speaker>

*Why this matters:*
<description>

This is too complex for auto-implementation — needs scoping before work starts.
Transcript: <transcript_url>

<oncall> please triage within <urgency == high ? "4h" : "24h">.
```

Where `urgency_emoji`: high = 🔴, medium = 🟡, low = 🟢

Output: `💬 Slack ping → <slack_channel> (<oncall>) [<urgency>]`

---

### GitHub Issue — Needs Human

For `complexity: complex` when Slack MCP is NOT connected.

Create issue in the team's first repo (NOT backlog_repo — this is a known issue, just complex):
- **title:** `[CallPilot] <item title>`
- **labels:** `callpilot`, `needs-human`, `<type>`
- **body:** same content as the Slack ping message above, formatted as Markdown

Output: `📌 Issue (needs-human) → <issue_url>`

---

### Backlog — GitHub Issues

When `backlog.type: github_issues` (default):

Create issue in `backlog.repo`:
- **title:** `[CallPilot] <item title>`
- **labels:** `callpilot`, `feature-request`
- **body:**
  ```
  ## Feature Request

  **Client:** <speaker> · <meeting title> · <date>

  > "<client_quote>"

  **Context:**
  <description>

  **Routing hint:** `<routing_hint>`

  ---
  > Auto-filed by [CallPilot](https://github.com/Manpreet-2002/callpilot)
  > Meeting: <transcript_url>
  ```

Output: `📋 Backlog issue → <issue_url>`

---

### Backlog — Linear

When `backlog.type: linear`:

Use Linear MCP to create an issue in the configured team:
- **title:** `[CallPilot] <item title>`
- **team:** `backlog.linear_team_id`
- **priority:** map urgency → Linear priority (high=1, medium=2, low=3)
- **description:** same body as GitHub Issues version above

If Linear MCP is not connected: fall back to GitHub Issues and log the fallback.

Output: `📋 Linear issue → <linear_url>`

---

### Fallback

When routing confidence < `routing.confidence_threshold`:

**`slack_channel`:** Post to `fallback.slack_channel`:
```
:callpilot: *CallPilot — Needs Triage*

Could not confidently route this item from *<meeting title>*:

*<item title>* (confidence: <score>, threshold: <threshold>)
Type: <type> | Complexity: <complexity> | Urgency: <urgency>

> "<client_quote>"
— <speaker>

Domain keywords tried: `<routing_hint>`

<fallback.mention> — please assign to the right team.
Meeting: <transcript_url>
```

**`github_issue`:** Create issue in `backlog.repo` with labels `callpilot`, `needs-triage`.

**`ping_both`:** Both of the above.

Output: `⚠️  Fallback → <action> for "<title>"`

---

## Confirmation Mode

In `confirm` mode, after extracting and routing all items from a meeting, show this preview before executing anything:

```
┌─ CallPilot Preview ──────────────────────────────────────────┐
│ Meeting: <title> (<date>)                                     │
│ Items found: <N>                                              │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  1. [🔴 BUG · SIMPLE → DRAFT PR] auth-service               │
│     "Fix random session expiry on Safari"                     │
│     > "users are getting logged out randomly, especially      │
│       on Safari. It happens maybe twice a week"               │
│                                                               │
│  2. [🟡 ENHANCEMENT · COMPLEX → SLACK #team-data] @raj       │
│     "Add CSV export to reports"                               │
│     > "it would be nice if we could export our reports        │
│       as CSV, not just PDF"                                   │
│                                                               │
│  3. [🟢 NEW FEATURE · COMPLEX → BACKLOG]                     │
│     "Mobile app for field team"                               │
│     > "we'd love a mobile app at some point"                  │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│ Proceed? (yes = execute all · no = skip · edit = modify)      │
└───────────────────────────────────────────────────────────────┘
```

**Responses:**
- `yes` / `y` / `✅` / no response after 60s: execute all items, increment `confirmed_runs`
- `no` / `n` / `skip`: skip this meeting, do NOT add to `processed_meeting_ids` (retry next run)
- `edit <N> <new instruction>`: modify item N's routing or action before executing

After 5 confirmed runs (state `confirmed_runs >= 5`): promote to `auto` mode and notify:
```
✨ CallPilot has completed 5 confirmed runs without corrections.
   Switching to auto mode — future runs will execute without confirmation.
   Override with `mode: confirm` in callpilot.yaml to keep reviewing.
```

---

## Run Summary

After all meetings are processed, output:

```
┌─ CallPilot ─────────────────────────────────────────────────────────────────┐
│  📞 <N> meeting(s) · <total> item(s) extracted                              │
├──────────────────────────────────────────────────────────────────────────────┤
│  ✅  Draft PRs       <count>                                                 │
│  💬  Slack pings     <count>                                                 │
│  📋  Backlog items   <count>                                                 │
│  📌  Needs human     <count>                                                 │
│  ⚠️   Fallback        <count>                                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ✅  "Fix session expiry on Safari"                                          │
│      → PR #47  github.com/org/auth-service/pull/47  [🔴 HIGH]               │
│                                                                              │
│  💬  "Add CSV export to reports"                                             │
│      → Slack  #team-data  @raj  [🟡 MEDIUM]                                 │
│                                                                              │
│  📋  "Mobile app for field team"                                             │
│      → Issue #23  github.com/org/product-roadmap/issues/23  [🟢 LOW]        │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

**If `summary_channel` is configured:** Post this same summary (formatted for Slack) to that channel after each run. This gives PMs and founders visibility into what was actioned from their calls without opening a terminal.

**Slack-formatted summary for `summary_channel`:**

```
:callpilot: *CallPilot — Run Complete*
📞 *<meeting title>* · <date>

Found *<N> item(s)*:
<for each item: emoji + title + action + link>
  ✅ "Fix session expiry on Safari" → <pr_url>
  💬 "Add CSV export to reports" → pinged @raj in #team-data
  📋 "Mobile app for field team" → <issue_url>

_<N> items from 1 call. 0 minutes of manual triage._
```

**If zero meetings found:**
```
── CallPilot ──────────────────────────────────────────────────
No new meetings since <last_run>.
─────────────────────────────────────────────────────────────
```

---

## Dry Run Mode

When invoked with `--dry-run`:

Run the full workflow — fetch transcripts, extract items, route — but do NOT execute any action. Instead, show the Confirmation Mode preview for each meeting and append:

```
[DRY RUN — no actions taken]
To execute for real, run: /callpilot
```

Dry run does NOT increment `run_count` or `confirmed_runs`. It does NOT update `processed_meeting_ids` or `last_run`.
