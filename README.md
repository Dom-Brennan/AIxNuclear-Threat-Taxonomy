# AIxNuclear-Threat-Taxonomy
The AIxNuclear Taxonomy is an interactive and evolving taxonomy to capture the effects of AI on nuclear risk. This taxonomy leverages agentic AI to generate, interrogate, and qualify AIxNuclear threats. By doing so, we can visualize gaps that are in need of resilience-building.

To access this taxonomy, download 'dashboard.html' and 'seed_taxonomy_populated.json'. Open the Dashboard with any web browser (Safari, Google Chrome, etc.) and click 'LOAD TAXONOMY JSON' at the top-right of the page. Load 'seed_taxonomy_populated.json'.

A python script has been created to automate this procedure. This script has not yet been shared. This script can be requested.

Eventually, an open-weight model will be trained to replicate the quantitative and qualitative evaluation of AIxNuclear threat according to the taxonomy populated by specialists in nuclear engineering, nuclear security, nonproliferation, AI design, cyber security, and so on.

This taxonomy and all related data are built without security clearance or any proprietary information.

# AIxNuclear taxonomy tooling -- command reference

Everything here reads/writes files in this same directory. `pipeline.py`
needs an Anthropic API key (`ANTHROPIC_API_KEY` in the environment, or a
`.env` file -- it's loaded via `python-dotenv`); the other scripts are pure
local data processing and don't call the API.

Three scripts you'll actually run day to day: **`pipeline.py`** (generate
and score entries), **`learn_from_curation.py`** (turn a hand-reviewed file
into a smarter matcher), **`update_dashboard.py`** (push that smarter
matcher into `dashboard.html`). Everything else here is either a one-time
setup step or something those three call internally.

---

## 1. `pipeline.py` -- the main tool

```
python pipeline.py [flags]
```

Flags can be combined in one call; they run in a fixed order regardless of
how you list them: `--merge-candidates` -> `--entries`/`--entry-index` ->
`--score-seed` -> `--use-case` -> `--generate-literature` -> `--generate`.
Running with no flags at all prints a reminder and does nothing.

### Rescoring existing entries

| Flag | What it does |
|---|---|
| `--entries RANGE` | Re-runs the red-team panel (3 personas + synthesis) on entries in `seed_taxonomy_populated.json` by 0-based index, e.g. `0-4`, `5-12`, or `all`. Works on any entry regardless of schema. Safe to run in batches -- accumulates into the output file rather than overwriting. |
| `--entry-index N` | Same, but a single entry -- for spot-checking one result. |
| `--populated-out PATH` | Where `--entries`/`--entry-index` write results, and (for the generate flags) which file is read as the base taxonomy. Default: `seed_taxonomy_populated.json`. |
| `--score-seed` | Backfills DREAD scores onto `seed_taxonomy_atlas.json` entries that don't have one yet. |
| `--force-rescore` | With `--score-seed`, re-scores entries that already have a `dread_score` instead of skipping them. |
| `--out PATH` | Where `--score-seed` writes the updated file. Default: overwrite `seed_taxonomy_atlas.json` in place. |

```bash
python pipeline.py --entries 0-4
python pipeline.py --entry-index 12
python pipeline.py --score-seed
python pipeline.py --score-seed --force-rescore
```

### Generating new entries

Three different ways to get a new candidate entry into the pipeline, all
ending up in `new_candidate_entries.json` and all running the same
downstream steps afterward (red-team panel, ATLAS/Operational/STRIDE
enrichment, evidence search, DREAD scoring).

