# Workbook conventions

This workbook is the hands-on companion to
[flezzle-rs](https://github.com/nverhaaren/flezzle-rs): a series of projects
where the code is presented in a deliberately incomplete state and you
implement a key component yourself. It exists so the repo owner builds and
understands the core pieces (and can check AI-authored work), and doubles as
a study/onboarding path into flezzle-rs. Not every flezzle-rs change gets a
workbook project — only the parts worth building by hand.

## Project layout

```
projects/
  NN-short-slug/
    README.md      # goal, background, tasks, definition of done, references
    flezzle-rs/    # git submodule pinned at the project's start commit
    notes/         # optional: solution write-ups, retrospectives, diagrams
```

`NN` is a two-digit sequence number (`01`, `02`, …), matching the flezzle-rs
milestone number.

## Milestone branches in flezzle-rs

Each workbook project corresponds to a milestone in flezzle-rs, built as a
pair of branches sharing history:

- `milestone/NN-start` — the **start commit**: compiles, but key pieces are
  stubbed with `TODO(project-NN)` markers and the tests are red.
- `milestone/NN-complete` — branched from the start commit; the AI's
  reference implementation, tests green. **Spoilers live here** — don't look
  until you're done (or want to be done).

Work on your own branch (suggested name: `milestone/NN-attempt`). Inside the
project's submodule, HEAD is already detached at the start commit, so plain
`git switch -c milestone/NN-attempt` branches from the right place — no ref
name needed (the `milestone/NN-*` branch names live on the `nverhaaren-ai`
fork, not in a fresh submodule clone). When the milestone is finalized (after
feedback and comparison), the agreed-upon result merges to `main` of
flezzle-rs, and the workbook project merges to `main` here.

**Merging rule (important):** milestone branches must reach `main` via merge
commit or fast-forward — **never squash** — so the start commit stays
reachable from `main`. The submodule pin and this workbook's value depend on
that history being permanent. After the upstream merge, the start commit is
tagged `workbook/NN-start` in flezzle-rs.

## Submodules

Each project pins flezzle-rs at its start commit via a submodule whose URL
points at the upstream repo:

```bash
git submodule update --init projects/NN-short-slug/flezzle-rs
```

While a milestone is under review, its start commit exists only on the
`nverhaaren-ai` fork branches — but the checkout above still works, because
GitHub serves fork-network objects by SHA from the upstream URL. Two caveats:

- Don't rely on that for permanence: the pin is only durably safe once the
  milestone merges upstream (hence the no-squash rule and the tag).
- To *name* the milestone branches inside a submodule (e.g. to diff against
  `milestone/NN-complete`), add the fork as a remote:

```bash
git -C projects/NN-short-slug/flezzle-rs remote add fork \
    https://github.com/nverhaaren-ai/flezzle-rs.git
git -C projects/NN-short-slug/flezzle-rs fetch fork
# then e.g.: git diff fork/milestone/NN-complete
```

## Checking your work

1. `cargo test` goes green. The tests are part of the start commit —
   **don't modify them to pass**; they're the exercise's contract.
2. The game plays correctly (`cargo run`) — each project README lists what
   "correct" looks like.
3. Compare your implementation against `milestone/NN-complete`, note
   divergences, and record feedback. The final merged version may be either
   one, or a blend.

## Workflow summary

One project at a time: the AI prepares the milestone branches and the
workbook project → the owner implements from the start commit → feedback and
comparison → final result merges to `main` of both repos → next project.
