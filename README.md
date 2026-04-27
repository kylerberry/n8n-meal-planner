# Family Meal Planner

An automated weekly meal planning system powered by n8n, Claude, Spoonacular, Supabase, and Telegram. Proposes real, popular recipes every Saturday, lets the family vote via Telegram buttons, then generates full recipes and a consolidated grocery list once enough meals are approved.

## How It Works

```
Saturday 9am (or /planmeals)
  Spoonacular → 24 popular recipes
  Claude Haiku → selects best 4 based on family preferences + ingredient overlap
  Telegram → photo album + voting buttons

Async (anytime)
  Family taps ✅ or ❌ in Telegram
  First tap wins (atomic PostgreSQL RPC)
  ❌ = "never suggest this again"
  2 approvals → auto-triggers finalize

Finalize (auto-triggered)
  Claude → full recipe for each approved meal
  Claude → semantic grocery list deduplication
  Telegram → per-meal recipe cards + consolidated grocery list

2am daily
  Stale pending proposals (>8 days) auto-expired
```

## The 4 Workflows

### 1. Propose (`Meal Planner Propose.json`)

**Trigger:** Saturday 9am cron OR `/planmeals` Telegram command

**Flow:**
- Guards against duplicate proposals (checks if active proposals already exist for the current week)
- Fetches family profile (allergies, dislikes, equipment, cuisine preferences, target approvals)
- Fetches last 3 weeks of meal history (for repeat avoidance)
- Calls Spoonacular `complexSearch` API for 24 popular recipes matching family constraints
- Claude Haiku selects the best N candidates (default: 4) optimized for ingredient overlap
- Code node scores all k-subsets for shared ingredients, surfaces top "budget combos"
- Inserts proposals into Supabase with status `pending`
- Sends photo album + interactive inline keyboard to Telegram group

**Key nodes:** Schedule Trigger, Execute Workflow Trigger, Check Active Proposals, Search Spoonacular, Format Spoonacular Results, Generate Candidates (Claude), Structured Output Parser, Parse + Score Overlap, Insert Proposals, Format Telegram Message, Send Photo Album, Send to Telegram

### 2. Approval Handler (`Meal Planner Approval Handler.json`)

**Trigger:** Telegram webhook (receives all bot updates)

**Flow:**
- Routes incoming updates: `/planmeals` command vs. approval button tap vs. unknown
- **`/planmeals` path:** Checks for existing proposals → sends "already proposed" or "generating..." → triggers Workflow 1
- **Approval path:** Pre-checks if target approvals already met → calls atomic `approve_proposal()` RPC → shows toast → counts approvals
- **Deny path:** After successful deny, fetches the proposal and inserts it into `meal_history` with `feedback: 'rejected'` (prevents future re-suggestion)
- When target approvals reached: sends "Meals locked in!" message → triggers Workflow 3

**Key nodes:** Telegram Webhook, Parse Callback, Is Planmeals?, Is Valid Callback?, Approve Proposal RPC, First Click Wins?, Is Deny?, Get Denied Proposal, Insert Rejected History, Count Approvals, Trigger Finalize

### 3. Finalize (`Meal Planner Finalize.json`)

**Trigger:** Called by Workflow 2 when approvals are locked in (also has a test webhook)

**Flow:**
- Fetches family profile + approved proposals for the most recent week
- Claude Haiku generates full recipe (ingredients with sections, numbered steps with timing, prep-ahead notes) for each approved meal
- Merges recipe data back into proposal rows, updates Supabase
- Code node consolidates grocery list (mechanical merge with unit normalization)
- Claude Haiku semantically deduplicates the grocery list (merges "shredded cheese" + "shredded cheddar cheese", normalizes "tsp" vs "teaspoon")
- Formats per-meal recipe messages (title, source URL, description, per-serving macros, full instructions)
- Sends messages sequentially via SplitInBatches (guarantees Telegram delivery order)
- Inserts approved meals into `meal_history` for future repeat avoidance

**Key nodes:** Execute Workflow Trigger, Get Approved Proposals, Generate Full Recipe (Claude), Recipe Output Parser, Merge Recipe Data, Consolidate Grocery List, Deduplicate Grocery List (Claude), Format Final Message, Send One at a Time, Send Meal Plan, Insert meal_history

### 4. Nightly Cleanup (`Meal Planner Nightly Cleanup.json`)

**Trigger:** Daily at 2am

**Flow:**
- Queries for pending proposals older than 8 days
- If any found, sets their status to `expired`

**Key nodes:** Schedule Trigger, Get Stale Proposals, Any Stale?, Expire Proposals

## Prerequisites

