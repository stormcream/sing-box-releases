# sing-box CLI releases

[![Build sing-box CLI and release](https://github.com/stormcream/sing-box-releases/actions/workflows/build-cli.yml/badge.svg?branch=dev)](https://github.com/stormcream/sing-box-releases/actions/workflows/build-cli.yml)

Unofficial [sing-box](https://github.com/SagerNet/sing-box) CLI builds with the fallback outbound, published from GitHub Actions.

本仓库提供基于 [stormcream/sing-box](https://github.com/stormcream/sing-box) `my-stable` 分支构建的 **sing-box CLI** 非官方二进制，并通过 GitHub Actions 完成编译、归档、构建溯源和发布。

> 本仓库只包含构建工作流，不包含 sing-box 源码，也不构建图形客户端。最新版本见 [Releases](https://github.com/stormcream/sing-box-releases/releases/latest)。

## 与上游的关系

- 构建源码：[stormcream/sing-box@my-stable](https://github.com/stormcream/sing-box/tree/my-stable)
  - 基于官方 [SagerNet/sing-box](https://github.com/SagerNet/sing-box) 稳定版
  - 额外提供 fallback 出站（[配置文档](https://github.com/stormcream/sing-box/blob/my-stable/docs/configuration/outbound/fallback.md)）
- CI 结构参考并派生自 [reF1nd/sing-box-releases](https://github.com/reF1nd/sing-box-releases)
- 当前工作流使用 **Go 1.26.10**

每次发布的确切源码版本以 `build-manifest.json` 中的 `source.sha` 为准。

## 下载

前往 [Releases](https://github.com/stormcream/sing-box-releases/releases) 下载。Release tag 为 `v{version}`，例如 `v1.14.4`。

### 产物命名

| 平台 | 产物 | 说明 |
|---|---|---|
| Linux | `sing-box-<version>-linux-<arch>-<variant>.tar.gz` | arch: `arm64` / `amd64` / `amd64v3`；variant: `purego` / `glibc` / `musl` |
| Windows | `sing-box-<version>-windows-<arch>.zip` | arch: `arm64` / `amd64` / `amd64v3` |
| Windows 7 | `sing-box-<version>-windows-<arch>-legacy-windows-7.zip` | arch: `amd64` / `386` |
| macOS | `sing-box-<version>-darwin-<arch>.tar.gz` | arch: `arm64` / `amd64` / `amd64v3` |
| macOS 10.13 | `sing-box-<version>-darwin-amd64-legacy-macos-10.13.tar.gz` | 兼容旧版 macOS |
| Android | `sing-box-<version>-android-<arch>.tar.gz` | arch: `386` / `amd64` / `amd64v3` / `arm` / `arm64` |
| 清单 | `build-manifest.json` | 版本、源码 commit、每个文件的 SHA-256 和大小 |

每个压缩包内包含 `sing-box`（Windows 为 `sing-box.exe`）和 `LICENSE`。Linux/Windows 的 `purego` 变体会额外附带 `libcronet.so` / `libcronet.dll`。

`glibc` / `musl` 变体使用 CGO 和 NaiveProxy 工具链；`purego` 变体不依赖系统 libc，通用性更好。

## 构建矩阵

工作流一次构建 23 个组合：

| 平台 | 架构 | 变体 | 数量 |
|---|---|---|---|
| Linux | `arm64` / `amd64` / `amd64v3` | `purego` / `glibc` / `musl` | 9 |
| Windows | `arm64` / `amd64` / `amd64v3` | `purego` | 3 |
| Windows 7 | `amd64` / `386` | legacy | 2 |
| macOS | `arm64` / `amd64` / `amd64v3` | 默认 | 3 |
| macOS 10.13 | `amd64` | legacy | 1 |
| Android | `386` / `amd64` / `amd64v3` / `arm` / `arm64` | CGO | 5 |

特殊工具链（以下脚本位于构建源码仓库 `stormcream/sing-box` 的 `.github/` 下）：

- **Windows ARM64**：使用 `.github/patch_go_for_windows_arm64.sh` 打 CPU 特性补丁，确保 AES/SHA 使用硬件加速。
- **Windows 7**：使用 `.github/setup_go_for_windows7.sh` 安装带兼容补丁的 Go 1.26.10。
- **macOS 10.13**：使用 `.github/setup_go_for_macos1013.sh` 安装专用 Go 工具链。

## 发布流程

工作流文件：`.github/workflows/build-cli.yml`。仅支持手动触发，不会在 push/tag 时自动运行。

### 触发

GitHub → Actions → **Build sing-box CLI and release** → Run workflow。

或使用 GitHub CLI：

```bash
gh workflow run build-cli.yml \
  --repo stormcream/sing-box-releases \
  --ref dev \
  -f version=1.14.5 \
  -f prerelease=false
```

### 输入

| 输入 | 必填 | 说明 |
|---|---|---|
| `version` | 是 | 版本号，用于产物文件名、编译进二进制的 `constant.Version` 和 `v{version}` tag |
| `source_sha` | 否 | 指定 sing-box commit（40 位小写 hex）；留空则使用 `my-stable` 当前 HEAD |
| `prerelease` | 否 | 是否发布为 pre-release，默认 `false` |

### 工作流步骤

1. **resolve_source**：checkout 源码（`source_sha` 或 `my-stable`），校验版本号/SHA，并固定本次构建使用的 commit。
2. **build_cli**：23 个矩阵任务构建（最多 13 个并行）；恢复 Go module 缓存；按平台写入构建 tags；归档并上传 artifact（保留 7 天）。
3. **release**：下载全部 artifact，生成 `build-manifest.json`（含 SHA-256、大小），生成 GitHub 构建溯源证明，上传 manifest artifact，生成 release notes，删除同版本已有 release/tag 后创建 release。
4. **cache_gc**：清理过期缓存，把 Actions cache 控制在 9GB 以内。

工作流只使用仓库自带的 `GITHUB_TOKEN`，无需额外 secrets。请确保仓库 **Settings → Actions → General → Workflow permissions** 允许 **Read and write permissions**，否则无法创建 release。

同一版本的新运行会取消进行中的旧运行；发布前会先删除同版本已有的 release 和 tag，因此重跑同一版本会得到指向本次运行 commit 的全新 release。

## 校验下载

### 使用 build-manifest.json

```bash
gh release download v1.14.4 --repo stormcream/sing-box-releases
jq -r '.artifacts[] | "\(.sha256)  \(.path)"' build-manifest.json | sha256sum -c -
```

### 校验构建溯源

```bash
gh attestation verify sing-box-1.14.4-linux-amd64-purego.tar.gz \
  --repo stormcream/sing-box-releases
```

`build-manifest.json` 主要字段：

```json
{
  "schema_version": 1,
  "workflow": ".github/workflows/build-cli.yml",
  "repository": "stormcream/sing-box-releases",
  "head_branch": "dev",
  "head_sha": "...",
  "run_id": 123456789,
  "run_attempt": 1,
  "version": "1.14.4",
  "source_sha": "...",
  "source": {
    "repository": "stormcream/sing-box",
    "branch": "my-stable",
    "ref": "my-stable",
    "sha": "..."
  },
  "built_at": "2026-10-24T01:10:15Z",
  "artifacts": [
    {
      "path": "sing-box-1.14.4-linux-amd64-purego.tar.gz",
      "sha256": "...",
      "size": 31708258
    }
  ]
}
```

## 复现某次构建

从对应 release 的 `build-manifest.json` 取出 `source.sha`，触发工作流时填入 `source_sha`，并使用一个不同于已有 release 的版本号：

```bash
SOURCE_SHA="$(jq -r '.source.sha' build-manifest.json)"
gh workflow run build-cli.yml \
  --repo stormcream/sing-box-releases \
  --ref dev \
  -f version=1.14.4-rebuild.1 \
  -f source_sha="$SOURCE_SHA" \
  -f prerelease=true
```

工作流会把所有构建任务固定到该 commit，并把解析结果写回新的 manifest。

> 注意：`version` 直接决定 release tag；如果使用与已有 release 相同的版本号，工作流会先删除原 release 和 tag。复现历史构建时建议加 `-rebuild.N` 后缀。

## 本地构建

```bash
git clone --branch my-stable https://github.com/stormcream/sing-box.git
cd sing-box

# 建议使用 Go 1.26.10
TAGS="$(cat release/DEFAULT_BUILD_TAGS),with_purego"
CGO_ENABLED=0 go build -trimpath -tags "$TAGS" \
  -ldflags "-X github.com/sagernet/sing-box/constant.Version=dev $(cat release/LDFLAGS) -s -w -buildid=" \
  -o sing-box ./cmd/sing-box
```

这是 Linux 上最接近发布产物的 purego 命令；实际发布还涉及 CGO/musl、NaiveProxy/Chromium 工具链和平台专用 Go 补丁，建议直接复用工作流。

> `purego` 构建包含 NaiveProxy 出站，运行时需要与二进制同目录的 `libcronet.so`（发布包中已附带）。如果不需要 NaiveProxy，可把 `DEFAULT_BUILD_TAGS` 换成 `DEFAULT_BUILD_TAGS_OTHERS`。

## 仓库结构

```text
.github/workflows/build-cli.yml  # 构建与发布工作流
.gitignore
LICENSE
README.md
```

## 上游与致谢

- [SagerNet/sing-box](https://github.com/SagerNet/sing-box)
- [stormcream/sing-box](https://github.com/stormcream/sing-box)
- [reF1nd/sing-box-releases](https://github.com/reF1nd/sing-box-releases)

## License

本仓库及构建产物遵循 [GNU General Public License v3.0](LICENSE)。sing-box 源码版权归其作者所有。

## 免责声明

本仓库提供的是非官方构建，与 SagerNet、reF1nd 及其维护者无隶属关系。请自行评估使用风险，并遵守所在地区的法律法规。
