# v560ns-builder — Mobile CI/CD

GitHub Actions pipelines that build the **v560ns** mobile app (iOS + Android) and ship it to Diawi / TestFlight / Google Drive, using a **reusable-workflows + composite-actions** architecture.

> The real app source is **not** in this repo. Every build clones it from a **private Git server** (accessed over SSH; the host and key come from repository secrets) into `real-code/`, then builds with **Fastlane**. This repo only holds the CI glue.

## How a build works

```
entry workflow (workflow_dispatch)
  ├─ notify-start            → POST "build started" to the team webhook
  └─ build job(s)            → runs on ubuntu (Android) or macOS (iOS)
       1. setup-ssh          → write deploy key + SSH config for the private Git host
       2. checkout-private   → shallow-clone the private app source into real-code/
       3. setup-env          → pull the env config from Passbolt for the chosen environment
       4. prepare-mobile     → Ruby + Fastlane + gems
       5. fastlane <lane>    → build, sign, upload
       └─ notify-on-failure  → POST failure alert if any step fails
```

Within an entry workflow the jobs have no `needs:` between them, so `notify-start` and the build job(s) start **in parallel**.

## Layout

```
.github/
├── actions/                     # composite actions (shared steps)
│   ├── setup-ssh/               # SSH config for the private Git host
│   ├── checkout-private-repo/   # shallow clone private source → real-code/
│   ├── setup-env/               # generate env file from Passbolt
│   ├── prepare-mobile-build/    # Ruby 3.4.4 + Fastlane + gem cache
│   └── notify-on-failure/       # webhook alert on failure
└── workflows/
    ├── build-aab.yml            # ENTRY: Android AAB (release)
    ├── internal-test-build.yml  # ENTRY: iOS AdHoc + Android APK (internal testing)
    ├── testflight-upload.yml    # ENTRY: iOS → TestFlight
    ├── reusable-build-aab.yml   # Android AAB
    ├── reusable-build-apk.yml   # Android APK
    ├── reusable-ios-adhoc.yml   # iOS AdHoc → Diawi
    ├── reusable-ios-testflight.yml  # iOS → TestFlight
    └── reusable-notify-start.yml    # build-start webhook
```

## Entry workflows (`workflow_dispatch`)

