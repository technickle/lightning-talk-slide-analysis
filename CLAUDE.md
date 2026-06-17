# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A throwaway-style exploratory harness that tags every slide of a **native Google Slides** lightning-talk deck (a multi-year series of monthly sessions) with structured metadata via a vision model, then forward-fills session dates and emits a CSV for analysis.

## Commands

Dependencies are managed with `uv` (Python 3.12, see `pyproject.toml` / `.python-version`).

```bash
uv sync                              # install deps into .venv
export ANTHROPIC_API_KEY=sk-...      # only needed when BACKEND = "anthropic"
```

The pipeline lives entirely in **`lightning_talks_analysis.ipynb`** — run it cell by cell. There is no test suite, linter config, or build step. `main.py` is an unused scaffold stub — ignore it.

## Backends

The notebook runs against either of two vision backends, switched via `BACKEND` in the config cell:

- **`"lmstudio"`** (default) — a local VLM served by LM Studio (OpenAI-compatible API at `localhost:1234`). Keep `CONCURRENCY = 1` (serialized inference). Uses LM Studio's `json_schema` structured-output mode (`SLIDE_SCHEMA`).
- **`"anthropic"`** — Claude via the `anthropic` SDK. Bump `CONCURRENCY` to ~6.

The exact Slides text is fed into every call as grounding so a small local model reads real names/titles instead of OCR-guessing.

## Pipeline architecture (4 stages)

1. **Exact text** — `presentations.get` returns the slide list and text layer. `collect_text` recurses through shapes, **grouped elements**, and **table cells** to reconstruct each slide's exact text. This text is fed into every vision call as grounding so the model reads real names/titles instead of OCR-guessing — important for weaker local models.
2. **Thumbnails** — `pages.getThumbnail` renders a server-side PNG per slide (no local LibreOffice/poppler). Resumable: skips PNGs already on disk.
3. **Vision tagging** (`tag_slide`) — each PNG + its grounding text → one JSON record per slide, appended to `slides.jsonl`. Each record also keeps `exact_text` (the verbatim grounding) for auditing. Resumable: reloads `slide_index` values already in the JSONL and only processes the rest. This is why output is append-only JSONL, not a single document.
4. **Table build** — sorts rows by `slide_index` and **forward-fills** `session_date`/`session_label` from MARKER slides onto the talk slides that follow them, then writes `slides.csv` via pandas (new record keys flow into the CSV automatically). Prints a `[check]` warning on any backwards date jump (a likely misread or out-of-order slide).

### Key domain concept: marker slides

The deck has two slide kinds. **Marker** slides are session title/divider cards carrying the month/date; everything between two markers is a talk slide inheriting the prior marker's date. The vision prompt is built around this distinction (`is_marker`, `session_label`, `session_date`), and stage 4's forward-fill depends on it. `session_date` uses ISO `YYYY-MM`.

