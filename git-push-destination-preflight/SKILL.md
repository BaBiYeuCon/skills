---
name: git-push-destination-preflight
description: Pre-push gate re-resolving destination, credential path, publish set and history before any git push. Use when asked to push a commit or branch, when push failed with 403 / non-fast-forward / "Permission denied (publickey)", when origin was absent or its URL/protocol changed mid-task, when histories share no merge base, when asked to switch accounts or whether an account may push, or when naming a new remote branch. Covers remote/pushURL re-reads, gh auth status / helper / ssh-agent / push-permission checks without secrets, merge-base and divergence counts, publish-set and visibility disclosure, branch-name collision checks, one explicit src:dst push, and ls-remote remote-ref verification (not the tracking cache), plus a read-only HTTPS probe when SSH auth fails. Stops with the exact blocker instead of retrying, force-pushing, adding remotes or changing credentials. Commit provenance: skill-artifact-commit-provenance; exit codes: shell-command-result-verification; key/credential lifecycle is out of scope.
---

# Git Push Destination Preflight

## Overview

A push writes to shared state, and its destination is not stable across a task.
Measure four facts immediately before pushing: where it goes (remote, URL,
pushURL, protocol), who sends it (credential path and account), whether the
histories are related, and exactly which commits become visible on the remote.
If any is unresolved or contradictory, stop and report the exact blocker. A
successful `git commit` says nothing about any of them.

## Trigger shape

- "Push this commit", "push to origin", "push the branch", "publish it".
- A push failed with 403, exit 128, `non-fast-forward`, `fetch first`, or
  `Permission denied (publickey)`.
- `git remote -v` was empty earlier and a destination exists now;
  `.git/config` mtime is newer than the task start; the URL or protocol changed.
- Local and remote histories share no merge base.
- "Switch to the account that can write" while only one account is
  authenticated.
- The owner names a new remote branch, or a push has returned success and the
  remote ref must be verified rather than assumed.
- Commit provenance just passed and the owner now asks to push.

## Gate 0 - resolve the destination at push time

Run these now, not from memory:

```sh
git rev-parse --abbrev-ref HEAD
git branch -vv
git remote -v
git config --show-origin --get-regexp '^(remote\..*\.(url|pushurl)|branch\..*\.(remote|merge)|push\.)'
stat -f '%Sm %N' .git/config 2>/dev/null || stat -c '%y %n' .git/config
git rev-parse --abbrev-ref --symbolic-full-name '@{u}'
```

- An empty `git remote -v` is not proof there is no destination. Measured: the
  list was empty with exit 0 at 06:16:34, `origin` was added at 06:16:47 and
  fetched at 06:16:55, and the push then targeted that URL. Re-run after any
  failure and read `.git/config` directly.
- Read URL and pushURL. They can differ; a push uses the push URL.
- Read the protocol. HTTPS uses the git credential helper or `gh`; SSH uses
  ssh-agent and keys. Gate 1 depends on it.
- A remote can change between turns. Measured here: HTTPS at the failed push,
  SSH at a later measurement (config mtime 06:23:01). Never reuse an earlier
  URL or protocol.
- Capture the exact destination string and branch mapping in the report.
  Transcript tool entries do not preserve command arguments, so quote the
  command you are about to run.

Do not add, rename, edit or delete a remote on your own. If one appeared
mid-task, report its `.git/config` mtime and say the author is unattributed;
ask whether it is the intended target.

## Gate 1 - identify the credential path without reading secrets

```sh
gh auth status
git config --show-origin --get-all credential.helper
ssh-add -l
```

- `gh auth status` reports the active account, protocol, scopes and a masked
  token. That is identity inspection, not a secret read.
- Run `ssh-add -l` only for an SSH destination. Exit 1 with "The agent has no
  identities" is a result, not a command failure.
- Never run `gh auth token`, never query the keychain
  (`security find-internet-password` or equivalents), never print, request or
  accept a token in chat, and never change the credential helper.
