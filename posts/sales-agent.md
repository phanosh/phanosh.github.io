# I Built a Sales Agent With Claude Code and a JSON File

I'm not a salesperson. I'm a founder who has to do sales, which is a different thing entirely. The main failure mode isn't bad pitches — it's losing track of 18 conversations happening in parallel across months, forgetting who said what, and sending a follow-up that reveals you've clearly forgotten the context of the last call.

So I built a small agent to handle that. Here's how it works.

---

## The data model

Everything lives in a single `pipeline.json` file. No database, no SaaS CRM. Each client is a keyed object with roughly 18 top-level fields — standard CRM stuff like `status`, `deal_value`, `contacts`, `tags` — plus two arrays that do the heavy lifting: `emails` and `follow_ups`.

Each email entry stores 13 fields: `date`, `from`, `to`, `cc`, `subject`, `body_full`, `body_summary`, `direction`, `sentiment`, `intent`, `action_items`, and `attachments`. The last five are AI-generated and left blank at ingestion time. More on that in a second.

The file currently has 18 clients and 1,329 emails. It's about 512KB. Completely manageable.

---

## Email ingestion

Emails come in via the Gmail API. A Python script (`gmail_sync.py`) runs daily at 10am via a macOS launchd plist. It authenticates with OAuth2 using a stored `token.json`, then for each client it builds a Gmail search query:

```
(from:contact@email.com OR to:contact@email.com) after:YYYY/MM/DD
```

The `after` date comes from `metadata.last_gmail_sync` in the pipeline — so each run is incremental. For deduplication, it checks `gmail_id` first (for emails ingested via the API), then falls back to a `(subject, date)` pair match for older emails that were originally imported from an `.mbox` file.

New emails are appended with the AI fields (`body_summary`, `sentiment`, `intent`, `action_items`) left blank. The ingest step is intentionally dumb — it just gets the data in.

---

## The slash commands

The enrichment layer lives in Claude Code as a set of slash commands. Each command is a markdown file with detailed instructions that Claude follows as a structured agent task. There are four of them:

**`/fresh`** — goes through every email added since the last run and fills in the blank AI fields. `direction` is rule-based (`outbound` if the `from` address contains `@2050-materials.com`). Everything else — `body_summary`, `sentiment`, `intent`, `action_items` — is inferred from `body_full`. It also updates client-level fields: `status`, `latest_status.summary`, `latest_status.next_action`, and `latest_status.last_contact_date`. Incremental by design: it reads `metadata.last_fresh_run` as a cutoff date and skips any email on or before it.

**`/enrich`** — fills in the business/CRM layer: `deal_value`, `product_interest`, `source`, `tags`, `notes`, `expected_close`, `follow_ups`, and `deal_stages`. The static fields (`tags`, `notes`, etc.) are fill-if-empty — set once, not overwritten. The dynamic fields (`follow_ups`, `deal_stages`, `expected_close`) are re-evaluated on every run from new emails only, which prevents duplicates accumulating over time. Also incremental via `metadata.last_enrich_run`.

**`/research`** — runs web searches for each client and appends findings to a `research` array. It derives 2–4 search queries per client based on deal stage, industry, and the content of recent emails, then stores structured results: `title`, `url`, `source`, `description`, and a `relevance` field explaining why this specific article matters for this deal. Skips any client where `researched_at` is already set.

**`/weekly-bd`** — generates a weekly markdown report. It reads only `latest_status` fields (not raw emails), writes a status section per client, and drafts outreach emails for deals where `next_action` implies reaching out. It applies logic to skip draft generation when the next action is internal (e.g. "wait for their decision") or the deal is closed. Output is a dated `.md` file: `weekly-bd/YYYY-MM-DD_weekly-bd.md`.

---

## Scheduling

Two launchd plists handle the automation:

- `com.2050materials.gmailsync.plist` — runs `gmail_sync.py` at 10:00am daily
- `com.2050materials.weeklybd.plist` — runs every Monday at 12:00pm

The weekly BD job calls a small shell wrapper (`weekly_bd_run.sh`) that sets the correct `PATH` (the `claude` binary lives in `~/.local/bin`, which launchd doesn't know about) and then runs:

```bash
claude --print "/weekly-bd"
```

That's it. Claude Code runs non-interactively, executes the skill, writes the file, exits.

---

## The write-after-each-client pattern

One thing that turned out to matter: every slash command writes `pipeline.json` to disk after finishing each client, not at the end of the full run. This was a deliberate design decision. With 18 clients and hundreds of emails, a full run can take several minutes. If something fails halfway through, you don't lose everything. The incremental cutoff dates also mean re-running is safe — already-processed clients are skipped cleanly.

---

## What it actually does

Each morning: new emails are pulled in and the raw data is stored. When I open Claude Code, I run `/fresh` (fills AI fields for new emails only, usually fast) then `/enrich` (updates CRM fields from new context). On Mondays the weekly report generates automatically. When I add a new prospect, I run `/research` to get recent news and context before reaching out.

The output of the Monday report is a markdown file with a one-paragraph pipeline overview, per-client status sections, and draft outreach emails for deals that need a nudge that week. I open it, edit the drafts lightly, and send.

---

## Next steps

Make `/research` status-aware. Right now it's gated by a `researched_at` flag per client, so you have to manually clear it if a deal status changes and the research angle shifts. The cleaner trigger is re-running research automatically whenever `status` changes.

Fix the Gmail search to catch thread participants, not just the original contact. If someone new at the company writes in on the same thread, it currently misses them. Known gap, next to fix.

---

The whole thing is about 600 lines of Python, four markdown files, two plists, and a shell script. The JSON file is the database, Claude Code is the AI layer, and the slash commands are the interface. It's been running for a few weeks and it's already changed how I approach the pipeline — less reactive, more structured.
