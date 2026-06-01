# CallPilot

Auto-process meeting transcripts and route bugs, feature enhancements, and new feature requests to the right team — opening draft PRs, pinging developers on Slack, or filing backlog issues.

**Announce at start:** "Running CallPilot — processing recent meeting transcripts."

---

## Prerequisites

Before running, verify these MCP tools are available in the session:
- GitHub MCP (`mcp__github__*` tools) — for creating PRs and Issues
- Slack MCP (`mcp__slack__*` or `mcp__claude_ai_Slack__*` tools) — for pinging teams

If either is missing, stop and output:
```
CallPilot requires GitHub and Slack MCP connections.
Run /mcp to check connected servers, then reconnect and try again.
```

---

## Workflow

Follow these steps exactly, in order, every time CallPilot is invoked:

1. **Check prerequisites** — verify GitHub MCP and Slack MCP tools are available
2. **Read config** — read `callpilot.yaml` from CWD using the Read tool; if it doesn't exist, stop with: "No callpilot.yaml found. Copy examples/callpilot.example.yaml and configure your teams."
3. **Read/initialize state** — read `callpilot.state.json` or initialize it (see State Management)
4. **Fetch transcripts** — use the adapter matching `transcript_source.type` (see Transcript Adapters); filter out meeting IDs already in `processed_meeting_ids`
5. **Report discovery** — output: "Found <N> new meeting(s) to process."
6. **For each new meeting:**
   a. Run extraction (see Extraction and Classification) — get list of items
   b. For each item: route it (see Routing), then execute the appropriate action (see Actions)
   c. Add the meeting ID to `processed_meeting_ids`
7. **Update state** — write updated `callpilot.state.json`
8. **Print run summary** — see Run Summary

---

## State Management

State is stored in `callpilot.state.json` in the current working directory.

**On startup — read or initialize state:**

1. Check if `callpilot.state.json` exists in CWD
2. If it exists, read it. Expected shape:
   ```json
   {
     "last_run": "2026-06-02T14:30:00Z",
     "processed_meeting_ids": ["meet_abc123", "meet_def456"]
   }
   ```
3. If it does not exist, initialize with:
   ```json
   {
     "last_run": "1970-01-01T00:00:00Z",
     "processed_meeting_ids": []
   }
   ```

**On completion — update state:**

After processing all meetings in a run, write updated state:
```json
{
  "last_run": "<ISO8601 timestamp of this run's start>",
  "processed_meeting_ids": ["<all previously processed IDs plus newly processed ones>"]
}
```

Keep `processed_meeting_ids` to a maximum of 500 entries. If it grows beyond 500, drop the oldest entries (keep the 500 most recent).

---

## Transcript Adapters

### Fireflies Adapter

When `transcript_source.type: fireflies`:

**Endpoint:** `POST https://api.fireflies.ai/graphql`
**Auth:** Bearer token using the value of `transcript_source.api_key` (expand env var if prefixed with `${`)

**Fetch transcripts since `last_run` (run via Bash tool):**
```bash
curl -s -X POST https://api.fireflies.ai/graphql \
  -H "Authorization: Bearer $FIREFLIES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { transcripts(fromDate: \"<last_run>\") { id title date duration participants { displayName email } summary { overview action_items } sentences { index speaker_name text start_time } transcript_url } }"
  }'
```

Replace `<last_run>` with the ISO8601 value from state.

**Response path:** `data.transcripts` — array of meeting objects.

**Filter:** Remove any meeting whose `id` is already in `processed_meeting_ids`.

**Assemble transcript text** from `sentences` array:
```
<speaker_name>: <text>
<speaker_name>: <text>
...
```

---

### Otter Adapter

When `transcript_source.type: otter`:

**Step 1 — Get access token:**
```bash
curl -s -X POST https://otter.ai/api/v1/auth/token \
  -H "Content-Type: application/json" \
  -d "{\"client_id\":\"$OTTER_CLIENT_ID\",\"client_secret\":\"$OTTER_CLIENT_SECRET\",\"grant_type\":\"client_credentials\"}"
```
Extract `access_token` from response.

**Step 2 — List speeches since last_run (convert ISO8601 to Unix timestamp):**
```bash
curl -s "https://otter.ai/api/v1/speeches?created_after=<unix_timestamp>" \
  -H "Authorization: Bearer <access_token>"
```

Response includes speeches with `id`, `title`, `created_at`, and `transcripts` array (each: `speaker`, `text`, `start_offset`).

**Filter:** Remove any speech whose `id` is already in `processed_meeting_ids`.

**Assemble transcript text** from `transcripts` array, sorted by `start_offset`:
```
<speaker>: <text>
```

---

### Zoom Adapter

When `transcript_source.type: zoom`:

**Step 1 — Get access token (Server-to-Server OAuth):**
```bash
curl -s -X POST "https://zoom.us/oauth/token?grant_type=account_credentials&account_id=$ZOOM_ACCOUNT_ID" \
  -H "Authorization: Basic $(echo -n "$ZOOM_CLIENT_ID:$ZOOM_CLIENT_SECRET" | base64)"
```
Extract `access_token` from response.

