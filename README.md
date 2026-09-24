# AIxNuclear-Threat-Taxonomy
The AIxNuclear Taxonomy is an interactive and evolving taxonomy to capture the effects of AI on nuclear risk. This taxonomy leverages agentic AI to generate, interrogate, and qualify AIxNuclear threats. By doing so, we can visualize gaps that are in need of resilience-building.

To access this taxonomy, download 'dashboard.html' and 'seed_taxonomy_populated.json'. Open the Dashboard with any web browser (Safari, Google Chrome, etc.) and click 'LOAD TAXONOMY JSON' at the top-right of the page. Load 'seed_taxonomy_populated.json'.

A python script has been created to automate this procedure. This script has not yet been shared. This script can be requested.

Eventually, an open-weight model will be trained to replicate the quantitative and qualitative evaluation of AIxNuclear threat according to the taxonomy populated by specialists in nuclear engineering, nuclear security, nonproliferation, AI design, cyber security, and so on.

This taxonomy and all related data are built without security clearance or any proprietary information.



# AIxNuclear taxonomy tooling -- command reference

Everything here reads and writes files in this same directory. `pipeline.py`
needs an Anthropic API key (`ANTHROPIC_API_KEY` in the environment, or a `.env`
file loaded via `python-dotenv`). The other scripts are local data processing
and don't call the API.

Two things you'll use day to day: **`pipeline.py`** (generate and score
entries) and **`dashboard.html`** (review and edit them).

## Data files

| File | Role |
|---|---|
| `seed_taxonomy_populated.json` | **The** taxonomy. Every command reads from it unless `--output` names an existing file (see below). |
| `new_candidate_entries.json` | Newly generated entries waiting for review. Folded into the taxonomy with `--merge-candidates`. |
| `operational_taxonomy.xlsx` | **Single source of truth** for the Operational (OP) codes, names, groups and descriptions. Read directly by `taxonomy_xlsx_loader.py`; edit the workbook itself to change the taxonomy. |
| `atlas_full_catalog.json` | MITRE ATLAS technique catalog (from `mitre-atlas/atlas-data`, `dist/ATLAS.yaml`). Refresh periodically from upstream. |
| `dread_calibration_log.jsonl` | Append-only log of every LLM DREAD score next to the human-reviewed score (see `future_dread_calibration.md`). Created on the first scored entry. |

## Entry schema

```json
{
  "use_area": "...", "use_case": "...", "facility_domain": ["..."], "use_status": "identified | proposed",
  "incurred_insecurity": "Overview of the insecurity ...\n\nSubtitle for a threat -- implicit, exploitable\nOne or two precise sentences ...",
  "ai_vulnerability_atlas": [{"atlas_id": "AML.T0020", "atlas_name": "...", "atlas_tactics": ["..."]}],
  "ai_vulnerability_operational": [{"op_id": "OP.C0202", "op_name": "...", "op_group": "OP.G02"}],
  "stride": ["..."], "evidence": [...],
  "dread_score": {"damage": {"score": 3, "why": "..."}, ..., "total": 14},
  "red_team_panel": {...}
}
```

`incurred_insecurity` is one text field. It opens with a short overview, then
gives each main threat a subtitle line ending in its type (`-- adversarial`,
`-- implicit`, or `-- implicit, exploitable`) followed by one or two precise
sentences: the AI failure mechanism and its facility consequence, and for
implicit threats, how an adversary could capitalise on them. The pipeline
writes plain text. In the dashboard you can bold or underline parts of it, in
which case it is saved with a small sanitized subset of HTML (`<b>`,
`<strong>`, `<u>`, `<br>`, `<div>`, `<p>`); the pipeline strips that markup
back to plain text wherever it sends the field to the model.

The ATLAS and Operational codes are picked by the red-team synthesis step
directly from the two catalogs, and you add or remove them in the dashboard's
matrix views. ATLAS codes mark adversarial threats, Operational codes mark
implicit ones, and entries carrying both are where they cross over. `stride`
is derived from the ATLAS tactics of the entry's ATLAS codes.

---

## 1. `pipeline.py` -- the main tool

```
python pipeline.py [flags]
```

Flags can be combined in one call; they run in a fixed order regardless of how
you list them: `--merge-candidates` -> `--entries`/`--entry-index` ->
`--use-case` -> `--generate-literature` -> `--generate`. Running with no flags
prints a reminder and does nothing.

