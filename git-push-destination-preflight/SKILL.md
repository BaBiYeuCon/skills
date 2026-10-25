---
name: git-push-destination-preflight
description: Pre-push gate and post-push proof for git push. Use when asked to push a commit or branch; when a push failed with 403 / non-fast-forward / forced update / "Permission denied (publickey)"; when origin appeared or its URL/protocol/pushURL changed; when histories share no merge base; when push.followTags or remote.<name>.push may carry extra refs; when an account cannot push; or when naming a remote branch. Covers remote/pushURL re-reads, identity checks without secrets, merge-base and divergence counts, publish-set and visibility disclosure, branch-name collision checks at the push URL, one explicit src:dst push, exit code captured outside a pipeline, and ls-remote verification at the push URL with full object IDs, plus a read-only HTTPS probe when SSH auth fails. Stops with the exact blocker instead of retrying, force-pushing, adding remotes or changing credentials. Commit provenance: skill-artifact-commit-provenance; exit codes: shell-command-result-verification; key/credential lifecycle is out of scope.
---

# Git Push Destination Preflight

## Overview

A push writes to shared state, and its destination is not stable across a task.
Measure four facts immediately before pushing: destination (remote, URL,
pushURL, protocol), sender (credential path, account), history relationship,
and the exact commits that would become visible. If any is unresolved or
contradictory, stop and report the exact blocker. A successful `git commit`
says nothing about any of them, and a successful push report says nothing about
where it landed: query the push URL and compare full object IDs after it.

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
- `push.followTags` is set, `remote.<name>.push` exists, or the remote has a
  pushURL that differs from its fetch URL.
- Commit provenance just passed and the owner now asks to push.

## Gate 0 - resolve the destination at push time

Run these now, not from memory:

```sh
git rev-parse --abbrev-ref HEAD
git branch -vv
git remote -v
git config --show-origin --get-regexp '^(remote\..*\.(url|pushurl)|branch\..*\.(remote|merge)|push\.|url\.)'
stat -f '%Sm %N' .git/config 2>/dev/null || stat -c '%y %n' .git/config
git rev-parse --abbrev-ref --symbolic-full-name '@{u}'
```

- An empty `git remote -v` is not proof there is no destination. Measured: it
  was empty with exit 0 at 06:16:34, `origin` appeared with a `.git/config`
  mtime of 06:16:47 and was fetched at 06:16:55, and the push then targeted
  that URL. The mtime dates the write, not its author. Re-run after any
  failure and read `.git/config` directly.
- Read URL and pushURL. They can differ; a push uses the push URL. Capture it
  once and use it for every remote measurement in Gates 3 and 4:

  ```sh
  PUSH_URL="$(git remote get-url --push <remote>)"; echo "$PUSH_URL"
  ```

  Redact any `user:token@` part before it enters a report; the URL that proves a
  push is its host and path, never a credential.
- Read the protocol. HTTPS uses the git credential helper or `gh`; SSH uses
  ssh-agent and keys. Gate 1 depends on it.
- URL rewrites can replace the effective URL without changing the remote
  entry: `insteadOf` for fetches, `pushInsteadOf` for pushes. The `url\.` part
  of the query above lists both; check it before trusting `remote.<name>.url`.
- `@{u}` is the only proof of tracking; a `refs/remotes/...` ref can exist
  without it. Exit 128 `no upstream configured` is a result, not a failure.
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
# HTTPS destination
gh auth status
git config --show-origin --get-all credential.helper

