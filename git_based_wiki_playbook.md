## Summary

Use this playbook to create and run a Git-based team wiki using plain Markdown/HTML, pull requests, and lightweight automation.

## What this gives you

- **Single source of truth** in Git.
- **Searchable documentation** across all operational guides.
- **Reviewable changes** using pull requests.
- **Repeatable structure** for onboarding, architecture, runbooks, standards, and pipelines.

## Setup process

### Step 1: Create the repository structure

- Create top-level wiki folders by topic area.
- Keep folder names stable and descriptive.
- Add one README per section with links to key pages.

### Step 2: Define documentation standards

- Require frontmatter metadata (`title`, `category`, `tags`, `original_file`).
- Use consistent headings and page naming.
- Keep source traceability when content is curated from multiple documents.

### Step 3: Add starter templates

- Create templates for runbooks, architecture notes, and incident summaries.
- Include checklist sections for completeness and handover quality.

### Step 4: Add validation automation

- Add a basic quality script to check links, metadata, and structure.
- Run checks in CI so broken links or malformed pages are caught early.

### Step 5: Establish curation workflow

- Ingest raw notes/documents into draft pages.
- Synthesize cross-source pages for common workflows.
- Keep indexes updated so users can find information quickly.

### Step 6: Operate with PR-based governance

- Require pull requests for all documentation changes.
- Assign page-family owners for onboarding/runbooks/architecture.
- Track meaningful updates in per-page change logs.

## Plan-first delivery model (to reduce rework)

### Phase 1: Build the base

- Set up folder structure, standards, templates, and checks.
- Load historical source files into a stable raw source area.
- Confirm traceability rules before generating or curating any pages.

### Phase 2: Publish curated foundations

- Create a small set of high-value synthesis pages first (for example onboarding, runbook, catalog).
- Use historical sources to populate sections with reusable, merged content.
- Validate navigation/search paths before scaling out.

### Phase 3: Add enhancements

- Add optional automation (ingest helpers, field-population helpers, indexing helpers).
- Improve coverage iteratively after core pages are useful and stable.
- Track enhancement backlog in a plan and execute in priority order.

## What not to do (anti-patterns)

### Anti-pattern 1: Bulk convert every source file into standalone wiki pages

- **Why this fails:** creates noise, duplicates, and weak discoverability.
- **Better approach:** use raw historical sources as evidence and build curated pages that synthesize across sources.

### Anti-pattern 2: Start without a phased plan

- **Why this fails:** causes backtracking, deletions, and inconsistent structure.
- **Better approach:** lock base architecture first, then curate, then automate enhancements.

### Anti-pattern 3: Spend AI effort on deterministic formatting work

- **Why this fails:** burns credits on tasks scripts can do reliably.
- **Better approach:** script mechanical tasks and reserve AI for synthesis/judgment.

### Anti-pattern 4: Publish screenshots without sanitization

- **Why this fails:** increases security/privacy risk.
- **Better approach:** redact first, then publish only minimal visual context required.

## Lessons learned from avoidable rework

- Creating one Markdown file per historical document can require large cleanup later.
- Reprocessing content repeatedly without a stable curation target wastes time and AI credits.
- Delaying validation/check automation causes corrective loops that could be prevented early.
- Missing clear script-vs-AI boundaries leads to inconsistent output quality and cost.

## Script-first vs AI-first task split

### Use scripts for

- File discovery, extraction, conversion, and metadata stamping.
- Index updates, link checks, and structure validation.
- Repeatable payload shaping (for example JSON/field mapping shells).
- Bulk operations that follow fixed rules.

### Use AI for

- Cross-document synthesis and summarization into user-ready guidance.
- Identifying missing context across multiple sources.
- Drafting readable runbooks/playbooks from technical evidence.
- Producing first-pass text for business process workflow fields where judgment/phrasing matters.

### Practical rule

- If a task is deterministic and repeatable, automate with script.
- If a task needs interpretation or synthesis, use AI with clear constraints.

## Where human guidance is critical

### Information architecture decisions

- Decide the top-level wiki categories and when content belongs in a curated page vs a raw source area.
- Decide when multiple historical sources should be merged into one page rather than published separately.

### Quality and editorial decisions

- Review whether generated summaries actually answer user tasks, not just mirror source documents.
- Decide which sections need business-friendly wording versus verbatim technical detail.
- Confirm final titles, tags, and navigation labels are useful to future readers.

### Risk and sanitization decisions

- Review screenshots and examples for sensitive information before publication.
- Decide what source references should remain explicit and what details should be generalized/redacted.

### Automation boundary decisions

- Decide which workflows are stable enough to script.
- Decide where AI should only draft and where a human must approve before publication or submission.

