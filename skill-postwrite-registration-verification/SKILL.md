---
name: skill-postwrite-registration-verification
description: Verify a newly written or updated Skill before reporting it available. Use after editing SKILL.md or resources, when asked to execute /skills reload or confirm registration, or when skill-creator step 6 is the only closing guidance. Preserve a package baseline, check approved-file scope, recursively validate package Markdown links and anchors, separate citation structure from meaning, test authorized commands with positive and negative controls, load the exact name through the Skill tool, then compare the complete file set and hashes again. Distinguish in-app slash commands from shell probes and report validation gaps without silently changing unrelated resources.
---

# Skill Post-Write Registration Verification

## Overview

Writing `SKILL.md` is not registering a skill. Keep package integrity, command
execution, citation meaning and runtime loading as separate claims. Run the four
passes and the final hash comparison before reporting completion; disclose every
unexecuted check and known limitation.

## Trigger shape

- A skill directory was just written or edited, or resources were moved into it.
- The user replies `execute /skills reload` or asks if the new skill took effect.
- `skill-creator` step 6 ("restart the session or run `/skills reload`") is the
  only closing guidance available.
- A slash command has to be answered from an agent session that has no shell
  equivalent.

## Before writing - baseline and approved scope

Record the approved target and exact relative paths allowed to change. Before
editing, preserve the package in an authorized temporary directory outside the
package, with its timestamp, complete sorted file list and SHA-256 per file.
Keep the baseline immutable; do not re-snapshot it to hide drift. If verification
starts after writing and no earlier baseline exists, report scope preservation
as NOT CONFIRMED rather than reconstructing one.

Compare added, modified and deleted paths against that baseline after editing.
Require every difference to be approved and every untouched file to retain its
hash. Report the actual lists, not just a count. Symlinks are not permission to
read outside authorized roots; record their targets and any unverified content.
If another writer intervenes, preserve both versions and use
`skill-dir-concurrent-write-reconcile`; do not restore the whole package.

## Pass 1 - package, resources and citations

Reuse the shared checker without changing its defaults or copying it:

```sh
node /Users/khoi/.qoder/skills/verify-skill-source-session/scripts/check_skill.mjs --check-refs <skill-dir>
```

Require `VERDICT: OK`. It checks frontmatter boundaries, hyphen-case `name`,
folder/name match, single-line description ≤1024 chars, body <500 lines, to-do
markers and extraneous docs. Its link and inline-path checks cover `SKILL.md`,
not recursive resource links or heading anchors. Read each written file in full
for wording and claim boundaries that the checker cannot assess.

### Recursive resource links and anchors

Inventory all package Markdown, including nested resources, not only changed
files. Resolve each local link relative to the file containing it; validate the
linked file and fragment, then inspect linked in-package Markdown too. Track
visited files so cycles terminate. Include same-file fragments and referenced
resource paths; report files, links and anchors actually checked.

Match anchors against real heading IDs or explicit IDs using the renderer's
convention, including duplicate headings and encoded fragments. If the syntax
or renderer is unsupported, report that item as unchecked rather than passing
it with a guessed slug. A top-level file-existence check is not this pass.

Check external local targets only within authorized read scope and do not recurse
into sibling packages or follow network links automatically. Report excluded
links and inaccessible targets explicitly; do not widen permissions to turn a
partial check green. Keep temporary validators and fixtures out of the package.

### Citation structure is not citation meaning

```sh
node /Users/khoi/.qoder/skills/verify-skill-source-session/scripts/check_citations.mjs <skill-dir> --root <project-dir>
```

Use only an authorized source root. Take the citation count from the output;
zero citations means not applicable, not semantic validation. The checker proves
file existence and line bounds, not that the cited text supports the claim.
Read the relevant span and compare the actual wording, not just the first line.

Record structural and semantic results separately. For stale historical pointers,
identify the affected reference and the exact limitation; a shifted range may
still partly cover the right symbol. If that reference is outside the approved
edit set, preserve it, flag it in an approved file or the report, and exclude it
from verified-current claims. Fix in-scope errors and repeat the affected checks.
Never silently rewrite history or call the entire package semantically verified
while known stale or unread citations remain.

## Pass 2 - documented commands

Inventory commands in `SKILL.md` and its referenced resources. Classify them
before execution: read-only, temporary-fixture writes, or state-changing/live.
Resolve placeholders to authorized paths and retain the exact invocation, cwd,
stdout/stderr and actual exit status for each executed command, including the
probe below. Capture the tested process's status before filtering its output.

Run authorized read-only commands for real; keep positive and negative fixtures
inside the approved temporary directory. Documentation is not authorization to
render private files, call models, spend credits, start services or alter retained
evidence. Mark such commands DO-NOT-RUN with the reason. A safe wiring probe can
be recorded separately but cannot replace execution evidence; even an import may
have side effects, so inspect it first. Report those commands as documented, not
executed. A clean package check proves nothing about command execution.

## Pass 3 - controls

A check that returns nothing (`0`, no lines, exit 1 with empty output) is not
evidence until the same predicate fires where it must:

- On a deliberately broken copy (bad `name`, dangling link), the checker must
  print `VERDICT: PROBLEM` and exit non-zero.
- On a known-good sibling skill it must print `VERDICT: OK`.
- Exercise a nested-resource broken link and a missing anchor, as well as valid
  anchors and a cycle, to prove resource traversal rather than top-level checks.