- Match the check to the protocol. An HTTPS 403 means the authenticated account
  lacks write access. An SSH failure is a key or agent problem and is not fixed
  by switching gh accounts.
- Account-switch request: enumerate the authenticated accounts. If only one
  exists, stop and report that no second account is available. `gh auth login`,
  `gh auth switch`, `gh auth setup-git`, creating an account or entering a token
  are credential or config changes that need the owner's explicit direction.
  With `credential.helper=osxkeychain`, `gh auth switch` alone may not change
  the credential git sends.
- Measure push permission and repo visibility before Gate 3, not by attempting
  the push and reading its 403. When `gh` is authenticated and can read the
  repo:

  ```sh
  gh api repos/<owner>/<repo> \
    --jq '{push: .permissions.push, private: .private, visibility: .visibility}'
  ssh -T git@github.com   # success prints "Hi <account>!" and exits 1; 255 = no accepted key
  ```

  - HTTPS: the `gh` account is the account git sends, so `push:false` means
    stop. Measured: the probe returned `push:false` while the push was still
    attempted and failed 403, exit 128 - the attempt was avoidable.
  - SSH: the sending identity is the key's account, which need not be the `gh`
    account. `ssh -T` maps a working key to an account; `gh api` reports the
    `gh` account's permissions only. If they differ, the key account's
    permission is `UNKNOWN`; it is not inherited from either side.
  - If the probe errors or the account cannot see the repo, report
    `PERMISSION_UNKNOWN` and require the owner's explicit go-ahead naming the
    repo and the account. Unknown is not authorized.
  - A probe is not the push result: permission can change before the attempt,
    so the 403 triage still applies.

## Gate 2 - prove the histories are related

```sh
REMOTE=origin; BRANCH=main; UP="$REMOTE/$BRANCH"
git merge-base HEAD "$UP"; echo "merge_base_rc=$?"
git rev-list --left-right --count "$UP...HEAD"
git merge-base --is-ancestor HEAD "$UP"; echo "local_ancestor_rc=$?"
git merge-base --is-ancestor "$UP" HEAD; echo "remote_ancestor_rc=$?"
git rev-list --max-parents=0 HEAD
git rev-list --max-parents=0 "$UP"
git reflog show --date=iso "$UP"
```

- Empty merge-base with exit 1 means unrelated histories. Measured: left/right
  `15 4`, roots `30a826a` (remote) and `91d6051` (local). A plain push is
  non-fast-forward; a force-push would overwrite 15 remote commits.
- A remote-tracking ref can be stale. Its reflog date is the measurement.
  `git fetch <remote> <branch>` refreshes it but mutates local remote-tracking
  refs, so classify the fetch and run it only against a confirmed remote.
- Both ancestor checks exiting 1 alongside an empty merge-base is a consistent
  unrelated-history result. An exit 0 means fast-forward is possible in that
  direction; report which.
- No merge base means no push. Never substitute `--force`,
  `--force-with-lease`, ref deletion or a branch rename.
- An empty result is not proof until the same command produces output where it
  must: `git merge-base HEAD HEAD` returns a commit hash.
- Distinguish the two stop cases: a merge base with both ancestor checks
  exiting 1 is divergence; no merge base is unrelated history. Both forbid a
  plain push and a force push.

## Gate 2a - disclose every commit the push would publish

```sh
REMOTE=origin; BRANCH=main; UP="$REMOTE/$BRANCH"
git log --format='%h %an <%ae> %s' "$UP"..HEAD
git log --format='%h %an <%ae> %s' HEAD --not --remotes   # if "$UP" does not exist
git status --short
```

- A push publishes commits, not a task. List every commit that would become
  reachable on the remote, with author and subject. If the remote branch does
  not exist or has no merge base, use the `HEAD --not --remotes` form, state
  that the tracking refs may be stale, and name the base you compared against.
- Keep the explicit `HEAD`. Measured: `git log --not --remotes` alone returned
  empty with exit 0 while seven local commits were unrelated to the only
  remote ref; the same command with `HEAD` in front listed all seven. An empty
  publish set is a claim, so re-run the form that has a positive revision.