## Automation pattern to prefer

### Key aspects of scripts in general

- Scripts should handle **repeatable**, **deterministic**, and **testable** tasks.
- Scripts should prefer **structured input** and **structured output**.
- Scripts should be safe to rerun without creating duplicate or conflicting content.
- Scripts should support both **batch execution** and **one-off manual use**.
- Scripts should make human review points obvious rather than hiding assumptions.

### Key script types to include in a Git-based wiki

- **Single-document ingest script**  
  Reads one source document, extracts content, and proposes placement/metadata.
- **Batch ingest script**  
  Processes a historical document set into a stable evidence base.
- **Quality/validation script**  
  Checks links, metadata, structure, and required fields.
- **Workflow field population script**  
  Builds draft text for business process workflow fields from supporting documents.

### Recommended interface

- Keep Python tools usable in two modes:
  - **structured JSON config** for repeatable runs and automation
  - **explicit CLI options** for ad hoc/manual runs

### Why this matters

- JSON config makes runs reproducible and easier to hand off.
- CLI options are faster for one-off use.
- Both approaches reduce hidden assumptions and make it easier to separate scriptable work from human review steps.

### Key aspects of scripts specifically for this workflow

- **Historical-source aware**  
  Scripts should treat historical documents as the evidence base, not as pages to publish one-for-one.
- **Section-oriented output**  
  Scripts should help populate target sections of curated pages or workflow fields instead of generating document clones.
- **Traceability preserved**  
  Scripts should keep source references so humans can verify where content came from.
- **Human review built in**  
  Scripts should emit review checkpoints for wording, ownership, risk, and sanitization.
- **Override-friendly**  
  Scripts should allow category/type/field overrides without code changes.

### Example JSON-driven pattern

```json
{
  "documents": {
    "implementation": "C:\\path\\Implementation_Plan.docx",
    "uat": "C:\\path\\Test_Plan.docx",
    "risk": "C:\\path\\Risk_Assessment.xlsx",
    "rollback": "C:\\path\\Rollback_Plan.docx",
    "communication": "C:\\path\\Communication_Plan.docx",
    "screenshots_zip": "C:\\path\\workflow_screencaptures_blank_process.zip"
  },
  "overrides": {
    "type": "Standard",
    "state": "New",
    "category": "Business Process",
    "configuration_item": "Workflow Support Service"
  },
  "outputs": {
    "json": "workflow_payload.json",
    "markdown": "workflow_payload.md"
  }
}
```

### Example CLI-driven pattern

```bash
python tools/workflow_form_populator.py ^
  --implementation "C:\path\Implementation_Plan.docx" ^
  --uat "C:\path\Test_Plan.docx" ^
  --risk "C:\path\Risk_Assessment.xlsx" ^
  --rollback "C:\path\Rollback_Plan.docx" ^
  --communication "C:\path\Communication_Plan.docx" ^
  --category "Business Process" ^
  --configuration-item "Workflow Support Service"
```

## Minimal operating model

### Weekly

- Triage new raw docs.
- Curate high-value updates into existing pages.
- Fix broken links and stale references.

### Monthly

- Review top-used pages for accuracy.
- Consolidate duplicate or overlapping pages.

### Quarterly

- Archive superseded content.
- Reconfirm ownership and update navigation/index pages.

## Suggested screenshots to include

### Core setup screens

- Repository root showing wiki folder structure.
- PR view with a docs-only change and reviewer comments.
- CI/check run showing docs quality pass.

### Authoring and discoverability screens

- Editing a Markdown page in browser or VS Code.
- Search results for a key term across the repo.
- Topic index page linking to core runbooks.

### Curation workflow screens

- Raw source folder with incoming files.
- Curated page showing source references.
- Diff view of a page update with changelog entry.

## Screenshot sanitization checklist

- Blur/redact **user names**, **emails**, **hostnames**, **IP addresses**, **URLs**, **ticket numbers**, **tenant IDs**, and **environment names** if sensitive.
- Remove browser tabs/bookmarks that reveal unrelated systems.
- Crop to the smallest area needed to explain the step.
- Prefer sample or dummy records over production records.
- Re-check images after export to ensure redactions are permanent.

## Publishing checklist

- [ ] No corporate-specific network or environment details.
- [ ] No credentials, secrets, or identifiable contact details.
- [ ] Links resolve and navigation entries are updated.
- [ ] CI/documentation checks pass.

## Change log

- 2026-07-22: Initial sanitized playbook for creating a Git-based team wiki.
- 2026-07-22: Added anti-patterns, avoidable rework lessons, phased plan-first workflow, and script-vs-AI decision guidance.
- 2026-07-22: Added human-critical guidance points and preferred Python automation interface patterns.