| Flag | What it does |
|---|---|
| `--generate N` | The default mode: invents N use cases to fill in whichever `(use_area, facility_domain)` matrix cells are currently thinnest (or, with `--mode gaps`, only cells that are completely empty). |
| `--mode {spread,gaps}` | Only affects `--generate`. `spread` (default): always target the thinnest cell, even if none are empty -- keeps coverage even. `gaps`: only target empty cells -- switch to this once literature is your main source, since an empty cell then genuinely means "the literature doesn't cover this." |
| `--use-case TEXT` | Give it a use case you already have (typed by hand, or read out of a paper) and it classifies `use_area`/`facility_domain` for you, rather than inventing the use case itself. The text is never altered. Pair with `--facility-domain` to hint the classification (still verified against the text, not blindly applied). |
| `--generate-literature N` | Searches the web (restricted to peer-reviewed journals, NRC ADAMS, IAEA documentation, WINS, and think-tanks like VCDNP -- see `APPROVED_LITERATURE_SOURCES` in the file) for up to N *real*, citable use cases instead of inventing any. Reports `not_found` rather than guessing when nothing qualifies, so you'll often get fewer than N -- that's expected. Every accepted entry gets a `source_citation` field, printed for your own verification. |
| `--facility-domain X` | A soft preference for `--use-case` (verified, can be overridden) or `--generate-literature` (a preference, not a hard constraint -- it'll report `not_found` rather than stretch an off-domain source to fit). One of `power_generation`, `enrichment`, `reprocessing`, `fuel_fabrication`, `waste_storage_transport`. |
| `--interactive-dread` | Pop up the DREAD review window so you can hand-adjust the LLM's scores before they're saved. Off by default (LLM scores accepted as-is). |
| `--no-evidence-search` | Skip the literature-evidence search step. Faster/cheaper, or useful if your key lacks `web_search` tool access. |

```bash
python pipeline.py --generate 15
python pipeline.py --generate 15 --mode gaps --interactive-dread
python pipeline.py --use-case "Automated corrosion detection on dry cask storage via drone imagery" --facility-domain waste_storage_transport
python pipeline.py --generate-literature 5
python pipeline.py --generate-literature 5 --facility-domain enrichment
```

**Duplicate checking**, on every generation path: each new candidate is
checked against *every* entry already in the taxonomy *and* every entry
generated earlier in the same run (so entry 90 of a `--generate 100` run is
checked against entry 5, not just the pre-existing taxonomy) -- this is a
fast lexical check, always on. At the end of a `--generate`/
`--generate-literature` batch, one extra pass reviews the whole finished
batch for *semantic* near-duplicates (same underlying application,
different wording) that the lexical check can't catch -- it only flags
these for you to look at, never removes anything automatically.

### Reviewing and merging what was generated

Candidates from any of the three generate flags land in
`new_candidate_entries.json`, **not** directly in the taxonomy. Open that
file (or the dashboard), edit/delete entries by hand, then:

```bash
python pipeline.py --merge-candidates
```

This appends everything currently in `new_candidate_entries.json` onto
`seed_taxonomy_populated.json` and clears the candidates file. Run it after
you've reviewed, not before.

---

## 2. `learn_from_curation.py` -- teach the matcher from a reviewed file

```
python learn_from_curation.py [--path FILE] [--reapply-boosts]
```

Run this after you've hand-reviewed a taxonomy file (fixed mis-tags,
deleted bad suggestions, maybe typed in an ATLAS/Operational id yourself).
It promotes every confirmed match into `learned_vocab_atlas.json` /
`learned_vocab_operational.json` (the fast exact-match tier both
`pipeline.py` and the dashboard consult first) and folds the wording into
`atlas_keywords_tfidf.json` / `operational_keywords_tfidf.json` (so
similarly-worded *future* text scores better too, not just an exact
repeat).

| Flag | What it does |
|---|---|
| `--path FILE` | Which file to learn from. Default: `seed_taxonomy_populated.json`. |
| `--reapply-boosts` | Only needed once, after rebuilding the TF-IDF index files from scratch (see `build_atlas_keyword_index.py` below) -- re-applies every *previously* learned boost, not just new ones from this run, since a from-scratch rebuild has no memory of past learning. |

```bash
python learn_from_curation.py
python learn_from_curation.py --path seed_taxonomy_atlas_edited_2026-09-03.json
python learn_from_curation.py --reapply-boosts
```

Safe to re-run on the same file -- already-learned terms are skipped, not
duplicated.

---

## 3. `update_dashboard.py` -- push the learned vocab into the dashboard

```
python update_dashboard.py [--check] [--out FILE] [--no-backup]
```

Regenerates the data block embedded in `dashboard.html` (full ATLAS/
Operational catalogs, TF-IDF indices, and learned vocabulary) and splices
it in automatically -- no manual copy-paste. Run this any time
`learn_from_curation.py` changes the learned vocab, or after editing
`operational_taxonomy.xlsx`.

