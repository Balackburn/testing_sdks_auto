<div align="center">

# 🛠 Sydka

**Automated builder and publisher of patched iOS SDKs for Theos**

[![Build Patched iOS SDK](https://github.com/Balackburn/Sydka/actions/workflows/build-ios-sdk.yml/badge.svg)](https://github.com/Balackburn/Sydka/actions/workflows/build-ios-sdk.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](#)

</div>


Theos requires a patched iOS SDK containing `.tbd` stubs generated from a real device's `dyld_shared_cache`. Apple doesn't ship these. Sydka automates the entire pipeline: downloads the correct Xcode and IPSW, extracts the cache, generates stubs with [`tbd`](https://github.com/leptos-null/tbd), and publishes the result as a GitHub Release.

Runs daily. Covers **iOS 9.3 → iOS 26.x**.


## Download

Grab a pre-built SDK from [**Releases**](../../releases):

```bash
tar -xJf iPhoneOS18.2.sdk.tar.xz -C $THEOS/sdks/
```

## Build manually

**Via GitHub Actions:** Actions → Build Patched iOS SDK → Run workflow → enter a version or tick *Build all*.

**Locally** (macOS + Homebrew required):

```bash
./build_sdk.sh --ios 18.2   # single version
./build_sdk.sh --all        # everything (skips existing)
```

## Secrets (required for Xcode download)

Sydka authenticates with Apple via Fastlane `spaceauth`. Sessions last between 1~30 days.

```bash
# Install Fastlane, then:
fastlane spaceauth -u your@apple.com
```

Add to **Settings → Secrets → Actions**:

| Secret | Value |
|--------|-------|
| `APPLE_ID` | Your Apple ID email |
| `FASTLANE` | The full `FASTLANE_SESSION` string from above |

Re-run `spaceauth` and update the secret whenever builds fail with an auth error.


## Files

| File | Purpose |
|------|---------|
| `build_sdk.sh` | Core build script |
| `map_sdks.py` | Generates `sdk_map.json` from 8 sources |
| `sdk_map.json` | Maps every Xcode version to its bundled iOS SDK |
| `.github/workflows/build-ios-sdk.yml` | CI pipeline |