The vision model is instructed **never to guess a speaker name** (`null` when absent). `topics` are constrained to a **controlled vocabulary** (the `TOPICS` list in step 5, enforced via a `json_schema` enum so even weak local models can't emit off-list values; `"other"` is the escape hatch). **`TOPICS` order is meaningful** — roughly least→most technical (personal → domains → data/engineering) — so step 15's z-axis reads as a gradient; reordering doesn't affect validity (enum membership is unchanged, so no re-tagging needed). Specific detail goes in a separate free-form `keywords` field. Edit `TOPICS` to retune the taxonomy.

**Known model failure + the reconciliation rules (all in step 7):** the model (especially the local VLM) over-flags `is_marker` on per-talk title cards and misfiles presenter names into `session_label` or `title`. Step 7 corrects this against the deck's saving invariant — a *genuine* session marker **always** carries a valid `YYYY-MM` `session_date`:
- Only dated markers (`is_real_marker`) define sessions and drive the forward-fill.
- Dateless "markers" are demoted to talk slides; a stranded short single-line `session_label` is rescued into `speaker`.
- A `title` that exactly matches a known speaker (roster built from all `speaker` values) and has no speaker yet is moved into `speaker` — this keeps real titles intact.
- Speaker names are normalized to **first name only** (first names are unique in this deck); multi-speaker strings (`& / , and +`) are preserved, not collapsed. Two edit points at the top of the cell: `SPEAKER_OVERRIDES` (keyed by `slide_index`, for one-off misreads) and `SPEAKER_ALIASES` (keyed by raw string, for multi-speaker/canonical fixes that apply everywhere).
- `present_date` is the forward-filled `session_date_ff` as a real `datetime64`, pinned to the 15th of the month (only pre-first-marker cover slides are `NaT`).

This reshapes only the CSV — `slides.jsonl` stays as the raw model output (audit trail), so re-running step 7 is idempotent.

**Manual edits:** `slides.csv` is an *output* of step 7 (rebuilt from `slides.jsonl` each run). To hand-correct rows (marker flags, speaker names) and feed them into the views, edit the CSV then run **step 7b** ("Reload manual edits"), which loads the CSV into `df` with dtype coercion — and do *not* re-run step 7 (it overwrites). Step 7b does not recompute the date forward-fill, so it takes `present_date` as-is.

## Auth and secrets

OAuth flow lives in `get_service`. Requires a GCP **Desktop OAuth client** saved as `credentials.json` in the project root (an input you supply; Slides API must be enabled on the project). First run opens a browser for consent and caches `token.json` in the workdir (`lt_work/token.json`) — note these two files live in **different folders by design** (root input vs. generated cache). If `getThumbnail` returns 403, add `https://www.googleapis.com/auth/drive.readonly` to `SCOPES` and delete `token.json` to re-consent.

`credentials.json`, `token.json`, and `lt_work/` are gitignored.

## Graph (step 9)

Builds a `networkx` graph from the talk rows (cover slide + month markers dropped): one `talk` node per slide wired to its `speaker`, `topic`, and `tech` nodes. Node ids are **type-prefixed** (`talk:`/`speaker:`/`topic:`/`tech:`) so a topic and a tool with the same name don't collide; topic/tech keys are **case-folded** (display label preserved) so `Python`/`python` merge. Multi-speaker strings are split into individual speaker nodes. Exported as GraphML for Gephi/Cytoscape. Step 10 renders the same graph interactively with **pyvis** (colored by type, sized by degree) to a self-contained offline HTML embedded in the notebook.

Step 11 is a **bipartite projection**: collapse the talk in the middle to link speakers directly to topics, edge weight = # of that speaker's talks carrying the topic. Emits the full weighted list as `speaker_topic.csv` and an interactive pyvis graph filtered to topics seen in ≥ `TOPIC_MIN_TALKS` talks.

Step 12 is a **speaker activity timeline** (matplotlib): one row per speaker, line from first→last talk, dot per talk, count labeled. Deliberately shows count-in-tenure-context rather than a per-speaker rate — observed first→last span is only a *lower bound* on real tenure (talks are observed, employment dates are not), so a rate would make one-talk speakers look hyper-active. A true rate needs real start/end dates (an HR input not in this data).

Step 13 ranks the **most wide-ranging speakers** by variety = count of distinct controlled `topics` covered, with talk count shown alongside (variety partly tracks volume, since there are only ~17 buckets).

Step 14 measures **breadth independent of volume**: Simpson's (Gini–Simpson) diversity on each speaker's topic distribution — P(two random talks differ in topic), for speakers with ≥ `MIN_TALKS`. 0 = specialist, →1 = wide-ranging. Only meaningful *because* topics are now a controlled vocabulary; on the old free-form tags every speaker scored ~0.97 (no discrimination).

Step 14b generates a cute/SFW one-line **speaker blurb** per person by feeding their own talk titles + summaries to the local model; cached to `speaker_blurbs.json` (editable; only missing speakers regenerate unless `REGEN_ALL`). The step-15 data cell attaches each blurb to its speaker, and the Three.js hover popup shows it under the name.

Step 14c **reworks talk summaries** via the local model so they describe the talk directly rather than the slide ("This slide announces…" → plain description); cached to `summary_revised.json` (editable; incremental unless `REGEN_ALL_SUMMARIES`). The step-15 data cell prefers the revised text. The step-5 `summary` prompt instruction is also written to avoid slide-meta language for future tagging runs. Both 14b and 14c are non-destructive overlays keyed off the data — they don't touch `slides.jsonl`/`slides.csv`.

Step 15 is the **3D talk space, rendered in Three.js**, dark background. It is **split**: the view is a static, hand-editable page **`docs/index.html`** (tracked, published via GitHub Pages); the **notebook cell only emits the data** to `docs/talk_space_data.js` (`window.TALK_DATA = {...}`), which the page loads via a classic `<script src>` (works under `file://` and when served). `docs/` is the GitHub Pages root (`main` → `/docs`, with `.nojekyll`); committing it publishes the viz — note the data file carries speaker names, blurbs, and talk descriptions. Edit visuals in the HTML directly; rerun the cell after re-tagging to refresh the data. **Meaningful axes** (no force layout): **`x` = date** (decimal year), **`y` = speaker** (each a categorical slice — a y-plane of all their talks; world-Z), **`z` = primary topic** as discrete layers in `TOPICS` order, least→most technical, `"other"` excluded (world-Y/up). Multi-speaker talks are duplicated into each co-presenter's slice; slices ordered by first appearance. Each speaker is a **white 3D star polyhedron** (a *stella octangula* — two tetrahedra merged via `BufferGeometryUtils.mergeGeometries`, shaded by an ambient + directional light; real geometry so it rotates, unlike the old camera-facing sprite) at the centroid of its slice. Talk dots are **yellow**, drawn as an **`InstancedMesh` of small spheres** (real world geometry — so dots and stars scale identically under perspective *and* ortho zoom; `PointsMaterial` point sizes don't track ortho zoom, which made dots look tiny in the flat presets). Per-instance scale = dot radius (selected dots enlarge), per-instance color = yellow×brightness for the fade. The `DOT` config at the top of the script holds the radii (`base`/`sel`/`fade`, world units) + faded brightness. Speaker dimming is per-mesh `material.color`.

