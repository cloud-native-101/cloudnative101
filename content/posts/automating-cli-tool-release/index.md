---
keywords:
- cloud native
- cloud native 101
- goreleaser
title: "Cloud Native Tools 101: 如何自动化发布 CLI 工具"
subtitle: "云原生小技巧: 如何自动化发布 CLI 工具？"
description: 本文详解如何在云原生时代自动化发布 CLI 工具，着重于跨平台兼容性、Makefile 自动化构建与 GitLab CI/CD 发布流程，适合开发者优化构建发布。
date: 2023-12-04T08:18:00+08:00
draft: false
author: LQ
toc: true
categories:
- cloud-native
tags:
- 工程效率
- 云原生小技巧
---

![cover](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/automating-cli-tool-release/6964e503-54a2-490c-bd68-5de88cee5332.jpeg)

在云原生时代，CLI 工具已成为开发者日常工作中不可或缺的一部分。然而，将开发好的 CLI 工具分享给大家使用，如果仅依赖手动发布，不仅效率低，且易出错，特别是在处理多架构和多平台兼容性时尤为明显。

那么，我们如何才能实现 CLI 工具的自动化发布呢？本文旨在探讨这一问题，并提出一套实用的解决方案。

在接下来的分享中，我将主要以 Golang 举例。需要指出的是，我们将讨论的自动化构建和发布的原则是通用的，适用于所有编程语言。因此，无论大家使用哪种语言编写工具，这些实践都将具有重要的参考价值。

## 编写构建脚本

在自动化构建的世界中，编写一个稳定且跨平台兼容的构建脚本是关键。Golang 提供了强大的跨平台构建能力，而 `go build` 命令是实现这一目标的核心。例如：

```bash
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build -o foo-darwin-amd64 -v
```

这个命令示例突显了 Golang 在生成特定平台和架构二进制文件方面的灵活性。在构建脚本中，我们需要进一步扩展这种灵活性，以支持多平台构建需求。

1. **参数化和默认值设置**：
    - `OUTPUT_DIR` 和 `BINARY_NAME` 的设定允许用户自定义输出目录和二进制文件的名称，加强了脚本的通用性。
    - `BUILDPATH` 变量用于指定构建路径，是脚本运行的必需参数，保证了构建过程的稳定性。

2. **跨平台和架构支持**：
    - 通过设置 `BUILD_GOOS` 和 `BUILD_GOARCH` 变量，脚本能够灵活地处理不同操作系统和架构的构建需求，增加了适用性。
    - 这些变量的默认值通过 `go env` 获取，但也可以通过参数覆盖，提供了灵活性。
3. **动态输出路径**：
    - `OUT` 变量根据是否为发布版（`IS_RELEASE`），动态调整输出文件的命名和路径。这样的设计使得脚本能够根据不同的使用场景（如开发测试或正式发布）输出不同格式的文件名。
4. **特殊情况处理**：
    - 对 Windows 平台的特殊处理（`.exe` 扩展名）是必要的，因为 Windows 系统下的可执行文件通常需要这个扩展名。

### gobuild.sh 脚本

下面的 `gobuild.sh` 脚本是对上述原则的实践，将跨平台构建的复杂性转化为简单的命令行操作：