- [n8n](https://n8n.io) (self-hosted or cloud)
- [Supabase](https://supabase.com) project (free tier works)
- [Telegram Bot](https://core.telegram.org/bots#botfather) added to a group chat
- [Spoonacular API key](https://spoonacular.com/food-api) (free tier: 150 requests/day)
- [Anthropic API key](https://console.anthropic.com/) (Claude Haiku)

## Setup

### 1. Database (Supabase)

Run the migrations in order against your Supabase project:

```bash
# From the supabase/ directory, or paste into the Supabase SQL Editor
psql < migrations/20260419000000_init.sql
psql < migrations/20260419000001_add_recipe_url_columns.sql
psql < migrations/20260419000002_add_target_approvals.sql
```

This creates:
- `family_profile` — single-row config table (members, allergies, dislikes, equipment, cuisine prefs)
- `proposals` — weekly meal candidates with status tracking
- `meal_history` — finalized/rejected meals for repeat avoidance
- `approve_proposal()` — atomic RPC function for first-click-wins approval

Then seed your family profile:

```bash
# Edit supabase/seed.sql with your family's details first
psql < seed.sql
```

### 2. Telegram Bot

1. Create a bot via [@BotFather](https://t.me/BotFather)
2. Add the bot to your family group chat as admin
3. Register the command: `/setcommands` → `planmeals - Generate new meal proposals`
4. Get your group's `chat_id` (send a message, then check `https://api.telegram.org/bot<TOKEN>/getUpdates`)

### 3. n8n Credentials

Create these credentials in n8n (Settings → Credentials):

| Credential | Type | Notes |
|-----------|------|-------|
| Supabase | Supabase API | Project URL + service role key |
| Anthropic | Anthropic API | Your Anthropic API key |

### 4. n8n Environment Variables

Set these in n8n (Settings → Environment Variables):

| Variable | Value |
|----------|-------|
| `TELEGRAM_CHAT_ID` | Your group chat ID |
| `TG_USER_MAP` | Optional: `{"username":"Display Name"}` |

### 5. Import Workflows

1. Import all 4 JSON files from the `workflows/` directory into n8n
2. **Search and replace** these placeholders in each workflow:

| Placeholder | Replace With |
|------------|--------------|
| `YOUR_TELEGRAM_BOT_TOKEN` | Your Telegram bot token |
| `YOUR_SPOONACULAR_API_KEY` | Your Spoonacular API key |
| `YOUR_SUPABASE_CREDENTIAL_ID` | The ID of your Supabase credential in n8n |
| `YOUR_ANTHROPIC_CREDENTIAL_ID` | The ID of your Anthropic credential in n8n |
| `WORKFLOW_1_PROPOSE_ID` | The n8n ID of your imported Propose workflow |
| `WORKFLOW_3_FINALIZE_ID` | The n8n ID of your imported Finalize workflow |

3. Update credential references in all Supabase and Anthropic nodes to point to your credentials

### 6. Register Telegram Webhook

Point your bot's webhook at Workflow 2's webhook URL:

```bash
curl "https://api.telegram.org/botYOUR_TOKEN/setWebhook?url=https://your-n8n-domain.com/webhook/meal-planner-approval"
```

### 7. Activate Workflows

Activate all 4 workflows in n8n. The system is now live:
- Workflow 1 fires every Saturday at 9am
- Workflow 2 listens for Telegram updates 24/7
- Workflow 3 is triggered by Workflow 2 when approvals lock in
- Workflow 4 cleans up stale proposals nightly at 2am

## Configuration

All configuration lives in the `family_profile` table:

| Column | Type | Purpose |
|--------|------|---------|
| `members` | jsonb | Family members with ages and notes |
| `allergies` | text[] | Hard constraints (non-negotiable) |
| `dislikes` | text[] | Soft avoidance |
| `equipment` | text[] | Available kitchen equipment (e.g., `air_fryer`, `instant_pot`) |
| `cuisine_prefs` | jsonb | `{liked: ["Mexican", "Italian"], disliked: []}` |
| `portion_notes` | text | Serving notes for Claude |
| `target_approvals` | int | Meals needed per week (default: 2). Proposal count = 2x this value |

## Project Structure

```
meal-planner/
├── README.md
├── CLAUDE.md                 # Claude Code project instructions
├── DECISIONS.md              # Architecture decision log
├── meal-planner-n8n-spec.md  # Original build spec
├── supabase/
│   ├── seed.sql              # Family profile seed data
│   └── migrations/
│       ├── 20260419000000_init.sql
│       ├── 20260419000001_add_recipe_url_columns.sql
│       ├── 20260419000002_add_target_approvals.sql
│       └── 20260425000001_expire_stale_proposals.sql
└── workflows/
    ├── Meal Planner Propose.json
    ├── Meal Planner Approval Handler.json
    ├── Meal Planner Finalize.json
    └── Meal Planner Nightly Cleanup.json
```

## Cost

| Service | Cost |
|---------|------|
| n8n | Self-hosted (or free cloud tier) |
| Supabase | Free tier |
| Spoonacular | Free tier (150 req/day, we use ~1/week) |
| Claude Haiku | ~$0.05/week (~5 calls) |
| Telegram | Free |

## License

MIT
