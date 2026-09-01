# Competitive research agent

Runs six narrow research questions against a subject using the Anthropic server-side web search tool, and returns a claim ledger where each claim carries a source tier, an as-of date, and a confidence label computed from those two things.

## Run

```bash
pip install -r requirements.txt

make serve         # http://127.0.0.1:8000, click "Load sample"
make demo          # CLI, no API key needed
make verify        # 18 assertions
make demo-strict   # same, with --min-confidence medium
make eval          # 14 tiering + confidence assertions, offline and free

export ANTHROPIC_API_KEY=...
python -m scout.cli "Linear" --domain linear.app
python evals/run_eval.py --live   # adds the fabrication probe
```

## What the code does

`scout/agent.py` sends one call per question with `{"type": "web_search_20250305", "max_uses": 6}` attached, and parses the response into `Claim` objects with `statement`, `url`, `as_of` and `status`. The prompt requires `status: "not_found"` rather than a guess when evidence is absent, and caps quotes at 15 words.

`scout/sources.py` grades each claim:

- **Tier** from the URL. A vendor's `/pricing`, `/docs`, `/changelog`, `/security` are primary. That same vendor's `/solutions`, `/why-us` and homepage are tertiary — same domain, marketing register, not evidence. G2, Capterra, Medium, Reddit are tertiary; Reuters, FT, SEC, trade press are secondary.
- **Confidence** from tier plus age: primary under 6 months is high, primary older is medium, secondary under a year is medium, tertiary is always low, and anything undated is low regardless of source.

`scout/verify.py` holds a grounding check with a deliberately hostile prompt — it is told to assume the claim is unsupported until the source text forces otherwise, and to mark a failure "material" if a reader acting on the claim would decide differently. **This module is written but not yet wired into the CLI or the eval runner.** It is a function you can call, not a step that runs.

## Interface

`make serve` runs a FastAPI app. Enter a subject and its official domains, pick which of the six questions to ask, get the claim ledger grouped by question.

Every confidence label states the rule that produced it — "primary, dated 2026-07-15 — primary source, graded on age" — sitting directly under the claim. That is the difference between computed confidence and asserted confidence: a reader can disagree with the grading instead of only being able to disagree with the claim. A three-way filter drops everything below medium or high when the teardown is going into a document other people will read.

`Not found` renders as a dashed red card, not an omission.

## Where things are

| File | Lines | What it holds |
| --- | --- | --- |
| `scout/sources.py` | ~110 | Tier rules, date extraction, confidence table |
| `scout/agent.py` | ~170 | Question set, search loop, claim parsing, Markdown/JSON rendering |
| `scout/verify.py` | ~55 | Grounding prompt and rate calculation — **not yet called by anything** |
| `scout/cli.py` | ~50 | Arguments, `--min-confidence` filtering, drop count |
| `scout/server.py` | ~60 | FastAPI app, `POST /api/research`, `GET /api/questions` |
| `scout/web/index.html` | ~250 | Full UI including the visible confidence derivation |
| `scout/stub.py` | ~40 | Fixed claim set covering every grading path |
| `evals/run_eval.py` | ~115 | 14 tiering/confidence assertions, live fabrication probe |
| `tests/smoke.py` | ~75 | 18 assertions across grading, rendering and the API |

## Evals

**Tiering regression (offline, free, currently 14/14).** Eight URL→tier assertions and six (tier, date)→confidence assertions. These are ordinary code and ordinary code drifts; if `vendor.com/solutions` ever grades as primary, every confidence label downstream is wrong.

**Fabrication probe (`--live`).** Researches a company that does not exist. Every claim should return `status: not_found`. Any confident statement about Vantrelle Systems is a fabrication. This is pass/fail, not a score. **Not yet run.**

**Grounding rate.** The function exists in `verify.py`. The harness to run it over a teardown does not. Unmeasured.

## Limits

- The grounding check is unwired, so a claim can cite a real page that says something adjacent and nothing catches it. This is the largest gap.
- `as_of` is the page's date, not the date of the fact it describes. A 2026 post about 2023 pricing grades as fresh.
- Undated documentation pages grade low even when current.
- Vendor-funded comparison content on a neutral-looking domain is not detected.
- The offline stub returns the same claims for every question, so `make demo` shows the grading working but proves nothing about research quality.

## Model

Sonnet at temperature 0, six searches per question, six questions per teardown. Server-side search means no scraping infrastructure.
