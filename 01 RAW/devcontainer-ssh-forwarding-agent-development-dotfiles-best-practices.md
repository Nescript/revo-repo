# Dev Container 使用最佳实践：宿主机 SSH 转发、Agent 开发环境与 dotfiles

研究日期：2026-08-25

研究目标：整理 Dev Container 中的宿主机 SSH agent 转发、Agent 开发环境部署和 dotfiles 使用方式，给出适用于 Windows、WSL、Docker Desktop 与 Linux 容器的配置路径、验证方法和安全边界。

## 一、研究范围与判断方式

本报告优先使用 Dev Container、Docker、Microsoft OpenSSH、Microsoft WSL、VS Code、GitHub 和 Zed 的官方文档或官方仓库源码。

文中的判断分为四类：

1. **来源直接支持**：官方文档或官方源码明确说明了行为、配置项或限制。
2. **跨来源归纳**：多份官方资料共同支持一条工作流判断。
3. **实践建议**：根据资料和开发环境边界设计出的建议，落地前仍需要在目标主机和目标工具版本中验证。
4. **待验证事项**：资料没有给出跨平台保证，或结果取决于主机、Docker 后端、编辑器和 CLI 的组合。

## 二、核心结论

1. 运行期 SSH 访问应优先使用宿主机 ssh-agent 转发。私钥保留在宿主机，容器通过 SSH_AUTH_SOCK 请求 agent 签名。VS Code Dev Containers 在宿主机 agent 已运行时会自动转发本地 agent，通常无需把 .ssh 目录挂入容器。[VS Code：Sharing Git credentials with your container](https://code.visualstudio.com/remote/advancedcontainers/sharing-git-credentials)

2. Windows 原生 OpenSSH agent、WSL 中的 Linux agent、Docker Desktop 和 Linux 容器处于不同的运行边界。应先确定 VS Code、devcontainer CLI 和 Docker daemon 分别运行在哪一层，再决定使用自动转发、Unix socket 挂载或 WSL 内的 agent。不能把一个平台的 socket 路径直接当作所有平台的通用配置。

3. Docker Desktop 官方提供的 /run/host-services/ssh-auth.sock 挂载方案适用于 Docker Desktop for Mac 和 Linux。Docker 官方页面没有把这条路径作为 Windows 配置方案。Windows 原生场景优先使用 VS Code 的自动转发，或把开发入口放到 WSL，使容器侧收到 Linux Unix socket。[Docker Desktop：Networking how-tos](https://docs.docker.com/desktop/features/networking/networking-how-tos/)

4. 构建期取私有依赖和运行期访问 Git 需要分开设计。Docker BuildKit 使用 RUN --mount=type=ssh 与 docker build --ssh 临时提供 SSH agent；运行期使用 devcontainer.json 的 mount 和环境变量。构建期 agent mount 不会自动成为运行期容器中的 agent。[Docker：Build secrets](https://docs.docker.com/build/building/secrets/)，[Dockerfile reference](https://docs.docker.com/reference/dockerfile)

5. Agent 开发环境建议分为项目运行时、团队容器配置、个人工具配置、凭据和持久化状态五层。项目运行时和团队所需工具进入镜像、Dockerfile、Features 或项目级 devcontainer.json。个人 shell、编辑器和 Agent CLI 偏好放入用户控制的 dotfiles。SSH agent、provider 登录状态和私有配置通过宿主机或用户级机制注入。

6. remoteUser、dotfiles 安装用户、日常终端用户和 Agent CLI 用户应保持一致。Dev Container 规范说明，remoteUser 影响终端、任务、调试器和生命周期脚本；Linux 下设置用户时，工具可以同步 UID/GID 以减少 bind mount 权限问题。[Dev Container JSON reference](https://github.com/devcontainers/spec/blob/main/docs/specs/devcontainerjson-reference.md)

7. VS Code 的 dotfiles 设置属于用户设置范围，适合保存个人仓库地址、目标路径和安装命令。项目级容器配置应保存团队共享的镜像、依赖、Features、端口、工作区和编辑器扩展。不要把 provider token、SSH 私钥、Agent auth 状态或个人 dotfiles 地址写入团队项目配置。[VS Code：Developing inside a Container](https://code.visualstudio.com/docs/devcontainers/containers)

## 三、运行边界与数据流

| 层 | 主要职责 | 典型数据 | 推荐放置位置 |
| --- | --- | --- | --- |
| Windows、macOS、Linux 或 WSL 宿主机 | 保存私钥并运行 SSH agent | 私钥、agent 进程、SSH 配置、已知主机 | 宿主机用户目录或系统凭据机制 |
| Docker client 或 Dev Container 工具 | 创建容器并建立 agent 转发 | mount 参数、SSH_AUTH_SOCK、remote user | 编辑器用户设置、devcontainer.json 或 CLI 参数 |
| Linux Dev Container | 使用 agent 完成 Git、SSH 和签名操作 | agent socket、Git、OpenSSH client | 容器运行时，避免把私钥复制进文件系统 |
| 项目工作区 | 保存源代码和可审查的配置 | 源代码、测试、.devcontainer | 宿主机 bind mount 或明确的工作区 volume |
| Agent 工具和缓存 | 提供 CLI、技能、语言运行时和缓存 | 工具二进制、包缓存、个人设置、日志 | 镜像、Feature、dotfiles 或明确命名的 volume |
| 凭据与登录状态 | 访问模型、代码托管和私有服务 | API token、auth 文件、registry 登录信息 | 宿主机凭据管理、短期 secret 或用户级状态目录 |

Docker 官方说明，容器 writable layer 随容器删除而丢失。需要跨容器保留的数据应放到 bind mount 或 named volume。bind mount 适合宿主机和容器共同编辑的文件，volume 适合由 Docker 管理并跨容器保留的数据。[Docker：Storage](https://docs.docker.com/engine/storage)

## 四、宿主机 SSH agent 配置

### 4.1 Windows 原生 OpenSSH

Microsoft 文档说明，Windows 的 ssh-agent 默认可能处于禁用状态。可以在管理员 PowerShell 中启用服务，再在用户 PowerShell 中加载密钥：

~~~powershell
# 管理员 PowerShell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
Get-Service ssh-agent

# 普通用户 PowerShell
ssh-add $env:USERPROFILE\.ssh\id_ed25519
ssh-add -l
~~~

密钥文件路径只作为示例，实际文件名应按当前账号已有密钥调整。Microsoft 说明，Windows agent 在当前 Windows 账号的安全上下文中保存私钥，并向 SSH client 提供签名服务。[Microsoft Learn：Key-Based Authentication in OpenSSH for Windows](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement)

确认宿主机访问代码托管服务：

~~~powershell
ssh-add -l
ssh -T git@github.com
~~~

GitHub 文档说明，ssh -T git@github.com 用于确认公钥认证是否成功。首次出现主机指纹提示时，应先与 GitHub 公布的指纹核对，再接受主机密钥。[GitHub：Testing your SSH connection](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/testing-your-ssh-connection)

在 VS Code 本地窗口中使用 Dev Containers 时，官方文档建议先保证 Windows agent 正常运行。Dev Containers 扩展会在检测到本地 agent 后自动转发它。[VS Code：Sharing Git credentials with your container](https://code.visualstudio.com/remote/advancedcontainers/sharing-git-credentials)

### 4.2 Linux、macOS 和 WSL 内的 Linux agent

在 Linux、macOS 或 WSL 中，开发进程应读取当前 shell 的 SSH_AUTH_SOCK：

~~~bash
eval "$(ssh-agent -s)"
ssh-add "$HOME/.ssh/id_ed25519"
printf 'SSH_AUTH_SOCK=%s\n' "$SSH_AUTH_SOCK"
ssh-add -l
ssh -T git@github.com
~~~

长期使用时，应结合当前系统的 keychain、登录服务或受控 shell 初始化方式，避免每个终端都启动一个新的 agent。VS Code 官方文档给出了 Linux 登录 shell 中维护 agent 的示例，也提醒 Windows 和 Linux 在 agent 未运行时会报错。[VS Code：Sharing Git credentials with your container](https://code.visualstudio.com/remote/advancedcontainers/sharing-git-credentials)

WSL 有独立的 Linux 文件系统和用户环境。Microsoft 文档说明，Windows 文件系统通过 /mnt/c 访问，WSL 用户目录位于独立的 Linux 文件系统中；WSL 中的 Git 和 Windows 中的 Git 也需要分别考虑配置和凭据。[Microsoft Learn：Get started using Git on WSL](https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-git)

当开发入口是 WSL 时，推荐采用以下路径：

1. 在 WSL 中启动或连接 Linux ssh-agent。
2. 在 WSL 中运行 ssh-add -l 和 ssh -T git@github.com。
3. 使用 Docker Desktop 的 WSL integration，让 WSL 终端能够调用 Docker。
4. 从 WSL 目录打开 VS Code 或运行 Dev Container 工具，让工具继承 WSL 中的 SSH_AUTH_SOCK。
5. 在容器内重新执行 ssh-add -l 和 GitHub SSH 测试。

Docker 官方说明，Docker Desktop 的 WSL 2 backend 支持在启用 integration 的 WSL distribution 中直接使用 Docker CLI。[Docker Desktop：WSL 2 backend on Windows](https://docs.docker.com/desktop/features/wsl/)

### 4.3 SSH 远程主机场景

如果 Docker daemon 位于另一台 SSH 主机，需要同时检查两段 SSH 配置：

~~~sshconfig
Host dev-docker.example.com
    ForwardAgent yes
~~~

远端 SSH server 需要允许 agent forwarding。GitHub 文档将 ForwardAgent yes 作为客户端配置，将 AllowAgentForwarding 作为服务器端配置，并明确警告不要对所有主机使用 Host *。只对可信且确实需要访问私有仓库的主机开启转发。[GitHub：Using SSH agent forwarding](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/using-ssh-agent-forwarding)

本地 Docker Desktop 的 Dev Container 创建流程没有经过 SSH server 时，重点是本地 agent 是否可被编辑器或 CLI 看见。此时不应为了容器内 Git 访问去开启 Windows sshd 服务。

## 五、devcontainer.json 中的 SSH 转发

### 5.1 VS Code 自动转发，推荐作为第一选择

使用 VS Code Dev Containers 时，项目配置可以保持简洁：

~~~jsonc
{
  "name": "agent-dev",
  "image": "ghcr.io/<org>/<dev-image>:<version>",
  "remoteUser": "vscode"
}
~~~

主机侧先完成：

~~~powershell
ssh-add -l
ssh -T git@github.com
~~~

容器侧验证：

~~~bash
printf 'SSH_AUTH_SOCK=%s\n' "$SSH_AUTH_SOCK"
test -n "$SSH_AUTH_SOCK"
test -S "$SSH_AUTH_SOCK" || test -e "$SSH_AUTH_SOCK"
ssh-add -l
ssh -T git@github.com
git ls-remote git@github.com:<org>/<private-repository>.git
~~~

ssh -T 成功访问 GitHub 时通常会显示账号问候语，并以退出码 1 结束，因为 GitHub 不提供 shell。验证脚本应检查输出或认证结果，避免只检查退出码。[GitHub：Testing your SSH connection](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/testing-your-ssh-connection)

如果 VS Code 的 Source Control 面板或终端看不到 agent，先确认以下边界：

1. VS Code 是从 Windows、WSL 还是另一台远程主机启动。
2. ssh-add -l 在启动 VS Code 的同一环境中是否有身份。
3. 项目使用的 Docker context 是否与预期一致。
4. 容器中的 SSH_AUTH_SOCK 是否指向实际存在的 socket。
5. 容器的 remoteUser 是否与预期的终端用户一致。

### 5.2 Linux、macOS 或 WSL 的手动 Unix socket 挂载

当工具没有自动转发能力，且宿主机已经提供可访问的 Unix socket 时，可以使用 Dev Container 规范支持的 mounts 和 containerEnv：

~~~jsonc
{
  "name": "agent-dev",
  "image": "ghcr.io/<org>/<dev-image>:<version>",
  "remoteUser": "vscode",
  "mounts": [
    "source=${localEnv:SSH_AUTH_SOCK},target=/tmp/ssh-agent.sock,type=bind"
  ],
  "containerEnv": {
    "SSH_AUTH_SOCK": "/tmp/ssh-agent.sock"
  }
}
~~~

这段配置只适用于启动该工具的宿主环境能够解析 ${localEnv:SSH_AUTH_SOCK}，并且 source 路径在创建容器前已经存在。Dev Container 规范允许在 mount 字符串中引用本地环境变量；Docker --mount 默认要求 bind source 已存在，路径缺失时会创建失败。[Dev Container JSON reference](https://github.com/devcontainers/spec/blob/main/docs/specs/devcontainerjson-reference.md)，[Docker：Bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)

containerEnv 会让容器内的所有进程都看到目标路径。remoteEnv 只影响 Dev Container 工具、终端和相关子进程。规范建议优先使用 containerEnv，需要跟随客户端变化的值可以考虑 remoteEnv。[Dev Container JSON reference](https://github.com/devcontainers/spec/blob/main/docs/specs/devcontainerjson-reference.md)

如果 agent 可选，项目应在宿主机初始化阶段给出清晰提示，避免无条件绑定一个不存在的 socket。规范中的 initializeCommand 在宿主机执行，适合做路径检查或生成平台专用的 Compose override；该命令的 shell 语法需要按启动工具所在平台分别实现。[Dev Container JSON reference](https://github.com/devcontainers/spec/blob/main/docs/specs/devcontainerjson-reference.md)

### 5.3 Docker Desktop for Mac 和 Linux 的 host-services socket

Docker 官方针对 Docker Desktop for Mac 和 Linux 给出的配置是：

~~~jsonc
{
  "mounts": [
    "source=/run/host-services/ssh-auth.sock,target=/run/host-services/ssh-auth.sock,type=bind"
  ],
  "containerEnv": {
    "SSH_AUTH_SOCK": "/run/host-services/ssh-auth.sock"
  }
}
~~~

Docker 官方文档明确把这条方案列在 Mac 和 Linux 小节中。该文档没有把它作为 Windows 原生 Docker Desktop 的通用配置，因此 Windows 配置不应直接复制这条路径。[Docker Desktop：Networking how-tos](https://docs.docker.com/desktop/features/networking/networking-how-tos/)

### 5.4 Windows 原生 Docker Desktop 的平台边界

Windows 的 OpenSSH agent 使用 Windows 服务和 Windows 侧的 agent 传输机制。公开的 VS Code 文档给出了 Windows agent 启动方式，并说明 Dev Containers 扩展会自动转发本地 agent。Docker 官方的 host-services socket 示例只覆盖 Mac 和 Linux。基于这两条资料，推荐顺序如下：

1. 使用 VS Code Dev Containers 时，启用 Windows ssh-agent，在同一 Windows 用户环境中运行 ssh-add -l，让扩展自动处理转发。
2. 如果开发入口在 WSL，使用 WSL 内的 Linux agent，从 WSL 启动 VS Code 或 Dev Container CLI。
3. 如果使用独立的 devcontainer CLI、Zed 或其他工具，先验证该工具对 Windows agent 的支持。若工具无法把 Windows agent 转换为容器可用的 Unix socket，改用 WSL agent 或 HTTPS credential helper。
4. 不把 C:\Users\<user>\.ssh 直接挂载到 Linux 容器，也不把 Windows agent 的命名管道路径写成未经验证的 Linux bind mount。

第 3 项属于跨平台实践建议。目标工具、Docker backend 和版本不同，Windows named pipe 的处理方式也可能不同，当前报告没有给出一个可跨工具保证的 npipe 配置。

## 六、SSH 转发的验证流程与故障定位

### 6.1 最小验证顺序

按以下顺序执行，可以把问题定位在宿主机、编辑器、容器 mount 或远程 Git 认证中的一层：

~~~text
宿主机 agent
  -> 启动编辑器或 CLI 的环境
  -> Docker/Dev Container 创建日志
  -> 容器中的 SSH_AUTH_SOCK
  -> 容器中的 ssh-add -l
  -> 容器中的 ssh -T git@github.com
  -> 目标仓库的 git ls-remote
~~~

建议命令：

~~~bash
printf 'SSH_AUTH_SOCK=%s\n' "$SSH_AUTH_SOCK"
test -n "$SSH_AUTH_SOCK"
test -S "$SSH_AUTH_SOCK" || test -e "$SSH_AUTH_SOCK"
ssh-add -l -E sha256
ssh -vT git@github.com
git remote -v
git ls-remote git@github.com:<org>/<private-repository>.git
~~~

### 6.2 常见症状

| 症状 | 优先检查 | 处理方向 |
| --- | --- | --- |
| 宿主机 ssh-add -l 报 agent unavailable | agent 服务、当前 shell、当前用户 | 启动对应平台 agent，再加载密钥 |
| 容器中 SSH_AUTH_SOCK 为空 | 编辑器启动环境、remoteEnv、自动转发日志 | 从拥有 agent 的环境启动工具，或设置正确的容器变量 |
| socket 路径存在，ssh-add -l 没有身份 | socket 指向另一套 agent、WSL 与 Windows agent 混用 | 在启动工具的同一环境中重新检查 ssh-add -l |
| Docker 创建时提示 bind source 不存在 | agent 是否启动、localEnv 是否传入 | 启动 agent，或让配置先检查 socket 再创建 mount |
| ssh-add -l 有身份，GitHub 仍拒绝 | Git remote URL、GitHub 公钥、主机指纹、网络 | 使用 ssh -vT 和 git remote -v 区分认证与 URL 问题 |
| VS Code pull/sync 卡住 | 带 passphrase 的 SSH key 与远程 Source Control 流程 | 按官方限制改用命令行 Git，或改用 HTTPS credential helper |
| Root 中能用，普通用户中不能用 | dotfiles、agent 环境和 HOME 的归属 | 让 remoteUser、dotfiles 安装用户和日常用户保持一致 |

VS Code 文档记录了带 passphrase 的 SSH key 在远程 pull 和 sync 中可能导致卡住的限制，并建议使用命令行 Git 或 HTTPS 作为绕过路径。[VS Code：Developing inside a Container](https://code.visualstudio.com/docs/devcontainers/containers)

## 七、构建期 SSH 与运行期 SSH

### 7.1 构建期访问私有依赖

Docker BuildKit 的 SSH mount 用于让某条 RUN 指令临时访问 SSH agent：

~~~dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.21

RUN apk add --no-cache git openssh-client

RUN --mount=type=ssh,id=default,required=true \
    git clone --depth 1 git@github.com:<org>/<private-dependency>.git /opt/private-dependency
~~~

构建时传入 agent：

~~~bash
ssh-add -l
docker buildx build --ssh default .
~~~

Docker 文档说明，RUN --mount=type=ssh 支持通过 SSH agent 或 key 为构建提供认证，required=true 可以在凭据缺失时让该步骤失败。SSH mount 只在对应的 RUN 指令期间可见。私钥和 agent socket 不应通过 COPY、ARG 或 ENV 写入镜像。[Dockerfile reference](https://docs.docker.com/reference/dockerfile)，[Docker：Build secrets](https://docs.docker.com/build/building/secrets/)

使用 Docker Compose 构建时，可以在 build 配置中声明：

~~~yaml
services:
  dev:
    build:
      context: .
      ssh:
        - default
~~~

Docker Compose Build Specification 同样支持把 default agent 或指定的 socket、PEM 文件传给构建器。[Compose Build Specification](https://docs.docker.com/reference/compose-file/build/)

### 7.2 运行期访问 Git

运行期容器需要：

1. 容器内安装 openssh-client 和 Git。
2. 容器中存在可访问的 agent socket。
3. SSH_AUTH_SOCK 指向该 socket。
4. Git 使用 SSH 风格 remote URL。
5. 远程主机密钥已经经过核对并写入受控的 known_hosts。

构建期 --ssh 解决镜像构建过程中的一次性访问。它不会替运行中的 Dev Container 建立长期 agent 转发。运行期需要独立的 devcontainer.json 配置或编辑器自动转发。

### 7.3 凭据安全边界

Docker 官方说明，build arguments 和普通环境变量不适合传递 secret，因为它们可能进入镜像历史、最终镜像或构建证明。需要在构建期间使用敏感数据时，应使用 secret mount 或 SSH mount。[Docker：Build secrets](https://docs.docker.com/build/building/secrets/)

运行期也应保持以下边界：

- 私钥只留在宿主机或硬件、系统凭据机制中。
- 容器只拿到 agent socket，不挂载整个 .ssh 目录。
- 只给可信代码和可信工具转发 agent。
- 只对指定的可信远程主机开启 ForwardAgent yes，不要使用 Host *。
- 为不同环境考虑使用不同密钥、短期凭据或 HTTPS credential helper。
- 不把 provider API key、Agent auth 文件或个人登录状态写入镜像、项目仓库和公共日志。

GitHub 明确说明，转发后的服务器无法直接读取私钥，但在连接期间可以借助 agent 以用户身份发起签名操作，因此应限制可信主机范围。[GitHub：Using SSH agent forwarding](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/using-ssh-agent-forwarding)

## 八、Agent 开发环境部署

### 8.1 五层部署模型

| 层 | 放入内容 | 推荐实现 |
| --- | --- | --- |
| 项目运行时 | ROS、编译器、系统库、语言运行时 | 基础镜像或 Dockerfile |
| 团队工具 | Git、调试器、常用 CLI、共享语言工具 | Dev Container Features 或预构建镜像 |
| 项目连接配置 | workspace、端口、项目扩展、生命周期脚本 | 项目级 devcontainer.json |
| 个人 Agent 环境 | shell、编辑器偏好、个人 CLI、技能目录、别名 | 用户管理的 dotfiles 或用户级 bootstrap |
| 凭据与个人状态 | SSH agent、模型登录、私有 registry、缓存 | 宿主机凭据机制、短期 secret、明确的用户 volume |

这样划分可以让项目成员获得一致的编译和测试环境，也能让个人 Agent 工具保持独立。项目容器应能在没有个人 token 和私有 dotfiles 的情况下完成基础构建、测试和静态检查。

### 8.2 镜像、Dockerfile、Features 和 lifecycle

Dev Container 规范支持 image、Dockerfile、Docker Compose、Features、mounts、用户、环境变量和生命周期脚本。规范说明，devcontainer.json 的重点是丰富一个开发容器，复杂的多容器编排应交给 Compose 等编排格式。[Dev Container JSON reference](https://github.com/devcontainers/spec/blob/main/docs/specs/devcontainerjson-reference.md)

推荐分工：

1. 基础镜像提供 Linux 发行版和稳定系统工具。
2. Dockerfile 安装项目必须的系统依赖、编译器和固定版本工具。
3. Features 提供可复用的语言、Git、GitHub CLI 或调试工具。
4. customizations 保存项目需要的编辑器扩展。Zed 官方 Dev Container 文档支持在 customizations.zed.extensions 中声明 Zed 扩展。[Zed：Dev Containers](https://zed.dev/docs/dev-containers)
5. postCreateCommand 用于容器首次创建后的项目初始化，例如创建虚拟环境、安装项目依赖和生成本地构建目录。
6. postStartCommand 用于每次启动都需要执行的轻量操作，例如启动开发服务。
7. 需要在所有开发者之间一致的工具优先进入镜像、Dockerfile 或 Feature，避免依赖某个用户的 shell 初始化文件。

Dev Container 规范区分了 onCreateCommand、updateContentCommand、postCreateCommand、postStartCommand 和 postAttachCommand。其中 postCreateCommand 在容器首次分配给用户后执行，postStartCommand 在容器每次成功启动时执行；生命周期脚本失败时，后续脚本会被跳过。[Dev Container JSON reference](https://github.com/devcontainers/spec/blob/main/docs/specs/devcontainerjson-reference.md)

预构建镜像适合稳定的团队工具和基础运行时。VS Code 官方建议预构建开发镜像，以减少每次打开容器的构建时间，并通过固定工具版本降低上游更新带来的变动。[VS Code：Developing inside a Container](https://code.visualstudio.com/docs/devcontainers/containers)

Docker 官方进一步建议把基础镜像固定到 digest，以获得可重复构建。例如：

~~~dockerfile
FROM ubuntu:24.04@sha256:<verified-digest>
~~~

实际 digest 应通过团队允许的镜像来源和更新流程维护。[Docker：Building best practices](https://docs.docker.com/build/building/best-practices/)

### 8.3 工作区、缓存和持久化

工作区源代码通常使用 bind mount，让宿主机编辑器、容器编译器和 Git 看到同一份文件。编译输出、包缓存、ROS 构建目录和日志需要按写入频率与共享需求分别处理：

- 源代码、测试和项目配置使用 workspace bind mount。
- 高写入量且需要跨容器保留的缓存使用 named volume。
- 需要宿主机直接查看的地图、日志和构建产物使用明确的宿主机目录。
- 临时密钥、一次性下载和运行期中间文件使用 tmpfs 或容器临时层。
- 对每个持久化目录写清楚所有者、备份策略和删除影响。

Docker 文档将 bind mount 定义为宿主机与容器共享路径，将 volume 定义为由 Docker 管理且可跨容器保留的数据。[Docker：Storage](https://docs.docker.com/engine/storage)

### 8.4 用户与权限

容器中 Agent CLI、shell、dotfiles 和项目构建应尽量使用同一个普通用户。推荐：

~~~jsonc
{
  "remoteUser": "vscode",
  "updateRemoteUserUID": true
}
~~~

remoteUser 会影响 VS Code server、终端、任务、调试器和生命周期脚本。Linux bind mount 场景下，UID/GID 同步可以减少宿主机文件和容器用户之间的权限冲突。[Dev Container JSON reference](https://github.com/devcontainers/spec/blob/main/docs/specs/devcontainerjson-reference.md)，[VS Code：Add a non-root user](https://code.visualstudio.com/remote/advancedcontainers/add-nonroot-user)

Windows Docker Desktop 中的 Linux bind mount 权限表现与原生 Linux 不同。VS Code 文档说明，Windows 挂载文件在容器中可能显示为 root 所有，但指定的容器用户仍可以读写。该行为应在目标项目中通过创建文件、编译和 Git 操作验证。[VS Code：Add a non-root user](https://code.visualstudio.com/remote/advancedcontainers/add-nonroot-user)

### 8.5 Agent 环境的最小项目配置

下面示例只包含团队共享内容，不包含个人 dotfiles 地址和认证信息：

~~~jsonc
{
  "name": "agent-dev",
  "image": "ghcr.io/<org>/<dev-image>:<version>",
  "remoteUser": "vscode",
  "features": {
    "ghcr.io/devcontainers/features/common-utils:2": {}
  },
  "workspaceMount": "source=${localWorkspaceFolder},target=/workspaces/<repo>,type=bind",
  "workspaceFolder": "/workspaces/<repo>",
  "customizations": {
    "vscode": {
      "extensions": [
        "<team-required-extension>"
      ]
    },
    "zed": {
      "extensions": [
        "<team-required-extension>"
      ]
    }
  },
  "postCreateCommand": [
    "bash",
    "-lc",
    "./scripts/bootstrap-project.sh"
  ]
}
~~~

team-required-extension 和 bootstrap-project.sh 只表示项目占位符。实际仓库应根据团队工具和脚本决定是否保留。项目初始化脚本应可重复执行，并避免把用户 token 写入日志。

## 九、dotfiles 部署

### 9.1 VS Code 用户设置

VS Code Dev Containers 支持在 User Settings 中配置：

~~~json
{
  "dotfiles.repository": "<owner>/<private-dotfiles-repository>",
  "dotfiles.targetPath": "~/dotfiles",
  "dotfiles.installCommand": "bootstrap.sh"
}
~~~

这三项分别指定 dotfiles 仓库、容器内目标路径和安装入口。VS Code 文档说明，这组设置属于用户设置，之后创建容器时会使用该 dotfiles 仓库。[VS Code：Developing inside a Container](https://code.visualstudio.com/docs/devcontainers/containers)

私有仓库有两条认证路线：

1. 使用 SSH URL，让创建容器时的 Git clone 通过已转发的 SSH agent 完成。
2. 使用 HTTPS URL 或仓库简写，让 Dev Containers 使用可用的 HTTPS credential helper。

完整的 devcontainers/cli dotfiles 源码显示，仓库简写会转换为 https://github.com/<repo>.git，目标路径中不存在仓库时执行 shallow clone，之后进入目标目录运行安装命令。未指定安装命令时，CLI 会按顺序查找 install.sh、install、bootstrap.sh、bootstrap、script/bootstrap、setup.sh、setup 和 script/setup。[Dev Containers CLI：dotfiles.ts](https://github.com/devcontainers/cli/blob/main/src/spec-common/dotfiles.ts)

实践中建议显式设置 dotfiles.installCommand，减少安装入口自动发现带来的歧义。安装入口应：

- 可重复执行。
- 不包含真实 token、私钥和登录状态。
- 以当前 remoteUser 的 HOME 为准。
- 能在没有 systemd user manager 的普通容器中完成基础初始化。
- 对需要在线登录的步骤给出提示，不把登录结果写入项目仓库。

### 9.2 个人配置与项目配置

| 配置内容 | 推荐位置 | 原因 |
| --- | --- | --- |
| 基础镜像、系统库、编译器 | Dockerfile、预构建镜像 | 团队和 CI 需要一致 |
| 语言、Git、调试器等共享工具 | Features 或镜像 | 可复用、可版本化 |
| 工作区路径、端口、项目脚本 | 项目级 devcontainer.json | 与项目行为直接相关 |
| 项目需要的 VS Code 或 Zed 扩展 | customizations | 方便新成员进入项目 |
| 个人 shell、编辑器偏好、Agent CLI 习惯 | User Settings 的 dotfiles | 属于个人工作流 |
| 私有 dotfiles 仓库地址 | 编辑器 User Settings 或本地 CLI 配置 | 避免进入团队项目 |
| SSH 私钥、模型 token、provider 登录状态 | 宿主机凭据机制或用户级状态 | 降低泄露和误提交风险 |

Root 容器和普通用户容器的 HOME 不同。若 dotfiles 安装阶段使用 root，工具和配置可能写入 /root；日常终端使用 vscode 时，会读取 /home/vscode。因此应先决定实际工作用户，再统一 remoteUser、dotfiles 安装用户、Agent CLI 用户和个人状态目录。

### 9.3 Zed 的当前边界

Zed 官方文档说明，仓库中存在 .devcontainer/devcontainer.json 时，Zed 可以通过 Dev Container 打开项目，并支持 customizations.zed.extensions。Zed 使用 devcontainer CLI 创建容器；修改 devcontainer.json 后，当前版本不会自动重建或重新加载已有容器，需要手动停止或结束旧容器后重新打开。[Zed：Dev Containers](https://zed.dev/docs/dev-containers)，[Zed：Run Your Project in a Dev Container](https://zed.dev/blog/dev-containers)

截至本报告研究日期，Zed 的官方 Dev Container 文档没有列出 VS Code 的 dotfiles.repository、dotfiles.targetPath 和 dotfiles.installCommand 设置。因此，Zed 中的 dotfiles 自动安装能力属于待验证事项。推荐使用以下方式：

1. 把项目容器配置放入项目的 .devcontainer/devcontainer.json。
2. 把个人 dotfiles 放在用户控制的 bootstrap 或 CLI 流程中。
3. 使用与 remoteUser 一致的用户运行 bootstrap。
4. 在 Zed 连接容器后单独执行用户级初始化。
5. 不把个人仓库 URL、API key 和认证文件加入项目配置。

## 十、Windows、WSL、Docker Desktop 与 Zed 的推荐流程

### 10.1 推荐决策

| 使用入口 | agent 位置 | 容器配置 | 推荐度 |
| --- | --- | --- | --- |
| Windows VS Code + Docker Desktop | Windows ssh-agent | VS Code 自动转发 | 首选 |
| WSL VS Code + Docker Desktop WSL integration | WSL Linux agent | VS Code 自动转发或 Unix socket mount | 首选 |
| Linux/macOS + Docker Desktop | 本机 Unix agent | /run/host-services/ssh-auth.sock 或普通 Unix mount | 首选 |
| Zed + Docker Desktop Windows | Windows agent 或 WSL agent | 先验证 Zed 与 devcontainer CLI 的 agent 路径 | 需要验证 |
| CI 或无交互构建 | CI 提供的短期 agent 或 key | BuildKit --ssh 或 HTTPS credential helper | 按 CI 安全策略 |

### 10.2 建议的实施顺序

1. 先在宿主机验证 agent 和 GitHub SSH。
2. 再用与日常工作相同的编辑器和 Docker context 创建一个最小 Linux 容器。
3. 让容器使用普通 remoteUser，确认 HOME、PATH 和 SSH_AUTH_SOCK。
4. 先验证 ssh-add -l，再验证 ssh -T，最后验证私有仓库的 git ls-remote。
5. 基础链路稳定后，再加入 Agent CLI、共享技能、语言工具和项目初始化脚本。
6. 最后加入持久化 volume、缓存、图形显示、GPU、ROS 设备或多容器服务。
7. 每次修改 .devcontainer 后重建或重新创建容器，避免把旧容器状态当作新配置结果。

## 十一、验收清单

### 宿主机

- ssh-add -l 能列出预期的公钥指纹。
- ssh -T git@github.com 能完成目标账号认证。
- 工作区 Git remote 使用预期的 SSH 或 HTTPS 形式。
- Docker Desktop、WSL integration 和当前 Docker context 与工作流一致。

### 容器

- command -v git ssh ssh-add 均能找到可执行文件。
- SSH_AUTH_SOCK 非空，目标路径存在。
- ssh-add -l 能从转发的 agent 读取身份。
- ssh -T git@github.com 能完成认证测试。
- git ls-remote 能访问目标私有仓库。
- find / -name 'id_*' 等检查不会发现被复制进镜像的私钥文件。
- id、echo "$HOME" 和 command -v <agent-cli> 显示的是预期用户、home 和工具路径。

### 配置与持久化

- 镜像和 Features 的版本策略已记录。
- Dockerfile 没有使用 ARG、ENV 或 COPY 传递私钥和长期 token。
- workspace、构建输出、缓存和日志的持久化位置已经区分。
- dotfiles 安装入口可以重复运行。
- 容器销毁和重建后，项目源代码仍然存在，个人状态按预期保留或重新初始化。
- Zed 或 VS Code 的配置变更已通过重新创建容器验证。

本报告没有执行当前宿主机的 Docker、WSL、SSH agent、VS Code 或 Zed 验证。上述命令是目标环境的验收步骤，不能视为当前机器已经通过。

## 十二、待验证事项与适用边界

1. Windows 原生 OpenSSH agent 在独立 devcontainer CLI、Zed 和不同 Docker Desktop backend 中的 named pipe 转发，需要按具体版本验证。当前报告只把 VS Code 自动转发和 WSL Unix socket 作为优先路径。
2. Docker Desktop 的 /run/host-services/ssh-auth.sock 官方方案只在 Mac 和 Linux 文档范围内得到确认。Windows 不应直接套用。
3. 项目是否需要把 Agent CLI、技能目录和缓存放入镜像、dotfiles 或 volume，取决于工具的安装方式、团队共享需求和认证状态。应先做一次重建测试。
4. 远程 Docker host、Remote SSH、WSL 和本地 Docker Desktop 的 agent 流向不同。遇到问题时应记录启动工具的环境、Docker context、容器用户和 socket 路径。
5. Zed 当前官方文档未给出 VS Code dotfiles.* 等价配置，个人 dotfiles 应通过用户控制的 bootstrap 验证。
6. 私有 registry、模型 provider 和 Agent CLI 的具体认证变量属于各产品的单独契约，不能从通用 Dev Container 规范推导。

## 十三、参考来源

1. [Dev Container JSON reference](https://github.com/devcontainers/spec/blob/main/docs/specs/devcontainerjson-reference.md)
2. [Dev Container：Using Images, Dockerfiles, and Docker Compose](https://containers.dev/guide/dockerfile)
3. [Dev Container：Authoring a Dev Container Feature](https://containers.dev/guide/author-a-feature)
4. [VS Code：Sharing Git credentials with your container](https://code.visualstudio.com/remote/advancedcontainers/sharing-git-credentials)
5. [VS Code：Developing inside a Container](https://code.visualstudio.com/docs/devcontainers/containers)
6. [VS Code：Add a non-root user to a container](https://code.visualstudio.com/remote/advancedcontainers/add-nonroot-user)
7. [Dev Containers CLI：dotfiles.ts](https://github.com/devcontainers/cli/blob/main/src/spec-common/dotfiles.ts)
8. [Docker：Build secrets](https://docs.docker.com/build/building/secrets/)
9. [Dockerfile reference：RUN --mount=type=ssh](https://docs.docker.com/reference/dockerfile)
10. [Docker Compose Build Specification：ssh](https://docs.docker.com/reference/compose-file/build/)
11. [Docker Desktop：Networking how-tos](https://docs.docker.com/desktop/features/networking/networking-how-tos/)
12. [Docker Desktop：WSL 2 backend on Windows](https://docs.docker.com/desktop/features/wsl/)
13. [Docker：Bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)
14. [Docker：Storage](https://docs.docker.com/engine/storage)
15. [Docker：Building best practices](https://docs.docker.com/build/building/best-practices/)
16. [Microsoft Learn：Key-Based Authentication in OpenSSH for Windows](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement)
17. [Microsoft Learn：Get started using Git on WSL](https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-git)
18. [GitHub：Using SSH agent forwarding](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/using-ssh-agent-forwarding)
19. [GitHub：Testing your SSH connection](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/testing-your-ssh-connection)
20. [Zed：Dev Containers](https://zed.dev/docs/dev-containers)
21. [Zed：Run Your Project in a Dev Container](https://zed.dev/blog/dev-containers)
