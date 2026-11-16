# CycleScope iOS

Signs the iOS app CycleScope, whose code lives in [SphaeroX/Serienmessung](https://github.com/SphaeroX/Serienmessung) and is owned by its author, and uploads it to TestFlight under the Boundfox Studios Apple account, so Apple certificates and keys never enter the upstream repo.

## How a delivery starts

Upstream's `.github/workflows/ios.yml` builds unsigned and runs the XCTests on pull requests and on `main`. After a green `main` build, its job `deliver` mints a token of the GitHub App `cyclescope-ios-trigger` (permission Actions: Read and write, installed only on this repo) and dispatches this repo's `ios.yml` with the pushed commit and upstream's Flutter version. Every upstream push to `main` is delivered, not only tags.

The `deliver` job runs only once upstream has the variable `CYCLESCOPE_IOS_TRIGGER_CLIENT_ID`; set upstream's secret `CYCLESCOPE_IOS_TRIGGER_PRIVATE_KEY` before that variable.

## Contract

Upstream relies on this contract; changing it breaks upstream's `main` builds.

- The workflow file is exactly `.github/workflows/ios.yml` on the default branch.
- Its only trigger is `workflow_dispatch` with the inputs `sha` (40 lowercase hex characters, a commit on SphaeroX/Serienmessung's `main` branch; pull request heads are rejected) and `flutter-version` (`X.Y.Z`, currently `3.47.9`). Both are validated by regex before use; the check that `sha` is on upstream `main` follows the regex.
- The repo runs nothing but this Apple build and upload: no other workflows, no Dependabot, no CodeQL.

## What a run does

1. Selects Xcode 27.2 explicitly on GitHub's preview runner image `xcode-27` (uploads need Xcode 26 or newer).
2. Checks that `sha` is on upstream `main`, then checks out this repo, then upstream at `sha` into `app/`.
3. Installs Flutter in the input version and Ruby 4.0 with the locked gems; nothing is cached.
4. Generates the Xcode config: the version is the `FLUTTER_BUILD_NAME` Flutter derives from upstream's `pubspec.yaml` (the version without the `+build` part), the build number is this workflow's run number.
5. Runs the fastlane lane `ios release`: it imports the distribution certificate into a temporary keychain, fetches or creates the provisioning profile "CycleScope App Store" via the App Store Connect API key, switches only the `Runner` target to manual signing, archives a build for iPhone and iPad and uploads it to TestFlight without waiting for processing.

Signed artifacts are never uploaded as workflow artifacts, because the repo is public.

## Version already approved

If App Store Connect has already approved or released the `pubspec.yaml` version or a higher one (states `ACCEPTED`, `PENDING_APPLE_RELEASE`, `PENDING_DEVELOPER_RELEASE`, `PROCESSING_FOR_DISTRIBUTION`, `READY_FOR_DISTRIBUTION`, `REPLACED_WITH_NEW_VERSION`), the run still builds but skips the upload and reports a notice plus a job summary instead of failing. Fix: raise the version in upstream's `pubspec.yaml`.

Known gap: a version that was approved and later rejected by the developer passes the check; Apple then rejects the upload and the run fails.

## Failures

A run whose delivery job fails, times out or is cancelled while running opens the issue "iOS delivery failed" in this repo, or comments on it if it is already open. Close the issue once delivery works again. Apart from that issue nobody is notified, because the app token dispatches the runs.

## Secrets

Organization secrets visible to this public repo, each passed only to the step that needs it:

- `APPLE_CERTIFICATE_DATA`: Apple Distribution certificate as base64 `.p12`
- `APPLE_CERTIFICATE_PASSPHRASE`
- `APPLE_APP_STORE_CONNECT_KEY`: App Store Connect API key as base64 `.p8`
- `APPLE_APP_STORE_CONNECT_KEY_ID`
- `APPLE_APP_STORE_CONNECT_ISSUER_ID`
- `APPLE_TEAM_ID`

The API key needs the App Manager role with access to certificates, identifiers and profiles, because the profile is created on first use.

## Recovery

Start a run by hand with the upstream commit (it must be on upstream `main`) and the Flutter version from `FLUTTER_VERSION` in upstream's `.github/workflows/ios.yml`:

```sh
gh workflow run ios.yml -R BoundfoxStudios/cyclescope-ios -f sha=<commit> -f flutter-version=<version>
```

Each dispatched run gets a new build number, so delivering the same commit again is fine. "Re-run jobs" keeps the run number and with it the build number: if the failed run's upload already reached Apple, dispatch a new run instead. Runs share one concurrency group without cancelling the running one; GitHub keeps only the newest pending run, so a burst of upstream pushes delivers the latest commit.

## Certificate rotation

Update `APPLE_CERTIFICATE_DATA` and `APPLE_CERTIFICATE_PASSPHRASE` at the organization, then delete the profile "CycleScope App Store" in the Apple Developer portal; the next run recreates it. Without the deletion, the next run finds the old profile unusable, refuses to create a renamed one (`fail_on_name_taken`) and fails with "The name 'CycleScope App Store' is already taken". The same applies to every app that signs with the shared distribution certificate.

## Repository settings

Applied via the REST API:

- Actions restricted to `actions/checkout`, `subosito/flutter-action`, `ruby/setup-ruby` and `actions/cache` (nested inside `subosito/flutter-action`).
- Default workflow token read-only; Actions may not create or approve pull requests.
- Dependabot security and version updates off.
- CodeQL default setup off.

SHA pinning is not enforced by policy, see [AGENTS.md](AGENTS.md).

## One-time Apple setup

Prerequisites outside this repo:

- App ID `com.boundfoxstudios.apps.cyclescope` registered.
- App record in App Store Connect.
- Internal TestFlight group.
- App Store metadata and screenshots, maintained by hand in App Store Connect.
