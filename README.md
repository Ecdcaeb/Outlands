# Outlands

A 1.12.2 mod loader. It replaces the old ForgeGradle based build with a self contained one: the MCP pipeline
(decompile, access transformers, cleanup, rename), the patch set, the loader sources, the binary patches, the
installer and the launcher pack are all produced from this repository.

`README-zh.md` is the Chinese version of this file.

## Requirements

* JDK 25 (`java -version`)
* Nothing else: the Gradle wrapper downloads Gradle itself, and the patch/decompile tooling comes from maven

## Building

```
./gradlew setup      # decompile + patch Minecraft (only needed once, and after patch changes)
./gradlew build      # compile the loader, reobfuscate, generate binary patches, package
```

Artifacts land in `projects/outlands/build/libs/`:

| File | What it is |
|---|---|
| `outlands-<version>.jar` | the loader ("universal") jar |
| `outlands-<version>-installer.jar` | Forge's installer program (GUI + `--installClient`/`--installServer`) carrying our `install_profile.json`, `version.json` and the universal jar |
| `outlands-<version>-sources.jar` | loader sources (what userdev consumers inject) |
| `outlands-<version>-userdev.jar` | userdev artifact for mod development setups |
| `outlands-<version>-srg.jar` | reobfuscated classes (consumed by userdev tooling) |
| `outlands-<version>.zip` / `outlands-<version>-local.zip` | MultiMC/Prism instance pack (`-local` carries the jar itself) |

`-Prelease` builds the launcher pack against the published maven URL instead of pointing at the local jar.

### Version

CalVer, `YYYY.MM.DD`, in UTC, with nothing else in it - no commit distance, no dirty flag, no CI run number. Two
builds on the same day therefore produce the same file names and overwrite each other.

Only *releases* get a number: the first release of a day is `2026.10.23`, the next ones are `2026.10.23-2`,
`2026.10.23-3`, ... That suffix is decided when publishing, never by a local build. `-PcalverDate=2026.10.23
-PcalverSuffix=-2` reproduces a specific release locally.

## Dev runs

```
./gradlew runOutlandsClient
./gradlew runOutlandsServer
```

Both run out of `projects/outlands/run/`. The classpath is the project's runtime classpath (the mcp config's
libraries plus its `extra` jar for the resources inside the client jar), and FML is booted directly through
`net.minecraftforge.boot.MainClient` - no launcher sits in between, so nothing would fetch the vanilla assets.
`runOutlandsClient` downloads them first: the asset index and its 1305 objects (sounds, the 105 languages besides
`en_us`, the realms icons) go into `$GRADLE_USER_HOME/caches/outlands/assets`. It is a cache - fetched once, reused
afterwards, untouched by `clean` - and only the client run depends on it.

