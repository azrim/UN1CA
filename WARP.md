# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project overview

UN1CA is a bash-based build system for generating custom Samsung Galaxy firmware (flashable zips / images) from stock Odin packages. The tooling automates:

- Building and caching required low-level tools (android-tools, apktool, erofs-utils, img2sdat, samloader, signapk) into `out/tools`
- Downloading stock firmware from Samsung FUS for a configured source/target device
- Extracting, de-sparsifying, and mounting firmware partitions to produce clean filesystem trees and metadata
- Merging "source" and "target" firmware trees into a unified work directory per device
- Applying UN1CA patches and mods, rebuilding APKs/JARs, and generating partition images
- Assembling and signing a dynamic-partitions-based OTA zip (or just raw images)

All of this is orchestrated via small bash entrypoints in `scripts/` plus shared helpers in `scripts/utils/`.

## Core workflows and commands

### 1. Set up the build environment

From the repo root, source `buildenv.sh` for a specific device target (one subdirectory under `target/`, e.g. `r8q`, `a52sxq`, `a73xq`, `m52xq`):

```bash
source ./buildenv.sh <target_codename>
```

Behavior:

- Locates `SRC_DIR` (tree root) and exports key paths: `OUT_DIR=out`, `TMP_DIR`, `FW_DIR`, `TOOLS_DIR`, `WORK_DIR`, `APKTOOL_DIR`, etc.
- Enumerates available targets from `target/*` and will prompt interactively if no target argument is given.
- Runs `scripts/internal/gen_config_file.sh` to generate `out/config.sh` by combining:
  - Global ROM version logic from `unica/configs/version.sh`
  - Per-device settings from `target/<codename>/config.sh`
  - Shared SSI config from `unica/configs/{qssi,essi,mssi}.sh`
- Exports all variables found in `out/config.sh` (firmware identifiers, partition sizes, feature flags, etc.).
- Defines a convenience alias:

  ```bash
  unica <cmd>  # equivalent to: scripts/<cmd>.sh with logging & timestamped logs under out/target/<codename>
  ```

Always source `buildenv.sh` in a fresh shell before running any of the `scripts/*.sh` entrypoints.

### 2. Build external toolchain (one-time per environment)

UN1CA relies on several external tools that are vendored under `external/` and installed into `out/tools/bin`.

To (re)build them, after sourcing `buildenv.sh` run:

```bash
./scripts/build_dependencies.sh
```

This script:

- Ensures `out/tools/bin` contains all expected binaries and helper scripts (android-tools, apktool, erofs-utils, img2sdat, patched samloader, signapk wrapper and jars).
- Internally runs `cmake`, `make`, `gradle`, and `pip` in the corresponding `external/*` projects.

If you need to debug dependencies directly, the lower-level entry is:

```bash
./external/make.sh          # same logic used from build_utils.sh
./external/make.sh --check-tools  # exit 0 if everything already built
```

> CI reference: see `.github/workflows/build.yml` ("Build dependencies" step) for the authoritative package list (apt installs, kernel modules, etc.) when replicating the GitHub Actions environment.

### 3. Download stock firmwares

Firmware selection and credentials are configured indirectly via `out/config.sh`, generated when you source `buildenv.sh`. Relevant variables include:

- `SOURCE_FIRMWARE`, `TARGET_FIRMWARE` — base and target firmware strings in the form `MODEL/CSC/IMEI_OR_SN`
- `SOURCE_EXTRA_FIRMWARES`, `TARGET_EXTRA_FIRMWARES` — optional colon-delimited lists of extra firmwares to fetch

To download all required Odin packages for the configured device(s):

```bash
source ./buildenv.sh <target_codename>
./scripts/download_fw.sh [--force] [--ignore-source] [--ignore-target] [EXTRA_FIRMWARE ...]
```

Key details:

- Uses patched `samloader` from `out/tools/venv` to pull official Odin zips from FUS.
- Stores zips and `.md5` tars under `out/odin/<MODEL>_<CSC>/`.
- Validates `.md5` suffixes by recomputing MD5 over the payload.
- Tracks the last downloaded firmware in `.downloaded` and will skip or warn unless `--force` is used.

### 4. Extract and normalize firmwares

After the Odin zips are present, extract them into mountable images and filesystem trees:

```bash
source ./buildenv.sh <target_codename>
./scripts/extract_fw.sh [--force] [--ignore-source] [--ignore-target] [EXTRA_FIRMWARE ...]
```

This script (via `scripts/utils/firmware_utils.sh`):