Highlight transitions (lines, dot brightness, star brightness, overlay opacity) are **eased over ~0.5s** (`HL_DUR`) by `stepHighlight()` in the render loop; the tooltip fades via CSS opacity.

**Preset camera views** (buttons + number keys `0`–`3`): reset (perspective) and three orthographic views (speaker×date / date×topic / speaker×topic) that *flatten* by looking down an axis. Transitions tween position + ortho `zoom` (computed to fill). The active preset's button highlights and clears when the user manually orbits/zooms (an OrbitControls `change` not flagged `programmatic`). Loads Three.js from a CDN (needs internet); `OrbitControls`, raycaster hit-testing (reliable click), CSS2D billboard labels (light grey, no wall-snap). Hover previews a person (line-fan brightens, rest fade); **click** a talk/star to lock a speaker, or **click a z-axis topic label** to highlight that theme. One lock at a time, keyed `kind:value` — same unlocks, another switches. (Plotly did this before but its WebGL 3D had unreliable clicks / freezes / wall-snapping labels.)

Note: in the DataFrame, empty `speaker`/`title` are `NaN` (a *truthy* float), so node-building guards with `pd.notna` rather than `or`-fallback — otherwise you get a bogus `nan` node.

## Workdir layout (`./lt_work`)

`token.json` (auth cache) · `slides/slide_NNNN.png` (thumbnails) · `slides.jsonl` (append-only tag records, the resume ledger) · `slides.csv` (final table) · `lightning_talks.graphml` (step-9 graph) · `lightning_talks.html` (step-10 interactive view) · `speaker_topic.csv` + `speaker_topic.html` (step-11 projection) · `speaker_timeline.png` (step-12 timeline) · `speaker_variety.png` (step-13 topic variety) · `speaker_breadth.png` (step-14 Simpson diversity) · `speaker_blurbs.json` (step-14b LLM speaker blurbs) · `summary_revised.json` (step-14c reworked talk descriptions) · `talk_space_data.js` here is superseded — step 15 now writes `docs/talk_space_data.js`, and the view is `docs/index.html` (GitHub Pages root).