# SSH destination
ssh-add -l
ssh -o BatchMode=yes -T git@github.com   # contacts GitHub; may update known_hosts
```

- `gh auth status` reports the active account, scopes and a masked token. That
  is identity inspection, not a secret read. Its `Git operations protocol`
  line describes gh's own integration, not the repository's remote.
- Scopes are not permission. Measured: the denied account's token carried the
  `repo` scope and still received 403. Scopes describe what a token may ask
  for, not what an account may write to one repository; only the `permissions`
  probe or the push result speaks to write access.
- Run `ssh-add -l` only for an SSH destination. Exit 1 with "The agent has no
  identities" is a result, not a command failure. A listed key proves only that
  a key is loaded - not which account owns it, whether GitHub has it, or
  whether that account can write.
- `ssh -o BatchMode=yes -T git@github.com` maps an offered key to an account
  without a passphrase prompt. Success prints `Hi <account>!` and exits 1;
  exit 255 means no accepted key. It is a read-only probe, but it contacts
  GitHub and may add a host key to `known_hosts`.
- Never run `gh auth token`, never query the keychain
  (`security find-internet-password` or equivalents), never print, request or
  accept a token in chat, and never change the credential helper.
- Match the check to the protocol. On HTTPS, a 403 means the credential git
  sends lacks write access. On SSH, `Permission denied (publickey)` is a key or
  agent problem, while `ERROR: Permission to <owner>/<repo> denied to <user>`
  is an authenticated key whose account lacks write access - switching gh
  accounts does not change the key's account.
- Account-switch request: enumerate the authenticated accounts; if only one
  exists, stop. `gh auth login`, `gh auth switch`, `gh auth setup-git`, creating
  an account or entering a token need the owner's explicit direction. With
  `osxkeychain`, `gh auth switch` alone may not change the credential git sends.
- Measure push permission and repo visibility before Gate 3, not by attempting
  the push and reading its 403. When `gh` is authenticated and can read the
  repo:

  ```sh
  gh api repos/<owner>/<repo> \
    --jq '{push: .permissions.push, private: .private, visibility: .visibility}'
  ```

  - HTTPS: `gh api` reports the `gh` account; git sends whatever the helper
    returns (`gh auth setup-git` matches, `osxkeychain` can differ). When they
    match, `push:false` means stop. Measured 2026-10-26 06:47: `push:false`,
    `private:false`, `visibility:public` for `BaBiYeuCon`; the original push had
    no pre-push probe and failed 403, exit 128.
  - SSH: the sending identity is the key's account, which need not be the `gh`
    account. `ssh -T` maps an accepted key to an account; `gh api` reports the
    `gh` account's permissions only. If they differ, the key account's
    permission is `UNKNOWN`; it is not inherited from either side.
  - If the probe errors or the account cannot see the repo, report
    `PERMISSION_UNKNOWN` and require the owner's explicit go-ahead naming the
    repo and the account. Unknown is not authorized.
  - A probe is not the push result: permission can change before the attempt,
    so the 403 triage still applies.
- Repairing access - switching account or transport, generating or registering
  a key - is a separate owner-authorized task owned by
  `remote-push-access-preflight`. This gate observes and stops.

## Gate 2 - prove the histories are related

```sh
REMOTE=<remote>; BRANCH=<remote-branch>; UP="$REMOTE/$BRANCH"
git merge-base HEAD "$UP"; echo "merge_base_rc=$?"
git rev-list --left-right --count "$UP...HEAD"
git merge-base --is-ancestor HEAD "$UP"; echo "local_ancestor_rc=$?"
git merge-base --is-ancestor "$UP" HEAD; echo "remote_ancestor_rc=$?"
git rev-list --max-parents=0 HEAD
git rev-list --max-parents=0 "$UP"
git reflog show --date=iso "$UP"
```

- Empty merge-base with exit 1 means no common ancestor. Exit 128 with a fatal
  message means the ref is missing or unreadable - a measurement failure, not
  an unrelated-history result. Measured: left/right `15 4` at the push request
  and `15 10` at 06:47, roots `30a826a` (remote) and `91d6051` (local). A
  plain push is non-fast-forward; a force-push would overwrite 15 remote
  commits.
- A remote-tracking ref can be stale. Its reflog date is the measurement.
  `git fetch <remote> <branch>` refreshes it but mutates local remote-tracking
  refs, so classify the fetch and run it only against a confirmed remote.
- Both ancestor checks exiting 1 alongside an empty merge base is unrelated
  history; an exit 0 means fast-forward is possible in that direction - report
  which. A merge base with both checks exiting 1 is divergence; both cases
  forbid a plain and a force push. An empty result needs a control:
  `git merge-base HEAD HEAD` returns a commit hash.
- No merge base blocks a plain and a force push to that existing remote
  branch; never substitute `--force`, `--force-with-lease`, ref deletion or a
  branch rename. A new branch is a different destination, not a workaround: it
  needs the owner's explicit confirmation and the Gate 3 collision check.

## Gate 2a - disclose every commit the push would publish

```sh
REMOTE=<remote>; BRANCH=<remote-branch>; UP="$REMOTE/$BRANCH"
git log --format='%h %an <%ae> %s' "$UP"..HEAD
git log --format='%h %an <%ae> %s' HEAD --not --remotes   # if "$UP" does not exist
git status --short
```

- A push publishes commits, not a task. List every commit that would become
  reachable on the remote, with author and subject. If the remote branch does
  not exist or has no merge base, use the `HEAD --not --remotes` form, state
  that the tracking refs may be stale, and name the base you compared against.
- Keep the explicit `HEAD`. Measured 2026-10-26: `git log --not --remotes`
  alone returned empty with exit 0 while `git log HEAD --not --remotes` listed
  10 commits at 06:47 and 11 at 06:48, when another writer committed
  `9d47ba4`. An empty publish set is a claim, so re-run the form that has a
  positive revision.
- Stop and ask the owner if any commit is outside the commit set they approved.
  Measured at the push request: the four local-only commits were `91d6051`,
  `0e0465d`, `136f723` and the task commit `663b2d6`; the first three predate
  the task and their attribution was not measured. Disclose them; do not assume
  they are yours.
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
  PUSH_URL="$(git remote get-url --push <remote>)"; BRANCH=<base-branch>
  git ls-remote --heads "$PUSH_URL" "<new-branch>"
  git ls-remote --heads "$PUSH_URL" "$BRANCH"   # positive control
  ```

  Empty for the candidate and a ref for the control means the check
  discriminates. If the candidate already exists, stop and ask; never
  overwrite it. If the control is also empty, the remote has no refs or no such
  base branch and the check proves nothing: use a base ref that exists, or
  report the collision check as unmeasurable before pushing. Ask the push URL,
  not `ls-remote <remote>`: the latter reads the fetch URL, so a name held in
  the other repository can come back free.
