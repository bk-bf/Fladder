# Fladder — fork conventions

This is a fork of `Fladder-App/Fladder`. Our line is the **`kirill`** branch; it
is the default branch here and everything ships from it. Upstream's own `main`
is their release branch, which is why ours is not called that.

Upstream development happens on their `develop`. `upstream-mirror` is an
untouched copy of it, kept only as a reference — never commit to it.

The `upstream` remote is fetch-only; its push URL is deliberately invalid. Never
push to upstream, and never open a PR against it from an automated session.

## How this fork stays current

Nobody merges this by hand. A `mon` check named `fladder` runs
`.fork/bin/behind` in this checkout once an hour on ubuntuserver. While it exits
zero nothing happens and nothing is spent. When it exits non-zero, mon starts a
Claude Code session here — tagged `ci/cl`, in `acceptEdits` — which merges
upstream in, resolves the conflicts, verifies, and pushes. If it cannot, it
stops without pushing and says what it needs.

The session runs in this directory, which is why this file is the place merge
conventions belong: it is read before the work starts.

| | |
|---|---|
| Checkout | `~/Documents/Projects/Fladder` on ubuntuserver |
| mon | `~/Documents/Projects/mon`, config `~/.config/mon/monitors.json` |
| Reading it back | `mon sessions`, `mon show <id>`, `mon steer <id> "…"` |
| Also visible at | the dashboard's Monitor page |

### `.fork/bin/behind` is the trigger, and its output is load-bearing

It exits non-zero when `kirill` is behind `upstream/develop`, and prints how far
behind along with the upstream tip.

**mon hashes that output to decide whether this is news.** A failing check that
keeps saying the same thing is acted on once, not once per poll. So whatever it
prints must change only when there is genuinely something new to try — the
upstream tip, and nothing else. Adding a timestamp, a duration, a commit count
that moves on its own, or anything else that drifts would start a fresh merge
session every hour on a merge that already failed.

An upstream that cannot be reached exits non-zero on purpose rather than
reporting the fork as level. A fork nobody can ask about must not read as one
that is up to date.

## Merging upstream in

Merge `upstream/develop` into `kirill`. `git rerere` is enabled, so a conflict
resolved once replays automatically the next time the same one appears — do not
disable it, and do not resolve a conflict in a way you would not want repeated.

Where the two sides genuinely disagree about behaviour, keep this fork's
behaviour. That is what the fork is for.

## Generated files

Upstream **commits** its codegen output — about 95 files matching `*.g.dart` and
`*.freezed.dart`, plus `lib/routes/auto_router.gr.dart`. They are not ignored
and not regenerated on checkout. Codegen comes from freezed, riverpod_generator,
chopper, auto_route and swagger_dart_code_generator; `build.yaml` lists the
inputs.

**Do not regenerate as part of an upstream merge.** Running `build_runner` here
does not reproduce upstream's committed output byte for byte — the generators
resolved by a fresh `pub get` are newer than the ones upstream last ran, so the
output comes back reordered and rewrapped. Regenerating on every merge would
commit that churn, and the fork would then conflict with upstream on files
neither side meaningfully changed.

So treat generated files as ordinary files:

- On conflict, take **upstream's** version. It matches upstream's source, which
  is what the rest of the merge is bringing in.
- Regenerate only if the build actually fails because the output is stale, and
  then commit only the files that had to change.
- `pubspec.lock`: take upstream's, then `flutter pub get`.

Generating output for **our own** changes is different — that divergence is
deliberate, and the regenerated files get committed with the change that needed
them. That belongs on `dev`, not in a merge commit.

### Never stage while a build is running

`build_runner --delete-conflicting-outputs` removes the committed generated
files before writing them back. A `git add -A` in that window records ~112
deletions that nobody intended, and the commit looks ordinary — it is only
visible as a five-figure deletion count in `git show --stat`. This has already
happened once here. Check nothing is generating before staging:

```sh
pgrep -af build_runner
```

## Checking the result

```sh
.fork/bin/verify --build   # what a merge should pass before it is pushed
.fork/bin/verify           # analyze + test only, while iterating
.fork/bin/verify --regen --build   # only when stale codegen is the actual problem
```

`--build` compiles the Linux app. `--regen` rebuilds the generated files first
and is **not** part of a normal merge — see "Generated files" above for why.

The toolchain is pinned by `.fvmrc` to Flutter 3.35.7 and lives at
`~/opt/flutter/bin` on this host — it is not on `PATH` by default.

**Never push a merge that has not passed `--build`.** Compiling is the only
check that proves the result works; analyze and test between them do not.
Budget for it — a clean release build takes upwards of fifteen minutes, and the
first one on a machine also downloads the mdk-sdk media backend.

`flutter analyze` on the fork point reports **No issues found**, so treat any
analyze output at all as a regression this merge introduced, not as pre-existing
noise. It takes about 90 seconds.

`flutter test` is a weak signal — there are 2 test files and only one of them is
real. `test/pip_manager_test.dart` has 9 passing tests;
`test/widget_test.dart` is the `flutter create` boilerplate, unchanged since the
initial commit, which taps a counter this app has never had and has always
failed. `verify` skips it by name, so **do not run bare `flutter test`** and
conclude the merge broke something — that failure predates the fork. Anything
upstream adds later is picked up automatically.

If something makes the build impossible to run rather than failing it, say that
plainly instead of reporting the merge as checked. A merge that was never
compiled and one that compiled cleanly must not read the same way.

## Translations

`lib/l10n/*.arb` is Weblate output and makes up most of upstream's commit
traffic. Conflicts there are mechanical — take both sides' new keys. It is never
worth stopping a merge over.

## Changing the automation

There is no `mon edit`. A check is changed by removing it and registering it
again, and a monitor's identity is its name, source and project together — so
re-registering under the same name at a new path is correctly treated as a
different check, and the old one retires itself at its next tick. A new
registration is picked up within about ten seconds; no restart is needed.

```sh
MON=~/Documents/Projects/mon/mon
$MON rm fladder
$MON check fladder \
  --run '.fork/bin/behind' \
  --then '<what the session is asked to do>' \
  --project ~/Documents/Projects/Fladder \
  --mode acceptEdits --tag ci/cl \
  --title 'merge upstream Fladder into the kirill fork'
```

Removing a *tail* monitor (docker, journald, file) still wants a
`systemctl --user restart mon.service`; checks retire on their own.

## GitHub Actions on this fork

The four workflows are inherited from upstream and all active, but nothing has
run: they trigger on `master`, on tags, and on pull requests, none of which this
fork's `kirill` branch touches. Before relying on CI here, know that

- `build.yml` carries a nightly cron that builds whenever `develop` moved,
- the macOS and iOS legs need signing secrets this fork does not have and will
  fail if they ever fire,
- `prepare-release.yml` is written for upstream's release flow, not ours.

Local `.fork/bin/verify --regen --build` is what actually gates a merge here.

## What this fork changes

Nothing yet — `kirill` is upstream's `develop` plus this file and `.fork/`. Once
that stops being true, record what diverged and why, because that is exactly the
context a merge session needs to decide which side wins a conflict.
