# 贡献指南

## 各仓库贡献规则

本项目涉及多个代码仓库，由于仓库的来源和承载的业务不同，其贡献规则也各不相同。

<table>
  <thead>
    <tr>
      <th>仓库</th>
      <th>内容</th>
      <th>来源</th>
      <th>用途</th>
      <th>贡献规则</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><a href="https://atomgit.com/Harmonybrew/brew">brew</a></td>
      <td>Homebrew 客户端</td>
      <td>从上游 fork</td>
      <td>用户下载的鸿蒙版 Homebrew 包管理器来自这里。</td>
      <td>
          1. 使用英文提交信息、英文注释。提交信息规则与上游保持一致。<br>
          2. 尽可能减少对上游代码的修改，请勿提交针对个人偏好或自家企业定制需求的修改，请勿尝试“优化”上游代码。
      </td>
    </tr>
    <tr>
      <td><a href="https://atomgit.com/Harmonybrew/install">install</a></td>
      <td>Homebrew 安装脚本</td>
      <td>从上游 fork</td>
      <td>用户在鸿蒙系统上使用的安装脚本来自这里。</td>
      <td>
          1. 使用英文提交信息、英文注释。提交信息规则与上游保持一致。<br>
          2. 尽可能减少对上游代码的修改，请勿提交针对个人偏好或自家企业定制需求的修改，请勿尝试“优化”上游代码。
      </td>
    </tr>
    <tr>
      <td>
        <div style="min-width: max-content; display: block; white-space: nowrap;">
          <a href="https://atomgit.com/Harmonybrew/homebrew-core">homebrew-core</a>
        </div>
      </td>
      <td>formula 仓库</td>
      <td>绝大部分 formula 都是从上游搬运而来，也可以视为是上游的 fork</td>
      <td>用户下载的软件包是从这里面的 formula 构建得来。</td>
      <td>这个仓库有单独的文档：<a href="./contribute-formula.md">如何贡献 formula</a></td>
    </tr>
    <tr>
      <td><a href="https://atomgit.com/Harmonybrew/ci">ci</a></td>
      <td>流水线脚本</td>
      <td>新建</td>
      <td>
        维护流水线脚本。由于我们不使用 GitHub Action，而是使用 CodeArts Pipeline
        作为流水线平台，所以无法直接复用上游的流水线脚本，需要编写脚本自己实现流水线逻辑。
      </td>
      <td>由 Harmonybrew 维护团队例行维护，暂不接纳 PR 贡献。</td>
    </tr>
    <tr>
      <td><a href="https://atomgit.com/Harmonybrew/docs">docs</a></td>
      <td>项目文档</td>
      <td>新建</td>
      <td>维护项目文档。</td>
      <td>
        1. 使用中文提交信息、中文注释。<br>
        2. 每一句文档修改都会被维护团队严格审核，请确认自己的 PR 对社区有重要作用，否则会有很大概率会被拒收。
      </td>
    </tr>
    <tr>
      <td><a href="https://atomgit.com/Harmonybrew/pages">pages</a></td>
      <td>项目主页</td>
      <td>新建</td>
      <td>这里面的网页会被部署到 https://harmonybrew.atomgit.com</td>
      <td>使用中文提交信息、中文注释。</td>
    </tr>
    <tr>
      <td><a href="https://atomgit.com/Harmonybrew/.gitcode">.gitcode</a></td>
      <td>全局配置库</td>
      <td>新建</td>
      <td>当前仅存储组织级 Issue 模板，无其他内容。</td>
      <td>由 Harmonybrew 维护团队例行维护，暂不接纳 PR 贡献。</td>
    </tr>
    <tr>
      <td>
        <div style="min-width: max-content; display: block; white-space: nowrap;">
          <a href="https://atomgit.com/Harmonybrew/formula-migration-tool">formula-migration-tool</a>
        </div>
      </td>  
      <td>formula 自动化搬运工具</td>
      <td>新建</td>
      <td>本项目维护者开发的实用工具，旨在为用户提供便利。</td>
      <td>使用中文提交信息、中文注释。</td>
    </tr>
    <tr>
      <td>
        <a href="https://atomgit.com/Harmonybrew/uname-is-linux">uname-is-linux</a>
      </td>
      <td>系统标识伪装工具</td>
      <td>新建</td>
      <td>本项目维护者开发的实用工具，旨在为用户提供便利。</td>
      <td>使用中文提交信息、中文注释。</td>
    </tr>
    <tr>
      <td>
        <a href="https://atomgit.com/Harmonybrew/ohos-pip-autosign">ohos-pip-autosign</a>
      </td>
      <td>Python 三方库自动签名工具</td>
      <td>新建</td>
      <td>现已废弃</td>
      <td>/</td>
    </tr>
    <tr>
      <td>
        <a href="https://atomgit.com/Harmonybrew/musl-compat">musl-compat</a>
      </td>
      <td>用于补充 musl 符号的兼容垫片库</td>
      <td>新建</td>
      <td>本项目维护者开发的实用工具，旨在为用户提供便利。</td>
      <td>使用中文提交信息、中文注释。</td>
    </tr>
    <tr>
      <td>
        <a href="https://atomgit.com/Harmonybrew/custom-tap-example">custom-tap-example</a>
      </td>
      <td>自定义 tap 样例</td>
      <td>新建</td>
      <td>一个样例仓库，支撑相关教程文档。</td>
      <td>由 Harmonybrew 维护团队例行维护，暂不接纳 PR 贡献。</td>
    </tr>
    <tr>
      <td>
        <a href="https://github.com/orgs/Harmonybrew/repositories">ohos-xxx</a>
      </td>
      <td>各种鸿蒙移植版本的开源软件</td>
      <td>新建</td>
      <td>
        主要供 Harmonybrew 项目内部使用，用于构建
        <a href="https://atomgit.com/Harmonybrew/ci/blob/main/ci-runner/Dockerfile">ci-runner</a>
        镜像。但项目都是开源的，其他人想用也可以拿去用。这些仓库托管在
        GitHub 平台而不是 AtomGit 平台（为了使用 GitHub 的流水线）。
      </td>
      <td>由 Harmonybrew 维护团队例行维护，暂不接纳 PR 贡献。</td>
    </tr>
    <tr>
      <td>
        <a href="https://github.com/Harmonybrew/maintenance-tools">maintenance-tools</a>
      </td>
      <td>Harmonybrew 项目维护工具</td>
      <td>新建</td>
      <td>
         一些用于维护项目的小工具，为维护团队开展日常工作提供便利。普通用户无需关注。这个仓库托管在
         GitHub 平台而不是 AtomGit 平台（为了使用 GitHub 的流水线）。
      </td>
      <td>由 Harmonybrew 维护团队例行维护，暂不接纳 PR 贡献。</td>
    </tr>
  </tbody>
