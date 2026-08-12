# Fladder — fork conventions

This is a fork of `Fladder-App/Fladder`. Our line is the **`kirill`** branch; it
is the default branch here and everything ships from it. Upstream's own `main`
is their release branch, which is why ours is not called that.

Upstream development happens on their `develop`. `upstream-mirror` is an
untouched copy of it, kept only as a reference — never commit to it.

The `upstream` remote is fetch-only; its push URL is deliberately invalid. Never
push to upstream, and never open a PR against it from an automated session.

## Merging upstream in

Merge `upstream/develop` into `kirill`. `git rerere` is enabled, so a conflict
resolved once replays automatically the next time the same one appears — do not
disable it, and do not resolve a conflict in a way you would not want repeated.

Where the two sides genuinely disagree about behaviour, keep this fork's
behaviour. That is what the fork is for.

## Generated files are never merged

95 files matching `*.g.dart` and `*.freezed.dart` are build artifacts that
upstream commits. They conflict on almost every merge and the conflict carries
no information. Never resolve one by hand and never take one side.

Take ours, then rebuild them:

```sh
flutter pub get
dart run build_runner build --delete-conflicting-outputs
```

`pubspec.lock` is the same — regenerate it with `flutter pub get` rather than
merging it. Codegen comes from freezed, riverpod_generator, chopper,
auto_route and swagger_dart_code_generator; `build.yaml` lists the inputs.

Regeneration is only needed when a codegen input actually moved — anything under
`lib/**.dart`, `swagger/*.json`, `pubspec.yaml` or `build.yaml`. A
translations-only merge does not need it.

## Checking the result

```sh
.fork/bin/verify --regen --build   # what a merge should pass before it is pushed
.fork/bin/verify                   # analyze + test only, while iterating
```

`--regen` rebuilds the generated files and is needed whenever a codegen input
moved. `--build` compiles the Linux app.

The toolchain is pinned by `.fvmrc` to Flutter 3.35.7 and lives at
`~/opt/flutter/bin` on this host — it is not on `PATH` by default.

**Never push a merge that has not passed `--build`.** Compiling is the only
check that proves the result works; analyze and test between them do not.
Budget for it — a clean release build takes upwards of fifteen minutes, and the
first one on a machine also downloads the mdk-sdk media backend.

`flutter analyze` on the fork point reports **No issues found**, so treat any
analyze output at all as a regression this merge introduced, not as pre-existing
noise. It takes about 90 seconds.

`flutter test` is a weak signal — the suite is 2 files. Passing tests are not
evidence the merge is good.

If something makes the build impossible to run rather than failing it, say that
plainly instead of reporting the merge as checked. A merge that was never
compiled and one that compiled cleanly must not read the same way.

## Translations

`lib/l10n/*.arb` is Weblate output and makes up most of upstream's commit
traffic. Conflicts there are mechanical — take both sides' new keys. It is never
worth stopping a merge over.