- For each requested firmware (source, target, and extras):
  - Locates `BL_*.md5` and `AP_*.md5` in `out/odin/<MODEL>_<CSC>/`.
  - Extracts partition images (either raw, `.ext4`, `.lz4`, or `super.img`) into `out/fw/<MODEL>_<CSC>/`.
  - Converts sparse images to raw via `simg2img` when necessary.
  - For dynamic partitions (`super.img`), uses `lpdump`/`lpunpack` and, on Virtual A/B, takes `_a` slots.
  - Mounts images via `mount` or `fuse.erofs` (requires root/sudo) into a temp directory and copies contents into named partitions (`system`, `product`, `vendor`, `system_ext`, `odm`, `*_dlkm`, etc.).
  - Generates sorted `fs_config-<partition>` and `file_context-<partition>` files by walking the mounted tree.
  - Extracts and records kernel images (`boot`, `init_boot`, `vendor_boot`, `dtbo`) plus AVB metadata and partition sizes.
  - Extracts `vbmeta.img` and builds a patched copy `vbmeta_patched.img` when present.
- Tracks the extracted firmware version in `.extracted` and will refuse to overwrite newer extractions unless `--force` is specified.

### 5. Build the ROM for a device

With dependencies, downloads, and extraction done, the main entrypoint to assemble the ROM is:

```bash
source ./buildenv.sh <target_codename>
./scripts/make_rom.sh [--force] [--no-rom-zip]
```

Semantics:

- Without `--force`, `make_rom.sh` computes a hash over `unica/` and `target/<codename>/` sources and compares it to `work_dir/.completed` to decide whether a rebuild is needed.
- With `--force`, it always rebuilds, ignoring the cached work-dir hash.
- Stages:
  1. Ensures both source and target firmware trees exist in `out/fw/<MODEL>_<CSC>/`; if not, runs `download_fw.sh` and `extract_fw.sh` automatically.
  2. Invokes `scripts/internal/create_work_dir.sh` to assemble the per-device work directory under `out/target/<codename>/work_dir`:
     - Copies and merges partitions (`system`, `product`, `vendor`, `system_ext`, `odm`, `*_dlkm`) from the configured source and target firmware trees, respecting `TARGET_OS_BUILD_SYSTEM_EXT_PARTITION` (standalone vs. `system/system_ext`).
     - Normalizes SELinux `file_context` / `fs_config` files, symlinks, and metadata.
     - Copies kernel images from the target firmware and optionally includes `vbmeta_patched.img` depending on `TARGET_INCLUDE_PATCHED_VBMETA`.
  3. Applies device-specific patches from `target/<codename>/patches` and ROM-wide patches/mods from `unica/patches` and `unica/mods` via `scripts/internal/apply_modules.sh` and helpers in `scripts/utils/module_utils.sh`.
  4. If any APK/JARs were decoded to `$APKTOOL_DIR`, rebuilds them in parallel via `scripts/apktool.sh`.
  5. Persists a new work-dir hash in `work_dir/.completed`.
  6. Unless `--no-rom-zip` is passed, calls `scripts/internal/build_flashable_zip.sh` to generate a flashable OTA zip in `out/`.

For CI-like local builds (matching `.github/workflows/build.yml`):

```bash
# Example for r8q; see target/ for other codenames
source ./buildenv.sh r8q
./scripts/build_dependencies.sh
./scripts/download_fw.sh
./scripts/extract_fw.sh
./scripts/make_rom.sh --no-rom-zip
```

### 6. Building filesystem images / OTA zip manually (advanced)

Normally you should let `make_rom.sh` orchestrate everything. For advanced workflows, these lower-level entrypoints may be useful:

- `./scripts/build_fs_image.sh <fs> [options] <dir> <file_context> <fs_config>`
  - Creates a single partition image (`ext4`, `erofs`, or `f2fs`), optionally sparse and AVB-signed, from a directory tree plus metadata files.
  - Used internally by `build_flashable_zip.sh` for each partition.
- `./scripts/internal/build_flashable_zip.sh`
  - Consumes `$WORK_DIR`, filesystem metadata, and kernel images to:
    - Build partition images via `build_fs_image.sh`.
    - Build an `unsparse_super_empty.img` with `lpmake` and a `dynamic_partitions_op_list` matching the target's super group.
    - Convert images to `*.new.dat[.br]` using `img2sdat` and `brotli`.
    - Generate OTA metadata (`META-INF/com/android/metadata` and optionally `metadata.pb` using `protoc` and `ota_metadata.proto`).
    - Generate the edify `updater-script` with asserts, dynamic partition patching, and optional `postinstall.edify` for the device.
    - Sign the final zip with `signapk` using either AOSP testkeys or the UN1CA keys in `security/unica_*.{pk8,x509.pem}` if present.