</table>

## 急需援手

维护团队精力有限，如果你想参与贡献但还在寻找切入点，我们非常希望在以下方向得到你的支持：

- **自动升级 PR 修复**：Harmonybrew 流水线每天会通过 `brew bump` 自动生成软件版本升级 PR（脚本源码见 [auto_bump.py](https://atomgit.com/Harmonybrew/ci/blob/main/src/auto_bump.py)），在 Harmonybrew/homebrew-core 仓库中每天会有大量此类 PR 生成。由于版本升级可能会需要重做补丁或调整构建配置，并非所有自动升级 PR 都能顺利通过构建与测试。欢迎大家前往 PR 列表认领并修复这些失败的 PR。普通用户请直接提交新 PR 代替，待你的 PR 合并后，维护者会手动关闭对应的自动升级 PR。处理此类问题时请优先对比上游 Homebrew 的 formula。如果是非鸿蒙的问题，上游一般都已经修复，大多时候只需要同步上游最新的 formula 即可解决。注意：这也是 PR 贡献，同样需要遵循贡献指南，请勿使用 AI Agent 或自动化脚本批量提交未经验证的 PR。
- **文档国际化**：目前 Harmonybrew/docs 仓库的文档缺乏英文版本，需要熟悉 Homebrew 的开发者协助翻译，以便与上游社区交流。
- **软件包适配**：目前仍有大量上游 formula 尚未完成鸿蒙适配，未录入到我们的下游 Harmonybrew/homebrew-core 仓库中，需要技术能力强的开发者参与这方面的贡献。
- **兼容性测试**：核心 tap 中的软件包并未经过充分测试，仅在录入时在流水线中执行了 `brew test` 冒烟测试，这些软件包在 OpenHarmony 设备上的兼容性是未知的，需要有人在不同设备上对其进行功能测试，将发掘出的问题以 Issue 形式反馈至 Harmonybrew/homebrew-core 仓库。建议优先测试热门软件包，软件包热度可在 [下载数据统计](https://harmonybrew.atomgit.com/stats/) 页面获知。
- **AI 能力建设**：当前我们的 Formula 移植工作流程并未充分发挥业界主流 AI 能力，尚处于“各自为战”的碎片化状态：AI 能力的调用仍依赖开发者的个人经验在本地分散完成。我们希望能沉淀出可复用的 Prompt 资产、自动化 Agent 脚本或专用工具，构建统一的 AI 辅助工作流，帮助开发者更高效地输出 Formula 适配补丁。
- **Issue 认领**：请关注本社区的 [Issues](https://atomgit.com/org/Harmonybrew/issues)，尤其是归属于 Harmonybrew/homebrew-core 仓库的 Issue，维护团队在这部分 Issue 上几乎不会投入精力，主要依靠社区开发者互帮互助处理。
