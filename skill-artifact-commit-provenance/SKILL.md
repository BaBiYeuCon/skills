---
name: skill-artifact-commit-provenance
description: Commit an artifact or Skill package that lives outside the current project repository, and prove the committed bytes are the validated bytes. Use when asked to commit, stage or write a commit message for a path under ~/.qoder/skills or any file outside the working repo, when git reports 'not a git repository' for the artifact's directory while a different project repo is open, when AGENTS.md forbids committing the workspace, when a first commit in a shared directory would sweep in untracked sibling skills, or when a file changes after validation. Covers git -C rev-parse ownership detection, stopping for the owner's choice before git init in a shared config directory, re-reading and re-running the full validation suite after hash drift, staging exactly one repo-relative path, the H0 validated = H1 pre-commit = H2 index = H3 HEAD digest chain with H4 separate, and disclosing referenced resources left untracked. Not the in-file concurrent-edit merge, package checking or skill registration.
---

# Skill Artifact Commit Provenance

## Overview

Two questions decide every commit of an artifact outside the current project:
which repository may receive it, and whether the committed bytes are the bytes
that were validated. Answer both with measurements; a successful `git commit`
line answers neither.

## Trigger shape

- "Commit this", "stage it" or "write the commit message" for a `SKILL.md`,
  a skill package, a dotfile or any path outside the working repo.
- `git rev-parse` in the artifact's directory prints
  `fatal: not a git repository (or any of the parent directories)`.
- The open workspace repo is unrelated, has no commits, or its AGENTS.md or
  task rules forbid commits.
- The artifact sits in a shared directory whose siblings are untracked.
- The file changed after you validated it, or after you staged it.

## Workflow (in order)

1. **Resolve ownership from the artifact, not the cwd.**

   ```
   git -C <artifact-dir> rev-parse --show-toplevel
   ```

   Exit 0 prints the top-level directory. Exit 128 with `fatal: not a git
   repository` means no enclosing repo. Never infer the repo from the session
   cwd: `git status` there answers a different question and can hand you a
   repository that must not receive this commit.

2. **Classify authority.** Three cases, three behaviours:
   - The top-level is the repo you were told to commit into: proceed.
   - The path sits inside a different repo (dotfiles, monorepo): proceed only
     if that repo's own rules allow the commit.
   - AGENTS.md or the task forbids committing that repo: stop. Repo membership
     is not permission.

3. **Outside every repo: write the message, then ask the owner.** Present
   exactly these choices: (a) initialise a repo at a named directory and commit
   only this repo-relative path; (b) keep the message only, the file stays
   uncommitted; (c) commit into an owner-named existing repo, if its rules
   allow. Never run `git init` in a shared config directory on your own
   initiative. If the owner picks (a), state what init changes: HEAD becomes a
   root commit, every sibling directory now shows as an untracked entry, and
   `.gitignore` versus track-all remains the owner's decision.

4. **Re-measure before staging (drift gate).** Compare the current digest with
   H0, the hash recorded after validation:

   ```
   shasum -a 256 <artifact-path>
   ```

   Equal means H1 = H0. Different means the validated bytes are gone: re-read
   the whole file, confirm your own edits survived, read what was added, prove
   any newly referenced resource exists **and runs**, then re-run the complete
   validation suite and record the new H0. One passing check is not a
   re-validation.

5. **Stage exactly one path, then verify the staged set.**

   ```
   git -C <root> add -- <repo-relative-path>
   git -C <root> diff --cached --name-only -z | tr '\0' '\n'
   ```

   The staged list must be exactly that path, compared NUL-split. The default
   `--name-only` output C-quotes non-ASCII paths (`tệp.txt` arrives as
   `"t\341\273\207p.txt"`), so a literal comparison reports a false
   `STAGED_SET_NOT_EXACT`; `git -c core.quotePath=false` unquotes the display
   but still splits on newlines, so only `-z` is exact. Where untracked
   siblings exist, `git add -A` and `git add .` are forbidden.

6. **Check identity; never configure it.** `git config user.name` and
   `git config user.email` must resolve. Never change git config.

7. **Commit from a message file.** `git -C <root> commit -F <msg-file>`.