- Bound the ref set before sending. An explicit single refspec is necessary but
  not sufficient:

  ```sh
  git config --get push.followTags
  git config --get-all remote.<remote>.push
  ```

  Measured (git 2.55.0, throwaway repos): with `push.followTags=true`,
  `git push origin main:refs/heads/main` also carried an annotated tag
  (`* [new tag] vA -> vA`), and `--no-follow-tags` kept the next tag local.
  `remote.<remote>.push` refspecs are another source of extra refs. Report every
  ref the push carried, not only the one you asked for.
- Use the explicit mapping, for example
  `git push --porcelain --no-follow-tags <remote> <src>:refs/heads/<branch>`.
  No `--force`, no `--force-with-lease`, no tags or other refs unless asked,
  and no `-u` unless the tracking decision was made. `push.autosetupremote`
  can set an upstream without `-u`; check it and report any tracking change.
- Rehearse with `--dry-run` when the ref set is uncertain: measured, it wrote
  nothing - the remote SHA was identical before and after - and it still printed
  the extra refs the real push would carry. A rehearsal is not proof.
- Snapshot the remote heads and the local cache before and after:

  ```sh
  PUSH_URL="$(git remote get-url --push <remote>)"
  git ls-remote --heads "$PUSH_URL"
  git for-each-ref refs/remotes/<remote>
  ```

  The first is authoritative; the second is a local cache and can be stale.
- Record the command, stderr, exit code and both snapshots. Capture the exit
  code outside any pipeline (`git push … >push.out 2>push.err; rc=$?`):
  measured, `git push | tail` printed `push_exit=0` from `tail` while the same
  rejected push exited 1 when captured without the pipe. One attempt only; a
  failure goes to triage, not a retry loop.