- Stop and ask the owner if any commit is outside the commit set they approved,
  especially one added by another session or agent. Measured: the four local
  commits included `663b2d6`, added by a concurrent session, so the push scope
  was wider than the task.
- `git status --short` is not the publish set; it shows uncommitted work that
  will **not** be pushed. Report both.
- State the destination's visibility with the publish set. `private:false`
  makes the whole set readable by anyone at once, including commits from other
  sessions; a private repo limits it to its readers. If visibility is unknown,
  say so - "reachable" and "public" are different blast radii.
- If the owner approved a commit set, a later commit on the same branch is a
  new publish set and needs a new confirmation.

## Gate 3 - push once, on the named destination

- Push only after Gates 0-2a and the permission check are green and the owner
  named the remote and branch, or confirmed the resolved ones. The owner's
  instruction authorizes one push of that ref; it does not authorize a retry or
  a different ref.
- Before pushing a new branch, prove the name is free:

  ```sh
  REMOTE=origin; BRANCH=main
  git ls-remote --heads "$REMOTE" "<new-branch>"
  git ls-remote --heads "$REMOTE" "$BRANCH"   # positive control
  ```

  Empty for the candidate and a ref for the control means the check
  discriminates. If the candidate already exists, stop and ask; never
  overwrite it. If the control is also empty, the remote has no refs or no such
  base branch and the check proves nothing: use a base ref that exists, or
  report the collision check as unmeasurable before pushing.
- Use the explicit mapping, for example
  `git push origin main:refs/heads/<branch>`. No `--force`, no
  `--force-with-lease`, no tags or other refs unless asked, and no `-u` unless
  the tracking decision was made.
- Snapshot the remote heads and the local cache before and after:

  ```sh
  REMOTE=origin
  git ls-remote --heads "$REMOTE"
  git for-each-ref refs/remotes/<remote>
  ```

  The first is authoritative; the second is a local cache and can be stale.
- Record the command, stderr, exit code and both snapshots. One attempt only; a
  failure goes to triage, not a retry loop.

## Gate 4 - verify the remote ref, not the tracking cache

After the single push attempt:

```sh
REMOTE=origin; DEST=<remote-branch>; SRC=<source-ref>; BASE=<base-branch>
git ls-remote --heads "$REMOTE" "$DEST"
git rev-parse "$SRC"
git ls-remote --heads "$REMOTE" "$BASE"
```

- The pushed ref must exist on the remote and its SHA must equal the local
  source SHA. The base branch must still equal the pre-push SHA - unless the
  push targets the base branch itself, where its SHA is expected to become the
  pushed SHA. Any other change to the base means something else wrote to the
  remote and needs explanation before the push is reported.
- `refs/remotes/<remote>/<branch>` is a local cache. A failed push can leave it
  stale; a successful push may update it, but the remote query is authoritative.
  Report `REMOTE_REF` from `ls-remote`, not from `for-each-ref`.
- If SSH auth blocks `ls-remote`, query the public HTTPS URL directly for a
  read-only check:

  ```sh
  git ls-remote --heads https://github.com/<owner>/<repo>.git "$DEST"
  ```

  Do not change `origin`, the credential helper or the protocol just to verify.
  If the repo is private or the query fails, report `UNVERIFIED` instead of
  `PUSHED`.
- Positive control: query the known base branch and require a ref. An empty
  candidate result proves nothing until the same command returns the control.
- A successful push proves transfer only; it does not prove CI, review,
  correctness or acceptance.

## Failure triage