**Step 2 — List cloud recordings from last_run date to today:**
```bash
curl -s "https://api.zoom.us/v2/accounts/me/recordings?from=<YYYY-MM-DD>&to=<YYYY-MM-DD>" \
  -H "Authorization: Bearer <access_token>"
```

**Step 3 — For each meeting that has a transcript file (type: `audio_transcript`), fetch it:**
```bash
curl -s "<transcript_download_url>?access_token=<access_token>"
```
The response is in VTT format. Parse it: each cue block contains `<timestamp>` then `<Speaker Name>: <text>`.

**Filter:** Remove any meeting whose `uuid` is already in `processed_meeting_ids`.

---

### Manual Adapter

When `transcript_source.type: manual`:

1. Read `callpilot_transcript.txt` from CWD
2. If it doesn't exist, output: "No callpilot_transcript.txt found. Create this file with your meeting transcript and run CallPilot again." and stop.
3. Generate a synthetic meeting ID: `manual_<first 16 chars of sha256 of file content>`
   ```bash
   echo -n "$(head -c 200 callpilot_transcript.txt)" | sha256sum | cut -c1-16
   ```
4. Use the file contents directly as the transcript text. Speaker lines are detected by the pattern `Name: text` at the start of a line.
5. Use filename (without extension) as the meeting title, and current timestamp as the date.

---

## Extraction and Classification

For each new meeting, assemble the full transcript text from the sentences/cues array, then internally reason through the following prompt:

---

**EXTRACTION PROMPT — reason through this for each meeting:**

```
You are analyzing a meeting transcript between a startup team member and a client or stakeholder.

Your job:
1. Extract every distinct item the client or stakeholder mentioned that implies work needs to be done.
   Ignore: social pleasantries, praise, scheduling discussion, vague wishes with no specificity.
   Include: bug reports, complaints about existing functionality, requests for changes, requests for new things.

2. For each extracted item, determine:
   - type: exactly one of [bug, feature_enhancement, new_feature]
   - complexity: exactly one of [simple, complex]
   - title: a concise 5–10 word summary (imperative, e.g. "Fix session expiry on Safari")
   - description: 2–3 sentences explaining what the client wants and why it matters to them
   - client_quote: the verbatim sentence(s) from the transcript that best captures this item
   - speaker: the name of the person who raised it
   - routing_hint: 8–12 key domain words that will help match this item to an engineering team

Classification rules:
  bug              → something is broken, failing, producing errors, or crashing
  feature_enhancement → an existing feature that needs to be extended, improved, or made more flexible
  new_feature      → something that doesn't exist yet and would require building from scratch

Complexity rules:
  simple → self-contained change affecting 1–3 files, clearly scoped, no cross-service coordination needed
  complex → touches multiple services, requires design decisions, has ambiguous scope, or needs clarification before work can begin

Return ONLY a valid JSON array. No prose, no markdown fences. Example:
[
  {
    "type": "bug",
    "complexity": "simple",
    "title": "Fix random session expiry on Safari",
    "description": "Users are being unexpectedly logged out on Safari browser approximately twice per week. The issue appears session or cookie-related and is specific to Safari.",
    "client_quote": "users are getting logged out randomly, especially on Safari. It happens maybe twice a week.",
    "speaker": "Alice Chen",
    "routing_hint": "login logout sessions Safari authentication cookie session expiry"
  }
]
```

---

**After extraction:**
- If the array is empty: log `No actionable items found in: <meeting title>` and move to the next meeting.
- If extraction returns invalid JSON: log `Extraction parse error for: <meeting title> — skipping` and do NOT add the meeting ID to `processed_meeting_ids` (it will be retried next run).

---

## Routing

For each extracted item, route it to a team using this algorithm:

1. Load all team `name` + `description` pairs from `callpilot.yaml`
2. Assess which team's description best matches the item's `routing_hint` + `description` using semantic reasoning
3. Assign a confidence score from 0.0 to 1.0
4. If the top score ≥ `routing.confidence_threshold` (default: 0.75): route to that team
5. If the top score < `routing.confidence_threshold`: use the fallback strategy (see Actions → Fallback)

Output your routing decision before executing the action:
```
→ Routing "<title>" to team: <name> (confidence: <score>)
```
or
```
→ Routing "<title>" to fallback (confidence below threshold: <score>)
```

---

## Actions

Use this matrix to decide which action to take:

| type                 | complexity | action                    |
|----------------------|------------|---------------------------|
| `bug`                | `simple`   | Create Draft PR           |
| `bug`                | `complex`  | Slack Ping                |
| `feature_enhancement`| `simple`   | Create Draft PR           |
| `feature_enhancement`| `complex`  | Slack Ping                |
| `new_feature`        | `simple`   | Backlog Issue             |
| `new_feature`        | `complex`  | Backlog Issue             |
| any (low confidence) | any        | Fallback                  |