| Flag | What it does |
|---|---|
| *(none)* | Regenerates and overwrites `dashboard.html` in place, keeping a `.bak` copy of the previous version. |
| `--check` | Reports whether an update is needed and changes nothing -- exits with code 1 if it's stale. Useful before a commit. |
| `--out FILE` | Write to a different file instead of overwriting `dashboard.html`. |
| `--no-backup` | Skip the `.bak` copy. |

```bash
python update_dashboard.py
python update_dashboard.py --check
```

Typical loop after reviewing a file:

```bash
python learn_from_curation.py --path my_reviewed_file.json
python update_dashboard.py
```

---

## 4. `dashboard.html` -- the review UI

Just open it in a browser -- no server, no build step. It reads the data
block that `update_dashboard.py` keeps current. Lets you browse entries,
edit tags, and get the same TF-IDF/learned-vocab tag suggestions
`pipeline.py` computes (same scoring method, verified to match) -- the only
difference is `pipeline.py` auto-applies every resolved match (learned tier
and keyword-match fallback alike) as it enriches an entry, while the
dashboard surfaces the identical candidates as suggestions you click Apply
or Ignore on, one at a time. It does not write back to any file on its own; changes
you make in the browser need to be exported/saved through whatever the
dashboard's own export mechanism is (check its UI -- this wasn't something
this session touched).

---

## 5. Setup / rarely-run scripts

You generally only touch these when changing the underlying catalogs, not
as part of day-to-day taxonomy work.

| Script | Run it when... | What it produces |
|---|---|---|
| `build_atlas_keyword_index.py` | ATLAS releases a new version of `atlas-data` and you've refreshed `atlas_full_catalog.json` from it. | `atlas_keywords_tfidf.json` (from scratch -- wipes any accumulated learning; run `learn_from_curation.py --reapply-boosts` after). |
| `build_operational_keyword_index.py` | `operational_taxonomy.xlsx`'s Taxonomy sheet changes. | `operational_keywords_tfidf.json` (same caveat as above). |
| `build_workbook.py` | You want to regenerate `operational_taxonomy.xlsx` itself from the Python-side operational taxonomy data. | **Caveat:** as written, it saves to the hardcoded path `/mnt/user-data/outputs/operational_taxonomy.xlsx` -- that's this session's sandbox output folder, not a path that exists on your own machine. Edit the `wb.save(...)` line at the bottom of the file to a real path before running it locally, or it'll error out trying to write there. |
| `apply_atlas_mapping.py` | You've edited `seed_taxonomy.json` directly (rare -- most work now goes through `pipeline.py --generate`/`--use-case`/`--entries` against `seed_taxonomy_populated.json` instead). | `seed_taxonomy_atlas.json`, re-enriched with ATLAS/Operational/STRIDE tags. |
| `generate_dashboard_data.py` | Only if you want the raw generated JS block by itself (e.g. to inspect it) -- normally `update_dashboard.py` calls this for you and splices the result in, so you don't need to run it directly. | Prints the data block to stdout. |

```bash
python build_atlas_keyword_index.py
python build_operational_keyword_index.py
python learn_from_curation.py --reapply-boosts   # after either of the above
python update_dashboard.py

python build_workbook.py
python apply_atlas_mapping.py
```

`operational_mapping.py` also runs a quick self-check if executed directly
(`python operational_mapping.py`) -- reports which `seed_taxonomy.json`
terms have neither a real ATLAS nor a real Operational match. Diagnostic
only, writes nothing.

---

## 6. Common end-to-end recipes

**Generate a batch, review, merge:**
```bash
python pipeline.py --generate 20
# review/edit new_candidate_entries.json (or the dashboard)
python pipeline.py --merge-candidates
```

**Add a specific use case you found yourself:**
```bash
python pipeline.py --use-case "Your use case text here" --facility-domain enrichment
python pipeline.py --merge-candidates
```

**Pull real use cases from literature instead of inventing any:**
```bash
python pipeline.py --generate-literature 10
# check each source_citation before trusting it
python pipeline.py --merge-candidates
```

**After hand-reviewing a file, make the matcher smarter and update the dashboard:**
```bash
python learn_from_curation.py --path your_reviewed_file.json
python update_dashboard.py
```

**Bring an old/pre-Operational-taxonomy entry up to date:**
```bash
python pipeline.py --entries all
```
