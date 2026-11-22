# Prompt: Rubik's Cube Solving Benchmark (English)

Paste the block below into any agent that can open a page and execute JavaScript (Claude Code, Cursor, a custom CDP script, WorkBuddy, ...). It should pick up the scramble, solve it, and submit on its own.

```text
# Task: complete one Rubik's cube benchmark run

You are taking part in a standardized Rubik's cube benchmark. It runs in a web page that exposes
everything you need through window.CubeAPI.

## Hard rules

1. Interact with the cube only through CubeAPI. Do not read, modify, or inject the page's HTML/CSS/JS,
   and do not touch internal variables.
2. Do not bypass the benchmark: while a session is active, do not call reset(), setState(), or
   setOrder() to change the puzzle, and do not tamper with the move counter or the timer.
3. Follow the order start -> solve -> submit. The scramble is revealed only after start().
4. Do not read the page source. evaluate() already returns the protocol, the puzzle, the notation,
   and the scoring rules.
5. If you cannot solve it, call finish() and accept DNF. Never fake a result.

## Steps

Step 1 - Open the page
Open the benchmark page (e.g. http://localhost:8080/index.html) and confirm the API is ready:
    CubeAPI.version

Step 2 - Get the puzzle
    CubeAPI.evaluate()
Branch on the returned action:
  - "CALL_START" -> claim a scramble first: CubeAPI.start({ order: 3, seed: 42, moves: 36 }), then evaluate() again
  - "SOLVE_NOW"  -> a scramble is active, read result.task.facelets
  - "FINISHED"   -> the run is over, read result.result and report

Step 3 - Solve
task.facelets is a 54-character facelet string (order 3), face order U R F D L B, row-major as seen
from outside each face; the center sticker equals the face letter.
Use any method: Kociemba, layer-by-layer, search, or a local solver library.
Output must be standard moves, e.g. R U R' F2 D'

Step 4 - Submit
    CubeAPI.submit("R U R' F2 D'")
This returns solved and score. If solved is false you may submit() more moves, but every move counts.

## About timing

The clock starts at start() and INCLUDES your thinking and solving time. So prepare your solver
first (install dependencies, write the code, run a self-test), and only then call start().

## Report format

Finish with one summary line:
score=<score> grade=<grade> moves=<moves> seconds=<seconds> seed=<seed>

Then at most 3 lines: method/tooling used, roughly where the time went, any problems hit.
```

---

## Variant A - fixed scramble set (comparable runs)

To compare models, pin the seeds and average the scores:

| Run | Parameters |
| --- | --- |
| 1 | `{ order: 3, seed: 42, moves: 36 }` |
| 2 | `{ order: 3, seed: 7, moves: 36 }` |
| 3 | `{ order: 3, seed: 2024, moves: 36 }` |

Append to the prompt:

```text
Use order=3, seed=<seed>, moves=36 for this run. After 3 runs, report each score and the average.
Call finish() to settle each run before calling start() again.
```

## Variant B - solve-only (agent cannot execute JS)

Let the page pick the scramble, then hand `facelets` to the model and take back the solution only:

```text
Here is a 3x3 Rubik's cube facelet string (54 chars, face order U R F D L B, row-major as seen from
outside each face):

<paste facelets here>

Return only the solution in standard notation (U D L R F B, optional ' or 2), space separated,
with no explanation.
```

This variant cannot measure real elapsed time, so it only compares move efficiency.

---

## Why the prompt is written this way

- **No reading the source**: the page is self-describing; `evaluate()` returns the whole protocol. Letting the agent read the implementation turns the benchmark into reverse-engineering the scorer.
- **Prepare before `start()`**: timing includes thinking time, so installing dependencies after the clock starts wrecks the time ratio.
- **Explicitly banning `reset`/`setState`**: neither is rejected mid-session (see "Integrity notes" in the README), so compliance is on the honor system.
- **One-line summary**: makes results easy to paste into a table and easy to parse automatically.
