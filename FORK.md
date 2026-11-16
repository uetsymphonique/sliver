# Sliver Fork — Branch Organization & Development Methodology

> General methodology for maintaining a Sliver fork focused on **agent execution changes** while preserving upstream operability. Applicable to any fork where custom implant behavior is developed on top of an actively maintained C2 codebase.

## 1. Problem Statement

The fork's purpose is narrow: change **how the agent executes modules** (shellcode streaming, reflective loading, in-memory crypto) while reusing Sliver's operational codebase — server infrastructure, C2 channels, client commands, protobuf transport, implant generation pipeline, and team server orchestration.

Upstream (BishopFox) ships features, bug fixes, and security patches on an irregular cadence. The fork must absorb those updates without repeatedly resolving the same conflicts in custom code.

## 2. Principles

1. **New code lives in new files.** A new Go package (`implant/sliver/shellcodemod/`, `client/command/shellcodemod/`) never conflicts with upstream because upstream doesn't know it exists.

2. **Core patches are append-only and minimal.** When a core file must be touched (handler map, protobuf definitions, constant blocks), changes are additions at the end of a block — never restructuring existing lines. This puts the patch in a region upstream is unlikely to modify in the same merge.

3. **Interface over implementation.** Where Sliver defines an interface (e.g. `extension.Extension`), implement it rather than replacing the dispatch mechanism. Custom implementations behind a stable interface survive upstream refactors of the concrete types.

4. **Don't fork what you don't change.** Resist the temptation to "clean up" or refactor upstream code. Every line you modify is a line that can conflict. If upstream code is ugly but functional, leave it.

5. **Track your surface area.** Maintain an explicit list of every upstream file you modify and what the patch does. During upstream sync, this list is the conflict checklist — not the full diff.

## 3. Branch Model

```
upstream/master ─────●────●────●────●────●────── (BishopFox releases)
                      \                    \
origin/master ─────────●─────────────────────●── (sync point, no custom code)
                        \                    \
origin/custom-agent ─────●────●────●──────────●─ (custom development)
                                               \
                                     origin/eval-build (optional, tagged)
```

### Branches

| Branch | Purpose | Who commits here |
|---|---|---|
| `origin/master` | Mirror of `upstream/master`. Receives upstream merges only. **No custom code.** | Sync operation only |
| `origin/custom-agent` | All custom development. Rebased onto `origin/master` after each sync. | Developer |
| `origin/eval-build` | Optional. Tagged snapshots for specific eval runs. Created from `custom-agent` at freeze points. | Developer (tag + branch) |

### Why rebase instead of merge

Merge preserves history but creates a growing web of merge commits that obscures what is custom vs. what is upstream. Rebase replays custom patches on top of the latest upstream, keeping the commit graph linear and the custom patch set explicit. Since the fork is private (not shared with upstream contributors), rebase is safe — there are no external consumers of the custom branch's commit hashes.

If the fork is shared among multiple developers, use merge instead of rebase and accept the messier history.

## 4. Remotes Setup

```bash
# Inside the sliver fork directory
git remote add upstream https://github.com/BishopFox/sliver.git
git remote set-url --push upstream no_push    # prevent accidental push to upstream

# Verify
git remote -v
# origin    https://github.com/uetsymphonique/sliver (fetch)
# origin    https://github.com/uetsymphonique/sliver (push)
# upstream  https://github.com/BishopFox/sliver.git (fetch)
# upstream  no_push (push)
```

## 5. Upstream Sync Procedure

```bash
# 1. Fetch upstream
git fetch upstream

# 2. Update the local mirror branch
git checkout master
git merge upstream/master --ff-only
git push origin master

# 3. Rebase custom work onto updated master
git checkout custom-agent
git rebase master

# 4. If conflicts arise — they will only be in tracked core patches (see §7)
#    Resolve, then continue
git rebase --continue

# 5. Force-push the rebased custom branch (private fork, safe)
git push origin custom-agent --force-with-lease
```

### When `--ff-only` fails

If `origin/master` has commits that aren't in `upstream/master` (shouldn't happen if the rule "no custom code on master" is followed), investigate before merging. This branch must remain a clean mirror.

## 6. Where Custom Code Goes

### New packages (zero conflict risk)

| Location | Content |
|---|---|
| `implant/sliver/shellcodemod/` | Shellcode module loader, in-memory crypto, module registry |
| `client/command/shellcodemod/` | Operator-side commands for the module system |
| `server/rpc/rpc-shellcodemod.go` | Server-side RPC handlers (if needed beyond passthrough) |
| `protobuf/shellcodemod/` | Separate .proto file for custom messages (alternative to appending `sliver.proto`) |

### Using a separate .proto file

Instead of appending messages to `sliver.proto` (which upstream modifies frequently), define custom messages in a new `protobuf/shellcodemod/shellcodemod.proto`. This eliminates conflict on the most-edited protobuf file. The generated Go code lives in its own package. The handler dispatch still needs a constant in `constants.go`, but the message definitions are isolated.

Trade-off: cross-referencing `commonpb.Request`/`commonpb.Response` from the new proto requires an import, and the implant's code generation templates may need awareness of the new package. Evaluate whether this isolation is worth the wiring cost — for 2-3 messages, appending to `sliver.proto` is simpler.

