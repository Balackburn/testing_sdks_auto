<div align="center">

# 🛠 Sydka

**Automated builder and publisher of patched iOS SDKs**  
*Xcode + IPSW dyld shared cache → ready-to-use `.sdk` for Theos and jailbreak toolchains*

[![Build Patched iOS SDK](https://github.com/Balackburn/Sydka/actions/workflows/build-ios-sdk.yml/badge.svg)](https://github.com/Balackburn/Sydka/actions/workflows/build-ios-sdk.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](#)

</div>

---

## What is this?

iOS tweak development with [Theos](https://theos.dev) requires a patched iOS SDK — one that includes `.tbd` stub files generated from the live `dyld_shared_cache` extracted from a real device firmware. Apple doesn't ship these stubs in Xcode.

**Sydka automates the entire pipeline:**

1. Keeps a live map of every Xcode version → iOS SDK version (`sdk_map.json`)
2. Downloads the correct Xcode for the target iOS version
3. Downloads the matching IPSW directly from Apple's CDN
4. Extracts the `dyld_shared_cache_arm64e` from the IPSW
5. Builds [`tbd`](https://github.com/leptos-null/tbd) from source and uses it to generate `.tbd` stubs
6. Patches the Xcode SDK with those stubs using the [Theos `create_patched_sdk.sh`](https://github.com/theos/sdks/blob/master/tools/create_patched_sdk.sh) tool
7. Publishes the result as a GitHub Release asset (`iPhoneOS{version}.sdk.tar.xz`)

The daily schedule keeps the map fresh and automatically builds any new SDK that Apple ships with a new Xcode release.

---

## Usage

### Download a pre-built SDK

Go to [**Releases**](../../releases) and download the archive for the iOS version you need:

```
iPhoneOS18.2.sdk.tar.xz
```

Extract it and drop the `.sdk` folder into your Theos SDK directory:

```bash
tar -xJf iPhoneOS18.2.sdk.tar.xz -C $THEOS/sdks/
```

### Trigger a build manually

1. Go to **Actions → Build Patched iOS SDK**
2. Click **Run workflow**
3. Enter the iOS version (e.g. `18.2`) or tick **Build all**

### Supported iOS versions

All versions with a confirmed Xcode mapping in `sdk_map.json` — currently **iOS 9.3 through iOS 26.x**.  
Run `python3 map_sdks.py --skip-xcodes` to regenerate the map locally.

---

## Files

| File | Purpose |
|------|---------|
| [`build_sdk.sh`](build_sdk.sh) | Core build script. Downloads Xcode + IPSW, extracts the dyld cache, builds `tbd`, patches the SDK. Run locally with `./build_sdk.sh --ios 18.2`. |
| [`map_sdks.py`](map_sdks.py) | Scrapes 8 sources to build the Xcode → iOS SDK version map and writes `sdk_map.json`. Exits `1` if the map changed (triggers a commit in CI). |
| [`sdk_map.json`](sdk_map.json) | Flat JSON mapping every known Xcode version to the iOS SDK it ships. Used by `build_sdk.sh` to auto-select the right Xcode for a given iOS target. |
| [`.github/workflows/build-ios-sdk.yml`](.github/workflows/build-ios-sdk.yml) | Three-job GitHub Actions workflow: **fetch** (refresh map) → **prepare** (build matrix) → **build** (parallel macOS runners). |

---

## How it works

```
┌─────────────────────────────────────────────────────────────────┐
│                    build-ios-sdk.yml                            │
│                                                                 │
│  ① fetch (Ubuntu)          ② prepare (Ubuntu)                  │
│  ┌──────────────────┐      ┌──────────────────┐                │
│  │  map_sdks.py     │─────▶│  Read sdk_map.json│               │
│  │  (8 sources)     │      │  Emit version     │               │
│  │  sdk_map.json    │      │  matrix as JSON   │               │
│  └──────────────────┘      └────────┬─────────┘               │
│                                     │                           │
│             ③ build (macOS 15 × N parallel runners)            │
│             ┌───────────────────────▼──────────────────────┐   │
│             │  build_sdk.sh --ios {version}                 │   │
│             │                                               │   │
│             │  1. xcodes  → download Xcode                 │   │
│             │  2. ipsw    → download + extract IPSW        │   │
│             │  3. ipsw extract --dyld → dyld_shared_cache  │   │
│             │  4. build tbd from source (leptos-null/tbd)  │   │
│             │  5. create_patched_sdk.sh (theos/sdks)       │   │
│             │  6. tar.xz → gh release upload               │   │
│             └───────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

Each build job starts with a release-asset check — if `iPhoneOS{version}.sdk.tar.xz` already exists on the release, the job exits immediately with success. This makes the daily schedule safe to run without rebuilding anything unnecessarily.

---

## Running locally

**Requirements:** macOS, Homebrew. The script installs `xcodes`, `ipsw`, `aria2`, and `unxip` automatically.

```bash
# Build a single SDK
./build_sdk.sh --ios 18.2

# Build all known SDKs (skips any that already exist locally)
./build_sdk.sh --all

# Refresh sdk_map.json (requires: pip install requests beautifulsoup4)
python3 map_sdks.py

# Inspect source-level detail / conflicts
python3 map_sdks.py --detailed --table
```

---

## sdk_map.json

`sdk_map.json` is a flat JSON object mapping Xcode version strings to iOS SDK version strings:

```json
{
  "7.3.1": "9.3",
  "12.4":  "12.2",
  "16.2":  "18.2",
  "26.3":  "26.2",
  ...
}
```

It is generated by `map_sdks.py`, which cross-references **8 independent sources** and resolves conflicts by source priority:

| Priority | Source |
|----------|--------|
| 1 | Local `xcodebuild -showsdks` (ground truth when available) |
| 2 | Apple Developer Documentation JSON API |
| 3 | Apple official support page (HTML table) |
| 4 | xcodereleases.com community JSON |
| 5 | Apple library archive — Xcode 8–9 release notes |
| 6 | Apple library archive — Xcode 4, 6, 7 chapter pages |
| 7 | Wikipedia — History of Xcode |
| 8 | Wikipedia — Xcode main article |

The map is committed back to the repo whenever new entries are discovered, keeping it always up to date.

---

## Secrets

| Secret | Purpose |
|--------|---------|
| `APPLE_ID` | Your Apple ID email address (`FASTLANE_USER`) |
| `FASTLANE` | A valid `FASTLANE_SESSION` cookie string (see setup below) |

Sydka uses **Fastlane `spaceauth`** to handle Apple's 2FA requirement. Rather than re-authenticating interactively on every runner, you generate a session token once on your local machine and store it as a secret. The token is valid for roughly 30 days.

### Generating your FASTLANE_SESSION

**Step 1 — Install Ruby and Fastlane** (macOS)

```bash
brew install ruby
export PATH="/opt/homebrew/opt/ruby/bin:$PATH"
gem install fastlane --no-document
export PATH="$HOME/.gem/ruby/$(ruby -e 'print RUBY_VERSION[/\d+\.\d+/]')/bin:$PATH"
fastlane --version
```

**Step 2 — Generate the session token**

```bash
fastlane spaceauth -u your_apple_id@example.com
# Follow the 2FA prompt, then copy the full FASTLANE_SESSION output string
```

**Step 3 — Add to GitHub**

Go to **Settings → Secrets and variables → Actions** and create:

- `APPLE_ID` → your Apple ID email
- `FASTLANE` → the full `FASTLANE_SESSION` string from step 2

> ⚠️ **Session expiry:** Fastlane sessions last between 1 and ~30 days. If builds start failing with an authentication error, re-run `fastlane spaceauth` and update the `FASTLANE` secret.

---

## Disk space

Each build peaks at ~14–20 GB (Xcode + IPSW + intermediate files). GitHub-hosted `macos-15` runners provide ~14 GB free. For older Xcode versions whose download alone exceeds that, use a self-hosted runner with more storage.