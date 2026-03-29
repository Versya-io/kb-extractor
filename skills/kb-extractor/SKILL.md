---
name: kb-extractor
description: >
  Scans a codebase and generates structured markdown knowledge base files optimized
  for AI support chatbots and marketing teams. Supports sub-commands: scan (full
  codebase extraction), db (resolve database-dependent gaps), compile (consolidate
  and translate all files into one), faq (generate FAQ from KB), tokens (estimate
  token count and generate deployment report), and all (run everything in sequence).
  Use when asked to document features, generate a knowledge base, extract product
  capabilities, map UI constraints, create support documentation, build an LLM-ready
  feature reference, generate FAQ, count tokens, or translate product docs.
allowed-tools:
  - Glob
  - Grep
  - Read
  - Write
  - Bash
---

# Knowledge Base Extractor

You are a senior product analyst and technical writer. Your job is to scan a codebase
and produce structured, LLM-optimized markdown knowledge base files that a support
chatbot can use to answer user questions with perfect accuracy — including knowing
what is impossible or restricted.

## Invocation

```
/kb-extractor <command> <output_dir> [language]
```

| Command | What it does |
|---------|-------------|
| `scan` | Full codebase scan → writes all KB files |
| `db-check` | Detects available DB connections, generates db-prompt.md — **never queries** |
| `db-run` | Executes queries to resolve gaps — **requires explicit flag** |
| `faq` | Generates 30+ FAQ questions from existing KB files |
| `compile` | Consolidates all KB files into one file, translates if language given |
| `tokens` | Estimates token count for all KB files and writes `token-report.md` |
| `all` | Runs scan → db-check → faq → compile → tokens in sequence |

**Examples:**
```
/kb-extractor scan .kb/
/kb-extractor db-check .kb/
/kb-extractor db-run .kb/ --container tabloide-db
/kb-extractor db-run .kb/ --mcp db-production
/kb-extractor db-run .kb/ --connection "postgresql://readonly:pass@host:5432/db"
/kb-extractor db-run .kb/ --manual
/kb-extractor compile .kb/ portuguese
/kb-extractor faq .kb/
/kb-extractor tokens .kb/
/kb-extractor all .kb/ spanish
```

Parse `$ARGUMENTS` to extract command, output_dir, flags, and optional language.
If no command is given, default to `scan`. If no output_dir is given, default to `.kb/`.

---

## COMMAND: scan

### Phase 1 — Orientation

Before scanning, orient yourself:

1. Read `package.json`, `composer.json`, or equivalent to understand the tech stack
2. Read `README.md` if it exists
3. Read `CLAUDE.md` or `.claude/` config if it exists
4. List top-level directories to understand project structure
5. Identify the frontend layer (`pages/`, `views/`, `screens/`, `components/`,
   `routes/`, Next.js `app/`, Laravel Blade, etc.)
6. Identify the auth/permissions layer (`middleware/`, `policies/`, `guards/`,
   `abilities/`, `roles/`, feature flags, plan gates)
7. Identify the API/backend routes layer

Write a one-paragraph internal summary of what this product does before proceeding.

---

### Phase 2 — UI Surface Inventory

Scan every screen, page, route, and modal the end user can interact with.

For each screen/page found:

```
Screen: [name]
Route: [path or identifier]
Purpose: [what the user does here — one sentence]
Key interactive elements: [list every button, input, toggle, dropdown, drag target]
Navigation from here: [where the user can go next]
Access control: [who can see this — all users / plan X / role Y / condition Z]
```

Use Glob to find all view/template/page files, then Read each one to extract elements.

---

### Phase 3 — Feature Extraction

For each meaningful feature discovered, produce a feature block.
A "feature" is anything a user can DO — not a code module or class.

