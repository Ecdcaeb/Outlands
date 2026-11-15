# Outlands

Outlands is a mod loader for Minecraft 1.12.2. This repository contains the loader, Minecraft patches and the build tooling used to create installers, development artifacts and launcher instance packs.

[中文](README-zh.md) · [Downloads and usage](#downloads-and-usage) · [Building and development](#building-and-development) · [Maintenance and releases](#maintenance-and-releases)

## Downloads and usage

Download a version from [GitHub Releases](https://github.com/Project-Outlands/Outlands/releases). Choose the file for your setup:

| Release file | Usage |
|---|---|
| `outlands-<version>.zip` | Import as a MultiMC/Prism instance. The launcher downloads the published loader and its dependencies. |
| `outlands-<version>-installer.jar` | Run the installer GUI, or use `--installClient` / `--installServer`. |
| `outlands-<version>.jar` | Universal loader JAR for integrations that use it directly. |

Test builds are available as artifacts from the [build workflow](https://github.com/Project-Outlands/Outlands/actions/workflows/BuildTest.yml).

For mod development, Maven coordinates are `top.outlands:outlands:<version>` at [Outlands Maven](https://maven.outlands.top/releases/). Available classifiers are `userdev`, `installer`, `sources`, `srg`, `dev-sources`, `devenv` and `mmc`. The `mmc` artifact is named `outlands-<version>-mmc.zip`; it contains the same pack attached to the GitHub release as `outlands-<version>.zip`.

See [CHANGELOG.md](CHANGELOG.md) for release history.

Report bugs in [Issues](https://github.com/Project-Outlands/Outlands/issues). Propose features in [Discussions](https://github.com/Project-Outlands/Outlands/discussions) after reading the [proposal guidelines](CFP_GUIDELINES.md).

## Building and development

### Requirements and build

Use JDK 25 for Gradle and compilation, with JDK 8 available for ForgeDev's tooling. CI installs both. The Gradle wrapper downloads Gradle; build dependencies and tools are resolved from Maven repositories.

Run from the repository root. On Windows, use `gradlew.bat` in place of `./gradlew`.

```sh
./gradlew setup      # Prepare the base and patched Minecraft sources
./gradlew build      # Compile, generate binary patches and package
```

Run `setup` on the first checkout and after changing patches. Build outputs are in `projects/outlands/build/libs/`:

| File | Contents |
|---|---|
| `outlands-<version>.jar` | Universal loader |
| `outlands-<version>-installer.jar` | Installer with the loader and installation profiles |
| `outlands-<version>-sources.jar` | Loader sources |
| `outlands-<version>-userdev.jar` | Mod development configuration and patches |
| `outlands-<version>-srg.jar` | Reobfuscated classes |
| `outlands-<version>-dev-sources.jar` | Loader and patched Minecraft sources |
| `outlands-<version>-devenv.zip` | Development mappings and metadata |
| `outlands-<version>-local.zip` | Local instance pack, including the loader JAR |

`./gradlew build -Prelease` creates `outlands-<version>.zip` instead of the local pack. It references the published loader download URL, so that loader version must be available remotely to use the pack. The flag alone does not publish anything.

Versions use UTC CalVer (`YYYY.MM.DD`). Local builds on the same day share filenames. Releases on the same day use `YYYY.MM.DD`, then `YYYY.MM.DD-2`, `YYYY.MM.DD-3`, and so on. To reproduce a release version locally:

```sh
./gradlew build -PcalverDate=2026.10.23 -PcalverSuffix=-2
```

### Development runs

```sh
./gradlew runOutlandsClient
./gradlew runOutlandsServer
```

Both use `projects/outlands/run/` as their working directory. The client task downloads Minecraft assets to `<Gradle user home>/caches/outlands/assets`; subsequent runs reuse this cache, and `clean` leaves it intact.

| Property | Purpose |
|---|---|
| `-PassetDirectory=<dir>` | Use an existing assets directory and skip automatic downloads |
| `-PassetsRepo=<url>` | Asset object host; defaults to `https://resources.download.minecraft.net/` |
| `-PassetsConnections=<n>` | Parallel asset downloads; defaults to 8 |
| `-PnativesDirectory=<dir>` | Override the native library directory |

### Repository layout

| Path | Contents |
|---|---|
| [src](src) | Loader sources and tests |
| [projects/mcpbase](projects/mcpbase) | Minecraft source preparation and patch tasks |
| [projects/outlands](projects/outlands) | Loader build, development runs and packaging |
| [patches/minecraft](patches/minecraft) | Minecraft source patches |
| [buildSrc](buildSrc) | Gradle plugins and tasks |
| [templates](templates) | Generated loader version template |
| [mmcpack-template](mmcpack-template) | Launcher instance templates |
| [tools/lwjgl2diff](tools/lwjgl2diff) | LWJGL comparison tooling |

## Maintenance and releases

### Minecraft patches

Maintain Minecraft changes in `patches/minecraft/**/*.java.patch`. These are unified diffs with `before/` and `after/` prefixes. From the root of an extracted base tree, use `git apply -p1` or `patch -p1`; IDE diff tools can also read them.

`setup` generates two source trees: `projects/mcpbase/minecraft` is the unpatched base, and `projects/mcpbase/patched` has the patches applied. Edit the patched tree, then regenerate and check the patch set:

```sh
./gradlew :mcpbase:makePatches
./gradlew :outlands:checkPatches
```

Use `:mcpbase:extractBase` or `:mcpbase:applyPatches` to regenerate the respective tree. Both trees are ignored by Git and removed by `:mcpbase:clean`; keep source changes in the patch files before cleaning. Compare the trees in an IDE or with:

```sh
diff -ru projects/mcpbase/minecraft projects/mcpbase/patched
```

<details>
<summary>Release setup, commands and recovery</summary>

### Release setup

The [release workflow](.github/workflows/Release.yml) builds and publishes releases. `./gradlew release` checks the checkout, computes the version and dispatches CI, then waits for its result. It does not upload local artifacts. The checkout must be on `main`, clean and at the same commit as `origin/main`.

Create a fine-grained GitHub personal access token restricted to `Project-Outlands/Outlands`:

- `Actions: Read and write` to dispatch and monitor the workflow.
- `Contents: Read` to let the local task confirm the GitHub release afterward, including in a private repository. Without it, CI can still complete, but local release confirmation may be unavailable.

Store the token outside the repository in `~/.gradle/gradle.properties` (Windows: `%USERPROFILE%\.gradle\gradle.properties`):

```properties
releaseToken=REPLACE_WITH_YOUR_TOKEN
```

Alternatively, supply `OUTLANDS_RELEASE_TOKEN` through your environment or secret manager, or pass `-PreleaseToken=<token>`. Avoid entering real tokens directly into shell commands: literal values in either command-line arguments or environment assignments can enter shell history. Keep credential files private and out of version control.

CI needs the `OUTLANDS_MAVEN_TOKEN` secret for Maven publishing; its publishing username is `fg`.

### Release commands

```sh
./gradlew release                               # Check, dispatch CI and wait
./gradlew release -PdryRun                      # Check and print the planned release without dispatching
./gradlew release -PnoWait                      # Dispatch and print the workflow URL
./gradlew release -PreleaseVersion=2026.10.23-2  # Resume a specific release
./gradlew releaseVersion                        # Print the next release version
```

Optional properties: `-PgitRemote=<name|path>` selects a remote instead of `origin`; `-PreleaseTimeout=<minutes>` changes the wait timeout (default 20 minutes).

CI checks the requested commit and version, builds, uploads to Maven, then creates the tag and GitHub release. Local builds resolve `mavenLocal()` first, while CI has no local artifacts. Publish required `top.outlands` tooling dependencies to Outlands Maven before releasing; use an init script that removes `mavenLocal()` when reproducing CI dependency resolution locally.

Release notes and [CHANGELOG.md](CHANGELOG.md) are generated by [git-cliff](https://git-cliff.org) using [cliff.toml](cliff.toml). To reproduce notes, pass the previous tag and target version explicitly:

```sh
git cliff 2026.10.23..HEAD --tag 2026.10.23-2
```

### Release recovery

| Failure | Recovery |
|---|---|
| Build or checks, before upload | Fix the cause and run `./gradlew release` again. No tag has been created. |
| Maven upload | Fix the cause and retry. If the version is partially uploaded, remove that version from Maven first; CI refuses partial publications. |
| Tag, GitHub release or changelog after upload | Run `./gradlew release -PreleaseVersion=<version>`. CI skips a complete Maven upload and reuses an existing tag. |

Tags are not rolled back automatically. Keep the existing tag when resuming a release. A pushed CalVer tag also triggers the workflow; this bypasses local version selection, so reserve that path for a version you have confirmed is free.

</details>
