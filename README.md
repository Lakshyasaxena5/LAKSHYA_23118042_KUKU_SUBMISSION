# Kuku promo finder

This project proposes and reviews short promos from timestamped microdrama
screenplays. It includes a local Streamlit interface and a command-line batch
pipeline. The default local configuration uses Ollama and needs no API key.

**Current evidence:** Two real local-model runs processed all 50 episodes. Both
are marked partial because some validated candidates have no accepted judge
score. The user reviewed the top ten from Run A. The real Streamlit interface
was opened locally and ran one episode from the same show; new-show format
generalization remains untested. See `SUBMISSION_STATUS.md`.

The code repository does not contain the confidential input pack, model
requests, raw outputs, or the reviewer's source-dependent checklist.

## Start the Streamlit app on Windows

Install Python 3.10 or newer and Ollama. Download the local model once with
`ollama pull qwen3:4b-instruct`. Open PowerShell in the project folder and run:

```powershell
py -3 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m streamlit run app.py
```

The browser will open the app. Upload a ZIP of screenplay `.txt` files, select
**Local Ollama**, and click **Find promos**. Running a complete show can take a
long time on a CPU. The interface provides source text, scoring details and CSV,
JSON, evaluation and audit downloads. Its input copy is saved to `.runtime/`;
outputs are saved under `raw_outputs/`, and both are excluded from Git.

To inspect an earlier CLI run in the app, place its `run_A` or `run_B` folder
under this project's `raw_outputs/`, upload the same screenplay ZIP, and choose
**Open saved results**. The app checks the parsed-show fingerprint before it
shows the results against the uploaded scripts.