## Gate 4 - verify the remote ref, not the tracking cache

After the single push attempt:

```sh
PUSH_URL="$(git remote get-url --push <remote>)"; DEST=<remote-branch>
SRC=<source-ref>; BASE=<base-branch>
git ls-remote --heads "$PUSH_URL" "$DEST"
git rev-parse "$SRC"
git ls-remote --heads "$PUSH_URL" "$BASE"
```

- Query the **push URL**, not `git ls-remote <remote>`. The latter reads the
  fetch URL; when `pushurl` differs it answers about the other repository.
  Measured: pushurl `B.git` received the commit, fetch URL `A.git` stayed
  empty, and `git ls-remote origin` printed nothing with exit 0.
- The pushed ref must exist on the remote and its SHA must equal the local
  source SHA. The base branch must still equal the pre-push SHA - unless the
  push targets the base branch itself, where its SHA is expected to become the
  pushed SHA. Any other change to the base means something else wrote to the
  remote and needs explanation before the push is reported.
- Compare full 40-character object IDs. The pushed commit must equal the commit
  proven by `skill-artifact-commit-provenance`; an abbreviated match is
  ambiguous.
- Read the `--porcelain` lines for every ref. A forced update (`+` or `forced
  update`) replaced remote history; report it as such and never as a proven push
  of the commit you validated. `Everything up-to-date` means the remote already
  held that SHA: not a failed push, and still not a substitute for the
  `ls-remote` reading.
- `refs/remotes/<remote>/<branch>` is a local cache. A failed push can leave it
  stale; a successful push may update it, but the remote query is authoritative.
  Report `REMOTE_REF` from `ls-remote`, not from `for-each-ref`. Measured: with
  `pushurl` naming a second repository, the push updated local `origin/main` to
  the pushed SHA while the fetch URL had no such branch - that ref was never
  evidence about the repository the fetch URL names.
- If SSH auth blocks `ls-remote`, query the public HTTPS URL directly for a
  read-only check:

  ```sh
  git ls-remote --heads https://github.com/<owner>/<repo>.git "$DEST"
  ```

  Do not change `origin`, the credential helper or the protocol just to verify.
  If the repo is private or the query fails, report `UNVERIFIED` instead of
  `PUSHED`. That URL must name the same repository as the push URL; when they
  differ it verifies only the fetch URL.
- Positive control: query the known base branch and require a ref. An empty
  candidate result proves nothing until the same command returns the control.
- A successful push proves transfer only; it does not prove CI, review,
  correctness or acceptance.

## Failure triage

| Symptom | Meaning | Action |
| --- | --- | --- |
| HTTPS `403` / `Permission ... denied to <account>` | The credential git sent lacks write access to this repo | Stop. Report URL, account, stderr, exit code. Offer a new branch on a writable remote, an owner-granted access change, or leaving the commit local. |
| SSH `ERROR: Permission to <owner>/<repo> denied to <user>` | The key authenticated as `<user>`, who lacks write access | Stop. Report the user, URL, stderr, exit code. Do not switch gh accounts; the key's account is the identity. Offer a new branch on a writable remote, a granted access change, or leaving the commit local. |
| `non-fast-forward`, `fetch first`, no merge base | Histories are unrelated or diverged | Stop. Show merge-base, left/right counts and roots. Never force. |
| `Permission denied (publickey)` | SSH key or agent problem | Stop. Report `ssh-add -l` and the key state. Do not switch gh accounts, edit config or add keys. A public remote can still be read over its HTTPS URL for ref verification; that is a read-only probe, not an auth workaround. |
| `repository not found`, `404` | Wrong URL, or no access to a private repo | Re-resolve the URL. Stop. |
| `push` returned success but the remote ref is missing or has a different SHA | The transfer report is not a ref measurement | Stop. Re-measure with `ls-remote`; report `PUSHED_UNVERIFIED`, not `PUSHED`. |
| `[new tag]` or an unrequested ref in the output | `push.followTags` or `remote.<name>.push` added refs | Stop. Report every ref carried; remove nothing without owner direction. |
| `ls-remote <remote>` empty with exit 0, or local `origin/<branch>` moved while the fetch URL lacks it | You measured the fetch URL; the push may go to another repository | Re-read `git remote get-url --push <remote>` and query that URL. |
| `No configured push destination`, no upstream | The remote or tracking is unset | Stop. Ask the owner for the destination. Do not add a remote silently. |

