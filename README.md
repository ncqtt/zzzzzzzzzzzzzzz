# build-kernel

GitHub Actions workflow for building Android kernels manually via `workflow_dispatch`.
Targets Samsung Galaxy A32 4G (SM-A325F) by default, but is fully configurable.

```
  kernel source
      |
      +-- toolchain setup (clang / gcc)
      |
      +-- ccache restore
      |
      +-- defconfig + optional config fragments
      |
      +-- make
      |
      +-- AnyKernel3 ZIP
      |
      +-- artifacts / release / telegram
```

---

## Requirements

Two repository secrets must be set before running:

```
Settings > Secrets and variables > Actions

  TELEGRAM_TOKEN    obtained from @BotFather
  TELEGRAM_CHAT_ID  your group or channel ID
```

---

## Usage

Go to `Actions > Build Kernel > Run workflow` and fill in the inputs.
The workflow never runs automatically on push.

---

## Inputs

### Device

| Input            | Required | Default               | Description              |
|------------------|----------|-----------------------|--------------------------|
| `device_name`    | yes      | Samsung Galaxy A32 4G | Human-readable name      |
| `device_codename`| yes      | SM-A325F              | Used in ZIP filename     |

### Kernel source

| Input           | Required | Default | Description                        |
|-----------------|----------|---------|------------------------------------|
| `kernel_repo`   | yes      | —       | `user/repo` or full URL            |
| `kernel_branch` | yes      | main    | Branch to clone                    |

### Architecture

| Input  | Required | Default | Options       |
|--------|----------|---------|---------------|
| `arch` | yes      | arm64   | arm64 / arm   |

### Defconfig

| Input           | Required | Default        | Description                                   |
|-----------------|----------|----------------|-----------------------------------------------|
| `defconfig`     | yes      | a32_defconfig  | File inside `arch/<ARCH>/configs/`            |
| `extra_configs` | no       | —              | Space-separated config fragments to merge     |

### Toolchain

| Input               | Required | Default       | Description                                          |
|---------------------|----------|---------------|------------------------------------------------------|
| `toolchain`         | yes      | google-clang  | See options below                                    |
| `clang_version`     | no       | r522817       | Google Clang tag (google-clang only)                 |
| `custom_clang_repo` | no       | —             | Repo URL for proton / neutron / aosp clang           |
| `use_llvm`          | no       | true          | Pass `LLVM=1 LLVM_IAS=1` (disable for old kernels)  |
| `gcc_repos`         | no       | —             | `aarch64_url arm32_url` space-separated; leave empty to use apt gcc |

Available toolchain options:

```
  google-clang    downloaded from android.googlesource.com
  proton-clang    kdrag0n/proton-clang (or custom repo)
  neutron-clang   latest release from Neutron-Toolchains
  aosp-clang      crdroidandroid prebuilt (or custom repo)
  gcc-only        system GCC, no Clang
```

### ccache

| Input         | Required | Default | Description         |
|---------------|----------|---------|---------------------|
| `ccache_size` | no       | 5G      | Max cache size      |

Cache is keyed by device codename, toolchain, defconfig, and commit SHA.
Fallback keys restore partial cache from previous runs.

### AnyKernel3

| Input              | Required | Default           | Description         |
|--------------------|----------|-------------------|---------------------|
| `anykernel_repo`   | yes      | osm0sis/AnyKernel3| `user/repo` or URL  |
| `anykernel_branch` | no       | master            | Branch to clone     |

### Output / ZIP

| Input          | Required | Default | Description                              |
|----------------|----------|---------|------------------------------------------|
| `image_name`   | no       | Image   | Image / Image.gz / Image.gz-dtb / Image-dtb |
| `include_dtb`  | no       | false   | Copy DTB files into ZIP                  |
| `include_dtbo` | no       | false   | Copy dtbo.img into ZIP                   |
| `zip_prefix`   | no       | Kernel  | ZIP filename: `<prefix>-<codename>-<timestamp>.zip` |

### Release

| Input             | Required | Default | Description                                      |
|-------------------|----------|---------|--------------------------------------------------|
| `create_release`  | no       | false   | Create a GitHub Release and attach the ZIP       |
| `release_tag`     | no       | —       | Tag name, e.g. `v1.0.0` (required if releasing) |

### Telegram

| Input               | Required | Default | Description                        |
|---------------------|----------|---------|------------------------------------|
| `send_zip_telegram` | no       | true    | Upload ZIP to Telegram (max 50 MB) |

Notifications are sent at build start, on success, and on failure.
If the ZIP exceeds 50 MB the bot sends a link to the Actions run instead.

---

## Build stages

```
  1  free disk space + install deps
  2  clone kernel source + AnyKernel3
  3  set up toolchain
  4  restore + configure ccache
  5  generate defconfig, merge fragments, compile
  6  package flashable ZIP
  7  upload to GitHub Artifacts (retained 7 days)
  8  create GitHub Release (optional)
  9  send Telegram notifications
 10  print ccache stats + disk usage
```

Timeout: 180 minutes. Runner: ubuntu-latest.

---

## Artifacts

On success, two artifacts are uploaded:

```
  build-log-<codename>   build.log, retained 3 days (uploaded on failure too)
  <ZIP_NAME>             flashable ZIP, retained 7 days
```

---

## Notes

- `kernel_depth` is fixed at `1` (shallow clone). Edit the workflow env to change it.
- `custom_clang_branch` defaults to `main`. Edit `CUSTOM_CLANG_BRANCH` in the env block to override.
- LLVM flags are skipped automatically when `toolchain` is `gcc-only`.
- Config fragments that are not found on disk are skipped with a warning, not a failure.