```bash
OUTPUT_DIR=${4:-"bin"}
BINARY_NAME=$(basename ${1})
BUILDPATH=./${1:?"path to build"}

BUILD_GOOS=${GOOS:-$(go env GOOS)}
BUILD_GOARCH=${GOARCH:-$(go env GOARCH)}
GOBINARY=${GOBINARY:-go}
LDFLAGS=$(version::ldflags)

if [ $# -ge 2 ] && [ -n $2 ]; then
  BUILD_GOOS=$2
fi

if [ $# -ge 3 ] && [ -n $3 ]; then
  BUILD_GOARCH=$3
fi

OUT=${OUTPUT_DIR}/${1:?"output path"}
if [ "${IS_RELEASE:-0}" == "1" ]; then
    OUT="${OUTPUT_DIR}/${BINARY_NAME}-${BUILD_GOOS}-${BUILD_GOARCH}"
    if [ "${BUILD_GOOS}" == "windows" ]; then
        OUT="${OUTPUT_DIR}/${BINARY_NAME}-${BUILD_GOOS}-${BUILD_GOARCH}.exe"
    fi
fi

CGO_ENABLED=0 GOOS=${BUILD_GOOS} GOARCH=${BUILD_GOARCH} ${GOBINARY} build \
        -ldflags="${LDFLAGS}" \
        -o "${OUT}" \
        "${BUILDPATH}"
```

这个脚本不仅适应了多平台和多架构的需要，还提供了足够的灵活性和可配置性，以适应不同的构建场景。

### 配合 Makefile 实现全自动化构建

进一步的，结合 `Makefile` 可以将构建过程自动化，提升效率：

```makefile
.PHONY: build-binaries

BUILD_SCRIPT_PATH := ./hack/gobuild/gobuild.sh
# 列出了需要构建的所有二进制文件，可管理多个项目的构建过程
BINARIES := cmd/foo cmd/bar

# 通过 ALLPLATFORMS 变量，我们定义了一系列目标平台和架构组合
ALLPLATFORMS := linux/amd64 linux/arm64 darwin/amd64 darwin/arm64 windows/amd64 windows/arm64

# 构建所有组合
build-binaries: $(foreach bin,$(BINARIES),$(foreach plat,$(ALLPLATFORMS),build-$(bin)-$(plat)))

# 构建规则模板
# 这个模板可以生成特定于每个二进制文件和平台组合的构建规则。
define BUILD_template
build-$(1)-$(2):
 IS_RELEASE=1 $$(BUILD_SCRIPT_PATH) $(1) $$(subst /, ,$$(word 1,$$(subst -, ,$(2)))) $$(subst /, ,$$(word 2,$$(subst -, ,$(2))))
endef

# 生成构建规则
# 我们自动为每个二进制文件和平台组合生成了具体的构建规则。
$(foreach bin,$(BINARIES),$(foreach plat,$(ALLPLATFORMS),$(eval $(call BUILD_template,$(bin),$(plat)))))
```

通过这个 `Makefile`，即使同时构建 `foo` 和 `bar` 这两个 CLI Tool 也变得异常简单。一条简单的命令 `make build-binaries` 就能触发整个构建流程，大大减少了人工干预，确保了构建过程的一致性和可靠性。

### 小结

通过上述详细的构建脚本和 Makefile 配置，我们可以看到，现代软件开发中自动化构建的强大功能和必要性。这种方法不仅提升了构建效率，也增强了软件的质量和稳定性。在云原生时代，自动化构建已成为提高开发团队效率和产品可靠性的关键策略。

## Release CLI tool on GitLab CI/CD

在构建脚本准备完毕后，接下来我们就可以将其集成到 CI 系统了，下面我以 `GitLab CI/CD` 为例。

在 GitLab CI/CD 的核心，是一系列定义明确的作业（Jobs），它们在代码提交时自动执行。对于完整的持续集成来说，这些作业通常包括构建（build）、测试（test）、代码审查（lint）等步骤。但在本文中，我们将重点关注自动发布流程。

### 触发自动发布的条件

自动发布流程是基于 Git 标签创建的。当开发者推送一个新标签到仓库时，GitLab CI/CD 会捕捉到这一事件，并启动预定义的发布流程。

```bash
rules:
  - if: $CI_COMMIT_TAG
```

这个条件确保只有在创建新标签时，才会启动后续的构建、上传和发布作业。

### Release Jobs

**步骤一：** 构建二进制文件，在 `build-binaries` 阶段，CI 会构建针对不同平台和架构的 CLI 工具二进制文件，确保构建过程的一致性和可重复性。