```markdown
## Feature: [Feature Name]

**What it does:** [One sentence in plain user language. No technical jargon.]

**How to access:**
1. [Exact step]
2. [Exact step]

**Works when:**
- User is on plan: [Free / Pro / All]
- User role is: [Admin / Member / All]
- Prerequisite state: [e.g. "at least one flyer created"]

**Does NOT work when:**
- [Explicit condition where this feature is unavailable or disabled]
- [Hard limit: e.g. "maximum 10 items — adding more shows error X"]

**UI state when unavailable:**
- [Button disabled / option hidden / modal blocked / redirects to upgrade]
- [Exact error message shown if known]

**What to suggest instead:**
- [If unavailable, what should the user do?]

**Known edge cases:**
- [Any non-obvious behavior discovered in the code]

**confidence:** high | medium | low
```

Actively search for constraints:
```bash
# Plan/role gates
grep -r "plan\|subscription\|role\|feature_flag\|gate\|ability\|can(" \
  --include="*.{js,ts,jsx,tsx,vue,php,rb,py}" -l

# Disabled UI states
grep -r "disabled\|isDisabled\|locked\|hidden\|readOnly" \
  --include="*.{jsx,tsx,vue,blade.php,erb}" -l

# Hard limits
grep -r "maxLength\|max_items\|limit\|quota\|minimum\|maximum" \
  --include="*.{js,ts,jsx,tsx,vue,php,rb,py}" -l

# Error messages
grep -r "toast\|error\|alert\|notify\|message" \
  --include="*.{js,ts,jsx,tsx,vue}" -l
```

---

### Phase 4 — DB Gap Detection

While scanning, whenever you encounter logic that depends on runtime database values
(feature flags stored in DB, plan limits from a plans table, dynamic config, tenant
settings), you CANNOT determine the value from code alone.

**Do NOT guess.** Collect every such gap and write `[output_dir]/db-gaps.md`:

```markdown
---
kb_type: db-gaps
status: pending
generated_at: [ISO date]
---

# Database-Dependent Gaps

These items were found in the codebase but their values depend on the database.
This file can be used in three ways:
1. Answer questions manually and paste answers back into this file
2. Pass this file as a prompt to an LLM running on the production machine
3. Connect via MCP to resolve automatically (run `/kb-extractor db [output_dir]`)

## Gaps

### GAP-001
**Feature affected:** [feature name]
**File:** [path:line]
**What the code checks:** `user.plan.features.includes('logo_customization')`
**Question:** Which plans have `logo_customization` enabled? What are the exact plan names?
**Answer:** _(fill in here)_

### GAP-002
**Feature affected:** [feature name]
**File:** [path:line]
**What the code checks:** `workspace.limits.max_flyers`
**Question:** What is the value of `max_flyers` for each plan tier?
**Answer:** _(fill in here)_
```

Number each gap sequentially. Quote the exact code expression being checked.

---

### Phase 5 — Constraint Map

Write `[output_dir]/constraints.md`:

```markdown
---
kb_type: constraint
product: [product name]
generated_at: [ISO date]
---

# Product Constraint Map

## Plan-Based Restrictions
| Feature | Free | Starter | Pro | Notes |
|---------|------|---------|-----|-------|
| [feature] | ✗ | ✓ | ✓ | [note] |
_(mark cells as "?" where answer is in db-gaps.md)_

## Hard Limits
| Item | Limit | Behavior when exceeded |
|------|-------|----------------------|
| [item] | [value] | [what happens] |

## Structurally Impossible Actions
Things users may ask about that simply do not exist in the product.
The chatbot must NEVER say "yes" to these.

- **[Action]:** Not possible. Suggest [alternative].

## Known Error Messages
| Message | When it appears | What the user should do |
|---------|----------------|------------------------|
| "[exact message]" | [trigger] | [resolution] |
```

---

### Phase 6 — Flow Documentation

For the top 10 most common user actions (infer from UI prominence and route names):

```markdown
---
kb_type: flow
---

## Flow: [Action Name]
**User goal:** [what they're trying to accomplish]

**Steps:**
1. Navigate to [screen]
2. Click [exact button label]
3. [What opens/happens]
4. Fill in: [field list with constraints]
5. Click [confirm]
6. Expected result: [success state]

**Failure states:**
- If [condition]: [what happens and what to do]
```

---

### Phase 7 — Glossary

Write `[output_dir]/glossary.md` with every product-specific term found in the UI:

```markdown
---
kb_type: glossary
---

# Product Glossary

| Term | What it means in this product | Notes |
|------|------------------------------|-------|
| [Term] | [Definition specific to this product, not generic] | [Nuance] |
```

---

### Phase 8 — Output Structure

```
[output_dir]/
  index.md
  features/
    [feature-slug].md
  flows/
    [flow-slug].md
  constraints.md
  glossary.md
  db-gaps.md          ← gaps requiring DB access
  meta.md             ← scan date, commit, scope, gap count
  gaps.md             ← sections needing manual documentation
```

After writing all files, print a summary:
```
✅ Scan complete
   Features documented: N
   Flows documented: N
   Constraints found: N
   DB gaps detected: N → see db-gaps.md
   Manual gaps: N → see gaps.md

Next steps:
  → Fill in db-gaps.md manually, OR
  → Run /kb-extractor db-check [output_dir] to detect DB connections, OR
  → Pass db-gaps.md as a prompt to an LLM on the production machine
  → Then run /kb-extractor faq [output_dir]
  → Then run /kb-extractor compile [output_dir] [language]
```

---

## COMMAND: db-check

Detect available database connections and generate `db-prompt.md`.

**This command NEVER executes any database query.** It only detects what is
available, presents options to the user, and writes `db-prompt.md` so the
user can decide how to proceed. No data is read from any database.

**Syntax:**
```
/kb-extractor db-check [output_dir]
```

**Example:**
```
/kb-extractor db-check .kb/
```

### Step 1 — Read pending gaps

Read `[output_dir]/db-gaps.md` and list all gaps with `status: pending`.

If there are no pending gaps:
```
✅ No pending gaps in db-gaps.md. Nothing to resolve.
```
Stop.

### Step 2 — Detect available connections (READ ONLY — no queries)

Run detection via Bash:

```bash
# 1. Check for database CLI clients
for cli in psql mysql sqlite3 mongosh redis-cli; do
  which $cli 2>/dev/null && echo "cli:$cli" || true
done

# 2. Check for Docker database containers
which docker 2>/dev/null && \
  docker ps --format "{{.Names}}\t{{.Image}}\t{{.Ports}}" 2>/dev/null | \
  grep -iE "postgres|mysql|mongo|redis|mariadb|sqlite|cockroach|cassandra" \
  || echo "docker:none"

# 3. Check for connection env vars (mask credentials)
printenv 2>/dev/null | \
  grep -iE "database_url|db_url|db_host|mongo_url|redis_url|mysql_url" | \
  sed 's/:\/\/[^@]*@/:\/\/***:***@/g'
```

### Step 3 — Always generate db-prompt.md

Generate `[output_dir]/db-prompt.md` with the environment confirmation
checklist and one entry per pending gap:

```markdown
# DB Gap Resolution Prompt
generated: [ISO date]
product: [product name]
pending gaps: [N]

---

## ⚠️ BEFORE YOU BEGIN — Confirm your environment

This prompt must be run against the PRODUCTION database.
Using a development or staging database will generate incorrect KB
constraints and cause the support chatbot to give wrong answers.

Please confirm before proceeding:
- [ ] I am running this on the production machine (or with production DB access)
- [ ] The database I will query is: [detected DB type if known]
- [ ] I understand that incorrect data here will affect chatbot accuracy

If you are NOT on the correct machine, stop and run this on the production server.

---

## Instructions

For each gap below, run the suggested query and paste the result into
the "Answer:" field in db-gaps.md.
Then run: /kb-extractor db-run [output_dir] --manual

---

## Gaps to resolve

### GAP-001
Feature affected: [feature name]
File: [path:line]
What the code checks: `[expression]`
Suggested query: [SQL or equivalent for the detected DB type]
What we need: [plain English description]
Answer: _(paste result here in db-gaps.md)_
```

### Step 4 — Present findings and next steps

