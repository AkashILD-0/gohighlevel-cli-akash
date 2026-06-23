# GoHighLevel CLI

A production-grade command-line interface for [GoHighLevel](https://www.gohighlevel.com/) — manage contacts, pipelines, workflows, calendars, conversations, and more from your terminal or from AI agents like Claude.

Built and maintained by [Infinite Labs Digital](https://infinitelabsdigital.com).

---

## What it does

| Command Group | What you can do |
|---|---|
| `contacts` | List, get, create, update, delete, search, tag |
| `opportunities` | List pipelines and deals, update stages |
| `calendars` | List calendars, check slots, manage appointments |
| `workflows` | List workflows, enroll/remove contacts, **create workflows programmatically** |
| `conversations` | Read threads, send SMS / Email / WhatsApp |
| `emails` | List campaigns and templates |
| `payments` | Transactions, invoices, orders |
| `forms` | List forms, read submissions |
| `social` | Social media accounts, posts, analytics |
| `locations` | Sub-account info, custom fields |
| `documents` | Contracts, proposals |

---

## Requirements

- Python 3.10+
- A GHL Private Integration API key ([how to get one](#configuration))
- Your GHL Location ID ([how to find it](#configuration))

---

## Installation

### Windows

```powershell
# 1. Clone the repo
git clone https://github.com/AkashILD-0/gohighlevel-cli.git
cd gohighlevel-cli

# 2. Create a virtual environment and install
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -e .

# 3. Copy the env template
Copy-Item .env.example .env
```

### Mac / Linux

```bash
git clone https://github.com/AkashILD-0/gohighlevel-cli.git
cd gohighlevel-cli
./install.sh
```

---

## Configuration

Copy `.env.example` to `.env` and fill in your values:

```env
GHL_API_KEY=pit-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
GHL_LOCATION_ID=YOUR_LOCATION_ID
GHL_FIREBASE_REFRESH_TOKEN=          # Optional — only needed for workflow creation
```

### Where to find each value

**`GHL_API_KEY`**
1. Log into GHL → **Settings → Integrations → Private Integrations**
2. Click **Create New**
3. Give it all the scopes you need (contacts, workflows, conversations, etc.)
4. Copy the token — it starts with `pit-`

**`GHL_LOCATION_ID`**
The long ID visible in your GHL URL after `/location/`, e.g.:
`app.gohighlevel.com/v2/location/YOUR_LOCATION_ID/...`

**`GHL_FIREBASE_REFRESH_TOKEN`** *(only needed for creating/updating workflows)*
See [Getting the Firebase token](#getting-the-firebase-token) below.

> **Security:** `.env` is gitignored — your keys never leave your machine.

---

## Usage

### Quick examples

```bash
# List contacts
ghl contacts list --limit 10

# Search for a contact
ghl contacts search "john smith"

# Create a contact
ghl contacts create --email patient@example.com --first-name John --last-name Smith

# List all workflows
ghl workflows list

# List open opportunities in a pipeline
ghl opportunities list --pipeline-id YOUR_PIPELINE_ID

# Send an SMS
ghl conversations send CONVERSATION_ID --type SMS --message "Thanks for your visit!"

# Send an email
ghl conversations send CONVERSATION_ID --type Email --message "We'd love your feedback"

# List available calendar slots
ghl calendars slots CALENDAR_ID --start 2026-07-01 --end 2026-07-07

# List recent transactions
ghl payments transactions --limit 20
```

### JSON output (for scripting / agents)

Add `--json` before any command for machine-readable output:

```bash
ghl --json contacts list --limit 50
ghl --json opportunities list
ghl --json workflows list
```

### Interactive REPL

Run `ghl` with no arguments to enter an interactive shell with autocomplete:

```bash
ghl
```

---

## Creating Workflows Programmatically

The `--experimental` flag unlocks workflow creation via GHL's internal API.

### Getting the Firebase token

1. Open [app.gohighlevel.com](https://app.gohighlevel.com) in Chrome (logged in)
2. Press **F12** → go to **Application** tab
3. Navigate: **IndexedDB → firebaseLocalStorageDb → firebaseLocalStorage**
4. Click the row, expand **value → stsTokenManager**
5. Copy the **`refreshToken`** value
6. Paste into `.env` as `GHL_FIREBASE_REFRESH_TOKEN=`

> Tokens auto-refresh once set. Re-run only if you get an "expired/revoked" error.

### Using the workflow builder

The `builders/` directory contains Python scripts that create full multi-step workflows programmatically. Each builder reads a campaign definition and pushes it to GHL as a DRAFT workflow.

**Example: create a Google Review request workflow**

```python
import os
from cli_anything.gohighlevel.utils.workflow_builder import (
    CampaignBuilder, email_step, sms_step, wait_step, tag_step, link_steps
)
from cli_anything.gohighlevel.utils.ghl_internal_client import InternalGHLClient, TokenManager

email_body = """
Hi {{contact.first_name}},

Thank you for your recent visit. We'd love to hear your feedback.

Leave a review here: https://g.page/r/YOUR_REVIEW_LINK/review

— Your Team
"""

steps = [
    wait_step("Wait 5 min", 5, "minutes"),
    email_step("Review Request", "We'd love your feedback!", email_body, from_name="Your Clinic"),
    wait_step("Wait 3 days", 3, "days"),
    sms_step("Follow-up nudge", "Did you get a chance to leave us a review? https://g.page/r/YOUR_REVIEW_LINK"),
    tag_step("Remove trigger tag", ["trigger-tag"], remove=True),
]

campaign = {
    "My Workflow": {
        "name": "My Workflow",
        "tag": "trigger-tag",        # GHL tag that fires this workflow
        "templates": link_steps(steps),
    }
}

client  = InternalGHLClient(TokenManager(), os.environ["GHL_LOCATION_ID"])
builder = CampaignBuilder(client)
result  = builder.build(campaign, folder_name="My Folder")
print(result)
# → {"workflows_created": 1, "steps_saved": 5, "triggers_created": 1, ...}
```

### Available step types

| Function | GHL action |
|---|---|
| `email_step(name, subject, text)` | Send email |
| `sms_step(name, body)` | Send SMS |
| `wait_step(name, value, unit)` | Wait (minutes / hours / days) |
| `tag_step(name, tags, remove=False)` | Add or remove contact tags |

All 56 GHL action types are supported — see `VERIFIED_ACTIONS` in `workflow_builder.py`.

---

## Using with Claude / AI Agents

This CLI is designed to be called by AI agents. Install the included skill:

```powershell
# Windows
New-Item -ItemType Directory -Force "$HOME\.claude\skills\gohighlevel-cli"
Copy-Item "cli_anything\gohighlevel\skills\SKILL.md" "$HOME\.claude\skills\gohighlevel-cli\SKILL.md"
```

```bash
# Mac / Linux
mkdir -p ~/.claude/skills/gohighlevel-cli
cp cli_anything/gohighlevel/skills/SKILL.md ~/.claude/skills/gohighlevel-cli/SKILL.md
```

Claude Code will then auto-discover the skill when you say anything GHL-related.

---

## Project structure

```
gohighlevel-cli/
├── cli_anything/
│   └── gohighlevel/
│       ├── gohighlevel_cli.py          # Main CLI (11 command groups)
│       ├── skills/SKILL.md             # Claude Code skill manifest
│       └── utils/
│           ├── ghl_client.py           # Public REST API client
│           ├── ghl_internal_client.py  # Internal Firebase API client
│           ├── workflow_builder.py     # Workflow DSL + step builders
│           └── repl_skin.py            # Interactive REPL
├── builders/                           # Ready-to-run workflow builder scripts
├── docs/
│   └── get-firebase-token.md           # How to get the Firebase token
├── .env.example                        # Environment variable template (no secrets)
├── install.sh                          # One-shot installer (Mac/Linux)
└── setup.py                            # Package definition
```

---

## Tested on

- Windows 11 + Python 3.13
- macOS + Python 3.10+

---

## License

MIT