**步骤二：** 上传构建产物，待构建完成后，`upload` 阶段负责将二进制文件上传到 GitLab 的包管理器或其他存储位置。这为后续的发布提供了必要的资源。

**步骤三：** 发布到 GitLab，最后，在 `release` 阶段，CI 使用 `release-cli` 工具自动创建发布，并将构建的二进制文件作为发布的资产。

### Create releases from .gitlab-ci.yml

下面的 `.gitlab-ci.yml` 脚本是对上述发布流程的实践：

```yaml
stages:
  ...
  - build-binaries
  - upload
  - release

build-binaries:
  stage: build-binaries
  image: golang:1.21.1
  rules:
    - if: $CI_COMMIT_TAG
  script:
    - echo "Building binaries for all platforms and architectures..."
    - make build-binaries
  artifacts:
    paths:
      - bin

upload:
  stage: upload
  image: curlimages/curl:latest
  rules:
    - if: $CI_COMMIT_TAG
  script:
    - echo "Uploading binaries..."
    - >
      for binary in ./bin/*; do
        curl --header "JOB-TOKEN: $CI_JOB_TOKEN" \
             --upload-file $binary \
             "${PACKAGE_REGISTRY_URL}/$(basename $binary)";
      done

release:
  stage: release
  image: registry.gitlab.com/gitlab-org/release-cli:latest
  rules:
    - if: $CI_COMMIT_TAG
  script:
    - echo "Creating a release for $CI_COMMIT_TAG"
    - |
      ASSET_LINKS=""
      for binary in ./bin/*; do
        LINK="{\"name\":\"$(basename $binary)\", \"url\":\"${PACKAGE_REGISTRY_URL}/$(basename $binary)\"}"
        ASSET_LINKS="${ASSET_LINKS},${LINK}"
      done
      ASSET_LINKS="[${ASSET_LINKS:1}]"
    - >
      release-cli create \
        --name "Release $CI_COMMIT_TAG" \
        --tag-name $CI_COMMIT_TAG \
        --description "Created using the release-cli: $CI_COMMIT_REF_NAME-$CI_JOB_ID" \
        --ref $CI_COMMIT_SHA \
        --assets-link "$ASSET_LINKS"
```

以上示例将会构建 CLI 工具二进制文件，并将其上传到 Gitlab 发布页面。用户可以从 Gitlab  Release 页面查找并下载适合其平台的二进制包即可。

![Release v0.0.1](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/automating-cli-tool-release/e20db937-9d5a-4672-a31d-d5279b38253a.png)

> 有关详细的 GitLab CI 流程，可以参考项目：<https://gitlab.com/lqshow/clireleaseautomator>

{{< gitlab projectID="52718263" baseURL="https://gitlab.com/" >}}

### 小结

这个流程大大简化了 CLI 工具的发布过程，使得开发者能够专注于代码开发，而不是后续的构建和发布环节。自动化这些步骤意味着每次发布都是快速、一致且无误的，从而提高了软件的整体质量和可靠性。

## Release CLI tool use GoReleaser