- Against throwaway copies, require the scope check to detect an unapproved
  modification, addition and deletion. Require the final comparison to reject a
  late edit or a changed file set, not just match hashes for surviving paths.

Print expected and actual status plus a meaningful output signal per command.
Keep intentional non-zero controls distinct from unexpected failures. Use the
existing `shell-command-result-verification` workflow rather than inventing a
second exit-status convention.

After the disk passes, save the complete validated file list and SHA-256 map,
plus the shared checker versions used. Require no unexplained drift during the
checks; a file edited after its check needs revalidation. Keep this validated
map separate from the pre-edit baseline.

## Pass 4 - registration (the one commonly missing)

Call the `Skill` tool with the new name. Registration is CONFIRMED only when
the launch contains `Launching skill: <name>` **and** the body that comes back
is that skill's own content.

- A file-based skill also emits `Base directory for this skill: <dir>` as a
  separate injected line, not inside the tool's `output` field. So
  `read_chat_session` records only `Launching skill: <name>` for the same call:
  a missing base-directory line in a transcript is a capture artifact, not a
  failed registration. Built-in `skill-creator` emitted no such line in the
  observed launch.
- A different skill's body coming back, an error, or a missing name all mean
  **not confirmed** - never report registration on an unexecuted or mismatched
  load.
- For an update, check that the returned body contains the revised sections,
  not merely the existing name. A stale body does not confirm the update.
- Once this pass succeeds there is nothing to reload; success in the same
  session is normal. Loading is a verification action, not a request to repeat
  the entire workflow recursively.

### Final comparison after loading

Immediately after the load and before reporting, compare the live package with
the validated map: require identical relative file sets and matching hashes for
every entry. Repeat the approved-scope comparison against the original baseline.
If either differs, preserve the evidence, identify the drift, revalidate the
affected content and retry loading; never overwrite the other writer's work or
relabel the old checks as current. Recheck shared checker hashes too if they
changed during validation.

A matching disk map proves observed disk identity at those times, not a hash of
the runtime's in-memory body or absence of all intervening writes. Report body
loading and disk identity separately. A successful launch with stale validation
does not justify a completed-update claim.

## Slash commands are not shell commands

`/skills reload`, `/skills list` and `/<name>` are in-app commands of the Qoder
application. They have no shell equivalent, and `qoder` / `qoder-cli` are
generally absent from `PATH`.

Probe with a sentinel so an absence is visible instead of silent:

```sh
command -v qoder qoder-cli || echo NO_CLI_COMMAND
```

- Probe now; do not quote a past absence as a permanent fact. A binary on
  `PATH` is environment-dependent.
- Answering `execute /skills reload`: attempt pass 4 first. Report the observed
  result, and add "run `/skills reload` in the app or restart, then retry" only
  when the load actually failed. Telling a user to reload an already-loaded
  skill creates doubt where there is none.
- Never run shell commands alone, get "command not found", and stop there - the
  load attempt is the answer, the shell probe is only its boundary condition.

## Hard rules

- No completed-update claim before the revised body loads and the final file
  set and hashes match. A name in the available list alone is insufficient.
- Re-list `/Users/khoi/.qoder/skills/` immediately before writing. Siblings are
  added by parallel sessions; if the target appeared, switch to the update flow
  and read the current file first - `create` never overwrites.
- A missing target directory is the normal create branch, not a failure.
- `Skill` load failure is reported as a failure; a slash-command suggestion is
  a fallback for the user, never a substitute for the executed check.
- Empty or zero results require a control before they count.
- Preserve both baseline and validation maps; neither is permission to expand
  the approved edit set or rewrite unrelated historical citations.
- Report incomplete checks and semantic caveats separately from registration.
  Do not call partial traversal recursive validation of the whole package.

## Report template

```text
BASELINE     <time, manifest and approved edit paths>
SCOPE        added=[...] modified=[...] deleted=[...]; untouched=<N verified>
PACKAGE      <observed verdict>
RESOURCES    files=<N> links=<N> anchors=<N>; excluded/unchecked=<items>
CITATIONS    structure=<result, measured count>; meaning=<verified/stale/unread>
COMMANDS     <exact invocation and cwd> -> expected=<code> actual=<code> signal=<text>
NOT EXECUTED <command and reason, or none>
CONTROLS     <positive and intentional-negative results; unexpected failures>
REGISTRATION <launch output>; revised body=<confirmed/not confirmed>
POST-LOAD    file set=<match/drift>; hashes=<N/N>; checker versions=<identity>
LIMITATIONS  <remaining caveats, or none>
VERDICT      <verified update | registered with limitations | not confirmed>
```

## Boundaries

Own closing verification, not authoring approval, command implementation or
concurrent-edit recovery. Reuse `verify-skill-source-session` for source and
citation evidence, `shell-command-result-verification` for process outcomes,
`bundled-script-verification-checklist` for script branches, and
`skill-dir-concurrent-write-reconcile` for foreign edits. Do not modify those
skills or their shared scripts unless separately authorized.

## Resources

- No bundled scripts. Reuse these read-only shared tools:
  - [Package checker](../verify-skill-source-session/scripts/check_skill.mjs).
  - [Citation checker](../verify-skill-source-session/scripts/check_citations.mjs).
- Related: `skill-creator` (authoring rules), `verify-skill-source-session`
  (passes 1-3 in depth), `session-sourced-skill-distillation` (whole
  session-to-skill pipeline), `chunked-write-edit-payloads` (writing style).