**Every entry the pipeline writes is fully scored in the same run:** red-team
panel (three personas plus synthesis) -> `incurred_insecurity` and
ATLAS/Operational codes -> STRIDE -> evidence search -> DREAD. There is no
separate scoring pass.

### `--output PATH` -- where results go

| | |
|---|---|
| Default | `seed_taxonomy_populated.json`, updated in place. |
| Writes | `--entries`/`--entry-index` results and `--merge-candidates`. |
| Reads | `PATH` if it already exists, otherwise `seed_taxonomy_populated.json`. |

Because an existing `PATH` is read back in, repeated runs with the same
`--output` accumulate into it: `--entries 0-4 --output output.json` followed
by `--entries 5-9 --output output.json` gives one file with the full taxonomy
and entries 0-9 rescored. A new `PATH` starts as a full copy of the taxonomy,
so unrescored entries are never dropped. The generate flags read the same base
for their grounding examples and duplicate checks. A relative `PATH` is
relative to the folder you run the command from.

### Rescoring existing entries

| Flag | What it does |
|---|---|
| `--entries RANGE` | Reruns the full pipeline on entries by 0-based index, e.g. `0-4`, `5-12`, or `all`. Only `use_area`, `use_case`, `facility_domain` and `use_status` are read, so it works on entries of any age or schema. Existing `use_case` evidence is kept; `vulnerability` evidence is replaced by the new search, since the insecurity it supported has been rewritten. |
| `--entry-index N` | Same, for a single entry -- for spot-checking. |

```bash
python pipeline.py --entry-index 12
python pipeline.py --entries 0-4 --output output.json
python pipeline.py --entries all
```

### Generating new entries

All generate flags write to `new_candidate_entries.json`, never straight into
the taxonomy, and run the same full pipeline on each new entry.

| Flag | What it does |
|---|---|
| `--generate N` | Fills the thinnest `(use_area, facility_domain)` matrix cells. Each entry is drawn from a real source found by web search for that cell; if nothing citable turns up, the slot is skipped rather than invented. |
| `--allow-unsourced` | With `--generate`: let the model write use cases from its own knowledge instead. These have no tied evidence -- treat them as hypotheses. |
| `--mode {spread,gaps}` | `--generate` only. `spread` (default) targets the thinnest cell even when none are empty; `gaps` targets only empty cells, so an unfilled cell means the literature doesn't cover it. |
| `--generate-literature N` | Searches the literature for up to N real, specific, citable use cases (see `PREFERRED_LITERATURE_SOURCES` and `SOURCE_POLICY` in `pipeline.py`). Reports `not_found` rather than guessing, so fewer than N is normal. The source becomes the entry's first `use_case` evidence item and is printed for you to verify. |
| `--use-case TEXT` | Classifies a use case you already have into `use_area`/`facility_domain` and runs the pipeline on it. The text is used verbatim. |
| `--facility-domain X` | A soft preference for `--use-case` (verified against the text) or `--generate-literature` (reports `not_found` rather than stretching an off-domain source). One of `power_generation`, `enrichment`, `reprocessing`, `fuel_fabrication`, `waste_storage_transport`. |

```bash
python pipeline.py --generate-literature 5
python pipeline.py --generate-literature 5 --facility-domain enrichment
python pipeline.py --generate 15 --mode gaps
python pipeline.py --use-case "Automated corrosion detection on dry cask storage via drone imagery" --facility-domain waste_storage_transport
```

**Duplicate checking** runs on every generation path. Each new candidate is
checked lexically against every entry in the base taxonomy and every entry
generated earlier in the same run. At the end of a `--generate` or
`--generate-literature` batch, one more pass reviews the batch for semantic
near-duplicates (same application, different wording). It only flags these
for you; it never removes anything.

### Options for any run

| Flag | What it does |
|---|---|
| `--interactive-dread` | Pops up the DREAD review window so you can adjust the LLM's scores before they're saved. Off by default. |
| `--no-evidence-search` | Skips the evidence search step. Faster and cheaper, or useful if your key lacks `web_search` access. |

### Reviewing and merging

Review `new_candidate_entries.json` (by hand or in the dashboard), delete or
fix entries, then:

```bash
python pipeline.py --merge-candidates                        # into seed_taxonomy_populated.json
python pipeline.py --merge-candidates --output output.json   # or into another file
```

This appends the candidates onto the `--output` taxonomy, skips any already
present, and clears `new_candidate_entries.json`.