Triggered from the GitHub UI (**Actions** tab → Run workflow) or via the REST API (see [Triggering via API](#triggering-via-api)).

### `build-aab.yml` — Android Build AAB
Builds a release **.aab** and uploads it (Google Drive).

| Input | Default | Notes |
|-------|---------|-------|
| `branch` | `develop` | Branch of the **private source** to build |
| `environment` | `production` | `staging` or `production` |

Jobs: `notify-start` ‖ `build-aab`

### `internal-test-build.yml` — Mobile Build Internal Testing
Builds iOS AdHoc **and** Android APK for internal testers.

| Input | Default | Notes |
|-------|---------|-------|
| `branch` | `develop` | Branch of the private source |
| `environment` | `staging` | `staging` or `live` |
| `xcode_version` | `16.4` | `16.4` or `26.3` |

Jobs: `notify-start` ‖ `ios-adhoc` ‖ `build-apk`

### `testflight-upload.yml` — TestFlight Upload
Builds iOS and uploads to App Store Connect / TestFlight.

| Input | Default | Notes |
|-------|---------|-------|
| `branch` | `develop` | Branch of the private source |
| `environment` | `staging` | `staging` or `live` |
| `xcode_version` | `16.4` | `16.4` or `26.3` |

Jobs: `notify-start` ‖ `ios-testflight`

## Reusable workflows (`workflow_call`)

| Workflow | Runner | Timeout | Fastlane lane | Output |
|----------|--------|---------|---------------|--------|
| `reusable-build-aab.yml` | `ubuntu-latest` | – | `android aab` | .aab → Drive |
| `reusable-build-apk.yml` | `ubuntu-latest` | – | `android apk` | .apk → Diawi/Drive |
| `reusable-ios-adhoc.yml` | `macos-26` / `macos-15` | 60m | `ios adhoc` | .ipa → Diawi |
| `reusable-ios-testflight.yml` | `macos-26` / `macos-15` | 30m | `ios testflight_upload` | → TestFlight |
| `reusable-notify-start.yml` | `ubuntu-latest` | – | – | webhook POST |

> macOS runner is chosen from `xcode_version`: **`macos-26`** when `26.3`, otherwise **`macos-15`**.

## Composite actions

| Action | Inputs | What it does |
|--------|--------|--------------|
| `setup-ssh` | `ssh_private_key`, `private_host` | Writes `~/.ssh/id_rsa`, adds the private host to `known_hosts`, and writes an SSH config for it (host + key supplied via secrets) |
| `checkout-private-repo` | `branch`, `private_repo_full_name` | `git clone --depth 1 --single-branch` the private source into `real-code/` |
| `setup-env` | `environment`, `passbolt_env_file`, `passbolt_private_key` | Writes the Passbolt bot key + `.env.passbolt`, runs `npx tsx scripts/passbolt-env.ts <staging\|production>` to materialise the env file, then deletes the sensitive files |
| `prepare-mobile-build` | – | `ruby/setup-ruby` **3.4.4**, caches gems, `bundle install` in `real-code/fastlane` |
| `notify-on-failure` | `project_name`, `webhook_url`, `job_name`, `branch`, `run_url` | POSTs a failure alert to the team webhook (runs only on `if: failure()`) |

## Secrets & Variables

All are configured at **repository level** (repo → Settings → Secrets and variables → Actions). The GitHub *Environments* are **not** used to scope secrets — the `environment:` key on jobs is only a selector fed into Fastlane + Passbolt. Secret **values** are not stored in this repo; they come from Passbolt / the team vault.

### Secrets (17)

| Secret | Used by |
|--------|---------|
| `SSH_PRIVATE_KEY` | setup-ssh (private-repo deploy key) |
| `PRIVATE_HOST` | setup-ssh (private Git host) |
| `PRIVATE_REPO_FULL_NAME` | checkout-private-repo (SSH URL of the app source) |
| `PASSBOLT_ENV_FILE` | setup-env |
| `PASSBOLT_PRIVATE_KEY` | setup-env |
| `ANDROID_KEYSTORE_BASE64` | Android AAB/APK (base64 of the keystore) |
| `MYAPP_UPLOAD_KEY_ALIAS` | Android signing |
| `MYAPP_UPLOAD_KEY_PASSWORD` | Android signing |
| `MYAPP_UPLOAD_STORE_PASSWORD` | Android signing |
| `APP_STORE_CONNECT_API_KEY` | iOS TestFlight |
| `MATCH_GIT_URL` | iOS signing (fastlane match) |
| `MATCH_PASSWORD` | iOS signing (fastlane match) |
| `TEAM_ID` | iOS builds |
| `FASTLANE_USER` | all build lanes |
| `DIAWI_API_TOKEN` | AdHoc/APK upload |
| `DRIVER_KEY_JSON` | Google Drive upload |
| `NOTIFICATION_WEBHOOK_URL` | notify-start / notify-on-failure |

### Variables (3)

| Variable | Value | Description |
|----------|-------|-------------|
| `PROJECT_NAME` | `v560ns-mobile` | Shown in notifications |
| `GG_DRIVE_FOLDER_ID` | *(Drive folder id)* | Upload destination |
| `KEYCHAIN_NAME` | `fastlane.keychain` | macOS keychain for iOS signing |

## Caching

| Cache | Key | Scope |
|-------|-----|-------|
| Ruby gems | `gem-cache-<fastlane/Gemfile.lock>` | all builds (via `prepare-mobile-build`) |
| Gradle | `<os>-gradle-<gradle+props+yarn.lock>` (only `~/.gradle/caches`, `~/.gradle/wrapper`) | Android AAB/APK |
| Yarn | `yarn-<yarn.lock>` (`node_modules`, `~/.cache/yarn`) | iOS builds |
| CocoaPods downloads | `cocoapods-<Podfile.lock>` (`~/Library/Caches/CocoaPods`) | iOS |
| Xcode compilation cache (CAS) | `xcode-compcache-<xcode>-<run_id>` | iOS, **Xcode 26 only** |

> **Xcode CAS** (LLVM content-addressed) only engages on `macos-26` when the app's `gym` passes `COMPILATION_CACHE_ENABLE_CACHING=YES`. It replaces ccache (dead on Xcode 26.3 — 0 invocations on a real 21-min compile). Verified warm cache cut native compile ~21m → ~6m. The iOS reusables also emit a **"Compilation cache stats"** step so you can confirm the CAS actually populated.

## Triggering via API

Entry workflows are dispatched with a `workflow_dispatch` REST call. Because this repo is **public**, a classic PAT with only the **`public_repo`** scope is enough (no `repo`/full control needed).

```bash
curl -X POST \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/bssteam02/v560ns-builder/actions/workflows/build-aab.yml/dispatches \
  -d '{"ref":"main","inputs":{"branch":"develop","environment":"production"}}'
```

- `ref` = branch of **this builder repo** holding the workflow file → `main`.
- `inputs.branch` = branch of the **private app source** to build.
- Other workflow files: `internal-test-build.yml`, `testflight-upload.yml`.

## ⚠️ Project-specific hardcoded values

Baked into the workflows for this app; update them when cloning this builder for a different app:

| Where | Value |
|-------|-------|
| `reusable-build-aab.yml` / `reusable-build-apk.yml` | Android keystore filename |
