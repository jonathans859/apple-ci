# apple-ci

Shared GitHub Actions workflows for every Apple app on this account:
`glinet-access`, `RemKeys`, `locatification`, `NVRS`, `RemSoundApple`,
`linkbumms`.

It exists because those six repos each carried their own near-identical
TestFlight pipeline, and they had drifted — different secret names, different
runner images, different `actions/checkout` versions, three different answers to
the code-signing problem, and four different build-number schemes. A single
change (migrating to one shared signing certificate) took six commits across six
repos. Under this repo it is one commit and a tag.

## What's here

| Workflow | Runs on | Does |
|---|---|---|
| `testflight.yml` | macOS | on every push: archive → export → upload to TestFlight (internal testers) |
| `release.yml` | ubuntu | on a published release: promote that commit's existing build to external testers |

TestFlight only. RemKeys' Developer ID + notarization lane and its Windows agent
stay in that repo — they're genuinely one-off, and the duplication was never
there.

## Using it

```yaml
name: TestFlight
on:
  push:
    branches: [main]
    paths-ignore: ['**.md', '.claude/**', 'LICENSE', '.gitignore']
  release:
    types: [published]

concurrency:
  group: testflight
  cancel-in-progress: false

jobs:
  ios:
    uses: jonathans859/apple-ci/.github/workflows/testflight.yml@v1
    secrets: inherit
    with:
      platform: ios
      project: GLiNetAccess.xcodeproj
      scheme: GLiNetAccess
      app-id: "6801759784"
      xcodegen-spec: project.yml
      artifact-name: GLiNetAccess-ipa

```

A release is a **separate workflow**, because it must not rebuild:

```yaml
name: Release
on:
  release:
    types: [published]

permissions:
  contents: write   # attach the signed build to the release
  actions: read     # find and download the build run

jobs:
  external:
    uses: jonathans859/apple-ci/.github/workflows/release.yml@v1
    secrets: inherit
    with:
      app-id: "6801759784"
      build-workflow: build.yml
      artifact-name: GLiNetAccess-ipa
      platforms: IOS
```

`secrets: inherit` works because every repo on the account uses the same names:

```
ASC_KEY_ID  ASC_ISSUER_ID  ASC_KEY_P8  APPLE_TEAM_ID
APPLE_DEV_CERT_P12  APPLE_DEV_CERT_P12_PASSWORD
```

This repo must be reachable from the others: Settings → Actions → General →
Access → *Accessible from repositories owned by jonathans859*.

## Reference a tag, never `@main`

A bad change here breaks every app at once. That is the real cost of
consolidating, and the mitigation is boring: consumers pin `@v1`, and bumping
them is deliberate and one at a time.

`glinet-access` deliberately tracks `@main` as the canary — this repo has no app
of its own, so a change is only ever exercised when a real repo runs it.

To release a change: merge to `main`, let the canary build, then move the tag.

```
git tag -f v1 && git push -f origin v1
```

**A re-run will not pick up a fix here.** GitHub resolves a reusable workflow
reference when the run is *created* and pins it for that run's lifetime, so
`gh run rerun` replays the old version of this repo and fails identically. After
changing anything here, trigger a **new** run — `gh workflow run <file> --ref main`,
or push a commit. (Same family as release-triggered workflows running the YAML
from the tagged commit.)

## Conventions this encodes

Decisions that were inconsistent across the six repos and are now settled here:

- **Signing.** Apple cloud signing for distribution; ONE shared Apple Development
  certificate imported from `APPLE_DEV_CERT_P12` for the archive step. It cannot
  be per-repo — Apple returns 409 when minting a second Development certificate
  while one is current, because the certificate identifies the team, not an app.
  Never cache a keychain: `actions/cache` entries are evicted after 7 days and a
  miss mints silently.
- **Verification.** A green build does not prove signing works. A revoked `.p12`
  still imports cleanly and still passes `security find-identity`. Check with
  `asc certificates list` after a build and confirm no new certificate appeared.
- **Promote, don't rebuild.** A release ships the binary the push already built
  and internal testers already ran — not a fresh archive nobody has seen. It also
  keeps a 10× macOS runner out of App Store Connect's processing wait.
- **Build numbers.** `<commit count>.0.<run attempt>` — monotonic on main and
  unique across re-runs. The release workflow does not recompute it (the attempt
  digit is unguessable from a tag); it asks App Store Connect for the newest build
  of the tagged marketing version instead.
- **Versions must reach the bundle.** `manageAppVersionAndBuildNumber` is off, and
  a caller using XcodeGen must map the settings into its plist:
  `CFBundleShortVersionString: $(MARKETING_VERSION)` and
  `CFBundleVersion: $(CURRENT_PROJECT_VERSION)` in `info.properties`. Without
  them XcodeGen emits a literal `1.0`/`1` and Xcode renumbers at export.
- **What to Test** is every commit subject in the push, not just the head one —
  the head commit is routinely a docs commit riding behind the real change. On a
  release it is the release body, and an empty body fails the job. TestFlight
  renders raw text, not markdown.
- **Tags.** `vX.Y.Z` raises the marketing version (first external build waits on
  Beta App Review); `vX.Y.Z-bN` re-releases the same version, usually review-free.
  The tag is validated against the project at that commit.
- **Runners.** Uploads need `macos-26` (App Store Connect enforces an SDK floor on
  uploads only); a caller's own tests can stay on the cheaper, stabler image.
  Every signing job carries `timeout-minutes`, because a signing step that raises
  a SecurityAgent GUI prompt hangs forever instead of failing.
- **macOS archives are ad-hoc signed.** Automatic signing wants a Mac development
  provisioning profile, and Apple will not issue one to a team with no registered
  Macs. `CODE_SIGNING_ALLOWED=NO` looks like the fix but strips the entitlements
  and the upload is rejected with 90296; an ad-hoc signature keeps them.

## asc

Uploads and distribution go through [`asc`](https://github.com/rorkai/App-Store-Connect-CLI)
rather than fastlane, which keeps Ruby off the runners entirely — no `Gemfile`,
no `Gemfile.lock` that a Windows machine with no Ruby cannot produce, no
`bundle install` on 10×-billed minutes.

The version is **pinned** (`asc-version`, currently `4.10.0`) and the binary's
checksum is verified before it is handed the account's Admin API key. asc is
young and ships every few days, with 5.0.0 deprecations already announced — never
track `latest`. Bump it here, let the canary build, then move the tag.

Known gap: `asc builds add-groups` has no explicit "notify testers" flag, where
fastlane's `pilot` had `--notify_external_testers`. App Store Connect's own group
notification setting governs it instead.