不难发现，上述整个流程相对来说还是比较繁琐的，准备脚本的过程也比较麻烦，现在介绍一个让这个过程不那么痛苦的工具 [GoReleaser](https://goreleaser.com/ "GoReleaser")。

GoReleaser 是一个变革性的工具，特别是对于以 Golang 编写的项目。相比于传统的手动配置和脚本编写，GoReleaser 提供了一种更高效和简洁的自动化发布方法。

### GoReleaser 的优势

GoReleaser 的设计理念是“一次配置，处处运行”，它通过一个单一的配置文件，即可控制整个发布流程。这个配置文件定义了如何构建二进制文件、如何打包它们、如何处理版本信息以及如何发布到各种平台。具体来说，GoReleaser 的优势包括：

1. **简化的构建过程**：通过预定义的模板，GoReleaser 能够自动构建针对不同平台和架构的二进制文件，无需编写复杂的脚本。
2. **灵活的打包和发布**：支持多种格式的打包选项，以及与主要代码托管平台的无缝集成。
3. **高度可配置**：从构建选项到发布设置，GoReleaser 允许高度定制化，以满足不同项目的需求。

### 配置和使用 GoReleaser

使用 GoReleaser 的第一步是在项目的根目录下创建 `.goreleaser.yml` 配置文件。通过 `goreleaser init` 命令可快速生成初始配置。这个文件涵盖了构建、打包和发布的全过程。

在配置好 `.goreleaser.yml` 之后，我们需要调整 `.gitignore` 加上 `dist`，因为 goreleaser 会默认把编译编译好的文件输出到 `dist` 目录中。

接下来我们看个例子：

```yaml
# .goreleaser.yml 示例
builds:
  - id: fooctl
    binary: fooctl
    main: ./cmd/fooctl
    ldflags:
    - -s -w
    - -X gitlab.com/lqshow/clireleaseautomator-with-goreleaser/version.gitVersion={{.Version}}
    - -X gitlab.com/lqshow/clireleaseautomator-with-goreleaser/version.gitCommit={{.ShortCommit}}
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64
  - id: barctl
    binary: barctl
    main: ./cmd/barctl
    ldflags:
    - -s -w
    - -X gitlab.com/lqshow/clireleaseautomator-with-goreleaser/version.gitVersion={{.Version}}
    - -X gitlab.com/lqshow/clireleaseautomator-with-goreleaser/version.gitCommit={{.ShortCommit}}
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64
```

这个简单清晰的配置文件事实上包含了我之前介绍的两块内容，相当于于省去了写 shell 脚本和 Makefile 文件，使整个过程更加灵活和高效。

### GitLab CI 中的 GoReleaser 集成

在 `.gitlab-ci.yml` 文件中，我们只需要定义一个简单的 release 作业：

```yaml
stages:
  - release

release:
  stage: release
  image:
    name: goreleaser/goreleaser
    entrypoint: ['']
  only:
    - tags
  variables:
    GIT_DEPTH: 0
    GITLAB_TOKEN: $GITLAB_TOKEN
  script:
    - goreleaser release --clean
```

只要查看运行日志，其实我们就会发现，`GoReleaser` 将自动执行，包括构建、上传和发布的整个流程。

> 具体详情：<https://gitlab.com/lqshow/clireleaseautomator-with-goreleaser/-/jobs/5669211977>

### 结果展示

通过 GoReleaser 发布的结果在 GitLab 的 release 页面清晰可见。如下图所示，每个构建的二进制文件都被自动上传并与相应的发布关联。

![Release v0.0.1](https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/automating-cli-tool-release/4b2c29d2-2ed7-4782-8d00-6ff8730ad45a.png)

> **详细项目可参考**：<https://gitlab.com/lqshow/clireleaseautomator-with-goreleaser>

{{< gitlab projectID="52722059" >}}

<center>
    <img src="https://cdn.jsdelivr.net/gh/cloud-native-101/files@main/imgs/emoji/nice.gif" width="40%" alt="nice" />
</center>

## 写在最后

本文探索的自动化构建和发布流程，不仅凸显了现代软件开发中效率和一致性的重要性，更是云原生时代快速迭代和持续交付文化的体现。这种自动化的力量使得开发者能够将更多的精力投入到创新和优化代码上，而非消耗在重复性和繁琐的任务上。

在技术不断进步的今天，自动化不再是一个选择，而是一种必要。它为我们提供了一种更加高效和可靠的方式来应对快速变化的开发需求，使我们能够更快地适应市场和用户的需求，推动软件开发行业不断前进。

好了，今天的分享就到这里，感谢你的阅读！🙌🏻😁📃 期待我们的下次见面！👋🚀
