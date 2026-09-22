<p align="center">
  <img src="./assets/banner.webp" alt="十八木" width="100%" />
</p>

<p align="center">
  <a href="./README.md">中文</a> · English
</p>

# 18trees-benchmark-pages-skill

**Turn a benchmark's raw data, paper, and analysis report into a GitHub-Pages-ready homepage and leaderboard.**

Everyone building a benchmark does the same thing: fork the 5.2k★ academic project page template, then hand-roll the benchmark-specific parts.

The problem is that the template has **none of it** — no data-driven tables, no sorting or filtering, no leaderboard entry point, no submission guide. It is a paper homepage template, not a benchmark template. So every team re-solves the same problems: how to generate tables from experiment results, how to update the leaderboard when a model changes, how someone else submits their results.

**This skill does the parts everyone rebuilds.**

> **▶ [See what the generated page looks like](https://monsterpppp.github.io/18trees-benchmark-pages-skill/)**
>
> A live demo built from a real evaluation run (6 models × 2,492 questions),
> rebuilt automatically by [`.github/workflows/pages.yml`](./.github/workflows/pages.yml).
> Source directory: [`site/`](./site/).

---

## The problem it solves

| Where you are today | The consequence |
|--------------------|-----------------|
| Results scattered across `experiments/*.jsonl`, `analysis/*.csv`, `REPORT.md` | Hand-copying numbers into HTML when it's time to publish |
| Adding a new model | Edit `<td>` in HTML, then discover the charts didn't follow |
| Leaderboard shows only your own six models | It reads like advertising, not a benchmark |
| A reader sees "75.6%" | No idea whether that is good — there is no random baseline |
| A reviewer asks how invalid answers were counted | You have to dig through the paper to answer |

---

## What this does

**1. Every number is recomputed from CSV — never hand-written into HTML.**

Each figure on the page comes from `data/*.csv`, rendered by `build_site.py`.
Want to change a number on the page? Change the CSV and re-run.

**It renders; it does not audit your data.** Structural problems (a missing column, a referenced
file that doesn't exist) block the build. Data-plausibility questions — `correct / n` disagreeing
with the primary metric, uneven denominators — only print a warning and the page is still produced.
Opt into strictness yourself with `--strict`.

**2. The sections a leaderboard needs are already built.**

No hand-written tables. Enable what you want in `site.yaml`; whatever you don't configure
simply doesn't render:

- **Leaderboard** — column sorting, filter by model/org, confidence intervals, top row marked
- **Per-dimension heatmaps** — **column-wise independent color scales**
  (difficulty differs across columns; one global scale flattens those differences away)
- **Random baseline anchor** — `baseline 25.0% ▏best 75.6% ▏+50.6pt`, appears once `baseline` is set
- **Evaluation protocol footnote** — denominator, invalid handling, CI method, aggregation weights,
  straight from `data/metrics.md`
- **Submission guide** — how someone else gets their model onto your board

**3. One generator, not a pile of HTML to hand-edit.**

```bash
python scripts/build_site.py --source docs --out _site
```

`site.yaml` is the only file you write by hand. Data changes → re-run → the page always matches the numbers.

---

## Install

### Claude Code

```bash
/plugin marketplace add MonsterPPPP/18trees-benchmark-pages-skill
/plugin install benchmark-pages@18trees-benchmark-pages-skill
```

### Manual

Copy `skills/benchmark-pages/` into `.claude/skills/` (project) or `~/.claude/skills/` (user).

For Codex, use `.codex/skills/`. See [AGENTS.md](./AGENTS.md) for details.

### Dependencies

```bash
pip install -r scripts/requirements.txt
```

PyYAML is the only dependency. The Tabulator table engine ships with the repo —
**generated sites are fully self-contained and work offline.**

---

## Usage

### 1. Gather three inputs

| Input | Typical form |
|-------|-------------|
| **rawdata** | `experiments/<run>/*.jsonl`, `analysis/canonical/*.csv` |
| **paper** | LaTeX source / PDF; or a finalized title, author list, abstract |
| **analysis report** | `analysis/REPORT.md`, `analysis/figures/*.png` |

### 2. Let the skill run its six phases

```
Phase 0  Inventory the inputs      ← what is data, what is narrative
Phase 1  Lock the evaluation terms ← denominator / invalid / CI / aggregation / baseline
Phase 2  Build three tables        ← leaderboard.csv + breakdown_*.csv + items.csv
Phase 3  Write site.yaml           ← the only file written by hand
Phase 4  Render + self-check       ← build_site.py (exits if numbers disagree)
Phase 5  Verify locally + deploy   ← http.server page by page → GitHub Pages
```

### 3. The site source directory

```
docs/
├── site.yaml                 the only file you write by hand
├── data/
│   ├── metrics.md            evaluation protocol
│   ├── leaderboard.csv       main board: one row per entrant
│   └── breakdown_*.csv       per-dimension tables
├── figures/
└── pdfs/
```

The output is `_site/`: `index.html` + `leaderboard.html` + `static/`, ready for GitHub Pages.

A complete, runnable example lives in [`site/`](./site/).

---

## Repository layout

```
├── skills/benchmark-pages/
│   ├── SKILL.md                    single source of truth for the rules
│   ├── references/                 input contract / leaderboard spec / site spec / deploy
│   ├── assets/template/            page templates (derived, CC BY-SA 4.0)
│   └── assets/vendor/tabulator/    table engine (MIT, shipped with sites)
├── scripts/
│   ├── build_site.py               the generator
│   └── build-dist.sh               single-file build
├── dist/benchmark-pages.md         single-file full-constraint version (generated)
├── site/                  demo site source directory (real data)
├── .github/workflows/pages.yml     build + publish the demo site
└── NOTICE.md                       upstream credits and license layering
```

---

## Development

```bash
# regenerate the single-file version after editing rules
bash scripts/build-dist.sh

# smoke test
python scripts/build_site.py --source site --out _smoke
python -m http.server 8000 --directory _smoke
```

**A passing generator does not mean a working page.** At minimum, verify in a browser that the
table renders, header clicks sort, narrow viewports do not overflow horizontally, and the model
column stays leftmost and visible. This step is mandatory after touching the template or generator.

Contribution directions are in [CONTRIBUTING.md](./CONTRIBUTING.md).

---

## Comparison with related projects

> Snapshot: 2026-09-22. Stars and last-commit dates from the GitHub API.

| Project | ★ | Last commit | License | What it is | What it lacks |
|---------|---|-------------|---------|-----------|---------------|
| [Academic-project-page-template](https://github.com/eliahuhorwitz/Academic-project-page-template) | 5,229 | 2025-09-04 | ⚠️ no LICENSE file | Academic paper homepage template; the whole field forks it | No tables, no leaderboard, no submission guide — you hand-roll the benchmark part |
| [swe-bench/swe-bench.github.io](https://github.com/swe-bench/swe-bench.github.io) | 15 | 2026-09-01 | CC BY-NC 4.0 | Homepage + five leaderboards on one site, generated with Python/Jinja2 | Tightly coupled to swe-bench; non-commercial license |
| [evalplus/evalplus.github.io](https://github.com/evalplus/evalplus.github.io) | 13 | 2024-12-26 | Apache-2.0 | `results.json` + hand-written JS sorting and filtering | Solves only "render my own board" — no paper page, no submission flow, no protocol disclosure |
| [vividvilla/csvtotable](https://github.com/vividvilla/csvtotable) | 1,183 | 2026-08-24 | MIT | CSV → self-contained sortable HTML | A table tool, not a benchmark tool: no baseline, no protocol, no per-dimension views, no deployment |
| [Tabulator](https://github.com/olifolkerd/tabulator) | 7,771 | 2026-09-15 | MIT | General-purpose table library | A component, not a product — this skill uses it for rendering but supplies the benchmark-specific conventions |

### Which to pick

| Your situation | Use |
|---------------|-----|
| A project page for an ordinary paper, no leaderboard | Academic Project Page Template directly |
| A one-off page; the data will never change | evalplus-style `results.json` + hand-written JS |
| Turn a CSV into a sortable table you can send to someone | csvtotable |
| **Leaderboard data keeps changing and every edit means a re-render** | **this skill** |

**That last row is its only target scenario.** The value here is not "a prettier page" —
it is that **you change the data, re-run once, and the page follows, with every number still
matching the CSV**. The leaderboard sections (sorting, heatmaps, baseline anchor, protocol
footnote, submission guide) come along for the ride: they always had to exist, everyone just
used to write them from scratch.

---

## License

This repository is licensed under **CC BY-SA 4.0**, because the page templates are derived from
[Academic Project Page Template](https://github.com/eliahuhorwitz/Academic-project-page-template)
and [Nerfies](https://nerfies.github.io/) (both CC BY-SA 4.0).

- **Template layer** (`assets/template/`) and pages generated from it: CC BY-SA 4.0,
  **the footer upstream backlinks must be preserved**
- **Generator and rule text** (`scripts/`, `SKILL.md`, `references/`): original work by 十八木,
  additionally licensed **MIT**
- **Tabulator**: MIT, © Oliver Folkerd

Full upstream credits, third-party notices, and license layering are in [NOTICE.md](./NOTICE.md).
