# Fork maintenance

This fork tracks `Fladder-App/Fladder` by cherry-picking individual upstream
commits onto `kirill`, rather than merging. There is deliberately **no merge
base** with upstream.

## Branches

| branch | what it is |
|---|---|
| `kirill` | our line — the default branch, everything ships from here |
| `upstream-mirror` | exact copy of `upstream/develop`, never edited, refreshed by `scan` |
| `develop`, `main` | inherited from the fork, unused |

`upstream` is fetch-only; its push URL is deliberately invalid.

## Running a triage pass

```sh
.fork/bin/scan                       # JSON lines: upstream commits awaiting a decision
.fork/bin/take <sha>                 # cherry-pick one, regenerate codegen, verify
.fork/bin/record <sha> <decision> <reason> [local-sha]
```

`take` exits `0` on success, `2` on a conflict in a real (non-generated) file
with the pick left staged for manual resolution, and `3` if verification failed
after a clean pick.

`scan` derives what is outstanding from `ledger.jsonl`, not from a pointer — so
a `deferred` commit keeps reappearing until it gets a real decision, and nothing
is silently walked past.

## What the ledger is for

`ledger.jsonl` records every upstream commit we have seen and what we did with
it. This is the mitigation for the structural weakness of cherry-picking: with
no merge base, git cannot tell you why a pick failed. The ledger can. When
`take` reports a conflict, grep the ledger for a `skipped` ancestor touching the
same files — backfilling that commit is usually the right fix and is far cheaper
than resolving the conflict by hand.

Expect this to get worse over time. That is inherent to the model, not a bug in
the tooling.

## Codegen

95 `*.g.dart` / `*.freezed.dart` files are committed upstream. They are never
cherry-picked — `take` restores ours and re-runs `build_runner`. `pubspec.lock`
is likewise regenerated. Do not resolve conflicts in generated files by hand.

## Verification

`flutter analyze` and `flutter test` run on every pick. Be aware the test suite
is **2 files** and proves almost nothing. The real gate is `flutter build linux`,
which needs apt packages not yet installed on this host; until then the fork's
GitHub Actions workflow is the backstop.