These are tightly coupled to the logic and variables defined in `out/config.sh` and should generally not be called without having sourced `buildenv.sh` first.

### Testing

There is no standalone unit/integration test harness in this repository. Functional validation is done by successfully building images/zips and flashing them on supported devices.

## Repository layout and architecture

High-level structure relevant to Warp:

- `buildenv.sh`
  - Entry script to "enter" the build environment.
  - Locates the tree root, sets core env vars, enumerates `target/*` devices, and builds `out/config.sh` from `unica/configs/version.sh`, `unica/configs/*.sh`, and `target/<codename>/config.sh`.
  - Defines the `unica` alias to call `scripts/<cmd>.sh` with consistent logging.
- `unica/`
  - `configs/` — global ROM versioning (`version.sh`) and shared SSI configs (e.g. `qssi.sh`, `essi.sh`, `mssi.sh`).
  - `patches/` — ROM-wide code patches applied after the work directory is assembled.
  - `mods/` — feature modules and modifications, applied on top of the merged firmware tree.
- `target/<codename>/`
  - `config.sh` — per-device configuration (firmware identifiers, partition sizes, feature and capability matrix, super partition layout, etc.).
  - `patches/` — device-specific patches (APK/JAR smali diffs, resource tweaks, property changes) applied before ROM-wide patches.
  - Optional `debloat.sh`, `sff.sh`, `postinstall.edify` scripts that fine-tune the resulting ROM and OTA behavior.
- `scripts/`
  - Top-level entrypoints: `build_dependencies.sh`, `download_fw.sh`, `extract_fw.sh`, `make_rom.sh`, `build_fs_image.sh`, `cleanup.sh`, `generate_ota_manifest.sh`, etc.
  - `internal/` — lower-level build plumbing (`gen_config_file.sh`, `create_work_dir.sh`, `apply_modules.sh`, `build_flashable_zip.sh`, `update_prebuilt_blobs.sh`, etc.). These scripts assume a fully configured environment and are used by the top-level entrypoints.
  - `utils/` — shared bash libraries providing the bulk of the "logic" in this project:
    - `log_utils.sh` — structured, indented logging (`LOG`, `LOGE`, `LOGW`, `LOG_STEP_IN/OUT`) with call-site info.
    - `common_utils.sh` — generic helpers for property file discovery and editing, SELinux label lookup, fs_config/file_context manipulation, sparse image detection, partition name validation, and generic `EVAL` wrapper.
    - `build_utils.sh` — dependency checks, integration with `external/make.sh`, and helpers for computing disk/image sizes.
    - `firmware_utils.sh` — firmware-specific logic: comparing Samsung build strings, parsing firmware identifiers, checking/unsparsing images, and FUS queries.
    - `module_utils.sh` — high-level utilities used by ROM patches/mods (apktool-based decode/patch/rebuild, Galaxy Store downloads, floating feature XML manipulation, hex patching, convenience wrappers around `SET_PROP`, etc.).
- `external/`
  - Contains submodules and source trees for third-party tools; `external/make.sh` knows how to build and install them into `out/tools/bin`.
- `prebuilts/`
  - Static assets and prebuilt blobs (Samsung-specific files) referenced by `ADD_TO_WORK_DIR` and other utilities when constructing the work dir.
- `security/`
  - AOSP test keys and optional UN1CA signing keys (`unica_platform.*`, `unica_ota.*`). Presence of the UN1CA keys affects whether builds are considered "official" and which keys are used for signing.
- `out/` (generated)
  - `tools/` — all compiled helper tools and Python virtualenv for `samloader`.
  - `odin/` — downloaded Odin firmware zips and `.md5` tars, plus `.downloaded` markers.
  - `fw/` — extracted firmware trees and metadata for particular `MODEL_CSC` combinations, plus `.extracted` markers.
  - `target/<codename>/` — per-target artifacts: `apktool/`, `work_dir/`, logs, and built images/zips.
  - `config.sh` — the canonical, generated configuration file that every build script relies on.

Understanding and reusing the helpers under `scripts/utils/` and the configuration model (`out/config.sh` + `target/<codename>/config.sh` + `unica/configs/*.sh`) is key to making non-trivial changes or adding new build features without breaking existing devices.
