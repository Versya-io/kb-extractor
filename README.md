# kb-extractor

A [Claude Code](https://claude.ai/code) skill that scans your codebase and generates a structured, LLM-optimized knowledge base — ready to power an AI support chatbot or brief a marketing team.

Most support bots hallucinate because they're trained on generic documentation that doesn't know your product's actual constraints. **kb-extractor reads your source code directly** to find what's possible, what's not, and why — then produces markdown files a chatbot can use to answer with precision.

> Built to power [Versya](https://versya.io) — a multi-channel behavioral automation platform. The skill runs locally on your machine; your code never leaves it.

---

## What it generates

```
.kb/
  index.md              ← product overview + chatbot usage guide
  features/
    logo-upload.md      ← one file per feature, with constraints
    template-editor.md
    ...
  flows/
    create-flyer.md     ← step-by-step flows for top user actions
    ...
  constraints.md        ← everything that is NOT possible, consolidated
  glossary.md           ← product-specific terminology
  faq.md                ← 30+ questions generated from the KB
  db-gaps.md            ← working file: gaps + resolution history (version this)
  db-prompt.md          ← ready-to-use prompt for LLM with DB access (disposable)
  kb-compiled-pt.md     ← single file ready for system prompt injection
  token-report.md       ← token counts, deploy recommendation, cost estimates
```

The most valuable output is `constraints.md` — a machine-readable list of things
users will ask about that your product simply cannot do. This is what prevents your
bot from saying "yes" to impossible requests.

`token-report.md` tells you exactly how to deploy the KB: direct system prompt
injection, large context, or RAG — with cost estimates per conversation volume.

---

## Install

**Via plugin marketplace (recommended):**
```
/plugin marketplace add Versya-io/kb-extractor
/plugin install kb-extractor-plugin@kb-extractor
```

**Manual install:**
```bash
mkdir -p ~/.claude/skills/kb-extractor
cp skills/kb-extractor/SKILL.md ~/.claude/skills/kb-extractor/SKILL.md
```

Verify installation inside Claude Code:
```
/skills
```

---

### Windows (WSL) note

Claude Code on Windows runs inside WSL as `root`. If the marketplace install fails
with a `Permission denied (publickey)` error, it means git is trying to use SSH
instead of HTTPS. Fix it with:

```bash
sudo bash -c 'cat >> /root/.gitconfig << EOF
[url "https://github.com/"]
    insteadOf = git@github.com:
EOF'
```

Then retry the install. This is a one-time setup per WSL environment.

---

## Usage

```
/kb-extractor <command> <output_dir> [language]
```

| Command | What it does |
|---------|-------------|
| `scan` | Full codebase scan → writes all KB files |
| `db-check` | Detects available DB connections, generates `db-prompt.md` — **never queries** |
| `db-run` | Executes queries to resolve gaps — **requires explicit flag** |
| `faq` | Generates 30+ FAQ questions from existing KB files |
| `compile` | Consolidates everything into one file, with optional translation |
| `tokens` | Estimates token counts and writes deployment + cost report |
| `all` | Runs scan → db-check → faq → compile → tokens in sequence |

**Run everything at once, in Portuguese:**
```
/kb-extractor all .kb/ portuguese
```

**Run step by step:**
```
/kb-extractor scan .kb/
/kb-extractor db-check .kb/
# review .kb/db-prompt.md, then run with explicit source:
/kb-extractor db-run .kb/ --container my-db-container
/kb-extractor db-run .kb/ --mcp db-production
/kb-extractor db-run .kb/ --connection "postgresql://..."
/kb-extractor db-run .kb/ --manual
/kb-extractor faq .kb/
/kb-extractor compile .kb/ portuguese
/kb-extractor tokens .kb/
```

---

## How database-dependent values are handled

Many product constraints live in the database, not the code — plan limits, feature
flags per tier, dynamic quotas. The skill is **database-agnostic** and supports
PostgreSQL, MySQL, MariaDB, MongoDB, Redis, SQLite, and others.

The DB flow is split into two commands by design — **detection and execution are
intentionally separate** to prevent the AI from querying the wrong database automatically.

### Step 1 — `db-check` (safe, always runs first)

```
/kb-extractor db-check .kb/
```

This command **never executes any query**. It only:
- Detects available connections (MCP, Docker containers, CLI clients, env vars)
- Generates `db-prompt.md` with an environment confirmation checklist
- Tells you exactly what `db-run` flags are available

### Step 2 — `db-run` (requires explicit flag)

```
/kb-extractor db-run .kb/ --container my-db-container
/kb-extractor db-run .kb/ --mcp db-production
/kb-extractor db-run .kb/ --connection "postgresql://readonly:pass@host:5432/mydb"
/kb-extractor db-run .kb/ --manual
```

`db-run` **refuses to run without a flag** — there is no auto-detection or default.
This prevents the AI from choosing a database on your behalf.

Before executing any query, it still confirms once:
```
About to resolve [N] gaps using: --container my-db-container
⚠️ Is this the PRODUCTION database? (yes / no)
```

| Flag | What it uses |
|------|-------------|
| `--container [name]` | Docker exec into named container |
| `--mcp [name]` | Read-only MCP server |
| `--connection "[string]"` | Direct CLI (psql, mysql, mongosh, etc.) |
| `--manual` | Reads answers already filled in `db-gaps.md` |

**Two files are always generated by `db-check`:**

- `db-gaps.md` — working file, version this with your code.
  Tracks every gap, resolution method, and status over time.
- `db-prompt.md` — disposable prompt with environment confirmation checklist,
  ready to paste into an LLM with production database access.

**`all` runs `db-check` but never `db-run`** — you always invoke `db-run` manually
after reviewing what was detected.

**Example MCP setup for two databases (`~/.claude/claude.json`):**
```json
{
  "mcpServers": {
    "db-local": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres",
               "postgresql://user:pass@localhost:5432/myapp_dev"]
    },
    "db-production": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres",
               "postgresql://mcp_readonly:pass@localhost:5433/myapp_prod"]
    }
  }
}
```

---

## How to use the output with a support chatbot

After running `tokens`, check `token-report.md` for the deployment recommendation.
The skill decides automatically based on the compiled file size:

**Option A — System prompt injection (compiled file under 80k tokens)**

Paste the contents of `kb-compiled.md` into your chatbot's system prompt:

```
You are a support agent for [Product].
Answer ONLY based on the knowledge base below.
If the answer is not in the KB, say: "I don't have that information — I'll flag it for the team."

=== KNOWLEDGE BASE ===
{contents of kb-compiled.md}
======================
```

**Option B — Large context injection (80k–200k tokens)**

Same as Option A, but monitor cost per conversation at scale.
The token report includes cost estimates for 1k, 10k, and 100k conversations/month.

**Option C — RAG (over 200k tokens)**

Use the individual files in `features/` and `flows/` as chunks for a vector store.
Each file is scoped to one feature or flow, making retrieval precise.
Always include `constraints.md` in full — it's small and always relevant.

---

## Example output

See the [`examples/`](examples/) folder for sample generated files from a
real SaaS product, including a `db-gaps.md` with pending questions and a
`faq.md` ready for chatbot use.

---

## Limitations

- **Runtime DB values** — plan limits and feature flags stored in the database
  are detected as gaps, not resolved automatically. Use `db-check` + `db-run`
  with MCP, Docker, CLI access, or fill in manually.
- **Local vs. production data** — if resolved from a local database, gaps are
  marked `resolved-*-local` and flagged for verification before publishing.
- **Dynamic behavior** — animations, JS-heavy interactions, and third-party
  widget behavior may not be fully captured.
- **Business logic intent** — the skill documents what the code does, not why.
  Edge cases with complex business rules may need manual review.
- **External feature flags** — tools like LaunchDarkly are detected but their
  current state requires `db-run` or manual input.

Items the skill cannot confirm are marked `confidence: low` — the chatbot
treats these as "I'll need to double-check that for you."

---

## Built by a founder, for founders

This skill was built to solve a real problem in [Versya](https://versya.io) —
a behavioral automation platform. The support chatbot was hallucinating on
edge cases because generic documentation doesn't capture what your product
actually cannot do. kb-extractor was the fix.

---

## Contributing

Issues and PRs welcome. If you build a knowledge base with this skill and want
to share the output structure as an example, open a PR adding it to `examples/`.

---

## License

MIT
