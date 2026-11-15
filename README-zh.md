# Outlands

Outlands 是 Minecraft 1.12.2 的模组加载器。本仓库包含加载器源码、Minecraft 补丁，以及生成安装器、开发产物和启动器实例包的构建工具。

[English](README.md) · [下载与使用](#下载与使用) · [构建与开发](#构建与开发) · [维护与发布](#维护与发布)

## 下载与使用

从 [GitHub Releases](https://github.com/Project-Outlands/Outlands/releases) 下载所需版本，按用途选择文件：

| 发布文件 | 用途 |
|---|---|
| `outlands-<version>.zip` | 导入 MultiMC/Prism 作为实例；启动器会下载已发布的加载器及其依赖。 |
| `outlands-<version>-installer.jar` | 运行图形安装器，或使用 `--installClient` / `--installServer`。 |
| `outlands-<version>.jar` | 通用加载器 JAR，供需要直接集成加载器的工具使用。 |

测试构建可从[构建工作流](https://github.com/Project-Outlands/Outlands/actions/workflows/BuildTest.yml)的产物中下载。

模组开发使用的 Maven 坐标为 `top.outlands:outlands:<version>`，仓库地址为 [Outlands Maven](https://maven.outlands.top/releases/)。可用分类器包括 `userdev`、`installer`、`sources`、`srg`、`dev-sources`、`devenv` 和 `mmc`。其中 `mmc` 产物名为 `outlands-<version>-mmc.zip`，与 GitHub 发布附件 `outlands-<version>.zip` 是同一个实例包。

版本记录见 [CHANGELOG.md](CHANGELOG.md)。

问题反馈请使用 [Issues](https://github.com/Project-Outlands/Outlands/issues)；功能提案请先阅读[提案指南](CFP_GUIDELINES.md)，再提交到 [Discussions](https://github.com/Project-Outlands/Outlands/discussions)。

## 构建与开发

### 环境与构建

使用 JDK 25 运行 Gradle 和编译，并为 ForgeDev 工具准备 JDK 8；CI 也会安装这两个版本。Gradle wrapper 会下载 Gradle，构建依赖和工具从 Maven 仓库获取。

在仓库根目录执行命令。Windows 下请将 `./gradlew` 换成 `gradlew.bat`。

```sh
./gradlew setup      # 准备 Minecraft 基线源码和应用补丁后的源码
./gradlew build      # 编译、生成二进制补丁并打包
```

首次检出和修改补丁后运行 `setup`。构建产物位于 `projects/outlands/build/libs/`：

| 文件 | 内容 |
|---|---|
| `outlands-<version>.jar` | 通用加载器 |
| `outlands-<version>-installer.jar` | 包含加载器和安装配置的安装器 |
| `outlands-<version>-sources.jar` | 加载器源码 |
| `outlands-<version>-userdev.jar` | 模组开发配置与补丁 |
| `outlands-<version>-srg.jar` | 重混淆后的类 |
| `outlands-<version>-dev-sources.jar` | 加载器与应用补丁后的 Minecraft 源码 |
| `outlands-<version>-devenv.zip` | 开发用映射与元数据 |
| `outlands-<version>-local.zip` | 包含加载器 JAR 的本地实例包 |

`./gradlew build -Prelease` 会生成 `outlands-<version>.zip`，替代本地实例包。它引用已发布的加载器下载地址，因此使用前需要确保对应版本已在远端发布。这个参数本身不会上传任何产物。

版本号采用 UTC 日期的 CalVer（`YYYY.MM.DD`）。同一天的本地构建使用相同文件名；当天的发布依次使用 `YYYY.MM.DD`、`YYYY.MM.DD-2`、`YYYY.MM.DD-3` 等版本号。要在本地复现某个发布版本号：

```sh
./gradlew build -PcalverDate=2026.10.23 -PcalverSuffix=-2
```

### 开发运行

```sh
./gradlew runOutlandsClient
./gradlew runOutlandsServer
```

两者的工作目录均为 `projects/outlands/run/`。客户端任务会将 Minecraft 资源下载到 `<Gradle 用户目录>/caches/outlands/assets`；后续运行复用缓存，`clean` 不会删除它。

| 属性 | 用途 |
|---|---|
| `-PassetDirectory=<dir>` | 使用已有资源目录，跳过自动下载 |
| `-PassetsRepo=<url>` | 资源对象下载地址，默认为 `https://resources.download.minecraft.net/` |
| `-PassetsConnections=<n>` | 资源并行下载数，默认为 8 |
| `-PnativesDirectory=<dir>` | 指定本地库目录 |

### 仓库结构

| 路径 | 内容 |
|---|---|
| [src](src) | 加载器源码与测试 |
| [projects/mcpbase](projects/mcpbase) | Minecraft 源码准备与补丁任务 |
| [projects/outlands](projects/outlands) | 加载器构建、开发运行与打包 |
| [patches/minecraft](patches/minecraft) | Minecraft 源码补丁 |
| [buildSrc](buildSrc) | Gradle 插件与任务 |
| [templates](templates) | 加载器版本信息模板 |
| [mmcpack-template](mmcpack-template) | 启动器实例模板 |
| [tools/lwjgl2diff](tools/lwjgl2diff) | LWJGL 对比工具 |

## 维护与发布

### Minecraft 补丁

Minecraft 改动保存在 `patches/minecraft/**/*.java.patch` 中。这些文件是带有 `before/` 和 `after/` 前缀的 unified diff。在提取出的基线源码根目录中，可使用 `git apply -p1` 或 `patch -p1` 应用补丁，也可通过 IDE 的差异工具查看。

`setup` 会生成两棵源码树：`projects/mcpbase/minecraft` 是未修改的基线，`projects/mcpbase/patched` 已应用补丁。修改 `patched` 中的源码后，重新生成并检查补丁集：

```sh
./gradlew :mcpbase:makePatches
./gradlew :outlands:checkPatches
```

使用 `:mcpbase:extractBase` 或 `:mcpbase:applyPatches` 可重新生成对应的源码树。两棵树均被 Git 忽略，并由 `:mcpbase:clean` 删除；清理前请先将源码改动保存到补丁文件。可在 IDE 中比较两棵树，或执行：

```sh
diff -ru projects/mcpbase/minecraft projects/mcpbase/patched
```

<details>
<summary>发布准备、命令与故障恢复</summary>

### 发布准备

[发布工作流](.github/workflows/Release.yml) 负责构建和上传。`./gradlew release` 检查本地仓库、计算版本号、触发 CI 并等待结果，不会上传本地产物。执行时必须位于 `main` 分支，工作区干净，且当前提交与 `origin/main` 一致。

创建一个仅能访问 `Project-Outlands/Outlands` 的 GitHub 细粒度个人访问令牌：

- `Actions: Read and write`：用于触发和监控工作流。
- `Contents: Read`：用于让本地任务在发布后确认 GitHub release，包括私有仓库中的 release。缺少此权限时 CI 仍可完成，但本地可能无法确认发布结果。

将令牌放在仓库外的 `~/.gradle/gradle.properties` 中（Windows：`%USERPROFILE%\.gradle\gradle.properties`）：

```properties
releaseToken=REPLACE_WITH_YOUR_TOKEN
```

也可以通过环境或密钥管理工具提供 `OUTLANDS_RELEASE_TOKEN`，或传入 `-PreleaseToken=<token>`。避免在命令中直接填写真实令牌：命令行参数和环境变量赋值中的明文值都可能进入 shell 历史。凭据文件应仅供自己访问，不要加入版本控制。

CI 需要 `OUTLANDS_MAVEN_TOKEN` secret 来发布 Maven 产物，发布用户名为 `fg`。

### 发布命令

```sh
./gradlew release                               # 检查、触发 CI 并等待
./gradlew release -PdryRun                      # 检查并打印发布计划，不触发 CI
./gradlew release -PnoWait                      # 触发后打印工作流地址并返回
./gradlew release -PreleaseVersion=2026.10.23-2  # 恢复指定版本的发布
./gradlew releaseVersion                        # 打印下一个发布版本号
```

可选属性：`-PgitRemote=<name|path>` 指定远端，默认为 `origin`；`-PreleaseTimeout=<minutes>` 修改等待超时时间，默认为 20 分钟。

CI 会检查指定提交和版本号，完成构建并上传 Maven，然后创建 tag 和 GitHub release。本地构建优先解析 `mavenLocal()`，CI 则没有本地产物。发布前需将所依赖的 `top.outlands` 工具发布到 Outlands Maven；在本地复现 CI 的依赖解析时，可用 init script 移除 `mavenLocal()`。

发布说明与 [CHANGELOG.md](CHANGELOG.md) 由 [git-cliff](https://git-cliff.org) 按 [cliff.toml](cliff.toml) 生成。复现说明时，显式指定上一个 tag 和目标版本：

```sh
git cliff 2026.10.23..HEAD --tag 2026.10.23-2
```

### 发布恢复

| 失败位置 | 处理方式 |
|---|---|
| 上传前的构建或检查 | 修复原因后重新运行 `./gradlew release`；此时尚未创建 tag。 |
| Maven 上传 | 修复原因后重试；若该版本已部分上传，需先从 Maven 删除该版本，CI 会拒绝部分发布的状态。 |
| 上传后的 tag、GitHub release 或更新日志步骤 | 运行 `./gradlew release -PreleaseVersion=<version>`；CI 会跳过已完整上传的 Maven 产物，并复用已有 tag。 |

tag 不会自动回退，恢复发布时请保留已有 tag。推送 CalVer tag 也会触发工作流，但会绕过本地版本选择，仅在确认版本号尚未使用时采用此方式。

</details>