8. **Prove the bytes.** Every value below is SHA-256 over content:
   - H0 — the digest recorded after validation.
   - H1 — pre-commit worktree: `shasum -a 256 <artifact-path>`.
   - H2 — index: `git -C <root> show :<rel> | shasum -a 256`.
   - H3 — HEAD: `git -C <root> show HEAD:<rel> | shasum -a 256`.

   Require H0 = H1 = H2 = H3. H4 is the worktree after the commit and is
   reported separately. H2 absent while H3 is present means the index no
   longer holds the target: the staged change is a deletion (`git rm --cached`
   on an intact worktree produces exactly this). That is a chain break, never
   a match, even when H0 = H1. `git ls-files -s` confirms mode, blob and
   stage, but its hash is a git object SHA-1 — never compare it with the
   SHA-256 digest. A commit exit code is not provenance.

9. **Report package completeness.** Enumerate every resource the committed file
   references (`scripts/…`, `references/…`, `assets/…`) and test each:

   ```
   git -C <root> ls-files --error-unmatch -- <ref>
   ```

   Exit 0 means tracked, exit 1 means untracked. Any untracked resource means
   the package is not committed, whatever the commit says.

## One-shot inspector

`--expect` is optional; without it the output is a measurement, not a proof.

```
# before staging: records H1 and shows that the target is not staged yet
node ~/.qoder/skills/skill-artifact-commit-provenance/scripts/commit-provenance.mjs <artifact-path>

# after staging, and again after the commit (the final proof)
node ~/.qoder/skills/skill-artifact-commit-provenance/scripts/commit-provenance.mjs \
  <artifact-path> --expect <sha256> [--require-complete] [--ref <repo-relative-path>]...
```

It prints `REPO`, `HEAD`, `H0`–`H3`, `CHAIN` (`h0=h1`, `h1=h2`, `h2=h3` as
`match`/`differ`/`absent`/`index-absent`/`head-absent`/`not-checked`),
`TARGET_STAGED` (`no`, `yes` or `deleted`), `STAGED_PATHS`, `STAGED_EXTRA`,
`STAGED_OK`, one `REF` line per resource, `PACKAGE`, `TRACKED_FILES`,
`UNTRACKED_FILES` and `VERDICT`; it writes
nothing. Both counts are recursive file counts from `git ls-files`, not the
entry count of `git status`. `--ref` accepts repo-relative paths only: an empty
value, an absolute path or any `..` segment is a usage error.

