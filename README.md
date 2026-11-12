# review-loop

Devin re-reviews a PR on every push and CI re-runs on every push, so a stack
converges by iteration: push, wait, read the new automated findings, fix what
is verifiably right, push again. This skill captures that loop — where Devin's
comments actually live in the API, how to read a failing check down to its
log, how long to wait before believing there is nothing new, which findings to
fix now versus save for a human, and the one hard rule: comments from a human
account are never touched inside the loop.

The parts that are easy to get wrong:

- Classify comments by **posting account**, not content. `user.type == "Bot"`
  is in scope; `"User"` is reported verbatim and left alone, even when an
  agent obviously wrote it on the human's behalf.
- A Bot finding that needs a design call is a Tricky AI comment: one decision
  per thread in the managing thread's decisions.md, with its id in
  `_support/tricky_comments.json` (the `decisions-md` skill).
- Findings are **inline** review comments (`pulls/N/comments`). The issue
  endpoint is empty, so a PR can look clean when it is not.
- A resolved finding is a **new** `✅ Resolved` comment, not an edit — count
  open ones by excluding those.
- A CI failure whose log names something outside the diff, on a check a
  sibling head with the same tree passed, is the environment: re-run it from
  the invocation page instead of pushing an empty commit.
- Wait **twice** the slowest measured latency, and re-widen when a later
  round is slower. A wait calibrated on an early fast sample will miss a later
  round and read as "done".
- Mutation-check fixes with `--nocache_test_results`, or a cached green result
  will pass against mutated code and a vacuous test will look real.

## Usage

Ask an agent to work the loop:

```
run the review loop on #4729 and #4797
address the Devin review on #4197
CI is failing on the sync PR, loop until green
```

Read the findings on one PR directly, with the author type that decides scope:

```bash
gh api --paginate repos/<owner>/<repo>/pulls/<n>/comments \
  -q 'sort_by(.created_at) | .[] | "[\(.created_at)] \(.user.type) \(.user.login) \(.path)\n\(.body)"'
gh pr checks <n> --repo <owner>/<repo>
```

## Install

Symlinked into the central skill store:

```bash
ln -s /Users/zhuoran/Programs/skills/review-loop-skill ~/.agents/skills/review-loop
```

One link, not two: `~/.claude/skills` is itself a symlink to `~/.agents/skills`
on this machine, so a second `ln -s` into it overwrites the first and leaves a
self-referential loop.
