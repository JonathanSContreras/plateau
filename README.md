# Plateau

A command-line tool that reads a lifting log written in markdown, decides what
each lift's working weight should be next session, and builds a printable
one-page gym card from the result.

**Status: design spec. No code yet.** This README is the contract the first
version gets built against. Anything marked *planned* does not exist.

## Why this exists

A progressive-overload plan is a handful of rules applied to a log. Run by hand,
the rules drift: a number gets bumped because the last session felt good, a
different machine gets compared to an old one, a lift that fell outside its rep
range gets printed at the best weight instead of the working weight.

Plateau makes the rules explicit and repeatable. Every number on the card comes
with a reason you can read, and anything it can't decide is flagged instead of
guessed.

## What it does

1. **Parses** a markdown training log into sessions, lifts and sets.
2. **Keys every lift to a station.** The same lift on two machines with different
   weight stacks is two series. They are never compared.
3. **Applies a rule set** to each lift's history and returns one of: raise,
   hold, chase reps, back off, or baseline.
4. **Explains** each decision in one line, with the sessions it used.
5. **Renders** the card as HTML and PDF, with the working weight and the reach
   weight for each lift.
6. **Archives** the card it replaces, so nothing is overwritten.

It does not generate workouts, track food or sleep, or talk to any service.
Everything runs locally on plain files.

## Concepts

| Term | Meaning |
|---|---|
| **Lift** | A named exercise slot on the plan, such as "Incline DB press". |
| **Station** | The specific machine, stack or implement. `lift@station` is the unit of history. |
| **Range** | The prescribed rep window, such as 8–12. |
| **Working weight** | The load the card prints. The heaviest load that has held the range. |
| **Reach** | A heavier load that has been touched but not held. Printed in small type. |
| **Increment** | The smallest sensible load change for that station. |
| **Hardest** | The lift the session named as the hardest. Optional, and it changes some decisions. |
| **Baseline** | A first entry at a station. It sets a number and never moves the card. |

## Input

### The log

One section per session, newest first or last, either works. Loads and reps are
parallel lists separated by ` · `.

```markdown
## 2026-03-02 Monday — Push A
felt: 7
hardest: Incline DB press

| Exercise | Station | Loads | Reps |
|---|---|---|---|
| Barbell bench press | smith-1 | 95 · 115 · 135 | 8 · 8 · 6 |
| Incline DB press | db-rack | 40 · 45 · 45 | 10 · 10 · 8 |
| Cable lateral raise | cable-a | 11 · 11 · 16.5 | 15 · 15 · 12 |
```

Rules for the parser:

- Loads and reps must have the same length. A mismatch is an error, not a guess.
- `felt` is 1–10 and optional. `hardest` is optional and must match a lift name.
- A station that is missing or empty makes the row **unkeyed**. The row is kept
  and flagged, and it cannot move a card number until you assign a station.
- Prose around the tables is ignored, so the log can hold notes.

### The plan

`plan.toml` describes the card: days, lifts, ranges, increments and starting
loads.

```toml
[plan]
name = "6-Day PPL"
units = "lb"

[[day]]
name = "Push A"
weekday = "mon"

  [[day.lift]]
  name = "Incline DB press"
  station = "db-rack"
  sets = 3
  reps = [8, 12]
  increment = 5
  rest = "2 min"
  start = 40

  [[day.lift]]
  name = "Cable lateral raise"
  station = "cable-a"
  sets = 3
  reps = [12, 15]
  increment = 5.5
  rest = "60 s"
  start = 11
```

`increment` is per station, because stacks differ. A cable stack that moves in
5.5 lb steps cannot be told to add 5.

## The rule set

Each rule reads the history of one `lift@station`, newest session first. The
first rule that matches decides. Defaults are listed here, and every threshold
lives in `rules.toml`.

| # | Condition | Decision |
|---|---|---|
| 1 | Fewer than one prior session at this station | **Baseline.** Record it. The card does not change. |
| 2 | Any set in the session has no rep count | **Flag.** Cannot decide, ask for the reps. |
| 3 | Reps at the working weight are inside the range, below the top | **Chase reps.** Hold the load. |
| 4 | Top of range reached on every working set at the working weight | **Candidate.** Mark it, do not raise yet. |
| 5 | Candidate repeated in the next session at that station | **Raise** by one increment. |
| 6 | Reps below the range and the lift was named hardest | **Back off** to the last load that held the range. |
| 7 | Reps below the range and the lift was not named hardest | **Hold.** Chase reps. Treat the miss as a choice. |
| 8 | New best load, reps inside the range | **Promote** it to working weight. |
| 9 | Load touched above the working weight with reps outside the range | **Reach.** The card prints the working weight and shows this in small type. |

Two behaviours sit above all nine:

- **Station change resets the series.** A lift that moves to a different station
  starts a new `lift@station` history with a baseline.
- **A lower session does not lower the card.** One session under the prior
  numbers is a hold, not a back-off. It takes rule 6 to move a weight down.

Every decision carries the sessions it used, so `explain` can show its work.

## Commands *(planned)*

```
plateau check   log.md                   # parse and validate, list flags
plateau next    log.md --plan plan.toml  # print the proposed working weights
plateau explain log.md "Incline DB press"
plateau card    log.md --plan plan.toml --out card/ --archive old/
```

### `check`

Reports what it could not read, without changing anything.

```
2026-03-02 Push A
  line 11: unkeyed row "Cable fly" (no station) — cannot move the card
  line 14: loads has 3 values, reps has 2 — fix the row
2 flags, 0 files changed
```

### `next`

```
Incline DB press @ db-rack     45  hold      chase 12 (reps 10 · 10 · 8)
Cable lateral raise @ cable-a  16.5 candidate top of range once, repeat to raise
Barbell bench press @ smith-1  135 baseline  first entry, card unchanged
```

### `explain`

```
Incline DB press @ db-rack
  2026-02-23  40 · 45 · 45   × 10 · 9 · 8
  2026-03-02  40 · 45 · 45   × 10 · 10 · 8
  Range 8–12. Reps at 45 are inside the range and below the top on both sessions.
  Rule 3: hold 45, chase 12.
  Reach: none.
```

### `card`

Writes `card.html` and `card.pdf`. If a card already exists in `--out`, the
outgoing pair is copied to `--archive` first, named
`card (rev YYYY-MM-DD, superseded).*`. Nothing is overwritten.

PDF rendering uses WeasyPrint. The HTML is the source, so the card can be tuned
by hand and rebuilt.

## The card

- One page, one column per day, lifts in plan order.
- Each row: lift, sets × range, rest, **working weight**, **reach** in small type.
- A lift with a pending candidate gets a quiet marker, and a lift flagged unkeyed
  prints its last good number with a note.
- Only weights change between revisions. Sets, ranges, rest and day structure
  change when `plan.toml` changes.

## Design

```
plateau/
  parse.py      markdown log  ->  Session, Lift, SetEntry
  model.py      dataclasses, no I/O
  stations.py   lift@station keys, increments, series history
  rules.py      the nine rules, pure functions over a series
  explain.py    turns a Decision into the readable trace
  render.py     Jinja2 template -> HTML -> WeasyPrint -> PDF
  cli.py        argument handling only
tests/
  fixtures/     synthetic logs, one per behaviour
```

Decisions that shape the code:

- `rules.py` is pure. It takes a series and a rule config and returns a
  `Decision`. No file access, no dates from the clock. That makes it testable
  by table.
- The parser never repairs input. It reports.
- Station is part of the key from the first line of the model, not an
  afterthought, because comparing across stations is the failure this tool
  exists to prevent.
- Units are stored as given. No conversion between lb and kg.

## Edge cases the tests must cover

- Reps and loads of different lengths.
- A lift logged with two sets when the plan prescribes three.
- A lift that appears twice in one session.
- A station renamed mid-history.
- A first entry at a new station that is far heavier than the old one.
- Top of range hit on some sets but not all.
- Reps in range at the working weight but the lift was named hardest.
- A candidate that fails to repeat.
- A back-off followed by a return to the old weight.
- A load that is not a multiple of the station's increment.
- Sessions out of date order.
- A plan lift with no history at all.

## Testing

- `pytest` for the parser, rules and rendering.
- Table-driven tests for `rules.py`: a series in, a `Decision` out.
- `hypothesis` property tests: the card never prints a load above any load that
  held the range; reordering sessions by date never changes the decision.
- Golden files for `next`, `explain` and `check` output.
- One render test that builds a PDF and checks the page count is 1.

## Privacy

Training logs are health data. The repo holds only **synthetic fixtures**. Real
logs live outside the repo, and `.gitignore` excludes `*.log.md` and `card/`.
Nothing leaves the machine.

## Roadmap

| Milestone | Scope |
|---|---|
| 0.1 | Parser, model, `check`. |
| 0.2 | Stations and series history. Rules 1–3 and 6–7, `next`. |
| 0.3 | Candidate and raise (rules 4–5, 8–9), `explain`. |
| 0.4 | HTML card and PDF, archive on overwrite. |
| 0.5 | `rules.toml` thresholds, plan validation, property tests. |

## Non-goals

- Generating workouts or choosing exercises.
- Comparing across stations, or across gyms.
- Autoregulation from sleep, food or bodyweight.
- Accounts, sync or a web interface.

## Requirements

Python 3.12, `uv`, Jinja2, WeasyPrint, pytest, hypothesis.

## License

Not chosen yet.
