# Lightning Talk Slide Analysis

An exploratory pipeline that reads a multi-year **Google Slides** deck of monthly
"lightning talk" sessions, tags every slide with structured metadata, and renders the
whole series as an interactive **3D "talk space."**

### → [**Live visualization**](https://technickle.nicklin.info/lightning-talk-slide-analysis/)

Each **yellow dot** is a talk, each **white star** is a speaker. The axes are meaningful:
**x = date**, **y = speaker** (each person is a slice), **z = primary topic** (ordered
least→most technical). Hover for a description, click a speaker/talk to isolate their
work, click a topic label to highlight a theme, or use the preset views (and the slow
**auto-orbit**) to explore. ~187 talks across ~26 speakers, 2022–2026.

## How it works

The pipeline lives in [`lightning_talks_analysis.ipynb`](lightning_talks_analysis.ipynb)
and runs in stages:

1. **Extract** — the Google Slides API returns each slide's exact text layer plus a
   server-rendered PNG thumbnail.
2. **Tag (image recognition)** — every slide image is passed to a **local vision model**
   (InternVL3 served via [LM Studio](https://lmstudio.ai/), with the exact slide text as
   grounding) which returns structured JSON per slide: is it a session divider, who's the
   speaker, what's the title, a description, content type, a **controlled-vocabulary topic**,
   free-form keywords, and tools mentioned.
3. **Reconcile** — session dates are forward-filled from the monthly divider slides onto the
   talks that follow; the model's known quirks (over-eager dividers, names landing in the
   wrong field) are corrected against the deck's invariants.
4. **Synthesize (text generation)** — the local LLM writes a short, playful one-line **blurb
   for each speaker** from their own talk titles, and **rewrites each talk description** to
   describe the talk directly.
5. **Render** — the talks are projected into the 3D talk space, exported as a self-contained
   [Three.js](https://threejs.org/) page published via GitHub Pages.

## Where the AI came in

This project leaned on AI across the whole stack:

- **Image recognition** — a local vision-language model read every slide (screenshots,
  diagrams, photos, title cards) to extract speakers, topics, and descriptions that aren't in
  the text layer.
- **Text synthesis** — a local LLM generated the speaker blurbs and cleaned up the
  machine-written talk descriptions.
- **Code development** — the notebook pipeline and the Three.js visualization were built in
  partnership with **Anthropic's Claude** (via Claude Code), iterating on the analysis,
  reconciliation logic, and the interactive renderer.

## Repo layout

| path | what |
|---|---|
| `lightning_talks_analysis.ipynb` | the full pipeline, run cell by cell |
| `docs/index.html` | the Three.js visualization (GitHub Pages root) |
| `docs/talk_space_data.js` | the data the viz loads (emitted by the notebook) |
| `CLAUDE.md` | architecture notes / guidance for working in the repo |
| `lt_work/` | local working dir — thumbnails, tags, CSV (gitignored) |

## Running it yourself

```bash
uv sync                                  # install deps (Python 3.12)
# enable the Google Slides API + save an OAuth desktop client as credentials.json
# load a vision model in LM Studio (or set BACKEND="anthropic" + ANTHROPIC_API_KEY)
```

Then open the notebook, set `PRESENTATION` to your deck's URL, and run the cells top to
bottom. The visualization regenerates into `docs/`; commit it to publish.

## Notes

- The tags are model output and aren't perfect — `slides.csv` can be hand-corrected and
  reloaded without re-running the vision pass.
- The published data includes speakers' first names and AI-generated descriptions of their
  talks.