| Property | Meaning |
|---|---|
| `-PassetDirectory=<dir>` | Use an existing assets directory (a launcher's, for example) instead of downloading |
| `-PassetsRepo=<url>` | Object host, defaults to `https://resources.download.minecraft.net/` |
| `-PassetsConnections=<n>` | Parallel downloads, default 8 |

## Repository layout

| Path | Contents |
|---|---|
| `projects/mcpbase` | MCP layer: decompile, access transformers, mcpcleanup, patch application. Generates the two source trees below `projects/mcpbase/minecraft` and `.../patched` |
| `projects/outlands` | loader sources (compiled against the patched Minecraft), reobf, binary patches, packaging, userdev, installer, launcher pack |
| `patches/minecraft` | the patched Minecraft sources, as git-applyable patches |
| `templates` | `LoaderVersion.java`, filled in with the current version at build time |
| `buildSrc` | the build plugins/tasks (`top.outlands.gradle.*`) |
| `legacy/fg6` | the previous ForgeGradle 6 build scripts, kept for reference only |

Published coordinates for mod developers: `top.outlands:outlands:<version>` (plus the `userdev`, `installer`,
`sources`, `srg`, `dev-sources`, `devenv` and `mmc` classifiers), from `https://maven.outlands.top/releases`.
The `mmc` classifier is the launcher pack (`outlands-<version>-mmc.zip`), i.e. the same zip a release attaches.

## Maintaining patches

Minecraft sources are modified through `patches/minecraft/**.java.patch`. They are plain unified diffs with
`before/` + `after/` prefixes and full hunk headers, so `git apply -p2`, `patch -p2` and IDE tooling all understand
them.

* `./gradlew :outlands:checkPatches` verifies that every patch still applies
* `:mcpbase:makePatches` regenerates the patch set from `projects/mcpbase/patched`

The unpatched base is materialized next to the patched tree, both MCP named, both generated (gitignored) and both
removed by `:mcpbase:clean`:

| Path | Contents |
|---|---|
| `projects/mcpbase/minecraft` | the base, no patches applied (`:mcpbase:extractBase`) |
| `projects/mcpbase/patched` | `patches/minecraft` applied (`:mcpbase:applyPatches`) |

So a patch can be read as a diff without any tooling: `diff -ru projects/mcpbase/minecraft projects/mcpbase/patched`,
or the two directories can be compared side by side in an IDE. `:outlands` compiles the patched tree directly.

The base tree is registered as a source set of `:mcpbase` so IDEs index it (`compileMinecraftJava` is disabled -
nothing compiles it, `:outlands` already compiles the patched tree); that is also what gives its imports a
classpath to resolve against.

## Releasing (maintainers)

> Local builds resolve `mavenLocal()` before anything else, CI does not. Anything this build takes from the
> `top.outlands` fork (the ForgeDev plugin, the mavenizer, ...) has to be published to `maven.outlands.top`, or the
> release build will fail where a local one succeeds. `--init-script` with `mavenLocal` removed is a good way to
> reproduce CI locally.

Releases are built and published by CI (`Release.yml`); nothing is ever uploaded from a laptop. `./gradlew release`
only *asks* CI to do it, so it never mutates the repository or publishes a local build.

### One-time setup

Create a fine-grained personal access token for your own GitHub account:

* Repository access: only `Project-Outlands/Outlands`
* Permissions: `Actions: Read and write` (nothing else)

Put it outside the repository:

```
echo 'releaseToken=github_pat_...' >> ~/.gradle/gradle.properties
# Windows: %USERPROFILE%\.gradle\gradle.properties
```

Alternatively pass `-PreleaseToken=...` or set `OUTLANDS_RELEASE_TOKEN` (environment variables are easier to leak
into shell history, the properties file is the recommended place).

### Commands

```
./gradlew release                                 # verify, compute the version, dispatch CI, wait for the result
./gradlew release -PdryRun                        # only verify and print what would be released
./gradlew release -PnoWait                        # dispatch and return immediately (prints the run URL)
./gradlew release -PreleaseVersion=2026.10.23-2   # finish a release that broke half way
./gradlew releaseVersion                          # just print the version a release would use
```

Before dispatching, `release` refuses to continue unless: you are on `main`, the working tree is clean, and `HEAD`
is exactly `origin/main`. It reports "behind" and "not pushed" separately and never runs `git pull` for you.

CI then re-checks the same things (the commit must still be `HEAD`, and the version must still be the next free one),
builds, uploads to maven, **and only then creates the tag** and the GitHub release. That ordering is what keeps a
failed release from consuming a version number.

### Changelog

Release notes and `CHANGELOG.md` entries are summarized by an OpenAI-compatible Chat Completions API, using the
non-merge commits between the previous release tag and `HEAD`. [`cliff.md`](cliff.md) defines the categories,
excluded commit types and writing rules. Related changes are combined into concise notes instead of listing
every commit separately.

CI adds the version and date header to `CHANGELOG.md`; the generated body is also used for the GitHub release.
An API error or empty response fails the workflow. Fix the configuration or service, then resume the release with
`./gradlew release -PreleaseVersion=<version>`.

### CI configuration

Configure these in the repository's Actions secrets and variables:

| Name | Storage | Purpose |
|---|---|---|
| `OUTLANDS_MAVEN_TOKEN` | Secret | Maven publishing token |
| `OPENAI_API_KEY` | Secret | API key for changelog generation |
| `OPENAI_ENDPOINT` | Variable | Full Chat Completions URL, for example `https://api.openai.com/v1/chat/completions` |
| `OPENAI_MODEL` | Variable | Model name; defaults to `gpt-4o-mini` |

The Maven publishing username is `fg` and is set directly in the workflow.

### When a release fails

| Where it failed | What is left behind | What to do |
|---|---|---|
| build / tests | nothing, no tag | fix, then `./gradlew release` (same version) |
| maven upload | nothing, no tag (a partial upload is detected before anything else) | fix the cause, re-run; if the version is half published, delete that version from maven first |
| tag / AI changelog / GitHub release / CHANGELOG | artifacts are on maven, tag may exist | `./gradlew release -PreleaseVersion=<version>` |

Tags are never rolled back automatically: a tag records which commit produced a version, and deleting it would not
remove anything from maven anyway. To continue a release whose tag already exists, use `-PreleaseVersion=<version>`.

### Escape hatch

Pushing a tag by hand still releases, and the version is read from the tag name:

```
git tag 2026.10.23 && git push origin 2026.10.23
```

This bypasses the version computation, so only use it when you know the number is free.