Exit codes: 0 `MATCH` or `MEASURED_ONLY`, 1 `DRIFT`, `STAGED_SET_NOT_EXACT` or
`CHAIN_BROKEN`, 2 usage, 3 `OUTSIDE_REPO`, 4 `CHAIN_INCOMPLETE`. A staged
deletion shows `TARGET_STAGED: deleted` with `H2_INDEX: absent` while `H3_HEAD`
is present, and returns `CHAIN_BROKEN` — never `MATCH`. Run it before staging
(records H1), after staging (H2 appears), after commit (H3 appears; a later
run's H1 is H4). Read the verdict, not just the exit code.

The digests prove content bytes only. A `.gitattributes` filter such as Git
LFS makes the stored blob (a pointer) differ from the worktree content, and a
file-mode-only change leaves every digest equal; neither is detected. A
symlinked artifact is resolved with `realpathSync`, so the `PATH` reported is
the link target and a target outside the repo returns `OUTSIDE_REPO`. A worktree
edit after the commit makes H1 (now H4) differ from H0 = H2 = H3: the commit
proof still holds, the worktree claim does not.

## Commit message rules

- The subject states the staged change, not the plan.
- A first commit of a file that concurrent sessions also edited must name their
  content in a separate paragraph; never present it as yours.
- If only the file was committed while its bundled resources stay untracked,
  say so.
- Do not claim tests, accuracy, registration, clinical approval or a winning
  engine unless this run measured it.

## Report template

```
REPO: <top-level> | none | forbidden (<parent>, <reason>)
AUTHORITY: <owner choice or rule>
H0/H1/H2/H3: <sha256> MATCH | DRIFT at <stage>
CHAIN: h0=h1 <match|differ>; h1=h2 <match|differ|absent>; h2=h3 <match|differ|absent|index-absent|head-absent>
TARGET_STAGED: no | yes | deleted
H4: <sha256> | not-measured
COMMIT: <short sha> | not created
COMMITTED_PATHS: <paths>
STAGED_EXTRA: none | <paths>
TRACKED_FILES / UNTRACKED_FILES: <n> / <n> (recursive)
PACKAGE: complete | partial (<untracked refs>) | not-checked
VERDICT: COMMITTED_PROVEN | MESSAGE_ONLY | BLOCKED (<reason>)
```

The inspector's `VERDICT` is mechanical; the report's is a conclusion. Map them
explicitly: `MATCH (h0=h1=h2=h3)` under `--require-complete` →
`COMMITTED_PROVEN`; `MEASURED_ONLY` or a pending index/head → `MESSAGE_ONLY`;
`DRIFT`, `STAGED_SET_NOT_EXACT` or `CHAIN_BROKEN` → `BLOCKED` naming the stage.

## Measured case (2026-10-25, re-measured 2026-10-26)

Artifact: `~/.qoder/skills/session-sourced-skill-distillation/SKILL.md`.

- Before the commit, `git -C ~/.qoder/skills rev-parse --show-toplevel` exited
  128 (`fatal: not a git repository`). The open workspace repo was `Qoder-CHC`
  on `qoder-handoff`, with no commits and 15 untracked top-level entries, and
  its AGENTS.md forbids committing. The agent stopped and asked instead of
  running `git init`.
- The owner chose to initialise a repo for the skills directory. Identity
  already resolved (BBYC); no config was changed.
- The pre-commit re-hash expected `b98dbba…` but found `2136534…` (191 → 212
  lines, 31 → 34 sibling directories). The agent re-read the whole file,
  confirmed its two edits survived (lines 59–61 and 209–211), and ran the newly
  referenced citation checker (`VERDICT: OK`, `citations: 0`) before committing.
- Commit `91d6051`: 1 file changed, 212 insertions; 34 sibling directories
  stayed untracked. `git show HEAD:… | shasum -a 256` equalled the worktree
  digest.
- Re-measured 2026-10-26: HEAD is still `91d6051`, `git ls-files` lists only
  that `SKILL.md`, and `?? session-sourced-skill-distillation/scripts/`
  remains. The committed file references the overlap matcher at
  `session-sourced-skill-distillation/scripts/check_skill_overlap.mjs` in its
  step 4 and Resources list, so the commit is proven while the package is
  partial.
- The drift-time re-validation is the cautionary part: duplicate-heading and
  citation checks were visible, but a full package check after the 212-line
  edit was not. Re-run everything.
- These counts drift. Re-measure them on every run.

## Hard rules

- Measure repo membership from the artifact's directory every time; a sibling
  repo is not a default target.
- AGENTS.md and task rules outrank convenience; repo membership is not
  permission.
- No `git init` in a shared directory without the owner's explicit choice;
  report the side effects afterwards.
- Drift means re-read the whole file and re-run the complete validation suite;
  update H0 only after it passes.
- Prove a newly referenced resource exists and runs before it enters the
  commit.
- Stage exact paths only, then verify the staged set.
- A first commit carrying concurrent content must name it.
- A file-level commit is not a committed package; list untracked referenced
  resources.
- H0 = H1 = H2 = H3 is the proof; the commit's exit code is not.
- No config changes, amend, reset, force-push or push.

## Boundaries / non-claims

- A proven commit says nothing about correctness, test coverage, skill
  registration or clinical validity.
- The owner's choice authorises one commit of one named path, not a tracking
  policy for the whole directory.
- Do not stage memory files, sibling skills or evidence directories that were
  not named.

## Related skills

- `skill-dir-concurrent-write-reconcile` — concurrent writes inside one skill
  directory; its Boundaries explicitly exclude committing.
- `concurrent-skill-md-edit-recovery` — merges an in-file edit conflict before
  you get here.
- `skill-postwrite-registration-verification` — proves a written skill
  actually loads.
- `verify-skill-source-session` — verifies claims about a session; this skill
  verifies committed bytes.
- `frozen-snapshot-immutability-rehash` — re-hashes frozen snapshot lists, not
  commits.
- `session-sourced-skill-distillation` — orchestrates skill creation and
  delegates commit proof here.

## Resources

- [scripts/commit-provenance.mjs](scripts/commit-provenance.mjs) — read-only
  inspector for repo ownership, H0–H3, the equality chain, `TARGET_STAGED`,
  staged extras, referenced-resource tracking and the verdict. `--help` prints
  usage.
