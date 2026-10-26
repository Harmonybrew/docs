# 如何贡献 formula

## 1 前言

向系统级软件仓库录入软件包并非易事。在业界，这项工作通常由深度用户或系统开发者完成，而非普通用户。

如果你目前的开发经验尚处于“只会调包，不会传包”的阶段，从未向官方 Homebrew 或主流 Linux 发行版（如 Debian、Arch Linux 等）贡献过软件包，我们不建议你直接尝试向 Harmonybrew 提交贡献。

向 Harmonybrew 录入软件包的过程并不比其他软件仓库简单。相反，由于操作系统环境的特殊性以及基础设施的现状，适配工作面临更多挑战。如果你在成熟平台上尚且无法独立完成贡献，在这里只会更加寸步难行。

如果你认为自己已经准备好迎接这些挑战，那就来吧。欢迎开启 OpenHarmony 命令行生态的拓荒之旅。

## 2 前置知识学习

在贡献 formula 之前，请务必掌握以下资料：

<table>
  <thead>
    <tr>
      <th>资料</th>
      <th>介绍</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><a href="https://docs.brew.sh/">Homebrew Documentation</a></td>
      <td>
        Homebrew 官方文档。在“跑”之前请先学会“走”，请务必先成为 Homebrew
        深度用户，透彻了解其底层运作机制。
      </td>
    </tr>
    <tr>
      <td>
        <div style="min-width: max-content; display: block; white-space: nowrap;">
          <a href="https://docs.brew.sh/Adding-Software-to-Homebrew">Adding-Software-to-Homebrew</a>
        </div>
      </td>
      <td>
        Homebrew 官方贡献指南。Harmonybrew 的贡献流程尽可能与上游官方保持一致。
      </td>
    </tr>
    <tr>
      <td><a href="../user/FAQ.md">常见问题</a></td>
      <td>
        常见问题集。里面有介绍到 Harmonybrew 项目细节及 OpenHarmony
        平台的相关知识。
      </td>
    </tr>
  </tbody>
</table>

## 3 环境准备

### 3.1 准备 Docker 环境

为了解决 formula 构建过程中极其复杂的工具链依赖问题，并确保你的代码在提交后能够顺利通过流水线构建，Harmonybrew 的所有开发、调试与构建工作**必须在统一的 Docker 容器中完成。**

这种“容器化开发”模式有以下优势：

- **环境对齐**：你的本地环境与流水线环境 100% 一致，“在我的机器上能跑通”即意味着“在流水线上能跑通”。
- **工具集成**：容器内已预置好 ohos-sdk、make 等开发工具，无需手动配置复杂的编译环境。
- **无损撤销**：所有实验性修改均在容器内进行，不会污染你的宿主机系统。

<br>

由于我们使用的容器是一个 arm64 架构的鸿蒙容器，因此你首先需要准备一个能够运行 arm64 容器的环境。

相关要求如下：

- **硬件要求**：Arm 服务器。
- **网络要求**：构建过程中可能需要访问国外网站（例如从 github.com 下载源代码、从 proxy.golang.org 下载 Go 模块），需要确保自己的构建机拥有国际互联网环境。请直接选购香港或国外区域的 Arm 服务器作为构建机。请勿使用非法的代理服务。
- **软件要求**：已安装 Docker。

### 3.2 启动并进入容器

从华为云 SWR 拉取最新 ci-runner 镜像并启动：

```sh
docker pull swr.cn-north-4.myhuaweicloud.com/harmonybrew/ci-runner:latest
docker run -itd --name=ohos swr.cn-north-4.myhuaweicloud.com/harmonybrew/ci-runner:latest

# 进入容器环境，后续的所有步骤均在容器中进行
docker exec -it ohos sh
```