```
🔍 DB Check complete — NO queries were executed.

  Pending gaps: [N]
  Written: [output_dir]/db-prompt.md

  Available connections detected:
    Docker containers: [list or "none"]
    CLI clients:       [list or "none"]
    MCP servers:       [list or "none"]
    Env vars:          [masked list or "none"]

  To resolve gaps, run ONE of the following:

  /kb-extractor db-run .kb/ --container [container-name]
  /kb-extractor db-run .kb/ --mcp [mcp-server-name]
  /kb-extractor db-run .kb/ --connection "[connection-string]"
  /kb-extractor db-run .kb/ --manual

  ⚠️  Use production data. Local data will generate incorrect KB constraints.
```

Stop here. Do not query anything.

---

## COMMAND: db-run

Execute database queries to resolve pending gaps in `db-gaps.md`.

**This command requires an explicit flag specifying the data source.**
Without a flag, it refuses to run and directs the user to `db-check`.

**Syntax:**
```
/kb-extractor db-run [output_dir] <--container name | --mcp name | --connection "string" | --manual>
```

**Examples:**
```
/kb-extractor db-run .kb/ --container tabloide-db
/kb-extractor db-run .kb/ --mcp db-production
/kb-extractor db-run .kb/ --connection "postgresql://readonly:pass@prod-host:5432/myapp"
/kb-extractor db-run .kb/ --manual
```

---

### Step 0 — Validate flags

If NO flag is provided, stop immediately:
```
❌ db-run requires an explicit flag to prevent accidental use of wrong database.

  Run /kb-extractor db-check .kb/ first to see available options, then:

  /kb-extractor db-run .kb/ --container [container-name]
  /kb-extractor db-run .kb/ --mcp [mcp-server-name]
  /kb-extractor db-run .kb/ --connection "[connection-string]"
  /kb-extractor db-run .kb/ --manual
```
Do not proceed under any circumstances without a flag.

---

### Step 1 — Read pending gaps

Read `[output_dir]/db-gaps.md` and collect all gaps with `status: pending`.

If there are no pending gaps, stop:
```
✅ No pending gaps in db-gaps.md. Nothing to resolve.
```

If `--manual` was passed and all Answer fields are already filled in,
skip to Step 4 to propagate answers.

---

### Step 2 — Confirm before executing

Regardless of the flag provided, always confirm once before running any query:

```
About to resolve [N] gaps using:
  Source: [--container name / --mcp name / --connection string / manual answers]

⚠️  Is this the PRODUCTION database?
    Using local/development data will generate incorrect KB constraints.

Proceed? (yes / no)
```

Wait for explicit "yes". If "no" or anything else, stop and suggest
running `/kb-extractor db-check` again.

---

### Step 3a — Docker container execution (`--container`)

Detect database type inside the container:
```bash
docker exec [container] printenv 2>/dev/null | \
  grep -iE "postgres|mysql|mongo|redis|db_user|db_pass|db_name|db_host"
```

Adapt query to detected DB type:
```bash
# PostgreSQL
docker exec [container] psql -U [user] -d [dbname] -t -A -c "SELECT ..."

# MySQL / MariaDB
docker exec [container] mysql -u [user] -p[pass] [dbname] -se "SELECT ..."

# MongoDB
docker exec [container] mongosh [dbname] --quiet --eval "db.[col].find(...)"

# Redis
docker exec [container] redis-cli GET [key]

# SQLite
docker exec [container] sqlite3 [path] "SELECT ..."
```

For each pending gap, run the query, write the answer, mark as
`resolved-docker-local` or `resolved-docker-production` based on user
confirmation in Step 2.

**Never write container credentials to any file.**
If a query fails, mark as `status: failed` — never guess.

Proceed to Step 4.

---

### Step 3b — MCP execution (`--mcp`)

Query via the named MCP server.
READ ONLY — never INSERT, UPDATE, DELETE, DROP, or any write operation.

For each pending gap:
1. Infer table/collection and field from the "What the code checks" expression
2. Query via MCP
3. Write answer, mark as `resolved-mcp`

Also write `db-prompt.md` as an audit record, marking each gap
`[resolved via MCP — no action needed]`.

Proceed to Step 4.

---

### Step 3c — CLI / connection string execution (`--connection`)

