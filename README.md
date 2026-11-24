# CSPB Mobile CI/CD Pipeline

Automated build, verification, packaging, and release pipeline for **Counter-Strike Point Blank (CSPB)** on Android using the Xash3D engine.

This repository compiles the native game client & server libraries from [JangOetama/CSPB-ANDROID-OPEN-SOURCE](https://github.com/JangOetama/CSPB-ANDROID-OPEN-SOURCE) using Android NDK, integrates them into the [FWGS/xash3d-android-project](https://github.com/FWGS/xash3d-android-project) launcher APK, signs the APK with full v1/v2/v3 signatures, validates 4 verification gates, and publishes releases.

---

## 🚀 How to Trigger Builds

1. **Push to `main` or `master`**: Automatically runs native library compilation, launcher APK build, signing, and verification.
2. **Pull Requests**: Validates changes across branches and forks without consuming or leaking repository secrets.
3. **Manual Trigger (`workflow_dispatch`)**:
   - Go to **Actions** -> **Build CSPB Mobile** -> **Run workflow**.
   - Choose variant: `Release` (default) or `Debug`.
4. **Publishing a Release**:
   - Create and push a version tag:
     ```bash
     git tag v1.0.0
     git push origin v1.0.0
     ```
   - Tags containing `-rc` or `-beta` (e.g. `v1.0.0-rc1`) are automatically tagged as **Pre-release** on GitHub Releases.

---

## 🔐 Keystore & Signing Secrets Setup

If secrets are not provided, the pipeline automatically generates a temporary throwaway self-signed keystore to ensure the APK is installable and testable. To use your own production release key:

1. **Generate a keystore** (if you don't already have one):
   ```bash
   keytool -genkeypair -v \
     -keystore cspb-release.keystore \
     -alias cspb-key \
     -keyalg RSA \
     -keysize 2048 \
     -validity 10000 \
     -storepass "YourStrongKeystorePassword" \
     -keypass "YourStrongKeyPassword" \
     -dname "CN=CSPB Mobile, OU=DevOps, O=Community, C=ID"
   ```

2. **Encode the keystore file to Base64**:
   ```bash
   base64 -w0 cspb-release.keystore > keystore_base64.txt
   ```

3. **Configure GitHub Secrets**:
   Go to **Settings** -> **Secrets and variables** -> **Actions** -> **New repository secret**:
   - `KEYSTORE_BASE64`: Paste the entire content of `keystore_base64.txt`.
   - `KEYSTORE_PASSWORD`: Keystore password (e.g. `YourStrongKeystorePassword`).
   - `KEY_ALIAS`: Keystore alias (e.g. `cspb-key`).
   - `KEY_PASSWORD`: Key password (e.g. `YourStrongKeyPassword`).

---

## 📦 Deliverables Produced

| Artifact | Content |
| :--- | :--- |
| `CSPB-Mobile-Launcher-<build_type>-<shortsha>.apk` | Installable APK containing `libclient.so`, `libserver.so`, and engine binaries. |
| `CSPB-Mobile-Launcher.apk` | Fixed-name copy of the launcher APK. |
| `CSPB-Native-Libs.zip` | Standalone native libraries organized by ABI (`<abi>/cl_dlls/libclient.so`, `<abi>/dlls/libserver.so`) and sample `autoexec.cfg`. |
| `SHA256SUMS.txt` | Cryptographic checksums for all deliverables. |

> **Note**: Game assets (CS 1.6 / CSPB models, maps, sounds, sprites) are not included in the APK due to distribution licenses and must be placed manually in your device game folder (e.g., `/sdcard/xash/cstrike` or `/sdcard/xash/cspb`).

---

## 🛠️ Optional Patch Workflow

To apply patches before compilation without maintaining long-term forks:

1. **CSPB Core Patches**: Place `.patch` files in `.github/patches/cspb/`.
   ```bash
   cd path/to/cspb-core
   git diff > 0001-fix-weapon-recoil.patch
   mv 0001-fix-weapon-recoil.patch .github/patches/cspb/
   ```
2. **Launcher Patches**: Place `.patch` files in `.github/patches/launcher/`.
   ```bash
   cd path/to/launcher-apk
   git diff > 0001-custom-launcher-ui.patch
   mv 0001-custom-launcher-ui.patch .github/patches/launcher/
   ```
Patches are validated with `git apply --check` and applied in alphabetical order.

---

## 🔍 Troubleshooting Guide

| Issue | Cause | Indicator in Log | Fix |
| :--- | :--- | :--- | :--- |
| **NDK Version Mismatch** | Engine makefile requires features not present in NDK version. | `ndk-build: command not found` or compile syntax errors. | Set `NDK_VERSION` in workflow `env` to tested r21e (`21.4.7075529`) or r19c (`19.2.5345600`). |
| **GCC 4.9 Removal** | Deprecated `NDK_TOOLCHAIN_VERSION=4.9` in `Application.mk`. | `Toolchain 4.9 is not installed`. | Workflow automatically comments this out; ensure Clang default is active. |
| **AGP / JDK Mismatch** | JDK 17 requires Gradle >= 7.3 and AGP >= 7.0. | `Unsupported class file major version 61` or `Gradle version X is incompatible with Java 17`. | Workflow automatically bumps wrapper to 7.6.4 and AGP if an outdated project is detected. |
| **Missing Submodule / LFS** | Required dependencies missing from checkout. | `CMake Error: Could not find .../xash3d-fwgs/CMakeLists.txt` or corrupt binary files. | Verified: `actions/checkout@v4` uses `submodules: recursive` and `lfs: true`. |
| **ABI-Specific Failure (arm64)** | 32-bit legacy pointer math or hardcoded types in CSPB core. | `cast to pointer from integer of different size` in `arm64-v8a`. | `arm64-v8a` is flagged `experimental: true` (`continue-on-error: true`). armeabi-v7a remains mandatory. |
| **Missing `libc++_shared.so`** | Shared STL configured (`APP_STL=c++_shared`) but library not bundled. | `readelf: NEEDED libc++_shared.so` and runtime `UnsatisfiedLinkError`. | Workflow detects `libc++_shared.so` in `readelf -d` and copies it from NDK sysroot. |
| **`extractNativeLibs` Crash** | Android 6.0+ (API 23+) does not extract `.so` from APK, breaking `dlopen` by path. | Engine log: `Failed to load client library: dlopen failed`. | Workflow enforces `android:extractNativeLibs="true"` and `useLegacyPackaging = true`. |
| **Signing / Alignment Error** | `apksigner` failure or unaligned zip entries. | `DOES NOT VERIFY` or `zipalign verification failed`. | Workflow runs `zipalign -p 4` prior to `apksigner sign` with full v1+v2+v3 signatures. |

---

## 📋 Repository Layout Assumptions

1. **CSPB Core Repository**: Contains `Android.mk` and `Application.mk` (preferably under `jni/`). The workflow discovers them dynamically, strips obsolete GCC/m32 flags, and passes relative paths to `ndk-build -C "$CSPB_DIR"`.
2. **Xash3D Launcher Repository**: A standard Android Gradle project with `gradlew`, `app/build.gradle` (or `app/build.gradle.kts`), and `app/src/main/AndroidManifest.xml`.
3. **Module Names**: Produces `libclient.so` and `libserver.so` (or hardfp variants) under `libs/<abi>/`.

---

## 🛡️ Actionlint Validation

This workflow is 100% compliant with `actionlint` and passes all static analysis checks.