这个 ci-runner 镜像是我们的流水线镜像。它是 [DockerHarmony](https://github.com/hqzing/dockerharmony) 的二次封装，里面预置了必要的开发工具。我们将这个镜像对外公开，作为标准化的开发环境供贡献者使用。用户可从 SWR 拉取现成的镜像，或使用 [源码](https://atomgit.com/Harmonybrew/ci/blob/main/ci-runner/Dockerfile) 自行构建出镜像。

> **注意**：镜像会持续更新。建议在每次工作前执行一次 `docker pull` 更新镜像，并删除已有的旧容器、重新创建容器，确保每次使用的是最新、最干净的开发环境。

### 3.3 更新 Homebrew

执行一次手动更新，将 Homebrew 包管理器和核心 tap 更新至最新版本：

```sh
brew update
```

> 在这个容器中，默认设置了环境变量 `HOMEBREW_NO_AUTO_UPDATE=1`，全局禁用 Homebrew 自动更新。此举是为了避免开发者在工作过程中因 Homebrew 自动更新而丢失本地改动。因此，你每次工作前需要手动执行更新操作，以同步最新的包管理器和核心 tap 代码。

## 4 新增或修改 formula

### 4.1 formula 准入规则

我们的核心 tap 准入规则与上游保持一致。无论是新增 formula 还是修改 formula，都需要确保最终提交的 formula 符合上游的 [Acceptable-Formulae](https://docs.brew.sh/Acceptable-Formulae) 标准。

包括但不限于：
- 必须是开源软件
- 必须能从源码构建，禁止录入预构建的二进制文件
- 源代码必须来自这个开源软件的官方，不能使用第三方社区维护的 fork 版本
- 不能过于冷门
- ......

<br>

> **特例**：`ohos-sdk` 是核心仓库中唯一的特例。这套工具链目前无法在 OpenHarmony 平台上完成自举，因其作为生态基石的特殊地位，我们直接分发官方社区的构建版本。

在上游规则的基础上，本项目还额外增加如下约束：
- 统一英文注释：所有 formula 均需使用英文注释，以保持与上游仓库的语言一致。
- 遵循最小化修改原则：无论是基于上游 formula 进行适配，还是在现有 formula 基础上修改，都需确保改动点达到最少，使 formula 文件尽可能接近它在上游 Homebrew/homebrew-core 仓库中的状态，以便后续进行上游同步。
- 按需录入：请勿录入你自己并不使用的软件包。Harmonybrew 不以软件包数量为运营导向，相反，我们一直在刻意控制收录规模。软件包的录入绝非一次性工作，它将带来源源不断的后续维护成本。在维护者与贡献者数量有限的情况下，我们希望避免因过多冷门/低频软件包而分散精力。

### 4.2 新增 formula

**A. 搬运上游已有的 formula**

如果开源软件在上游 [Homebrew/homebrew-core](https://github.com/Homebrew/homebrew-core) 中已存在，**必须直接搬运上游 formula，严禁从零手写**。

这样做的目的是确保构建配置、依赖关系与上游保持一致，避免因打包差异导致连锁依赖失效。即便需要针对 OpenHarmony 环境做适配，也应在上游版本的基础上进行修改，并遵循最小化修改原则。若有特殊情况必须全量重写，请在 PR 中注明原因。

以 `sevenzip` 为例，搬运流程如下：

```sh
# 进入核心 tap 目录
cd $(brew --repo harmonybrew/core)

# 从 Homebrew 上游下载 formula 文件到对应的首字母子目录 (s)
# 若 Aliases 目录存在相关软链接，需一并创建
curl -fL -o Formula/s/sevenzip.rb https://raw.githubusercontent.com/Homebrew/homebrew-core/refs/heads/main/Formula/s/sevenzip.rb
ln -s ../Formula/s/sevenzip.rb Aliases/7-zip
ln -s ../Formula/s/sevenzip.rb Aliases/7zip

# 从源码安装（触发本地实时构建）
brew install -y -s -v --include-test sevenzip

# 执行冒烟测试，验证功能
brew test sevenzip
```

> **适配提示**：若上游 formula 在 OpenHarmony 环境下构建失败，可以尝试禁用不支持的可选功能，或制作专门的适配补丁。参考：[补丁制作指南](./make-patch.md)。

**B. 手写全新的 formula**

若上游仓库不存在该软件，则需手写全新的 formula。

全新 formula 除了需要符合上文列出的准入规则外，还需要通过 `brew style` 和 `brew audit` 校验。当前流水线门禁不会对这方面做强校验，但维护者看到这类 PR 会手工进行检查。校验不通过的 PR，如无特殊情况将会被拒收。

如果你的软件包具备可移植性，建议先将其录入上游 [Homebrew/homebrew-core](https://github.com/Homebrew/homebrew-core)，再从上游搬运回来，而不是直接往下游仓库录入。如此一来，你的软件包不仅能覆盖更多操作系统，还能尽可能减少被我们拒收的可能性。因为从上游搬运过来的 formula 我们通常不会拒收。

如果你的软件包因闭源、无法从源码构建等原因不满足核心仓库准入规则，请通过自定义 tap 的方式进行分发。参考：[通过自定义 tap 分发软件包](./custom-tap.md)。

### 4.3 修改 formula

如果有修改 formula 的需求，请直接在 homebrew-core 仓库目录进行修改。

以 `sevenzip` 为例，修改流程如下：

```sh
# 进入核心 tap 目录
cd $(brew --repo harmonybrew/core)

# 对 formula 文件进行修改
vim Formula/s/sevenzip.rb

# 从源码安装（触发本地实时构建）
brew install -y -s -v --include-test sevenzip

# 执行冒烟测试，验证功能
brew test sevenzip
```

修改 formula 时，需要注意修订（revision）版本号的问题：
- 如果你升级了软件版本，需要检查 formula 中是否存在 `revision *` 这种修订版本号声明。如果有，需要将其删除。因为升级软件版本后需要将修订版本号清零。
- 如果你未升级软件版本，且你的更改会导致构建产物发生改变，你应该递增修订版本号。
- 如果你未升级软件版本，且你的更改不会导致构建产物发生改变（例如更改注释、调整 formula 格式、级联依赖不变的情况下进行重构建），则不需要递增修订版本号。

<br>

递增修订版本号的方法：
- 如果 formula 里面本身已经有 `revision *`，你需要把数字加一。
- 如果 formula 里面还没有 `revision *`，你需要给它添加 `revision 1`。写法可参考现有的其他 formula。

## 5 多设备验证

这一步是可选步骤。如果你的修改点只能在鸿蒙 PC 真机上完成验证，你就需要做这一步。

首先在 ci-runner 中将 formula 打包成 bottle：

```sh
brew install -y -v --build-bottle <formula>
brew bottle --skip-relocation <formula>
```

将压缩包下载到鸿蒙 PC 上，使用 `brew install` 离线安装这个压缩包：
```sh
HOMEBREW_DEVELOPER=1 brew install <path-to-tarball>
```

现在你可以开展你的验证工作。

验证完成后，使用 `brew uninstall` 正常卸载：
```sh
brew uninstall <formula>
```

## 6 提交 PR

### 6.1 配置 SSH 密钥对

为了能够推送代码，需生成 SSH 密钥并将其公钥录入 [AtomGit 设置中心](https://atomgit.com/setting/key-ssh)。

```sh
# 生成密钥对（建议直接一路回车）
ssh-keygen -t ed25519

# 查看公钥内容。请将其添加到你的 AtomGit 账号上。
cat ~/.ssh/id_ed25519.pub
```

### 6.2 推送代码至个人仓库

如果你尚未 fork 过 [Harmonybrew/homebrew-core](https://atomgit.com/Harmonybrew/homebrew-core)，请先进行 fork。

随后在容器内将个人仓库添加为远程地址，并提交代码：

```sh
# 进入核心 tap 目录
cd $(brew --repo harmonybrew/core)

# 给 git 做配置，设置你的用户名和邮箱
git config --global user.email "you@example.com"
git config --global user.name "Your Name"

# 添加你的个人远程仓库（以用户 hqzing 为例）
git remote add my-fork git@atomgit.com:hqzing/homebrew-core.git

# 规范提交信息
git add .
git commit -m "sevenzip 26.00 (new formula)"

# 推送至个人仓的新分支（请勿直接推送到 main 分支）
git push my-fork main:sevenzip
```

提交信息（Commit Message）规范：

| 场景            | 格式                                   | 示例                                        |
| :-------------- | :------------------------------------- | :------------------------------------------ |
| 新增 formula    | `<formula> <version> (new formula)`    | `make 4.4.1 (new formula)`                  |
| 版本升级        | `<formula> <version>`                  | `ldapvi 1.8`                                |
| 修复/增强       | `<formula>: <action>`                  | `tcl-tk: update livecheck blocks`           |
| 修订版本升级    | `<formula>: revision bump <reason>`    | `lanraragi: revision bump (perl 5.44.0)`    |

<br/>

> “修复/增强”和“修订版本升级”都有可能会递增修订版本号。两者的区别是：“修复/增强”通常会改变 formula 业务代码，而“修订版本升级”通常不会改变业务代码，主要用于级联依赖升级导致软件包需要配合做重构建的场景。例如，因 `perl` 的升级导致 `lanraragi` 需要重构建，Commit Message 就需要这么写：`lanraragi: revision bump (perl 5.44.0)`。

提交规则约束：
- 一个 PR 不允许包含多个 commit。
- 一个 commit 不允许新增/修改多个 formula 文件，只允许新增/修改单个 formula 文件及其关联资源（Patches、Aliases 等）。
- 如果你提 PR 是为了修复问题，还需要注意一个 commit 只能修复一个问题。

### 6.3 代码自查

在真正将 PR 提交到社区仓库之前，需要先按照 [Pull Request 自查指南](./self-check-pr.md) 对自己的代码进行自查，确保自己提交的是一个合格的 PR。

### 6.4 提交 PR

去 AtomGit 网页上操作，从自己的个人仓往 [Harmonybrew/homebrew-core](https://atomgit.com/Harmonybrew/homebrew-core) 提交 PR。我们在仓库中设置了 PR 模板，提交 PR 时网页上会展示 PR 模板，你需要按照模板填写信息。

PR 被提交后，维护者需要进行对它进行评审。维护者为 PR 添加 `request-ci` 标签即视为评审通过。评审通过后，流水线会自动执行门禁检查、构建、测试、打包等动作。构建结束后，机器人会将构建日志链接发至 PR 讨论区。

如果流水线执行成功：
- 机器人自动生成“替代 PR”：机器人会自动生成一个新的 PR，标题里面会注明是你这个 PR 的替代 PR，里面包含你的原始 commit 和机器人自动生成的一条 commit。
- “替代 PR”与原始 PR 均会被合并：机器人会定期扫描并合并构建成功的 PR，当前扫描周期为 10 分钟。机器人检查到构建成功后，会首先合并“替代 PR”。随后，你的原始 PR 也会随之自动变更为“已合并”状态。
- 软件包将会入库：在 PR 合并后，流水线会继续在后台运行，重新生成、上传包索引，这个过程一般需要几分钟时间。流水线上传最新包索引后，你还需要等待 CDN 缓存刷新，最长需要 10 分钟时间。当这两个更新动作都完成了，其他用户就可以下载到你录入或修改的软件包。

<br>

如果流水线执行失败（含门禁失败和构建失败）：
- 贡献者需要修改代码：贡献者需要根据检查结果进行整改，整改完毕后使用 `git push -f` 强制更新 PR（确保每个 PR 只有一条 commit，否则无法通过门禁）。PR 更新后等待维护者重新进行评审，评审通过后流水线会再次被触发。改代码期间需要为 PR 标题加上 `[WIP]` 的前缀，修改完成后再将其删除。当 PR 标题中有 `[WIP]` 这个前缀，AtomGit 平台会将这个 PR 标记为草稿状态。这样可以避免维护者错误地对修改中的 PR 进行检视。

<br>

> **为什么要生成“替代 PR”**：因为 AtomGit 平台的能力相比于 GitHub 还是有所欠缺，并不支持设置 “Allow edits by maintainers”，而我们的业务非常依赖该特性（机器人在跑完流水线后需往 PR 中添加新的 commit）。在平台能力有限的情况下，只能通过这种妥协方式实现。

## 7 进阶玩法：使用 AI Agent 进行 formula 适配

如今的大模型能力已经相当强大，即使不预设任何知识库和技能集，它们自己也知道如何完成 formula 的鸿蒙适配。

在我们的核心 tap 中，对于那些不能直接继承上游的 formula，里面有一部分便是纯靠 AI 完成的适配，例如 `util-linux`、`openssh`、`gcc` 等。

这里以 Codex 工具为例，演示如何基于 AI Agent 完成一个 formula 的适配：

1. 在 ci-runner 中安装 Codex。

   ```sh
   brew install -y codex
   ```

2. 为 Codex 配置模型。

   以 deepseek-flash 为例：

   ```sh
   # 先运行一次 Codex，然后 Ctrl + C 关闭它，目的是让它生成 `~/.codex` 目录
   codex

   # 运行 DeepSeek 官方提供的一键配置脚本
   # 这里没有使用 DeepSeek 官方文档里面的 `bash <(...)` 写法，而是换了一种写法，这样能兼容 mksh
   curl -fsSL https://cdn.deepseek.com/api-docs/codex-deepseek-setup.sh | bash
   
   # 根据脚本里面的提示，输入操作指令和 API Key
   # ...
   ```

   完整官方文档看 [这里](https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/codex/)。

3. 启动 Codex

   ```sh
   codex
   ```

4. 下发指令，让 AI 自动完成构建、测试、问题修复。

   例如：

   ```txt
   你现在正位于一个鸿蒙（OpenHarmony）设备上，里面有各种开发工具和一个鸿蒙移植版本的 Homebrew。

   我想将 Homebrew 上游的 util-linux 这个 formula 移植到我的平台上，请帮我完成移植：
   https://raw.githubusercontent.com/Homebrew/homebrew-core/refs/heads/main/Formula/u/util-linux.rb

   注意事项：
   1. 如果需要制作和引用补丁，写法可参考仓库中现有的 formula。例如需要对软件包自身源码打补丁可参考 gcc、cmake、perl；需要对 node_modules 打补丁可参考 deepseek-harness；需要对 Rust crate 打补丁可参考 codex、cc-switch-cli。
   2. 如果需要在 C/C++ 源代码中添加条件编译宏，可使用 __OHOS__ 这个宏来作为判断条件，鸿蒙平台上的编译器会自动传入这个宏。
   3. 这个 Homebrew 将 OpenHarmony 视为一个特殊的 Linux 的发行版，因为在编译器视角上 OpenHarmony 平台的目标三元组是 `<arch>-linux-ohos`。在 formula 中，`OS.linux?` 和 `OS.ohos?` 的返回值都是 true。如果产生了鸿蒙特有的适配项，你必须将改动点用 `if OS.ohos?` 或 `unless OS.ohos?` 保护起来，并提供配套注释，不要让它影响 Linux 路径或全局路径。
   4. 改完代码后需要根据这些文档进行自查：
     - https://raw.atomgit.com/Harmonybrew/docs/raw/main/zh-CN/contributor/contribute-formula.md
     - https://raw.atomgit.com/Harmonybrew/docs/raw/main/zh-CN/contributor/self-check-pr.md。
   5. 禁止推送代码和创建 PR，这些动作必须由人类完成。
   ```