Adapt to the DB type inferred from the connection string:
```bash
# PostgreSQL
psql "[connection-string]" -t -A -c "SELECT ..."

# MySQL
mysql -h [host] -u [user] -p[pass] [dbname] -se "SELECT ..."

# MongoDB
mongosh "[connection-string]" --quiet --eval "..."

# SQLite
sqlite3 [path] "SELECT ..."
```

**Never write the connection string to any file.**
Use it only within the Bash tool during this session.

Mark gaps as `resolved-cli-local` or `resolved-cli-production`.
Proceed to Step 4.

---

### Step 3d — Manual (`--manual`)

Read all Answer fields already filled in `db-gaps.md`.
If any are still empty, list them and tell the user:
```
[N] gaps still have empty Answer fields:
  GAP-003: [question summary]
  GAP-005: [question summary]

Fill them in db-gaps.md and re-run:
  /kb-extractor db-run .kb/ --manual
```
Stop — do not proceed to Step 4 with incomplete answers.

If all answers are filled in, proceed to Step 4.

---

### Step 4 — Propagate answers

1. Update all affected feature files — replace "?" with resolved values
2. Update `constraints.md` — fill in plan restriction table cells
3. Update `db-gaps.md` — mark all processed gaps with resolved status

If any gap was resolved from a local source, prepend warning to db-gaps.md:
```markdown
> ⚠️ WARNING: Gaps marked `resolved-*-local` were resolved from a LOCAL
> database. Verify against production before publishing this KB.
```

Print summary:
```
✅ DB gaps resolved

  Resolved: [N] gaps
  Source: [container / mcp / cli / manual]
  Environment: [production / local ⚠️]
  Files updated: features/X.md, constraints.md, db-gaps.md

  Next steps:
    /kb-extractor faq [output_dir]
    /kb-extractor compile [output_dir] [language]
```

## COMMAND: faq

Generate a FAQ from all existing KB files in `[output_dir]/`.

### Step 1 — Read all KB files

Read every file in `[output_dir]/` recursively.

### Step 2 — Generate questions

Think as the target user (non-technical, using the product day-to-day).
Generate questions in these categories:

**Getting started** — questions a new user asks in the first 10 minutes

**Common actions** — "How do I..." questions for the top flows documented

**Limitations and confusion** — questions triggered by hitting a wall: plan limits,
disabled features, unexpected behavior. These are the highest-value FAQ items
because they directly prevent support tickets.

**Pricing and plans** — what's available on each plan tier

**Errors and problems** — "Why is X not working?" based on known error messages

### Step 3 — Write answers

Each FAQ answer must:
- Be written in plain, friendly language (no jargon)
- Reference exact button labels and screen names from the KB
- For impossible actions: say so clearly and offer the alternative
- Be 2-5 sentences maximum

### Step 4 — Write output

Write `[output_dir]/faq.md`:

```markdown
---
kb_type: faq
product: [product name]
generated_at: [ISO date]
question_count: N
---

# Frequently Asked Questions

## Getting Started
### [Question]
[Answer]

## Common Actions
### [Question]
[Answer]

## Limitations
### [Question]
[Answer]

## Plans & Pricing
### [Question]
[Answer]

## Errors & Problems
### [Question]
[Answer]
```

Generate a minimum of 30 questions. Prioritize questions that touch constraints
and impossible actions — these cause the most support friction.

---

## COMMAND: compile

Consolidate all KB files into a single file, optionally translated.

### Step 1 — Read all files in order

Read every file completely, in this order:

1. `index.md`
2. `glossary.md`
3. `constraints.md`
4. `features/*.md` (alphabetical)
5. `flows/*.md` (alphabetical)
6. `faq.md`

Skip internal files: `db-gaps.md`, `db-prompt.md`, `meta.md`, `gaps.md`.

**Read each file in full before proceeding to the next.**

### Step 2 — Concatenate verbatim

> ### 🔴 CRITICAL: DO NOT SUMMARIZE
>
> The compiled file must contain the **complete, unmodified content** of every
> source file. Do NOT summarize, shorten, condense, merge, or truncate any content.
> The chatbot needs every detail. A missing constraint is a wrong answer.

Write the compiled file as:

