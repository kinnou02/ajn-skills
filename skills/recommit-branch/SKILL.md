---
name: recommit-branch
description: Use when a branch or feature is finished and its history has to be presentable before review — one commit carrying three ideas, a fix for a bug an earlier commit on the same branch introduced, a message that repeats the diff. Also use before opening a pull request, when a commit message asserts a number nobody measured, or when a reviewer cannot tell why a commit exists.
---

# Recommit Branch

Rewrite the history the branch owns into one commit per idea. The final tree stays identical, byte
for byte. Only the partition and the messages change.

This skill re-partitions the whole diff: it splits a commit that carries two ideas, reorders,
folds a later fix back into the commit that broke it, and rewrites every message.

## Where this sits

This is the last step before review, and the order is forced: **this skill cannot change the
tree.** Anything that alters what the code does or says has to land before you start.

1. Finish the work. Every change the branch means to ship is committed.
2. Simplify the code and question its assumptions. This changes the tree, so it cannot run after
   this skill.
3. **This skill.** The tree freezes; the history and the messages get rewritten.
4. Open the pull request. For what to do with the branch afterwards, see
   superpowers:finishing-a-development-branch.

Running this before step 2 wastes the work: a simplification lands afterwards, and the history is
messy again.

## Usage

```
/recommit-branch              # branch vs merge-base with the default branch
/recommit-branch <base-ref>   # explicit base
```

## Preconditions

Stop and say why when any of these fails:

- `git status --porcelain --untracked-files=no` is not empty. This skill stages files. It must
  never absorb work you staged.
- HEAD is the default branch.
- `git merge-base HEAD origin/HEAD` finds no base. Fall back to `origin/main` or `origin/master`
  when `origin/HEAD` is unset.
- The operator gave a test command, or the repo makes one obvious: a `test` target in the
  `Makefile`, `go test ./...`, the command the CI config runs. Every commit must pass it, so
  without it there is nothing to verify. Ask once, then keep using it.

## The one guarantee

The tree at the new HEAD equals the tree at the old HEAD. Not "the diff looks the same" — the same
tree object, which covers content and file modes:

```bash
OLD_HEAD=$(git rev-parse HEAD)
OLD_TREE=$(git rev-parse HEAD^{tree})
```

Record both before anything else. Everything below exists to keep that equality true.

## Set up

```bash
git branch backup/$(git branch --show-current)
git worktree add -B recommit/$(git branch --show-current) /tmp/recommit-wt "$OLD_HEAD"
git -C /tmp/recommit-wt reset "$BASE"
```

That last command is a mixed reset: HEAD and the index move to `<base>`, the worktree keeps the
final state. From here you build every commit by **subtracting** from a worktree that already holds
the answer, so the closing `git add -A && git commit` shuts the tree exactly, by construction.

Work in the scratch worktree, never on the branch. Two reasons, both load-bearing:

- The branch keeps working while you rewrite. A failed run costs nothing.
- A fresh worktree holds no untracked files. `git add -A` in the operator's worktree would sweep in
  whatever they left lying around, and the precondition above deliberately ignores untracked files.

## Partition

Read the whole diff and the existing messages before you group anything:

```bash
git log --reverse --stat "$BASE"..HEAD
git diff "$BASE"..HEAD
```

One commit is one idea. What that means here:

| Signal | What it becomes |
|---|---|
| A commit fixes a bug an earlier commit on this branch introduced | One commit. The bug never existed. |
| A commit mixes a refactor with the behavior change it enabled | Two commits, refactor first. |
| A pure rename or extraction | Its own commit, before the change that needed it. |
| A benchmark added for an optimization | Its own commit, **before** the optimization. |
| An inert test artifact: golden file, testdata, helper | Its own commit, before the code it covers. |
| A test that asserts new behavior | The **same** commit as that behavior, or after it. Never before. |
| A doc or readme fix unrelated to the code | Its own commit, last. |
| Two unrelated changes in one file | Two commits. See Staging. |
| An idea that cannot pass the tests alone | Merged with the neighbor that completes it. Say so in the report. |

The bench-before-optimization rule is what makes the numbers real: `git checkout <bench commit>` and
`git checkout <optimization commit>` run the *same* benchmark, so the before/after comes from the
branch itself, not from a rerun you have to trust.

Order by dependency, not by original date. A commit may only depend on commits before it.

Every commit has to pass the tests, so "prerequisites first" has one hard limit: an artifact may
precede the code only if it stays quiet. A golden file, testdata, a helper, or a benchmark compiles
and asserts nothing — `go test` skips benchmarks unless you pass `-bench` — so those can lead. A
test that asserts behavior the next commit introduces cannot: put it in the same commit as the
behavior. When the two rules seem to collide, green wins over tidy ordering.

## Staging

Whole-file staging is the default:

```bash
git -C /tmp/recommit-wt add <paths>
git -C /tmp/recommit-wt commit -F <message file>
```

When two ideas share one file, look in the branch's own history before you touch a patch. The
intermediate state you want is often already a blob some existing commit wrote:

```bash
git rev-parse <sha>:<path>                                   # does that state exist?
git -C /tmp/recommit-wt update-index --cacheinfo 100644,<blob>,<path>
```

`update-index --cacheinfo` stages an existing blob into the index and never touches the worktree, so
the final state stays intact. Match the mode: `100755` for an executable. Splitting a file whose two
ideas were separate commits upstream costs nothing this way — no patch, no hunk arithmetic. Check
this first; it is the common case, because the messy history you are rewriting usually made those
states already.

When no commit ever held the state you need, stage a hunk subset with a patch. `git add -p` is
interactive and unavailable, so generate the patch, cut it down to the hunks for this idea, and
apply it to the index only:

```bash
git -C /tmp/recommit-wt diff -- <path> > /tmp/recommit-wt-patch
# keep only the hunks belonging to this commit
git -C /tmp/recommit-wt apply --cached /tmp/recommit-wt-patch
```

**Regenerate the patch every round.** `git diff` compares the index to the worktree, so once a
commit lands, the index moved and yesterday's patch no longer applies. A pre-computed batch of
patches fails on the second commit, always.

The last commit closes the tree:

```bash
git -C /tmp/recommit-wt add -A
git -C /tmp/recommit-wt commit -F <message file>
```

`add -A` here is what picks up a deletion or a mode change you never staged by name. When it stages
nothing and `commit` refuses, the previous commit already closed the tree — that is the good case,
not an error. Drop the empty commit and go verify.

## Hypotheses

You verify what you can. You ask about the rest, and you do not wait.

Verify it yourself when the answer is in reach:

| Assumption | How you settle it |
|---|---|
| A benchmark claim | Run the benchmark. See Numbers below. |
| A metric name or its labels | `grep -rn 'promauto\|prometheus.New.*Vec\|MustRegister' --include='*.go' .` |
| "Nothing else calls this" | `grep -rn <symbol>` across the repo, callers included. |
| "This field is always set" | Read the writer, or the schema. |
| An error shape from a third party | The vendored source in `$(go env GOMODCACHE)`, or a captured response. |

Ask the operator only for what none of that reaches: a product decision, a threshold someone chose,
a claim about traffic you have no metric for.

Asking does not block the rewrite. Write the commit, mark the claim in the message, and carry the
question to the report:

```
Hypotheses: the mapping file never exceeds 50k rows (unverified — no metric records the row
count; ask the operator or add one)
```

When the answers arrive, rewrite that message and nothing else:

```bash
git commit --allow-empty -m "amend! <original subject>" -m "<new subject>" -m "<new body>"
```

An `amend!` commit does nothing on its own. It lands only on `git rebase -i --autosquash <base>`,
so who runs that depends on when the answer arrives:

- **Before you hand over.** Run the autosquash yourself, in the scratch worktree. Cheaper than
  rebuilding the commit, and the operator's branch is still untouched. Then prove the rewrite
  touched only messages, by comparing the tree of **every** commit against its pre-autosquash twin:

  ```bash
  paste <(for c in <old shas>; do git rev-parse $c^{tree}; done) \
        <(for c in <new shas>; do git rev-parse $c^{tree}; done)
  ```

  All pairs equal means check 1's result still holds and you skip re-running the tests. Check 2
  alone is weaker: it proves the tip survived and says nothing about the commits below it.
- **After you hand over.** This is a second run against the branch the operator now holds. It ends
  the same way the first did: verify, then print the autosquash command and let them run it.

Pass the original subject through unmodified, as one quoted argument; autosquash matches on that
string. One `amend!` per commit, never two.

Never turn an unverified claim into a stated fact. `Facts:` holds only what you ran, read, or
measured, with the command or the file that says so.

## Numbers

**Benchmarks.** Perf work on this branch ships its own benchmark, so the benchmark exists — find it
before you write a number. Run it at the commit that adds it and at the commit that optimizes,
in two detached worktrees, and paste what you got:

```bash
git worktree add --detach /tmp/bench-before <bench commit>
git worktree add --detach /tmp/bench-after  <optimization commit>
go test -run=XXX -bench=BenchmarkParseMapping -benchmem -count=6 ./...   # in each
```

`-count` under 5 is noise. Report the real output, both sides, and say which command produced it. If
an optimization commit carries no benchmark, that is a finding: say so and name the benchmark that
would settle it. Never estimate a speedup from reading the code.

**Prod projection.** Only from a metric that exists. Grep the registration site for the name and its
labels, and aggregate — hundreds of instances export the same series:

```promql
sum(rate(targeting_map_builder_rows_parsed_total[5m]))
```

Then state the projection as arithmetic the reader can check: "8.2 µs saved per row × 40k rows/s
= 0.33 CPU-seconds per second, about a third of a core across the fleet." When no metric measures
the input, say that, and name the metric that would.

## Message template

Subject line: one imperative sentence, about 70 characters, no `feat:` or `fix:` prefix. Then one
paragraph on what changes and why. Then only the sections that carry something:

```
Facts:       what you ran, read, or measured, with the command or file
Hypotheses:  what the commit assumes, with what would confirm or refute it
Validation:  the command a reviewer runs to check this commit
Impact:      what changes for a caller, an operator, or a user
Limitations: what this commit deliberately does not cover
Risks:       what breaks if a hypothesis above is wrong
```

**A section appears only when it carries content.** Never write `N/A`, never pad a section to fill
the shape. A one-line readme typo fix gets a subject line and nothing else — its diff already says
everything. Six sections on a two-line commit is the bloat the template exists to prevent.

What each one holds, by commit kind:

- **Optimization** — `Facts:` the before/after benchmark output and the prod projection.
  `Validation:` the benchmark command. `Risks:` the workload where the optimization loses.
- **Test or fixture** — `Facts:` what it covers that no existing test did, and how you know
  (the mutation it catches, the bug it would have caught). `Impact:` what a future change is now
  protected against.
- **Refactor** — `Facts:` that behavior is unchanged, and what shows it (the tests that pass
  untouched). `Impact:` what the next commit can now do.
- **Metric or log** — `Facts:` the registered name and labels. `Risks:` label cardinality.
- **Bug fix** — `Facts:` the root cause and every caller you checked. `Validation:` the test that
  fails before and passes after.

Write English, present tense, active voice. Don't paraphrase the diff — a body that says what the
code already shows is worse than no body.

## Verify

Four checks, in this order. All four must pass before you hand anything over. Do not move them
earlier to "fail fast": a build drops artifacts in the worktree, and the closing `git add -A` would
commit any the `.gitignore` misses. Build every commit first, then test.

```bash
cd /tmp/recommit-wt

# 0. Baseline: the tests pass at the tip, whose tree equals the original HEAD.
<test command>

# 1. Every commit passes the tests.
git rebase --exec '<test command>' "$BASE"

# 2. The tree is identical. Commit shas changed; the tree must not have.
test "$(git rev-parse HEAD^{tree})" = "$OLD_TREE" && echo IDENTICAL

# 3. Nothing was left behind.
git status --porcelain
```

Check 0 exists because a fresh worktree copies no ignored or untracked file: no `.env`, no
generated file the `.gitignore` covers, no ignored `vendor/`. The tip holds the same tree as the
original HEAD, so a failure there is the environment, never your partition. Stop and say which file
is missing. Skipping this check turns an absent `.env` into a diagnosis of "these commits don't
stand alone", and the fix folds commits together for no reason.

Check 1 stopping, once check 0 is green, does mean a commit does not stand alone. Fold it into the
neighbor that completes it (`git rebase --continue` after squashing, or rebuild that pair) and
rerun. Do not weaken the test command to get past it.

Check 2 failing means the partition lost or added content. The closing `git add -A` should make
that impossible, so look for a file you committed twice with different content, or a mode change.
Fix it in the scratch worktree; the operator's branch was never touched.

## Hand over

Print the commands and stop. The operator runs them:

```bash
git checkout <branch>
git reset --hard recommit/<branch>
git worktree remove /tmp/recommit-wt
git branch -D recommit/<branch>
git worktree remove /tmp/bench-before   # if you ran benchmarks
git worktree remove /tmp/bench-after
```

`backup/<branch>` already exists, from Set up. Say so, and say that `git reset --hard
backup/<branch>` undoes everything.

## Report

- The new history: one line per commit, subject plus the original shas it came from.
- Commits that got merged because neither half passes the tests alone.
- Every hypothesis you settled yourself, and how.
- Every open question, with what would answer it. These are the ones the operator owes you.
- Optimizations with no benchmark, and the benchmark that would settle each.
- The three verify results.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building commits forward from base | Mixed reset, then subtract. The closing `add -A` shuts the tree for free. |
| Trusting `git diff old new` as the guarantee | Compare tree shas. It catches mode changes a diff read hides. |
| Reusing a patch generated before the previous commit | Regenerate from `git diff` every round. |
| Running `git add -A` in the operator's worktree | Scratch worktree. Untracked files are not yours to commit. |
| Writing a bench number you did not run | Run it, two worktrees, `-count=6`. No benchmark means say so. |
| A PromQL projection against a metric nobody registered | Grep the registration site. Aggregate with `sum(...)`. |
| Filling all six sections on every commit | A section with nothing to say is deleted, not filled. |
| Turning an unanswered question into a `Facts:` line | It stays in `Hypotheses:`, marked unverified. |
| Blocking the whole rewrite on one open question | Commit, mark it, keep going. Fix the message with `amend!` later. |
| Reordering so a commit depends on a later one | Order by dependency. Prerequisites first. |
| Optimization commit placed before the bench that measures it | Bench first. That is what makes before/after real. |
| A behavior test placed before the commit it tests | Same commit as the behavior. Only inert artifacts lead. |
| Folding commits to fix a test that fails at the tip too | Run check 0 first. A missing `.env` is not a partition problem. |
| Leaving `recommit/<branch>` and the bench worktrees behind | Both in the hand-over commands. |
| Rewriting commits outside `<base>..HEAD` | The branch does not own them. Leave them. |
| Running the switch-over yourself | Print it. The operator runs it. |
