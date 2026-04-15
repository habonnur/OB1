# OB1 Compatibility Tagging + natejones.com Directory UI — Plan

**Status:** Draft / not for PR. Lives local on `claude/open-brain-skill-directory-e3bTx`.
**Author's note:** Planning doc — decisions, trade-offs, and concrete next steps. No code changes yet.

---

## Part 1 — Compatibility tagging for skills (and all contributions)

### The problem

The repo already has a `skills/` directory with a README index and per-skill `metadata.json`. The thing missing is a way to tell, at a glance or programmatically, **which skills need an Open Brain setup and which ones work standalone.**

Concrete example: `skills/panning-for-gold/metadata.json` describes itself as a *"Standalone skill pack that turns transcripts… into… Open Brain-ready outputs"*, but its metadata says `"open_brain": true` — because the schema forces it. A user who hasn't set up OB1 yet has no way to discover "skills I can try right now with zero infra."

### The actual blocker (code reference)

`.github/metadata.schema.json:58-63` currently hard-codes:

```json
"open_brain": {
  "type": "boolean",
  "const": true,
  "description": "Must be true. All contributions require an Open Brain setup."
}
```

Every contribution type — skills, recipes, extensions, primitives, schemas, dashboards, integrations — inherits this constraint. Loosening this field is the minimum change.

`CONTRIBUTING.md:178` also documents the "must be true" rule and will need a parallel update.

### Proposed tagging model

**Primary change — one field, one meaning:**

Change `requires.open_brain` from `{ type: "boolean", const: true }` to a two-value enum string:

```json
"open_brain": {
  "type": "string",
  "enum": ["required", "compatible"],
  "description": "required = needs an Open Brain setup to function. compatible = works standalone; OB1 is optional and enhances but is not needed."
}
```

Why an enum instead of keeping it boolean?
- Explicit, self-documenting in file (`"required"` reads better than `true`).
- Room to grow without breaking (could add `"enhanced"` later for "works standalone but meaningfully better with OB1" — deliberately deferred; two buckets now, three if the case emerges).
- Migration is mechanical: `true` → one of the two values after a review pass.

**Alternative considered and rejected:** keep boolean, flip `const: true` to plain `boolean`. Rejected because `"open_brain": false` on the line next to `"open_brain": true` is easy to misread; a string enum surfaces intent.

### Secondary signal — tags (optional, for search UX)

Reserve two well-known tag strings that the frontend can key off for fast filtering without parsing `requires`:

- `ob1-required`
- `standalone`

Authors are encouraged but not required to add them. The `requires.open_brain` field is the source of truth; tags are a UX convenience for search indexes that aren't reading full metadata.

### README surface change

Update each category's README table (e.g., `skills/README.md`) to add a **Works Without OB1?** column:

| Skill | What It Does | Works Without OB1? | Contributor |
| --- | --- | --- | --- |
| Panning for Gold | … | ✅ | @jaredirish |
| Auto-Capture | … | — | @jaredirish |

One glance, one answer. The check/dash is generated from `requires.open_brain`.

### Migration / audit plan

1. **Loosen schema.** Edit `.github/metadata.schema.json` to the enum form above.
2. **Update `CONTRIBUTING.md`.** Replace the "must be `true`" language at lines ~163–178 and in the example blocks at ~200 and ~227. Add a short "Which value should I pick?" rubric:
   - Pick `required` if your SKILL.md / recipe / extension reads or writes the `thoughts` table, calls the OB1 MCP server, depends on Supabase vector search, or assumes RLS is wired up.
   - Pick `compatible` if a user with no OB1 infra can paste the skill into Claude/Cursor/Codex and get useful output today.
3. **Audit pass.** Walk every `metadata.json` (70 files, per grep) and set the new value. First-pass guesses for skills (to be reviewed, not committed blindly):
   - `compatible`: panning-for-gold, heavy-file-ingestion, competitive-analysis, research-synthesis, meeting-synthesis, financial-model-review, deal-memo-drafting, claudeception, n-agentic-harnesses
   - `required`: auto-capture, autodream-brain-sync, weekly-signal-diff, work-operating-model
4. **Update templates.** `skills/_template/metadata.json` and every other `_template/metadata.json` under each category need the new field value and a comment pointing to the rubric.
5. **Update the PR workflow check.** `.github/workflows/ob1-review.yml` — confirm it revalidates cleanly against the new schema. No functional change expected, but worth confirming.
6. **Update `CLAUDE.md`** — no change. It doesn't assert the current rule anywhere specific.