| Symptom | Meaning | Action |
| --- | --- | --- |
| `403` / `Permission ... denied to <account>` | The authenticated account cannot write to this repo | Stop. Report URL, account, stderr, exit code. Offer a new branch on a writable remote, an owner-granted access change, or leaving the commit local. |
| `non-fast-forward`, `fetch first`, no merge base | Histories are unrelated or diverged | Stop. Show merge-base, left/right counts and roots. Never force. |
| `Permission denied (publickey)` | SSH key or agent problem | Stop. Report `ssh-add -l` and the key state. Do not switch gh accounts, edit config or add keys. A public remote can still be read over its HTTPS URL for ref verification; that is a read-only probe, not an auth workaround. |
| `repository not found`, `404` | Wrong URL, or no access to a private repo | Re-resolve the URL. Stop. |
| `push` returned success but the remote ref is missing or has a different SHA | The transfer report is not a ref measurement | Stop. Re-measure with `ls-remote`; report `PUSHED_UNVERIFIED`, not `PUSHED`. |
| `No configured push destination`, no upstream | The remote or tracking is unset | Stop. Ask the owner for the destination. Do not add a remote silently. |

After a failed attempt, report whether refs changed as `measured` (from the
remote-head `ls-remote` snapshots) or `inferred` (from the rejection message). A
rejection text alone is not a ref measurement; the local tracking cache is not
the remote ref.

Credential changes are outside this gate. If the owner explicitly authorizes
generating or adding a key, treat it as a separate credential task: use a
dedicated key, show only the public half, never print the private key, and
confirm the target account has write access. A key that authenticates proves
identity, not permission, and `gh ssh-key list` needs `admin:public_key`,
which the measured token lacked. Report such an action done only from its own
measurement (`ssh -T` output, the key listed on the account), never from having
been asked to do it. The no-passphrase choice in the measured case was an
owner-authorized non-interactive tradeoff, not a default.

## Options to offer after a stop

1. **New branch on a writable remote** - preserves the exact commit hash and
   leaves the remote default branch untouched.
2. **Reconcile histories** - merge or rebase onto the remote branch, then push.
   The commit hash changes; state that the previously proven commit no longer
   exists.
3. **Leave the commit local** - report its hash, branch and worktree path.

For an account request, name the account needed, the protocol, and that login or
`setup-git` is an owner-executed credential/config change. Do not create
accounts or enter tokens.

## Report template

```text
REPO: <top-level> | BRANCH: <local> -> <remote>/<branch>
DESTINATION: <remote> <url> (pushurl <url>) | protocol <https|ssh> | config mtime <time>
UPSTREAM: <remote>/<branch> | none
IDENTITY: <active account|none> | helper <...> | ssh agent <n identities|none>
PERMISSION: <push true|false|unknown> for <account> | VISIBILITY: <public|private|unknown>
HISTORY: merge-base <sha|none> (exit <n>) | left/right <remote>/<local> | roots <sha>/<sha>
PUBLISH_SET: <n commits; authors; non-task commits disclosed>
COMMAND: <exact push command, or not-attempted>
PUSH: not-attempted | attempted once (exit <n>) | <stderr summary>
REFS_UPDATED: measured <remote heads> | inferred none (<evidence>) | not-measured
REMOTE_REF: <ls-remote sha|unverified> | BASE_REF: <unchanged|changed|unverified>
VERDICT: BLOCKED (<reason>) | PUSHED_AND_VERIFIED (<sha>) | PUSHED_UNVERIFIED (<reason>) | LEFT_LOCAL
```

## Measured case (2026-10-25/26, ~/.qoder/skills)

- Commit `663b2d6` was proven locally (H0=H1=H2=H3) before the push request.
- The push turn began at 06:16:34. `git remote -v` was empty (exit 0). `origin`
  was configured at 06:16:47 and fetched at 06:16:55; the commit was created at
  06:16:11. No process attribution was measured.
- The push to `https://github.com/khoi/skills.git` was denied:
  `Permission ... denied to BaBiYeuCon`, 403, exit 128. Remote heads were
  measured with `ls-remote` before and after: `main` stayed `8a6423f`, no
  `local-skills-2026-10-26` branch appeared, and local HEAD stayed `663b2d6`.
  The local tracking ref was not refreshed; the remote heads, not the cache,
  were the measurement.