After a failed attempt, report whether refs changed as `measured` (from the
remote-head `ls-remote` snapshots) or `inferred` (from the rejection message). A
rejection text alone is not a ref measurement; the local tracking cache is not
the remote ref.

Credential changes are outside this gate; `remote-push-access-preflight` owns
that lifecycle. If the owner authorizes generating or adding a key, treat it as
a separate task: dedicated key, public half only, never print the private key,
and confirm the account has write access. A key that authenticates proves
identity, not permission; `gh ssh-key list` needs `admin:public_key`, which the
measured token lacked. Report it only from its own measurement (`ssh -T`, the
key listed on the account), never from having been asked. Passphrase policy is
an owner decision, not a default.

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
UPSTREAM: <remote>/<branch> from @{u} | none (exit 128)
IDENTITY_PATH: https (helper <...>, gh <account>) | ssh (agent <n identities>)
IDENTITY: <sending account|unknown> | key account <name|unknown>
PERMISSION: <push true|false|unknown> for <account> | VISIBILITY: <public|private|unknown>
HISTORY: merge-base <sha|none> (exit <n>) | left/right <remote>/<local> | roots <sha>/<sha>
PUBLISH_SET: <n commits; authors; non-task commits disclosed>
PUSH_URL: <url> | same as fetch URL
COMMAND: <exact push command, or not-attempted>
PUSH: not-attempted | attempted once (exit <n>, captured outside a pipeline) | <stderr summary>
REFS_BEFORE: <remote heads at push URL, before>
REFS_CARRIED: <refs from --porcelain> | not-measured
REFS_AFTER: <remote heads at push URL, after>
REFS_UPDATED: measured (<before> -> <after>) | inferred none (<evidence>) | not-measured
REMOTE_REF: <full sha at push URL|unverified> | BASE_REF: <unchanged|changed|unverified>
PROOF: <remote sha> = <validated commit sha> | differs | forced | not-measured
VERDICT: BLOCKED (<reason>) | PUSHED_AND_VERIFIED (<sha>) | PUSHED_UNVERIFIED (<reason>) | LEFT_LOCAL
```

## Measured case (2026-10-25/26, ~/.qoder/skills)

- Commit `663b2d6` was proven locally (H0=H1=H2=H3) before the push request.
- The push turn began 06:16:34; its first `git remote -v` was empty (exit 0).
  `origin` appeared with `.git/config` mtime 06:16:47 and was fetched at
  06:16:55; the commit was created 06:16:11. The mtime dates the write, not the
  author.
- The push to `https://github.com/khoi/skills.git` was denied
  `Permission ... denied to BaBiYeuCon`, 403, exit 128. No `ls-remote` snapshot
  preceded the attempt, so "no refs updated" is inferred from the rejection,
  not measured. Later read-only HTTPS queries returned `main 8a6423f`, no
  `local-skills-2026-10-26` branch, and the `main` control returned a ref.
- Histories: left/right `15 4` at the push request, `15 10` at 06:47; no merge
  base; roots `30a826a` (remote) / `91d6051` (local).
- The "switch account" request at 06:21:18: only `BaBiYeuCon` was
  authenticated, helper `osxkeychain`, no SSH identities at that turn, no
  second account. No credential or config change was made by this gate.
