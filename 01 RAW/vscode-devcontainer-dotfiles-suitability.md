# VS Code Dev Containers dotfiles 功能与本仓库的适用性

研究日期：2026-08-24

## 结论

适合个人开发容器初始化，建议以 VS Code User Settings 启用。当前仓库的根目录包含可执行的 `bootstrap.sh`，脚本在未传入参数时会把自身所在目录作为 chezmoi source，因此可以直接被 Dev Containers 的 dotfiles 安装流程调用。

这项功能适合把本仓库的完整个人环境带入容器。它会安装或校验 chezmoi、mise、Node、Pi、Pi packages、共享 skills，并应用 Neovim 与 Pi 配置。它仍需要一个独立的容器镜像或 `devcontainer.json` 来提供容器本身，个人设置也不会替团队成员或 CI 自动获得同样的配置。

## 官方行为

1. VS Code 官方文档把 `dotfiles.repository`、`dotfiles.targetPath` 和 `dotfiles.installCommand` 放在 User Settings 中，设置完成后，后续创建容器时使用该 dotfiles 仓库。项目仓库无需提交这些个人设置。

   来源：[Personalizing with dotfile repositories](https://code.visualstudio.com/docs/devcontainers/containers#_personalizing-with-dotfile-repositories)

2. Dev Containers CLI 的实现会在容器内执行 Git clone，默认目标为 `~/dotfiles`，进入目标目录后执行安装命令。安装文件缺少可执行位时，流程会先补上。未显式提供安装命令时，源码会按顺序查找 `install.sh`、`install`、`bootstrap.sh`、`bootstrap`、`script/bootstrap`、`setup.sh`、`setup` 和 `script/setup`。

   来源：[Dev Containers dotfiles implementation](https://github.com/devcontainers/cli/blob/main/src/spec-common/dotfiles.ts)

3. GitHub 简写仓库名会被实现转换为 HTTPS URL。使用完整的 SSH URL 时，源码会保留 SSH 形式。私有仓库仍需要容器内可用的 Git 凭据。官方支持通过 HTTPS credential helper 复用凭据，也支持转发本机 SSH agent。

   来源：[Dev Containers CLI options](https://github.com/devcontainers/cli/blob/main/src/spec-node/devContainersSpecCLI.ts)，[Sharing Git credentials with your container](https://code.visualstudio.com/remote/advancedcontainers/sharing-git-credentials)

4. 安装流程使用容器的远程用户 shell，并以该用户的 home 目录解析 `~`。因此同一配置在 root 容器中会落到 `/root`，在 `vscode` 等普通用户容器中会落到对应的 `/home/<user>`。这是根据 Dev Containers 的容器属性与安装源码，以及本仓库大量使用 `$HOME` 和 `os.homedir()` 得出的判断。

## 与本仓库的匹配情况

| 检查项 | 本仓库证据 | 判断 |
| --- | --- | --- |
| 根目录安装入口 | `bootstrap.sh` 位于仓库根目录，Git 文件模式为 `100755`；`.gitattributes` 要求 shell 文件使用 LF。 | 通过。扩展可以直接调用。 |
| 默认安装命令 | Dev Containers 默认候选列表包含 `bootstrap.sh`，本仓库没有更早的 `install.sh` 或 `install`。 | 通过。建议显式设置 `bootstrap.sh`，减少未来新增入口后的歧义。 |
| source 路径 | `bootstrap.sh:80-82` 在无参数时使用脚本所在目录；`bootstrap.sh:148-155` 接受包含 `.chezmoiversion` 的 source 目录。 | 通过。扩展 clone 到 `~/dotfiles` 后执行脚本，等价于 `--source-dir ~/dotfiles`。 |
| Linux 容器 | `bootstrap.sh:120-125` 要求 `uname` 结果为 Linux。仓库 README 将 Ubuntu、WSL 和其他 XDG Linux 作为支持范围。 | 通过，限 Linux 容器。 |
| 基础命令 | `bootstrap.sh:128-129` 要求 Git、curl、ssh；后续阶段还使用 `awk`、`sed`、`grep`、`sha256sum`、`ldd`、`install` 和 `mktemp`。 | 有条件通过。极简镜像需要补齐依赖。 |
| 用户权限 | chezmoi 和 mise 都优先安装到 `$HOME/.local/bin`，仓库计划也要求尽量避免管理员权限。 | 通过。推荐以实际开发用户执行，避免 root 安装后普通用户无法使用。 |
| 网络依赖 | `bootstrap.sh:97-117` 下载 chezmoi；`.chezmoiscripts/run_onchange_before_10-install-mise-and-tools.sh.tmpl:35-53` 下载 mise；`mise.toml:4-24` 声明 Node、Pi 和共享技能任务。 | 有条件通过。首次创建容器需要可访问 GitHub、mise、npm 及共享技能源。 |
| chezmoi 管理边界 | `.chezmoiignore:1-27` 排除 README、脚本、测试、认证文件、sessions、npm、node_modules、Git checkout、logs 和 cache。 | 通过。仓库本身的维护文件不会被当作 home 文件部署，认证与运行时目录保持在 Pi 管理范围外。 |
| 后台同步 | `README.md:82` 描述六小时同步；Linux 模板在 `systemd --user` 不可用时只输出提示并结束。 | 可用但需留意。普通容器通常没有持久的 systemd user manager，首次安装仍可完成，持续同步需要单独验证。 |
| 容器配置职责 | 当前仓库没有 `.devcontainer` 目录。官方 dotfiles 功能只负责容器创建后的个人化步骤。 | 需要补充容器配置或选择镜像。dotfiles 设置本身不会定义 Dockerfile、镜像、端口或 VS Code 扩展。 |

## 主要限制

### 私有仓库认证

扩展的 clone 发生在容器内，当前主机已经能够访问 GitHub 并不等于容器已经具备相同凭据。个人设置可以保存仓库位置，但凭据要通过官方支持的 credential helper 或 SSH agent 方式提供。

本仓库 README 使用 SSH 作为手动恢复路径。若继续使用 SSH，建议在本机 User Settings 中填写完整 SSH URL，并确认容器可以使用转发的 SSH agent。若使用 GitHub 简写形式，Dev Containers CLI 会改用 HTTPS，此时需要 HTTPS credential helper 在容器中可用。

### root 与普通用户

本仓库脚本将工具、chezmoi 配置、Pi 配置和共享技能写入执行用户的 home。root 容器下这些内容会进入 `/root`，普通用户连接后会看不到。推荐让 `remoteUser`、执行安装命令的用户和日常终端用户保持一致。

### 安装范围较大

本仓库面向完整的个人 Pi 环境。一次安装包含精确版本工具、多个 Pi package、共享技能同步、Neovim 配置、Pi launcher 和配置校验。对于只想获得 shell 配置的项目容器，这个入口会带来额外下载、启动时间和持久状态。

### 创建时初始化与后续更新

Dev Containers 的 dotfiles 流程带有一次性 marker，主要服务于容器创建阶段。仓库自己的同步脚本会尝试对 dotfiles checkout 执行 `git pull --ff-only`，再同步共享技能。容器销毁后，用户目录中的工具和缓存是否保留取决于容器或 volume 配置，不能只依赖这项设置获得跨容器持久化。

### 认证状态仍需单独恢复

仓库 README 将 `Configuration-restored` 与 `Fully-verified` 分开，`auth.json` 需要手动恢复或重新登录。容器初始化完成后，在线 provider 验证仍需单独执行，符合当前仓库的凭据边界。

## 建议的个人设置

下面的内容应放在 VS Code User Settings，使用实际私有仓库地址替换占位符，不写入项目仓库：

```json
{
  "dotfiles.repository": "git@github.com:<owner>/<private-repository>.git",
  "dotfiles.targetPath": "~/dotfiles",
  "dotfiles.installCommand": "bootstrap.sh"
}
```

如果容器使用 HTTPS credential helper，可以把第一项改为 GitHub 简写仓库名或 HTTPS URL。设置后先在一个可丢弃的 Linux 容器中验证，确认 Git、curl、ssh 已安装，确认私有仓库 clone 成功，再观察 `bootstrap.sh` 的各阶段输出。

## 最小验证清单

1. 使用与日常开发相同的 `remoteUser` 创建 Linux 容器。
2. 确认容器中存在 `git`、`curl`、`ssh`、`awk`、`sed`、`grep`、`sha256sum`、`ldd`、`install` 和 `mktemp`。
3. 确认 dotfiles clone 到预期的 `~/dotfiles`，并确认 `chezmoi source-path` 指向该目录。
4. 确认 `bootstrap.sh` 完成 Platform、Prerequisites、Chezmoi、Source configuration、Dry run、Apply 和 Skills sync 阶段。
5. 运行 `mise run verify`，再运行一次 chezmoi dry-run，确认第二次 apply 没有意外变更。
6. 确认 `~/.pi/agent/auth.json` 未被创建或覆盖，在线验证只在手动恢复凭据后执行。
7. 若需要六小时同步，单独验证 `systemd --user` 或改用容器生命周期命令。不要把持久同步能力视为 dotfiles 创建步骤的默认保证。

## 最终判断

本仓库可以采用 VS Code Dev Containers 的 dotfiles 功能作为个人容器的初始化入口，当前无需修改安装脚本，也无需把个人仓库地址写入项目级容器配置。采用前需要确认 Linux 基础镜像的命令依赖、远程用户、私有仓库认证和容器状态持久化策略。

这项功能适合个人工作流。若目标变为团队成员和 CI 使用同一套容器环境，应把容器镜像、Features、`devcontainer.json` 或等效的 CLI 参数单独纳入项目级方案。

补充：当前 `README.md:16`、`README.md:24` 和 `README.md:32` 已有手动恢复用仓库地址。本次研究没有新增个人地址；“无需写入”适用于新增的 `dotfiles.*` 设置，因为它们位于 VS Code User Settings。

## 参考来源

1. [VS Code：Personalizing with dotfile repositories](https://code.visualstudio.com/docs/devcontainers/containers#_personalizing-with-dotfile-repositories)
2. [Dev Containers CLI：dotfiles.ts](https://github.com/devcontainers/cli/blob/main/src/spec-common/dotfiles.ts)
3. [Dev Containers CLI：devContainersSpecCLI.ts](https://github.com/devcontainers/cli/blob/main/src/spec-node/devContainersSpecCLI.ts)
4. [VS Code：Sharing Git credentials with your container](https://code.visualstudio.com/remote/advancedcontainers/sharing-git-credentials)
5. 本仓库：[README.md](../../README.md)、[bootstrap.sh](../../bootstrap.sh)、[AGENTS.md](../../AGENTS.md)、[.chezmoiignore](../../.chezmoiignore)、[mise.toml](../../mise.toml)、[skills/sources.json](../../skills/sources.json)。