## Start the Streamlit app on macOS or Linux

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m streamlit run app.py
```

The local Ollama service and model must be running on the same computer. The
interface is intended for local, single-user review, not an unsecured public
deployment containing confidential scripts.

## Tests

```powershell
py -3 -m unittest discover -s tests -q
```

The suite includes mocked-model and mocked-UI tests. A passing test does not
establish editorial quality or replace an actual browser/model run.

## Solution note

`submission/Kuku_Promo_Solution_Final.pdf` is the two-page solution note built
from the actual 50-episode run and the separate manual audit. The corresponding
walkthrough is `submission/Kuku_Promo_Walkthrough_Script.md`.

## Command-line batch pipeline

The batch pipeline reads a screenplay ZIP or folder, proposes promos with a
configured model, checks source evidence, judges valid candidates, and writes
ranked results, evaluation, and manual-audit files.

**Actual-run status:** Two Stage 4 runs with the user's local model processed
all 50 scripts; both runs are partial because some judge results were missing or
rejected. Run A produced 81 ranked and 15 unjudged candidates; Run B produced
71 ranked and 12 unjudged candidates. Passing structural checks does not prove
editorial quality. Source-dependent raw outputs stay outside the repository.

## Windows — run the supplied show locally

This assumes you already installed Ollama and downloaded `qwen3:4b-instruct`.
If not, install Ollama from
https://ollama.com/download/windows, open a new PowerShell window and run
`ollama pull qwen3:4b-instruct`. This model does not require an API key.

1. Create an `input_pack` folder:

```powershell
New-Item -ItemType Directory -Force input_pack | Out-Null
```

Place the authorized screenplay ZIP in `input_pack` with the name `show.zip`,
then check the path:

```powershell
Test-Path ".\input_pack\show.zip"
```

The command should say `True`. The input pack is confidential and is not
included in this repository.

2. Run the checks, then start the full run:

```powershell
py -3 -m unittest discover -s tests -q
py -3 run_stage4.py --input ".\input_pack\show.zip" --config ".\config.stage4.local.json" --run-label A
```

The tests include a **simulated** HTTP 429; that message is expected and is not
an actual quota error. The second command runs the actual model on **all parsed
episodes**. Keep Ollama running and keep the laptop awake. Depending on hardware
and output, this may take tens of minutes or much longer. A GPU can help but is
not required. Do not assume a partial run succeeded because files exist.

3. To measure run-to-run stability, run a second, separately labelled pass and
compare its top 10:

```powershell
py -3 run_stage4.py --input ".\input_pack\show.zip" --config ".\config.stage4.local.json" --run-label B
py -3 compare_runs.py --a ".\raw_outputs\run_A\promos.json" --b ".\raw_outputs\run_B\promos.json"
```

The A/B labels change the prompts and cache keys. This produces separately
requested model results, although it does not prove statistical independence.
If a run is partial, the comparison explicitly says so. Stability is exact
matching of episode/start/end segments; similar alternative cuts are not counted.

4. Open the results:

```powershell
explorer.exe ".\raw_outputs"
```

Each `run_A` / `run_B` folder contains `promos.json`, `promos.csv`,
`ranked_report.md`, `eval_report.md`, `manual_audit.md`, `manual_audit.csv`,
`WHERE_AI_GOT_IT_WRONG.md`, and private raw request/response bodies. The
comparison writes `stability_report.md` and updates both evaluation reports.
Results, audit sheets, cache files and raw logs can contain screenplay text;
keep them private.

## Interruptions and troubleshooting

- Rerun the **same command and label** after an interruption. Completed calls are
  read from the disk cache; a new label intentionally requests new outputs.
- A model response that was successful HTTP but invalid JSON is cached too. If
  both JSON attempts were invalid, an identical rerun reuses them; use a new
  label to request fresh responses after reviewing the failure.
- The program stops after three consecutive episode-provider failures, saves a
  partial result, and can be resumed with the same label. Its `partial` status
  also covers missing judge results or an interrupted run.
- If Ollama is unreachable, open the Ollama app. If the model is missing, run
  `ollama pull qwen3:4b-instruct`. If the laptop runs out of memory, close other
  apps; a longer context or larger model needs more memory.
- `context_tokens` is 16384 in the local config. The request asks Ollama not to
  truncate its input. A very long new show can exceed that limit; increasing it
  requires more memory and may be impractical on a small laptop.
- Open `eval_report.md` and `run_state.json` to see exactly what was processed,
  what failed, and which candidates remain unjudged. Unjudged candidates appear
  as `unscored` in JSON and CSV; they are not assigned invented judge scores.

## Use a new show

Copy its screenplay ZIP into `input_pack` with a new name, then change the path
after `--input`. A folder of `.txt` files in the same format can also be used.
Episode timelines are separate and may restart at zero. The parser reports
malformed blocks and continues when possible; inspect all warnings.

## How the program works

1. `screenplay.py` parses metadata, scenes and timed events; `validators.py`
   checks cuts, duration, chronology and source evidence.
2. `window_options.py` builds a **limited** set of source-grounded 20–60 second
   options, reserving some starting options for actions. Simple keywords help
   find potential action hooks; they do not prove an action is compelling. The
   model selects options rather than calculating all timings itself. A limited
   shortlist can miss a better clip.
3. `stage4_prompts.py` asks the model to select candidates and provide editorial
   explanations. `stage4_logic.py` constructs some prevalidated two-episode
   options in batches of ten episodes; the model can choose none. This bounded
   method is not an exhaustive search across all episode combinations.
4. The judge's source-event excerpts contain only material inside each proposed
   clip, to reduce the observed error of treating events after the cut as included
   footage. It also receives the candidate's explanation. It uses the same
   configured model as the proposer, so the assessments are **not independent**.
   The human audit separately shows source events after the final cut, clearly
   marked as outside the promo.
5. Code validates the judge's hook event ID, then inserts the exact hook text from
   the source. It does not claim the model itself reproduced the line. Quoted
   excerpts in explanations must occur verbatim within one selected source event;
   unquoted paraphrases and factual relationships still require human review.
6. Code scores judged candidates, removes overlaps, applies diversity penalties,
   and exports the results. It does not assign judge scores to unjudged candidates.

## Scoring and diversity

All weights are editable in `scoring_weights` in `config.stage4.local.json` and
must sum to 1. The default score is 70% model judgments and 30% code features:

| Dimension | Weight |
|---|---:|
| Hook strength | 0.17 |
| Cliffhanger strength | 0.20 |
| Standalone clarity | 0.15 |
| Emotional intensity | 0.07 |
| Genre fit | 0.05 |
| Spoiler avoidance | 0.06 |
| Speech begins in the first three seconds | 0.08 |
| Dialogue density | 0.06 |
| Length near 35–55 seconds | 0.10 |
| Earlier episode | 0.06 |

Judge scores are mapped from 1–10 to 0–1. Spoiler risk is reversed: a 10 means
**high risk**, so spoiler avoidance is low. Dialogue density is the fraction of
the clip with timed speech (overlapping speech is counted once). Early-episode
preference uses the episode's position in the show. Two candidates with overlap
intersection-over-union of at least `dedup_iou_threshold` are deduplicated.

After that, a greedy re-ranking penalty discourages repeated episodes,
five-episode blocks and literal speaker-label sets. A five-episode block is a
**time proxy**, not a verified story arc; speaker aliases are not resolved.
The displayed final score is `max(0, base score − diversity penalty)`.
Scoring and diversity settings are transparent heuristics, not validated
predictions of audience engagement.

## Evaluation and human review

`eval_report.md` shows structural pass rate, average duration, opening-hook
timing, cuts inside timed dialogue, obvious ellipsis-continuation cuts, observed
rejections and model-self versus model-judge score agreement. If self-scores are
absent, agreement is `N/A`; it is never invented. Even when present, agreement
is between two assessments from the **same model**, not a human benchmark.

`manual_audit.md` contains the source events overlapping each of the top ten
clips, with blank verdict and notes. Full source descriptions that overlap a cut
are explicitly labelled; they may extend beyond the video segment.
`manual_audit.csv` has blank verdict and notes columns. The user must review the
actual material and fill these in. The script can identify cuts inside timed
dialogue and obvious ellipsis continuations, but cannot guarantee a cut is never
mid-sentence. Model rationales can contain unsupported unquoted assertions.

## Model configuration and privacy

- The recorded runs use `config.stage4.local.json`, Ollama and
  `qwen3:4b-instruct` on the local computer. The model can be changed through
  configuration; no different model is claimed as tested here.
- `episodes_per_minute` limits requests, including judging. Cache keys include
  the model, endpoint, request body and run label.
- `eval_report.md` reports available token counts and the partial-run status.
  Local inference has no per-token charge; hardware and electricity still matter.
- `.gitignore` excludes input packs, credentials, caches and raw outputs. Do not
  force-add or publicly upload confidential screenplays, prompts, or logs.