- Histories: left/right `15 4`, no merge base, roots `30a826a` / `91d6051`.
- The "switch account" request at 06:22:55: only `BaBiYeuCon` was
  authenticated, helper `osxkeychain`, no SSH identities at that turn, no
  second account. A dedicated ed25519 key was generated later (06:23:22) only
  after an explicit owner instruction; as of 06:29 SSH auth still failed (exit
  128), so the key was not an authorized identity on the target. The gate made
  no credential or config change.
- Re-measured 2026-10-26: `origin` is now `git@github.com:khoi/skills.git`
  (SSH), config mtime 06:23:01; `origin/main 8a6423f`, still no merge base, no
  upstream. Local `main` was `663b2d6` at 06:29 and `65fed42` by 06:32 (three
  more local commits from another writer; the remote did not move). The
  destination changed after the session; attribution unmeasured. Re-measure
  everything on every run.
- Read-only fallback measured 06:27-06:29: with `origin` on SSH and
  `git ls-remote origin` failing 128, `git ls-remote --heads
  https://github.com/khoi/skills.git` returned `main 8a6423f`; the candidate
  branch was empty and the `main` control returned a ref. Use this only to
  verify, not to change credentials.
- Exit-code trap measured in the same session: the first SSH `ls-remote`
  printed `ls_remote_exit=0` beside a fatal error; re-running the command alone
  returned 128. Capture the command's own status, not a pipeline's.

## Hard rules

- Resolve destination, protocol, identity, history and publish set immediately
  before pushing; never from an earlier turn or a summary.
- Empty `git remote -v` is not proof of no destination; inspect `.git/config`,
  its mtime and the reflog.
- Never read tokens, keychain entries or private keys. `gh auth status`, the
  credential-helper config and `ssh-add -l` are the only identity probes.
- Match the credential check to the resolved protocol.
- Measure push permission and visibility before the attempt. `push:false` for
  the sending account, or `PERMISSION_UNKNOWN` without the owner naming the
  account, stops the push.
- No merge base means no push; never force, delete or rename a remote ref
  without explicit owner direction naming that ref.
- One push attempt. A failure produces a report, not a retry.
- List the publish set and every author before pushing. Stop if any commit is
  not part of the approved task.
- Check a new branch name is free before pushing; never overwrite an existing
  remote ref.
- Verify the pushed ref with `ls-remote`; `refs/remotes/...` is a cache, not
  proof.
- Never generate or install credentials or keys from this gate; that is a
  separate explicit owner task.
- Never add or edit remotes, change credentials or git config, or alter refs on
  your own initiative.
- "No refs updated" is measured from remote heads, or it is labeled inferred.
- A successful push proves transfer only - not correctness, CI, review,
  accuracy or acceptance.

## Boundaries

- Commit byte provenance (H0-H3, staged set, package completeness) belongs to
  `skill-artifact-commit-provenance`; its hard rules explicitly exclude push.
  Run it before this gate; this skill does not re-prove commits.
- Exit-code harness correctness and `set -e`/`pipefail` behavior belong to
  `shell-command-result-verification`.
- Concurrent writes inside a skill directory belong to
  `skill-dir-concurrent-write-reconcile`; edit-conflict merging belongs to
  `concurrent-skill-md-edit-recovery`.
- Source-session evidence grading belongs to `verify-skill-source-session`.
- Credential and SSH-key lifecycle (generating, adding, rotating, passphrase
  policy) is outside this gate; it needs a separate explicit owner instruction
  and an account with write access. This gate only observes identity state.
- This skill never authorizes a push by itself. The owner's instruction
  authorizes one named ref, one destination.

## Related skills

- `skill-artifact-commit-provenance` - proves the committed bytes; excludes
  push.
- `shell-command-result-verification` - captures exit codes per branch,
  including non-zero results that are valid outcomes.
- `user-skill-overlap-audit` - checks for a duplicate before creating or
  expanding a skill.
- `skill-postwrite-registration-verification` - proves a written skill loads.

## Resources

None. No bundled scripts or references: the gate is a short sequence of
read-only `git`/`gh` measurements plus a stop decision, and a push helper would
add a write-capable surface this skill exists to prevent.
