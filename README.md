# Shift-notes

Granite 4.2 8B reads each note line and labels it with a formal constraint (or `NONE` for chatter). A small brute-force solver does all the reasoning: it counts the consistent schedules, which gives unique / ambiguous / inconsistent, and for inconsistent items it finds a minimal set of lines that contradict each other.

On the 60 visible items this scores about **50 to 55% macro exact match at 1x** (four runs: 58.3, 55.0, 50.0, 48.3), against 0.0% for the raw model. The curve is flat: 52% at 3x, 54% at 10x.

```
python run items.json --budget 1x --out answers.json     # 1x, 3x or 10x
python score.py visible_key.json answers.json
```
Python 3.11.4, `pip install -r requirements.txt`. `run` reads the endpoint, model and key from a `.env` next to it (not included).

## What it does

Everything is in the one file `run`.

1. **Header parse (regex).** People, block times, stations, who holds a station.
2. **Extraction (the model).** One call per item: the numbered note lines go in, one answer per line comes out, from a closed list: `BLOCK`, `NOT_BLOCK`, `STATION`, `NOT_STATION`, `BEFORE`, `DIRECTLY_BEFORE`, `BETWEEN`, `NONE`.
3. **Reply parse (string matching).** Tuples are checked against the header; anything invalid becomes `NONE`.
4. **Solve.** All 720 block-and-station worlds are enumerated; each constraint is a bitmask over them. 0 survivors is inconsistent, 1 is unique, 2 or more is ambiguous.
5. **Citations.** For inconsistent items, lines are deleted one by one, keeping each deletion that leaves the rest still contradictory. What remains is a minimal conflicting set, quoted verbatim.

At 3x/10x the prompt is sampled 3/10 times with a per-line plurality vote. Every call sends `X-Item-Id`, `temperature=1.0`, `top_p=0.95`, reasoning off. Failed calls are **not retried**, since a hidden retry would break "exactly one call" at 1x.

**Model-based or symbolic extraction?** Model-based. No regex touches the note wording; only the header is regex-parsed. The model's tuples are the solver's only input, so changing the model's output changes the answers.

## Ablation

Macro exact match on the 60 visible items, mean over `n` runs (temperature 1.0 makes single runs vary by about 5 points). U / A / I are per-kind rates.

| Configuration | 1x | 3x | 10x |
|---|---|---|---|
| **full system** | **54.4%** n=3 (U60 A48 I55) | **52.2%** n=3 (U67 A42 I48) | **54.2%** n=2 (U60 A52 I50) |
| no few-shot examples | 13.3% n=3 | 14.4% n=3 | 14.2% n=2 |
| no rules block | 28.3% n=3 | 31.1% n=3 | 33.3% n=2 |
| cite every extracted line (no minimal core) | 34.4% n=3 (I = 0) | 34.4% n=3 (I = 0) | 35.8% n=2 (I = 0) |
| no solver (model answers the item itself) | 0.0% n=3 | 0.0% n=3 | 0.0% n=1 |

Full-system runs: 1x 58.3 / 55.0 / 50.0, 3x 53.3 / 51.7 / 51.7, 10x 55.0 / 53.3. One more 1x run (the one behind the shipped `answers.json`) scored 48.3% and is not in the averages; adding it, the 1x mean is about 53%. `tools/ablate.py` re-runs the grid; finished runs are cached in `ablation_runs/` (delete the folder to start over). `n` is small because I stopped the sweep to save calls, so differences under about 5 points should be read as noise.

What the table says:
- **The examples matter most.** Without them the model rarely produces the format.
- **The rules block is worth about 20+ points**, mostly by telling the model what to ignore (counterfactuals, past arrangements, errands) and which way round `BEFORE` goes.
- **Minimal-core search is worth about 20 points at every budget.** Citing every line scores exactly 0% on inconsistent items, because a non-minimal set fails the citation rule.
- **The solver is the whole point.** Asking the model for the answer directly gives 0.0%, matching the brief.
- **Extra calls don't help.** 1x, 3x and 10x are within noise.

Confusion matrix, one full-system 1x run (rows are true kind):

| true \ declared | unique | ambiguous | inconsistent |
|---|---|---|---|
| unique | 12 | 7 | 1 |
| ambiguous | 1 | 16 | 3 |
| inconsistent | 3 | 2 | 15 |

The largest error is unique declared as ambiguous: the model dropped a constraint, so the solver correctly found several schedules. That is the honest-failure direction. The other errors come from a wrong constraint making a solvable item look contradictory, or a missing one hiding a contradiction.

## How I characterised the model, and why each part exists

Asked to solve an item outright, the model gets 0%: it cannot hold twenty hedged lines and a permutation constraint at once. So I asked a smaller question, whether it can translate one sentence at a time, and wrote `tools/diagnose.py`, which flags every extracted constraint that is false in the visible key's true assignment. The first prompt's errors were:

- **Direction flips** on "takes over from", "comes later than", "by the time X starts, Y has already been on".
- **`BETWEEN` argument order**: the wrong person ended up in the middle.
- **Stations read as blocks**: "not on calibration" became a `NOT_BLOCK`.
- **Invented constraints** from pure filler lines.
- **Copy slips**: a name that isn't in the line. This is the main remaining error, and prompting did not fix it.

I rewrote the prompt in response (a "who goes first" rule, station-word rules, 25 varied examples in invented names and wording, not copied from the items). One 1x run went from 41.7% to 58.3%.

Why each component is there: the **solver** does the counting the model can't, and gives the three kinds a formal meaning (nothing is guessed). The **closed vocabulary with `NONE`** makes output checkable and chatter a first-class answer. **Examples and rules** are measured above. **Minimal-core search** produces exactly what the citation rule asks for. **Voting** is the obvious use of a bigger budget.

**What I built and removed:** not much. The one real iteration was the prompt (above). I kept voting even though it showed no gain, so the 3x/10x numbers are real.
