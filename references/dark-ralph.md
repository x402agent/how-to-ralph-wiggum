# Dark Ralph: an OODA-loop trading adaptation

> **Prior art and credit.** The Ralph Wiggum harness pattern is
> [Geoffrey Huntley's](https://ghuntley.com/ralph/) work. He coined it,
> documented it, and put it through its paces well before anyone else.
> *Dark Ralph* is one adaptation among many — the loop, the discipline of
> fresh context per iteration, and the "small task, commit, repeat" insight
> are all his. If this page convinces you of anything, go read his original
> post and watch [the recent video](https://www.youtube.com/watch?v=O2bBWDoxO4s)
> first. This document assumes you've done that.

This chapter is an extension of the
[Ralph Playbook](../README.md). It does not replace any of it. Every
guideline in the main playbook (small scoped task, fresh context per
iteration, tight tests, branch isolation, low-blast-radius work) applies
here — *more* strictly, not less, because the artifact is a signed
transaction instead of a unit test.

---

## What changes when the loop trades

The original Ralph harness is roughly:

```
while true:
  agent <<< RALPH.md
  run tests
  commit
```

For autonomous trading, "run tests" isn't enough — the agent needs to
*observe* a moving environment, *orient* against its current state,
*decide* what (if anything) to do, and *act*. That's John Boyd's OODA
loop, and it slots cleanly into Ralph's outer iteration:

```
while true:
  Observe   — pull market state, account state, recent fills
  Orient    — diff vs last loop, classify regime, load policy from RALPH.md
  Decide    — agent <<< RALPH.md + observations
  Act       — execute decision (paper or signed tx) under guardrails
  Journal   — write decision + outcome to a git-tracked file
  Commit    — `git add journal/ && git commit -m "tick N"`
```

The Ralph guardrails are still the load-bearing pieces. The OODA shape is
just a more honest description of what the agent is being asked to do
when the "task" is "trade well" instead of "implement a spec."

## Why the Ralph guarantees still matter (more, not less)

The original playbook lists these as non-negotiable. None of them get
weaker when you point Ralph at a market:

- **Small, scoped task.** "Trade well" is not a task. *"Decide whether to
  open or close one position in $POOL given these observations, return a
  single action with size and reason"* is a task.
- **Fresh context per iteration.** A drifted, 200-message conversation is
  bad when writing code. It is *worse* when the next message resolves to
  a transfer. Fresh context per tick.
- **Tight tests / strong feedback loop.** The trading equivalent is a
  paper-trade ledger replayed against historical bars and a unit test
  suite over the decision function. If the loop can't beat a flat
  baseline on replay, it has no business signing anything.
- **Branch isolation.** Strategy changes go on a branch and get backtested
  before they touch the live config. No "just tweak the threshold on
  main."
- **Walk-away safety.** Geoff's "wake up to 37 commits" framing is
  beautiful for code. For trades it becomes "wake up to 37 fills" — which
  is great if 36 of them are correct and a disaster if 36 are the same
  bug. The kill-switch must fire before you wake up, not after.

## The minimum guardrails Dark Ralph adds

Treat these as additive to, not substitutes for, the Ralph rules above.

1. **Network gate.** The loop reads its RPC endpoint from env. Mainnet
   endpoints are rejected unless an explicit `MAINNET_OK=1` flag is set,
   and mainnet mode requires a separate config file the operator
   commits — never the agent.
2. **Paper mode is the default.** `--mode paper` is the only way to ship
   v0. The signing path doesn't exist in code yet; the seam is marked
   `TODO(future PR)` so reviewers can see exactly where it would go.
3. **Loss kill-switch.** N consecutive losing decisions (configurable,
   default 3) halts the loop and exits non-zero. The journal records the
   reason so the next iteration of the *operator* — not the agent — can
   diagnose it.
4. **Per-tick budget.** Hard caps on position size and on number of
   actions per loop. The agent cannot take "an action that re-balances
   the whole book" — only one bounded action per tick.
5. **Journal-to-git.** Every decision (and every *non*-decision — the
   agent saying "do nothing this tick" is a real output worth logging)
   is appended to `journal/ticks.jsonl` and committed. State lives in
   git, exactly as the original playbook says.
6. **No key handling in the agent.** Keys come from the operator's
   environment. The agent never reads, prints, or persists them. In v0
   the signing path is stubbed and the agent literally cannot reach a
   key.

## What Dark Ralph is *not* trying to be

- It is not a strategy. The decision function is intentionally dumb in
  v0 (a momentum-flavoured rule over mocked candles). The point of the
  v0 is the *harness*, not the alpha.
- It is not a "set it and forget it on mainnet" product. If anything the
  Ralph pattern should make you *less* comfortable doing that, because
  it makes how easily a tight loop can produce 37 wrong things in a row
  very, very legible.
- It is not a replacement for any part of Geoff's playbook. If a Dark
  Ralph rule and a Ralph Playbook rule disagree, the Playbook wins.

## Reference implementation

A runnable reference is in the sibling repo
[`x402agent/solana-ralphy`](https://github.com/x402agent/solana-ralphy)
under `agent/`:

- `agent/RALPH.md` — the per-tick prompt (small, scoped, fresh-context).
- `agent/loop.py` — the OODA driver with the guardrails above. Stdlib
  Python, no install step.
- `agent/tui.py` — a dark ANSI TUI that renders the loop in real time.
- `agent/journal/` — the git-tracked decision log.
- `agent/README.md` — operator instructions and the safety contract.

Read the README before you run it, and read it again before you ever
flip a mainnet flag.

## Contributing

If you've read Geoff's original work and you see places where this
adaptation drifts from the spirit of it, please open an issue or a PR.
Corrections from the people who built the pattern are the most
welcome kind.