- Re-measured 2026-10-26 06:47: `origin` and its push URL are
  `git@github.com:khoi/skills.git` (SSH), config mtime 06:23:01, no upstream
  (`@{u}` exit 128), no URL rewrite, `push.followTags` unset,
  `remote.origin.push` unset. `gh auth status` still names `BaBiYeuCon`
  (HTTPS preference) with the `repo` scope; `gh api` returns `push:false`,
  `private:false`, `visibility:public`. One ED25519 key
  `qoder-skills-2026-10-26` is loaded; `ssh -o BatchMode=yes -T git@github.com`
  prints `Permission denied (publickey)` and exits 255.
- Local `main` moved from `663b2d6` (06:16:11) through `65fed42` (06:29:52),
  `c7c5db6` (06:43:42) and `9d47ba4` (06:46:59) to `a839af7` (06:49:15); the
  remote head stayed `8a6423f`. The writers after the task were not measured;
  the destination changed after the push turn. Re-measure everything on every
  run.
- `git log --not --remotes` returned empty with exit 0 while
  `git log HEAD --not --remotes` listed 10 commits at 06:47, 11 at 06:48 and
  12 at 06:50 - the positive-revision form is required.
- Exit-code trap measured in the same session: the first SSH `ls-remote`
  printed `ls_remote_exit=0` beside a fatal error; re-running the command alone
  returned 128. Capture the command's own status, not a pipeline's.
- Post-push and ref-set proof measured 2026-10-26 in throwaway repos (git
  2.55.0), never against `khoi/skills`: `push.followTags=true` added
  `* [new tag] vA -> vA` to an explicit single-refspec push, and
  `--no-follow-tags` kept the next tag local; with `pushurl` = `B.git` and
  fetch URL `A.git`, the push landed in B, local `origin/main` updated to the
  pushed SHA, `git ls-remote origin` returned empty with exit 0, and A had no
  such branch; `--dry-run` left the remote SHA unchanged; `git push | tail`
  printed `push_exit=0` while the same rejected push exited 1 without the pipe.

## Hard rules

- Resolve destination, protocol, identity, history and publish set immediately
  before pushing; never from an earlier turn or a summary.
- Empty `git remote -v` is not proof of no destination; inspect `.git/config`,
  its mtime and the reflog.
- Never read tokens, keychain entries or private keys. `gh auth status`, the
  credential-helper config, `ssh-add -l` and `ssh -o BatchMode=yes -T
  git@github.com` are the identity probes; `ssh -T` contacts GitHub and may add
  a host key to `known_hosts`. `gh api ... .permissions` is a permission probe.
- Match the credential check to the resolved protocol.
- Token scopes are not repository write permission; only the `permissions`
  probe or the push result speaks to write access.
- Measure push permission and visibility before the attempt. `push:false` for
  the sending account, or `PERMISSION_UNKNOWN` without the owner naming the
  account, stops the push.
- No merge base blocks a push to that existing remote branch; never force,
  delete or rename a remote ref without explicit owner direction naming that
  ref. A new branch is a new destination requiring owner confirmation.
- One push attempt. A failure produces a report, not a retry.
- List the publish set and every author before pushing. Stop if any commit is
  not part of the approved task.
- Check a new branch name is free before pushing; never overwrite an existing
  remote ref.
- An explicit refspec is not a ref-set guarantee; read `push.followTags` and
  `remote.<name>.push`, and report every ref the push carried.
- `push.autosetupremote` or `-u` changes tracking; report any upstream change.
- Verify the pushed ref at the push URL with `ls-remote`, comparing full object
  IDs; `refs/remotes/...` is a cache, not proof.
- Capture the push's own exit code outside any pipeline. A forced update is not
  a proven push of the commit you validated.
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
  policy) is owned by `remote-push-access-preflight`; this gate observes
  identity state and stops. Its `ACCESS_PROVEN` is a precondition, never an
  authorization to push.
- This skill never authorizes a push by itself. The owner's instruction
  authorizes one named ref, one destination.

## Related skills

- `remote-push-access-preflight` - diagnoses and repairs push access; stops
  before this gate and never pushes.
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