### Scope explicitly NOT in this plan

- Building a separate new skill directory inside the repo. (Earlier version of this idea — dropped. The existing `skills/` folder is the directory.)
- Rewriting the metadata schema beyond this one field.
- Automating the audit. Humans should look at each skill and decide. It's ~15 skills.

---

## Part 2 — natejones.com OB1 directory UI (Astro plan)

This is a plan for a public browsing surface hosted on natejones.com. It's not built in this repo; it's a separate Astro project that consumes OB1's `metadata.json` files and renders a product-like directory. Written here so the data contract stays in sync with the repo.

### Goal in one sentence

A fast, searchable, category-aware directory of every OB1 contribution, with clear hierarchy and a visible "works without OB1" signal, deployed at something like `natejones.com/ob1` (or a subdomain).

### Stack (per Nate's existing site)

- **Astro** — static site generation, zero-JS-by-default, content collections for structured data.
- **Tailwind CSS** — utility-first.
- **Vercel** — hosting, preview URLs on PRs.
- **Plausible** — analytics.

Everything below fits inside that stack without adding new runtime dependencies. The one new library: **Fuse.js** for client-side fuzzy search (~6KB gz, runs fine on 70 entries; no backend needed).

### Information architecture

```
/ob1                         → landing: pitch, "start here" path, top-level categories
/ob1/browse                  → single unified directory (all categories, filterable)
/ob1/browse?category=skills  → preselected filter
/ob1/skills                  → category landing: skills-specific intro + skills grid
/ob1/skills/panning-for-gold → detail page (renders README.md from GitHub)
/ob1/recipes                 → same pattern
/ob1/extensions              → includes the ordered learning path (1–6)
/ob1/primitives              → …
/ob1/schemas                 → …
/ob1/dashboards              → …
/ob1/integrations            → …
```

Hierarchy is visible both ways: users can browse *by category* (`/ob1/skills`) or *across everything* (`/ob1/browse`). The unified view is the primary search surface; category pages are marketing/context-setting.

### Data contract (what the site expects from OB1)

A single build-time JSON file, `catalog.json`, shaped like:

```json
{
  "generated_at": "2026-04-15T00:00:00Z",
  "commit": "abc123",
  "entries": [
    {
      "slug": "panning-for-gold",
      "category": "skills",
      "name": "Panning for Gold",
      "description": "…",
      "compatibility": "compatible",     // derived from requires.open_brain
      "difficulty": "intermediate",
      "estimated_time": "5 minutes",
      "tags": ["skill", "brain-dump", "transcript", "evaluation", "workflow"],
      "author": { "name": "Jared Irish", "github": "jaredirish" },
      "version": "2.0.0",
      "requires": { "services": [], "tools": ["Claude Code or similar…"] },
      "requires_skills": [],
      "requires_primitives": [],
      "github_url": "https://github.com/NateBJones-Projects/OB1/tree/main/skills/panning-for-gold",
      "readme_url": "https://raw.githubusercontent.com/…/skills/panning-for-gold/README.md"
    }
  ]
}
```

### Build pipeline

Two options, in order of preference:

**Option A — Build-time fetch via GitHub Tree API (recommended).**
An Astro `astro.config` integration or a pre-build script (`scripts/pull-ob1-catalog.ts`) hits the GitHub Git Trees API once, filters to `*/metadata.json`, fetches contents in parallel, normalizes to the shape above, writes `src/data/catalog.json`. Commits the output into the natejones.com repo as a cached fallback (so builds don't fail if GitHub is slow). Refreshes triggered by a Vercel Deploy Hook fired from a GitHub Action in OB1 on push to `main`.

**Option B — Git submodule.**
Add OB1 as a submodule in the natejones.com repo; walk the filesystem at build. Simpler, but couples the two repos more tightly and requires the submodule ref to be bumped for every update.

Go with A. Submodule adds friction for content-only updates.

### Search

- **Library:** Fuse.js, client-side, index built at page load from `catalog.json`.
- **Fields indexed (weighted):** `name` (3), `description` (2), `tags` (1.5), `author.name` (1).
- **UX:** single text input at the top of `/ob1/browse`. Debounced (150ms). Results update live as a card grid below the filters.
- **No backend.** 70–200 entries is trivially small for client-side search. Don't reach for Algolia or Pagefind here; they're overkill and add hosting surface.

### Filters

Shown as pill toggles above the grid:

1. **Category** — multi-select: skills / recipes / extensions / primitives / schemas / dashboards / integrations.
2. **Compatibility** — segmented control: All | Works without OB1 | OB1 required.
3. **Difficulty** — multi-select: beginner / intermediate / advanced.
4. **Tags** — popover with multi-select; shows counts.

Filter state lives in URL query params so links are shareable (`/ob1/browse?compat=compatible&difficulty=beginner`).

### Card design (per entry)

- Top row: category badge (color-coded per category) + compatibility badge (green "Standalone OK" / neutral "OB1 required") + difficulty chip.
- Title (name) — links to detail page.
- Description — two-line clamp.
- Footer: estimated time · author (with GitHub link) · top 3 tags.
- Hover: subtle lift; focus ring for keyboard nav.

### Detail page

Rendered at `/ob1/<category>/<slug>`:

- Hero: name, description, compatibility + difficulty + time badges, author.
- Primary content: README.md rendered (Astro's built-in Markdown, fetched at build and committed into content collections; do not fetch client-side).
- Sidebar:
  - Requirements (services, tools).
  - Depends on → links to any `requires_skills` and `requires_primitives` entries.
  - "Used by" → reverse index (computed at build: entries whose `requires_skills` contains this slug).
  - "View on GitHub" button.
- Footer: version, created/updated dates, link to CONTRIBUTING.

### Extension learning path (special-case rendering)

Extensions have `learning_order` (1–6). The `/ob1/extensions` page renders them as a numbered path/flow diagram (Tailwind-styled stepper) in addition to the standard grid view. This is where the repo's curation shows up visually — it's the OB1 equivalent of a "getting started" track.

### Analytics (Plausible)

Custom events:
- `ob1_search` with `{ query_length, results_count }` (not the raw query — respect privacy).
- `ob1_filter_applied` with `{ filter, value }`.
- `ob1_entry_view` with `{ slug, category }`.
- `ob1_github_click` with `{ slug, category }`.

No cookies, no consent banner needed (Plausible default).

### Performance budget

- **LCP < 1.5s** on 4G. Astro ships ~0KB JS by default; the only client script is Fuse.js + filter state (<15KB gz total).
- **Catalog size:** ~70 entries × ~1KB each = ~70KB JSON. Fine to ship uncompressed; <20KB gzipped.
- **No images per card** in MVP. Category color swatches via Tailwind classes.

### Minimum viable slice (what to build first)

In order, shippable at each step:

1. `scripts/pull-ob1-catalog.ts` → writes `src/data/catalog.json` from the GitHub Trees API. Check the file in.
2. Astro content collection over `catalog.json`. `/ob1/browse` page: grid + search + category filter + compatibility filter.
3. Detail pages (`/ob1/[category]/[slug]`) rendering README.md from the repo.
4. Category landing pages (`/ob1/skills`, `/ob1/recipes`, …) reusing the browse grid with prefilter.
5. Plausible events.
6. GitHub Action in OB1 that pings a Vercel Deploy Hook on push to `main`.
7. Extensions learning-path view.

### Open questions for the site build (flag for Nate)

- Subpath (`natejones.com/ob1`) vs. subdomain (`ob1.natejones.com`) — affects Vercel project setup and Plausible site config. Subpath is simpler; subdomain gives OB1 its own analytics bucket and future flexibility.
- Is there an existing Astro repo for natejones.com we're adding `/ob1/*` routes to, or is this a standalone project? Plan assumes routes added to the existing site.
- Do we want a "Submit a contribution" CTA that deep-links to a GitHub new-file form with the `_template/metadata.json` prefilled? Nice-to-have for a later pass.

---

## Dependencies between the two parts

The site can ship *before* the tagging change lands — it just has to treat every entry as `compatibility: "required"` until the schema update merges. But the whole point of the UI is the compatibility filter, so Part 1 should land first or in parallel.

One data-contract note: whatever value the schema lands on (`"required"` / `"compatible"` string enum, or anything else), the site's `compatibility` field should mirror it exactly. No translation layer. Keep the source of truth in the repo.