---

### Action: Create Draft PR

For `bug` or `feature_enhancement` with `complexity: simple`.

**Step 1 — Understand the codebase:**
Use GitHub MCP to read the first repo in the routed team's `repos` list:
- List the top-level directory
- Read `README.md` (if present) for stack overview
- Read `package.json`, `requirements.txt`, `go.mod`, `Cargo.toml`, or `pyproject.toml` (whichever exists) to understand dependencies
- Use the item's `routing_hint` and `description` to identify the 2–4 most likely files to change; read those files in full

**Step 2 — Reason about the fix:**
Given the client quote, description, and code you've read — determine the minimal correct change. Think step by step. Prefer the smallest change that addresses the root cause. Do not refactor beyond what's needed.

**Step 3 — Create the branch and PR using GitHub MCP:**
- Branch name: `callpilot/<meeting-id-first-8-chars>-<kebab-case-title-max-40-chars>`
  Example: `callpilot/abc123-fix-session-expiry-safari`
- Apply the file changes on that branch
- Create a pull request with:
  - **title:** `[CallPilot] <item title>`
  - **draft:** `true` (always — human reviews AI code before merging)
  - **labels:** value of `pr.label` from config (default: `callpilot`)
  - **body:**
    ```
    ## Summary
    <item description>

    ## Client Feedback
    > "<client_quote>"
    — <speaker>, <meeting title> (<meeting date>)

    ## What Changed
    - <bullet: file changed and what was done>
    - <bullet: file changed and what was done>

    ## Review Notes
    This PR was auto-generated by CallPilot from a client call transcript.
    Please review carefully before merging.

    ---
    Meeting: <transcript_url>
    Generated: <ISO8601 timestamp>
    ```

Output: `✅ Draft PR opened: <pr_url>`

---

### Action: Slack Ping

For `bug` or `feature_enhancement` with `complexity: complex`.

Use Slack MCP to send a message to the routed team's `slack_channel`:

```
:rotating_light: *CallPilot — Action Required*

*<item title>*
Type: <type> | Complexity: complex | From: <meeting title> (<meeting date>)

*Client said:*
> "<client_quote>"
— <speaker>

*Why this needs your attention:*
<item description>

This item is too complex for auto-implementation and needs human scoping before work can begin.

Meeting transcript: <transcript_url>

<oncall> please triage within 24h.
```

Output: `💬 Slack ping sent to <slack_channel> (<oncall>)`

---

### Action: Backlog Issue

For `new_feature` at any complexity.

Use GitHub MCP to create an issue in `backlog_repo`:

- **title:** `[CallPilot] <item title>`
- **labels:** `callpilot`, `feature-request`
- **body:**
  ```
  ## Feature Request

  **Requested by:** <speaker> in *<meeting title>* (<meeting date>)

  **What they asked for:**
  > "<client_quote>"

  **Description:**
  <item description>

  ---
  Meeting transcript: <transcript_url>
  Generated by CallPilot on <ISO8601 timestamp>
  ```

Output: `📋 Backlog issue created: <issue_url>`

---

### Action: Fallback

When routing confidence is below `routing.confidence_threshold`.

Apply `fallback.strategy` from config:

**`slack_channel`** — send to `fallback.slack_channel`:
```
:callpilot: *CallPilot — Needs Triage*

Could not confidently route the following item from *<meeting title>*:

*<item title>*
Type: <type> | Complexity: <complexity>

*Client said:*
> "<client_quote>"
— <speaker>

Routing was ambiguous (confidence: <score>, threshold: <threshold>).
Routing hint used: `<routing_hint>`

Please assign to the right team.
Meeting: <transcript_url>

<fallback.mention>
```

**`github_issue`** — create an issue in `backlog_repo` with:
- title: `[CallPilot] NEEDS TRIAGE: <item title>`
- labels: `callpilot`, `callpilot-needs-triage`
- body: same context as the Slack message above

**`ping_both`** — do both of the above.

Output: `⚠️  Fallback triggered for "<title>" — <action taken>`

---

## Run Summary

After processing all meetings, output this summary:

```
── CallPilot Run Complete ────────────────────────────────────
Meetings processed: <N>
Items extracted:    <total>

  Draft PRs opened:  <count>
  Slack pings sent:  <count>
  Backlog issues:    <count>
  Fallback routed:   <count>

Details:
<for each item, one line with icon + title + what was done + link>
  ✅  "Fix session expiry on Safari"     → PR #42   github.com/org/auth-service/pull/42
  💬  "Add date range filter to search"  → Slack    #team-data (@raj)
  📋  "Mobile app for field team"        → Issue #7  github.com/org/product-roadmap/issues/7
─────────────────────────────────────────────────────────────
```

If zero meetings were found:
```
── CallPilot ─────────────────────────────────────────────────
No new meetings found since <last_run>.
Next run will check for meetings after <current_timestamp>.
─────────────────────────────────────────────────────────────
```
