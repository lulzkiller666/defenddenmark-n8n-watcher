# Defend Denmark / Iceland — Job Watcher (n8n)

Polls the Defend Iceland bug-bounty API every 30 minutes and posts a **Slack** message
whenever a **new program** appears (Denmark + Iceland). Runs entirely on your own n8n.

- **Data source:** `GET https://burp-api.defendiceland.is/customers` with header `X-Api-Key: <your key>`
- **New = a `programId` not seen before** (tracked in an n8n Data Table)
- No scraping, no login cookie, no expiry.

---

## Requirements
- **Self-hosted n8n** (the built-in **Data Tables** feature is used — available in recent n8n).
- A **Defend Iceland API key** (from your account — the same key the Burp extension uses).
- A **Slack** app/bot credential, and a channel to post to (invite the bot to it).

---

## Setup (≈5 minutes)

### 1. Import the workflow
n8n → **Workflows → Import from File** → select `Defend Denmark Job Watcher-SHARE.json`.
It imports **inactive**. Credentials / tables / channel come in blank — you fill them below.

### 2. Create the API-key credential
**Credentials → New → "Header Auth"**
- **Name:** `DefendDenmark API Key`
- **Header Name:** `X-Api-Key`
- **Value:** your API key

Then open the **Fetch Programs** node → *Credential for Header Auth* → select it.

### 3. Create two Data Tables
n8n → **Data Tables → New**, create both with these exact columns:

**`dd_known_programs`**
| column | type |
|---|---|
| program_id | string |
| name | string |
| country | string |
| payout | string |
| url | string |

**`dd_state`**
| column | type |
|---|---|
| singleton | string |
| denmark_count | number |
| iceland_count | number |
| session_alerted | boolean |

Then in the workflow, open **Get Known Programs** + **Remember Program** and point them at
`dd_known_programs`; open **Get State** + **Save State** and point them at `dd_state`.

### 4. Wire up Slack
Open each of the 3 Slack nodes (**Slack: New Program**, **Slack: Session Expired**,
**Slack: Session Restored**):
- Select your Slack credential.
- Set the channel (invite the bot to that channel first, or you'll get `channel_not_found`).

### 5. Seed the "known" table (so the first run doesn't alert on everything)
The table starts empty, so a first run would treat **all** current programs as new.
To seed silently:
1. Right-click **Slack: New Program** → **Deactivate** (a deactivated node just passes data through).
2. Click **Execute Workflow** once. This records every current program into `dd_known_programs`
   and creates the state row — **without** sending any Slack messages.
3. Right-click **Slack: New Program** → **Activate** again.

*(Alternative: skip this and just accept a one-time burst of "new program" messages on first run.)*

### 6. Go live
Toggle the workflow **Active**. It now checks every 30 min and only pings you on genuinely new programs.

---

## How it works
```
Every 30 min → Fetch Programs (API) → Get Known Programs → Get State
  → Detect New Programs (parse JSON, diff programId)
  → Route Action ─┬─ new_program      → Slack: New Program → Remember Program
                  ├─ session_expired  → Slack: Session Expired  (fires on API error / bad key)
                  ├─ session_restored → Slack: Session Restored
                  └─ update_state     → Save State
```
- **Detect New Programs** flattens `communities[].customers[]`, compares each `programId`
  against `dd_known_programs`, and emits an alert per new one (with name, country,
  in-scope/asset counts, and a program link).
- If the API returns no valid `communities` (e.g. invalid/expired key, or the API is down),
  it sends **one** "API error" Slack message and sets a flag so it doesn't repeat every run;
  it sends a "restored" message once it recovers.

## Tuning
- **Interval:** edit the **Every 30 Minutes** schedule node.
- **Alert content:** edit the text in the Slack nodes.

## Notes
- No secrets are stored in the workflow JSON — credentials live in n8n's encrypted store.
