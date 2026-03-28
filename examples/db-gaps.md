---
kb_type: db-gaps
status: partial
generated_at: 2026-03-21T00:00:00Z
total_gaps: 6
resolved: 2
pending: 4
---

# Database-Dependent Gaps

These items were found in the codebase but their values depend on the database.

**How to resolve:**
1. Fill in the `Answer:` fields below manually, OR
2. Pass this file as a prompt to an LLM with database access, OR
3. Run `/kb-extractor db .kb/` with a read-only MCP connection configured

---

## Gaps

### GAP-001 ✅ resolved-manual
**Feature affected:** Flyer creation
**File:** `src/workspace/limits.js:34`
**What the code checks:** `workspace.limits.max_flyers`
**Question:** What is the value of `max_flyers` for each plan tier?
**Answer:**
- Free: 5 flyers/month
- Starter: 30 flyers/month
- Pro: unlimited
- Annual Pro: unlimited

---

### GAP-002 ✅ resolved-manual
**Feature affected:** Logo upload
**File:** `src/features/logo.js:12`
**What the code checks:** `user.plan.features.includes('custom_logo')`
**Question:** Which plans have `custom_logo` enabled?
**Answer:**
- Free: ✗ (uses platform watermark)
- Starter: ✓
- Pro: ✓
- Annual: ✓

---

### GAP-003 ⏳ pending
**Feature affected:** Template library access
**File:** `src/templates/gate.js:56`
**What the code checks:** `template.tier <= user.plan.template_tier`
**Question:** What is the `template_tier` value for each plan? Which template
categories are restricted to paid plans only?
**Answer:** _(fill in here)_

---

### GAP-004 ⏳ pending
**Feature affected:** Export resolution
**File:** `src/export/resolution.js:89`
**What the code checks:** `plan.export_dpi`
**Question:** What is the export DPI for each plan tier?
(e.g. Free = 72dpi, Paid = 300dpi?)
**Answer:** _(fill in here)_

---

### GAP-005 ⏳ pending
**Feature affected:** Team members / seats
**File:** `src/workspace/team.js:23`
**What the code checks:** `workspace.limits.max_members`
**Question:** What is the maximum number of team members per plan tier?
**Answer:** _(fill in here)_

---

### GAP-006 ⏳ pending
**Feature affected:** Scheduled publishing
**File:** `src/publish/scheduler.js:17`
**What the code checks:** `featureFlags.scheduled_publish`
**Question:** Is scheduled publishing available on all plans or restricted?
If restricted, which plans have access?
**Answer:** _(fill in here)_
