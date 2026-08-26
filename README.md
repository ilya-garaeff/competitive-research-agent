# competitive-research-agent
competitive-research-agent — Competitive teardowns where every claim carries a source tier, an as-of date, and a computed confidence
# Competitive research agent

Produces a competitive teardown where every claim carries a **source tier**, an **as-of date**, and a confidence label derived from those two things — not from how sure the model sounds.

```bash
pip install -r requirements.txt
python -m scout.cli "Example Vendor" --domain example-vendor.com     # no API key needed
python -m scout.cli "Linear" --domain linear.app --min-confidence medium
```

---

## The problem

Ask any model for a competitive analysis and you get a confident, well-organised document. It will be wrong in two specific ways, and both are invisible on the page:

**It repeats marketing as fact.** The vendor's homepage says they are the leading platform for mid-market operations teams. That is a claim about their copywriting, not about their product. It arrives in the summary indistinguishable from their pricing page.

**It repeats things that stopped being true.** Pricing from eighteen months ago, an integration that was deprecated, a limitation that shipped a fix in March. Nothing in a fluent paragraph tells you which sentence is current.

Both failures share a root cause: the output has one register. Everything is stated in the same steady voice, so a reader cannot tell an SEC filing from a G2 review. A teardown that hides source quality is worse than no teardown, because it converts uncertainty into false confidence right before a roadmap decision.

## What this does differently

**Narrow questions, not one big prompt.** Six focused passes — what it does, what it costs, what shipped recently, what it integrates with, what independent sources criticise, how it handles data. Narrow questions produce checkable claims. "Tell me about Competitor X" produces an essay with nothing to verify.

**A claim ledger, not prose.** Each pass returns discrete statements with a URL and a date. Prose gets assembled at the end, from graded claims.

**Source tiering that knows marketing from documentation.** A vendor's own `/pricing` and `/docs` are primary sources. A vendor's own `/solutions` page is tertiary — it is the same domain, and it is not evidence.

| Source | Tier | Why |
| --- | --- | --- |
| `vendor.com/pricing`, `/docs`, `/changelog`, `/security` | primary | The company's own dated factual surfaces |
| `vendor.com/solutions`, `/why-us`, homepage | tertiary | Same company, marketing register |
| Reuters, FT, SEC filings, trade press | secondary | Independent, but reported at a moment in time |
| G2, Capterra, Medium, Reddit, listicles | tertiary | Aggregated, often undated, frequently seeded |

**Confidence is computed, not asserted.**

```
primary   + under 6 months  → high
primary   + older           → medium
secondary + under a year    → medium
secondary + older           → low
tertiary  + anything        → low
anything  + undated         → low
```

An undated page is low confidence no matter how good the outlet. This one rule removes most of the staleness problem, because staleness you cannot measure is staleness you should not trust.

**"Not found" is a valid answer.** The system prompt says so explicitly, and the report renders it in bold. A missing SOC 2 statement is a real finding. A guessed one is a liability.

## Model choice

Sonnet with the server-side `web_search` tool, capped at six searches per question. Server-side search means no scraping infrastructure, no crawler maintenance, and no robots.txt questions — for this scale that trade is obviously right.

Temperature 0. Research output should not vary run to run; if it does, one of the runs is wrong.

The verification pass (`scout/verify.py`) uses the same model with a deliberately hostile prompt: it is told to assume the claim is wrong and to look for reasons the page does not support it. Asking a model "is this correct?" in the same friendly register as the generation gets you agreement, not signal. Flipping the burden of proof is the cheapest quality improvement in the repo.

## Eval plan

```bash
python evals/run_eval.py          # tiering + confidence regression, free, offline
python evals/run_eval.py --live   # adds the fabrication probe
```

**1. Tiering regression (offline, free).** Fourteen assertions over URL→tier and (tier, date)→confidence. These grading rules are ordinary code, and ordinary code drifts. If `vendor.com/solutions` ever starts grading as primary, every confidence label downstream is wrong and nothing else in the suite would catch it.

**2. Fabrication probe (live).** Research a company that does not exist. Every claim should return `status: not_found`. Any confident statement about Vantrelle Systems' pricing is a fabrication, and one fabricated claim does more damage than ten missing ones. This is a pass/fail gate, not a score.

**3. Grounding rate (live).** Re-read each cited page and ask the hostile verifier whether it supports the claim. Reported as a rate plus a count of **material** failures — ones where a PM acting on the claim would decide differently than the source supports. The rate is interesting; the material count is the number that matters.

No published results table here on purpose. Grounding rates depend on the model version, the search index on the day, and the subject you research. A benchmark you cannot reproduce is worse than no benchmark.

## Cost and latency

Six questions, up to six searches each, one call per question: roughly 10–25 seconds and a few cents per teardown. The grounding pass adds one call per claim and is the expensive part, which is why it is opt-in rather than default.

`--min-confidence medium` drops low-confidence claims from the report and prints how many it dropped. Useful when the teardown is going into a document other people will read, where a low-confidence claim tends to lose its label somewhere between the doc and the meeting.

## Failure modes

| Failure | How it shows up | Mitigation |
| --- | --- | --- |
| Citation without support | Claim cites a real page that says something adjacent | The grounding pass; this is the largest remaining risk and it is only partly closed |
| Undated primary sources | Docs pages with no visible date | Graded low. Conservative, and it does mean genuinely current docs get penalised |
| SEO-seeded comparison pages | "Vendor X vs Y" content written by Vendor X | Tertiary hosts list catches the obvious ones; a vendor-funded page on a neutral-looking domain would not be caught |
| Search index gaps | Recent changes missing entirely | `not_found` rather than an invented answer, but absence is still absence |
| Date confusion | Page's publication date vs the date of the fact it describes | Not solved. `as_of` is the page date, which is an upper bound on freshness, not the fact's date |
| Quoting too much source text | Copyright exposure | Prompt caps quotes at 15 words and prefers paraphrase |

## UX decisions

- **Confidence label sits inline with every claim**, in monospace, before the source. It cannot be skimmed past, and it cannot be separated from the claim when someone copies a line into a doc.
- **Low-confidence claims are repeated in a closing section** headed "Read these before you act on them", with a sentence explaining that low confidence means unestablished, not false.
- **Claims are grouped by the question that produced them**, so the reader can see what was asked, not just what was answered — including which questions came back empty.
- **`--min-confidence` prints the drop count to stderr.** A quieter report should never also be a smaller one without saying so.
- **Runs with no API key.** The offline stub returns a fixed claim set that exercises every grading path — a dated docs page, a vendor marketing page, aged trade press, an undated aggregator, and a not-found — so a reviewer can see the grading working in one command.

## What I would do next

1. **Make grounding the default, not a flag.** It is the check that closes the actual gap; the cost argument is weaker than it looks at a few cents per teardown.
2. **Separate page date from fact date.** A 2026 blog post describing 2023 pricing currently grades as fresh. This is the biggest correctness hole left in the tiering.
3. **Diff mode.** Re-run against a stored teardown and report only what changed. Competitive research is a subscription, not a purchase, and the second run should be cheaper and more useful than the first.
4. **Vendor-funded content detection** for comparison pages hosted on neutral-looking domains.
