# CycleScope iOS – agent notes

This repo is nothing but the pipeline that signs the upstream Flutter app [SphaeroX/Serienmessung](https://github.com/SphaeroX/Serienmessung) with Boundfox's Apple Distribution certificate and uploads it to TestFlight. [README.md](README.md) documents operations (trigger, contract, secrets, recovery, certificate rotation). The contract in the README is binding: any change to the workflow file name, its trigger or its inputs breaks upstream's `main` builds and needs a coordinated change in SphaeroX/Serienmessung.

## Invariants

- `.github/workflows/ios.yml` is the only workflow; no Dependabot config, no CodeQL. The contract promises that the repo runs nothing but the Apple build and upload.
- The inputs are validated by regex in the first step and reach shell code only through `env`, never interpolated into `run:`. Interpolation would turn a crafted dispatch input into script injection in the job that holds the signing secrets.
- Only commits on upstream `main` are delivered: the step after the regex check asks the compare API whether `main` contains `sha` and rejects anything else, pull request heads included. A dispatch with a leaked trigger token therefore cannot get foreign code signed or reach the signing secrets.
- Secrets appear only as `env` of the steps that need them, which keeps them out of the Flutter setup and the config generation. Upstream code still runs inside the fastlane step (Xcode build phases, Dart build hooks) with the secrets in its env and the key files on disk; the one-job decision accepts that.
- The repo is public, so its logs are too: no `set -x`, no fastlane `--verbose`, no `SPACESHIP_DEBUG`, no uploaded signed artifacts or logs.
- Checkout order is this repo first, then upstream into `app/`, because a checkout into the workspace root deletes an existing `app/`.
- Both checkouts use `persist-credentials: false`, because upstream build code runs in the same job.
- Every action is pinned to a full commit SHA with the version as comment, and SHAs are looked up fresh before an update. Repository policy does not enforce this (see Decisions), so the workflow has to.
- The Xcode path is pinned explicitly and has to exist on the `xcode-27` image; uploads need Xcode 26 or newer. actionlint 1.7.12 does not know the `xcode-27` label yet and reports `runner-label`; that finding is expected and no reason to change the label.

## Decisions

Recorded so they are not proposed again.

- The job runs on `xcode-27` with Xcode 27.2 on purpose, although upstream's own CI builds on `macos-latest` with Xcode 26, so a toolchain difference shows up only here. The image is generally available since [2026-10-15](https://github.blog/changelog/2026-10-15-xcode-27-runner-image-generally-available); the "preview" badge in the actions/runner-images README is outdated.
- One job for build, signing and upload, no build/sign split: upstream code is trusted like own code.
- The failure report is a separate job only so `issues: write` never reaches the job with upstream code and secrets.
- `report-failure` fires on `failure` and on `cancelled` because a timeout ends as `cancelled`; a manual cancel of a running delivery therefore reports too. Pending runs replaced through the concurrency group create no jobs and report nothing.
- When App Store Connect already approved the version, the IPA is still built so signing and build regressions surface; only the upload is skipped, reported as notice and job summary, not as an issue.
- SHA pinning is not enforced as repository policy: `subosito/flutter-action` uses `actions/cache@v5` by tag internally, and the policy rejects nested tag references at job setup even when the step is skipped ([subosito/flutter-action#371](https://github.com/subosito/flutter-action/issues/371), the maintainer declined to pin). The workflow pins by SHA itself, and the allowlist contains `actions/cache` because it applies to nested actions too.
- Nothing is cached, neither the Flutter SDK nor the gems (no `bundler-cache`): upstream code runs in the same job and could poison a cache that is saved after it; the fresh installs cost about a minute each.
- `export_method` stays `"app-store"`: fastlane 2.242 rejects `"app-store-connect"`, while Xcode only logs a deprecation warning for the old value.
- Signing is manual with a pinned profile name: automatic signing on ephemeral runners mints development certificates until the account quota is exhausted.
- Only the `Runner` target is switched to manual signing: a project-wide `PROVISIONING_PROFILE_SPECIFIER` breaks the SwiftPM resource bundle targets of plugins.
- The version is `FLUTTER_BUILD_NAME` from `Generated.xcconfig`, which is what the archive carries, so the approval check and the upload cannot disagree; the build number is `github.run_number` of this workflow.
- Ruby 4.0: fastlane's own CI tests Ruby 4.0 on `macos-26` with Xcode 26.6 and on `xcode-27`.

## Updating

- When bumping fastlane, regenerate `Gemfile.lock` with `bundle lock` (never hand-edit it) and keep the platforms `arm64-darwin`, `x86_64-linux` and `aarch64-linux`.
- When changing the Xcode version, take the newest GA version listed in `images/macos/xcode-27-arm64-Readme.md` of [actions/runner-images](https://github.com/actions/runner-images).