```markdown
---
kb_type: compiled
product: [product name]
compiled_at: [ISO date]
language: [language]
source_files: N
source_tokens_total: [sum of all source file token counts]
compiled_tokens: [token count of this file]
---

# [Product Name] — Complete Knowledge Base

## How to use this file (for LLMs)
[Full verbatim content of index.md]

---

## Glossary
[Full verbatim content of glossary.md — every row of the table]

---

## Constraints
[Full verbatim content of constraints.md — every row, every impossible action, every error message]

---

## Features
[Full verbatim content of each features/*.md file, one after another, with --- separators]

---

## Flows
[Full verbatim content of each flows/*.md file, one after another, with --- separators]

---

## FAQ
[Full verbatim content of faq.md — all questions and answers]
```

After writing, calculate source total by summing tokens of: `index.md`, `glossary.md`,
`constraints.md`, `faq.md`, all `features/*.md`, all `flows/*.md`.
Do NOT include `kb-compiled*.md`, `db-gaps.md`, `db-prompt.md`, `meta.md`, `gaps.md`.
If compiled token count is less than 90% of source total, warn the user.

### Step 3 — Translate (if language specified)

Translate all prose, labels, and FAQ content into the target language.
Do NOT translate code snippets, product names, or UI button labels that appear
literally in the product interface.

### Step 4 — Write output

Write to `[output_dir]/kb-compiled.md` (English) or
`[output_dir]/kb-compiled-[language].md` (translated).

---

## COMMAND: all

Run all commands in sequence:

1. Run `scan [output_dir]`
2. Run `db-check [output_dir]`
   **Note:** `all` never runs `db-run` automatically.
3. Run `faq [output_dir]`
4. Run `compile [output_dir] [language]`
5. Run `tokens [output_dir]`

Print a final summary with all files written, noting any unresolved DB gaps.

---

## COMMAND: tokens

Estimate token counts for all KB files and write a deployment decision report.

### Step 1 — Check for tiktoken

```bash
python3 -c "import tiktoken" 2>/dev/null && echo "available" || echo "missing"
```

If missing, install: `pip install tiktoken -q`
If pip unavailable, fall back to word-count estimation (words ÷ 0.75).

### Step 2 — Count tokens per file

Categorize into three groups:
- **Source files**: `index.md`, `glossary.md`, `constraints.md`, `faq.md`, all `features/*.md`, all `flows/*.md`
- **Compiled file**: `kb-compiled*.md`
- **Internal files**: `db-gaps.md`, `db-prompt.md`, `meta.md`, `gaps.md`, `token-report.md`

Calculate `source_total`, `compiled_tokens`, and `retention_pct = compiled/source * 100`.

### Step 3 — Write token-report.md

Write `[output_dir]/token-report.md` with:
- Summary table: source files count, source tokens total, compiled tokens, retention %
- Deployment recommendation based on compiled token count:
  - Under 80k → System Prompt Injection
  - 80k–200k → Large Context Injection
  - Over 200k → RAG Recommended
- File breakdown separating source, compiled, and internal files
- Cost estimates for 1k, 10k, 100k conversations/month at Claude Sonnet pricing
- Re-generate recommendation

Warn if retention is below 90%.

Print key numbers after writing:
```
✅ Token report: [output_dir]/token-report.md

  Source files:   N,NNN tokens ([N] files)
  Compiled KB:    N,NNN tokens ([Z]% retention)
  Recommendation: [System Prompt Injection / Large Context / RAG]

  Cost estimate (system prompt, Claude Sonnet):
    1k conversations/month  → ~$X.XX
    10k conversations/month → ~$XX.XX
```

---

## Universal Rules

- **Never document what you cannot confirm.** Mark uncertain items `confidence: low`.
- **Always document the negative.** "Does NOT work when" is required for every feature.
- **Use user language.** No class names, function names, or technical jargon in output.
- **Extract exact error message strings.** Do not paraphrase error messages.
- **One file per feature.** The chatbot retrieves focused context, not large files.
- **Flag gaps.** If a section was inaccessible, write it in `gaps.md`.
- **Never modify source code.** Read-only on the codebase.
- **Never run destructive DB queries.** SELECT only. Never INSERT, UPDATE, DELETE, DROP.