### Evidence and source policy

Evidence search and `--generate-literature` look in `PREFERRED_LITERATURE_SOURCES`
first and fall back to other credible public sources only when those don't cover
the point; fallback sources are stored with `source_type: "Other"` so they stay
visible for review. News, press releases, blogs, vendor marketing, Wikipedia,
forums and AI-generated content are never accepted. Any citation whose URL was
not actually returned by the search is dropped as likely fabricated. Edit
`PREFERRED_LITERATURE_SOURCES` and `SOURCE_POLICY` in `pipeline.py` to change
what counts.

### DREAD scoring

`stride_dread.py` holds the rubric (`DREAD_SYSTEM`), adapted to nuclear
engineering and scored 1-5 per dimension. **Discoverability** means how easily
an attacker could find the vulnerability, not how likely the facility is to
notice a problem. Where the threats in an entry differ on a dimension, the
scorer rates the most severe credible one and names it in the justification.
With `--interactive-dread` you can adjust each score before it is saved; every
score, edited or not, is appended to `dread_calibration_log.jsonl`.

---

## 2. `dashboard.html` -- the review UI

Open it in a browser -- no server, no build step. Load
`seed_taxonomy_populated.json` or `new_candidate_entries.json`.

- **Dashboard tab:** coverage heatmap, STRIDE, DREAD, ATLAS, Operational, the
  ATLAS x Operations co-occurrence heatmap, and four crossover views: (A) a
  per-use-case scatter of ATLAS vs Operational code counts (click a dot to open
  the entry), (B) diverging bars by use area or facility domain, (C) a flow
  diagram from context through Operational group to ATLAS tactic, and (D) a
  chord diagram linking Operational groups and ATLAS tactics. To drop one,
  delete its panel markup, its `renderCrossover*` function and its call in
  `render()`.
- **Editor tab:** per entry, an *Incurred Insecurity* text box with bold and
  underline (Ctrl+B / Ctrl+U), matrix views for ATLAS and Nuclear Operations
  (click a box to tag or untag), STRIDE, DREAD and Evidence. An unscored
  entry's DREAD tab stays empty until you type a score. Nothing is written back
  automatically -- use *Export edited taxonomy* to download the
  edited file.

The dashboard is a static page, so it can't read the workbook itself. It
carries an embedded copy of the ATLAS and Operational catalogs (names,
descriptions, tooltips) and the STRIDE/DREAD text. After you edit
`operational_taxonomy.xlsx`, refresh `atlas_full_catalog.json`, or change
`stride_dread.py`, run:

```bash
python update_dashboard.py           # re-embeds the catalogs into dashboard.html
python update_dashboard.py --check   # only reports whether it's out of date
```

---

## 3. Supporting modules

These are imported by `pipeline.py` and the dashboard scripts; you don't run
them directly.

| File | Role |
|---|---|
| `llm.py` | The model name, `GenerationFailure`, and robust extraction of JSON from model responses, shared by every API call. |
| `taxonomy_xlsx_loader.py` | Reads the Operational taxonomy straight from `operational_taxonomy.xlsx`. |
| `operational_mapping.py` | Operational code/group lookup built from the workbook; validates OP codes the model picks. |
| `atlas_mapping.py` | ATLAS tactic names and descriptions, and validation of ATLAS IDs against `atlas_full_catalog.json` (drops any ID not in the catalog). |
| `stride_dread.py` | STRIDE derivation from ATLAS tactics, and DREAD scoring and rubric. |
| `dread_review.py` | The `--interactive-dread` popup. |
| `insecurity.py` | Converts stored `incurred_insecurity` (including dashboard markup) to plain text for prompts. |
| `generate_dashboard_data.py` | Builds the data block `update_dashboard.py` embeds. |

---

## 4. Common recipes

**Pull real use cases from the literature, review, merge:**
```bash
python pipeline.py --generate-literature 10
# check each printed source, edit new_candidate_entries.json
python pipeline.py --merge-candidates
```

**Rescore the whole taxonomy into a separate file, in batches:**
```bash
python pipeline.py --entries 0-9   --output output.json
python pipeline.py --entries 10-19 --output output.json
# ...
```

**Add a specific use case you found yourself:**
```bash
python pipeline.py --use-case "Your use case text here" --facility-domain enrichment
python pipeline.py --merge-candidates
```

**After editing the workbook:**
```bash
python update_dashboard.py
```