### Core files that must be patched (conflict surface)

| File | Patch type | Conflict risk |
|---|---|---|
| `protobuf/sliverpb/constants.go` | Append `Msg` constants at end of iota block | Low — upstream also appends |
| `protobuf/sliverpb/sliver.proto` | Append message definitions at end of file | Low-Medium — upstream adds messages too, but at EOF |
| `implant/sliver/handlers/handlers_windows.go` | Add entries to `windowsHandlers` map literal | Low — map entries are independent lines |
| `client/command/commands.go` (or equivalent registration) | Register custom command group | Low — registration is append-only |
| `server/rpc/rpc.go` (if custom server RPC needed) | Register RPC handler | Low |

**Total: ~5 files, ~2-5 lines each.** This is the entire conflict surface.

## 7. Patch Tracking

This section tracks every upstream file modified by the fork. During upstream sync, check this list first — if rebase reports a conflict, the patch description tells you exactly what to re-apply.

### Core Patches

_No patches yet — fork is at upstream parity._

<!-- Template for when patches are added:
| File | Change | Custom branch commit |
|---|---|---|
| protobuf/sliverpb/constants.go | +N Msg constants (MsgXxxReq, ...) | commit-hash |
| protobuf/sliverpb/sliver.proto | +N message definitions | commit-hash |
| implant/sliver/handlers/handlers_windows.go | +N handler map entries | commit-hash |
-->

## 8. Commit Discipline on the Custom Branch

### Separate "core patch" commits from "new code" commits

Structure the custom branch so that core patches are in **dedicated commits** with clear messages:

```
abc1234  core-patch: add shellcodemod handler entries
def5678  core-patch: add shellcodemod protobuf messages
1112131  feat: shellcodemod loader — VirtualAlloc + decrypt + VirtualProtect
2223242  feat: shellcodemod output callback
3334353  feat: client command — exec-shellcode-module
```

During rebase, only the `core-patch:` commits can conflict. New-code commits apply cleanly because they create new files. This separation makes rebase resolution predictable — you know which commits will need manual attention before you start.

### Keep core patches as small as possible

If a core patch grows beyond ~10 lines in a single file, reconsider the design. Can the logic move to a new file and the core patch shrink to a function call or map entry? Almost always yes.

## 9. Testing After Sync

After every upstream sync + rebase:

1. **Build the implant** — confirms Go compilation with updated dependencies
2. **Run upstream unit tests** — confirms custom patches don't break existing functionality
3. **Run custom tests** — confirms the module system still works against the updated implant
4. **Generate a test implant** — confirms the code generation pipeline incorporates custom packages

```bash
# From sliver root
make                        # build server + client
cd implant && go test ./... # run implant tests
# Generate and deploy a test implant to the lab range
```

## 10. When Upstream Breaks Your Patches

Scenarios and responses:

| Scenario | Response |
|---|---|
| Upstream adds entries to `windowsHandlers` map, causing merge conflict in adjacent lines | Resolve by keeping both: upstream's new entries + your custom entries. Trivial. |
| Upstream refactors `handlers_windows.go` structure (e.g. splits into multiple files) | Re-apply your map entries in the new structure. Use §7 to know exactly what to re-add. |
| Upstream changes the `Extension` interface | Adapt your implementation to the new interface. Since your code implements it rather than replacing it, the change is localized. |
| Upstream changes protobuf message numbering or iota block structure | Re-number your constants. Use §7 to find them instantly. |
| Upstream replaces `taskrunner` with a different execution model | Evaluate whether the new model serves your needs better. If yes, migrate. If no, keep your `shellcodemod` package — it's independent of `taskrunner`. |

## 11. Eval Build Snapshots

Before each evaluation run, create a tagged branch:

```bash
git checkout custom-agent
git checkout -b eval/q4-2026-run1
git tag -a q4-2026-run1 -m "Q4 2026 eval run 1 — shellcode streaming agent"
git push origin eval/q4-2026-run1 --tags
```

This preserves the exact implant source for reproducibility. Never rebase eval branches — their commit hashes are part of the evaluation record.

## 12. Decision Record: What Belongs in the Fork vs. External Tooling

| Change type | Where it belongs | Rationale |
|---|---|---|
| New execution primitive in the implant (shellcode loader, crypto, memory management) | **Fork** — `implant/sliver/shellcodemod/` | Must compile into the implant binary |
| Module compilation pipeline (source → shellcode blob) | **External tooling** — separate repo or `testlab-ael/resources/` | Build-time tooling, doesn't ship in the implant, no reason to couple to Sliver's build |
| Operator workflows (which modules to send, in what order) | **External tooling** or Sliver client scripts | Operational concern, changes per engagement |
| C2 channel modifications (new protocol, new encoding) | **Fork** — but reconsider whether Sliver's existing channels suffice | High conflict risk; Sliver's channel code is complex and frequently updated |
| Server-side orchestration (automated module deployment) | **Sliver gRPC API** — external script calling existing API | Avoids forking server code; Sliver's gRPC API is stable and well-documented |
