# VSCode clangd 扩展配置方法：与 ROS 1 / ROS 2 项目配合使用

研究日期：2026-08-31

研究目标：整理 VSCode clangd 扩展（llvm-vs-code-extensions.vscode-clangd）的安装与启用方式、与微软 C/C++ 扩展的冲突处理、编译数据库（compile_commands.json）的生成与发现规则、`.clangd` 配置语法、与 ROS 2（colcon）和 ROS 1（catkin_make / catkin_tools）的配合工作流、交叉编译场景下的 `--query-driver` 用法，以及常见问题排查方法。

## 一、研究范围与判断方式

本报告只使用第一手来源：clangd 官方文档（clangd.llvm.org 的 installation、config、troubleshooting、design、faq、guides 页面）、vscode-clangd 官方仓库（github.com/clangd/vscode-clangd）、colcon 官方文档与 colcon-cmake 官方 issue/PR、catkin_tools 官方文档与官方 issue、CMake 官方文档。社区博客与 StackOverflow 仅作为"社区实践"标注。

文中的判断分为三类：

1. **官方明确写明**：官方文档或官方 issue 中明确说明的行为、配置项或推荐做法，直接引用来源链接。
2. **社区实践**：官方文档没有给出，但社区广泛使用并有公开记录的做法（如把 `compile_commands.json` 软链接到 workspace 根目录、用 `jq` 合并多个 `compile_commands.json`），明确标注为社区实践。
3. **推断**：由官方规则组合推导出的结论（如"ROS 2 的 include 路径由 compile_commands.json 自动携带"），标注为推断。

## 二、核心结论

1. **最小可用配置只有两步**：安装 vscode-clangd 扩展（并按提示禁用微软 C/C++ 扩展的 IntelliSense 或卸载该扩展），让构建系统生成 `compile_commands.json` 并保证 clangd 能发现它。clangd 会在"源文件的所有父目录"以及"名为 `build/` 的子目录"中查找 `compile_commands.json`；CMake 项目用 `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` 生成，非 CMake 项目用 `compile_flags.txt` 或 Bear。[clangd：Installation / Project setup](https://clangd.llvm.org/installation)

2. **ROS 2（colcon）工作流**：执行 `colcon build --cmake-args -DCMAKE_EXPORT_COMPILE_COMMANDS=ON` 后，每个包在自己的构建目录生成 `compile_commands.json`；`colcon-cmake`（>= 0.2.24）还会在工作区级 `build/compile_commands.json` 生成汇总了所有包的编译数据库。当 VSCode 直接打开 colcon workspace 根目录时，clangd 的 `build/` 子目录搜索规则会自动发现这个汇总文件，**不需要软链接**。软链接到 workspace 根是社区实践，不是官方要求。[colcon：How to](https://colcon.readthedocs.io/en/released/user/how-to.html)，[clangd：Installation](https://clangd.llvm.org/installation)

3. **ROS 1 工作流**：`catkin_make -DCMAKE_EXPORT_COMPILE_COMMANDS=1` 会在 `build/` 生成单个全 workspace 的 `compile_commands.json`，开箱即用；`catkin build`（catkin_tools）则是每个包在 `build/<pkg>/` 下各自生成一份，且没有官方汇总功能（catkin_tools 维护者明确表示不会合并）。推荐做法是持久化配置 `catkin config --cmake-args -DCMAKE_EXPORT_COMPILE_COMMANDS=ON` 后构建，再（社区实践）用 `jq` 合并或软链接，或者用 `.clangd` 的 `CompilationDatabase` 字段 / `--compile-commands-dir` 指向对应目录。[catkin_tools issue #551](https://github.com/catkin/catkin_tools/issues/551)，[betwo/vscode-catkin-tools](https://github.com/betwo/vscode-catkin-tools)

4. **ROS 的 include 路径不需要手动加**：`compile_commands.json` 记录的是"机器可读的、精确的编译器调用"，ament / catkin 通过 CMake 传给编译器的 `-I` / `-isystem`（包括 `/opt/ros/<distro>/include`、build 目录下消息生成代码的 include 路径）都会写进编译命令，clangd 直接复用。只有不走 compile_commands.json 时（微软 C/C++ 扩展的 `c_cpp_properties.json` 方案）才需要手写 includePath。（推断，依据 [colcon：How to](https://colcon.readthedocs.io/en/released/user/how-to.html) 与 [clangd：design/compile-commands](https://clangd.llvm.org/design/compile-commands)）

5. **交叉编译 / 非 clang 工具链**：用 `--query-driver` 白名单真实编译器（如 `/usr/bin/*gcc*,/usr/bin/*g++*` 或交叉工具链路径），clangd 会执行该编译器提取内置 include 路径和 target triple。官方明确推荐 `--query-driver` 优先于手工 `-isystem` 加路径。注意 `--query-driver` 是 clangd 二进制自身的命令行参数，只能放在 VSCode 的 `clangd.arguments` 中，不能放进 `.clangd` 的 `CompileFlags: Add`。[clangd：Troubleshooting](https://clangd.llvm.org/troubleshooting)，[clangd：guides/system-headers](https://clangd.llvm.org/guides/system-headers)，[clangd issue #2512](https://github.com/clangd/clangd/issues/2512)

6. **clangd 二进制来源**：vscode-clangd 扩展在 PATH 找不到 clangd 时会提示自动下载（支持 x86-64 Linux / Windows / macOS）；系统包管理器（如 Ubuntu 的 `clangd` / `clang-tools` 包）安装的版本通常落后于 LLVM 官方发布节奏（每 6 个月一版），可在命令面板运行 "Check for clangd language server update" 更新扩展下载的版本，或用 `clangd.path` 指定系统版本。[vscode-clangd README](https://github.com/clangd/vscode-clangd)，[clangd：Installation](https://clangd.llvm.org/installation)

## 三、基础配置

### 3.1 安装 vscode-clangd 扩展

官方扩展 ID 为 `llvm-vs-code-extensions.vscode-clangd`，在 VSCode 扩展面板搜索 "clangd" 安装即可。[clangd：Installation](https://clangd.llvm.org/installation)，[Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=llvm-vs-code-extensions.vscode-clangd)

扩展需要一个独立的 `clangd` 语言服务器二进制：

- 如果 PATH 中找不到 clangd，扩展会提示自动下载（自动安装支持 x86-64 Linux、Windows、macOS）。[vscode-clangd README](https://github.com/clangd/vscode-clangd)
- 系统包管理器（如 `apt install clangd` 或 `apt install clang-tools`）装的版本跟随 LLVM 发布节奏（约每 6 个月一版），往往比扩展下载的 GitHub release 旧。[clangd：Installation](https://clangd.llvm.org/installation)
- 可通过命令面板的 "Check for clangd language server update" 更新扩展管理的 clangd；`clangd.path` 设置可指定自定义二进制路径（默认 `"clangd"`）。[vscode-clangd README](https://github.com/clangd/vscode-clangd)，[vscode-clangd docs/settings.md](https://github.com/clangd/vscode-clangd/blob/master/docs/settings.md)

### 3.2 与微软 C/C++ 扩展（ms-vscode.cpptools）的冲突

clangd 官方安装页明确要求"确保微软 C/C++ 扩展没有安装"。[clangd：Installation](https://clangd.llvm.org/installation)

实际使用中更常见的做法是保留 cpptools（它的调试器仍有用）但禁用其 IntelliSense：vscode-clangd 检测到 cpptools 时会弹出警告，提供 "Disable IntelliSense" 按钮，该按钮写入的设置是：

```json
{
  "C_Cpp.intelliSenseEngine": "disabled"
}
```

（官方扩展行为，见 [vscode-clangd PR #141](https://github.com/clangd/vscode-clangd/pull/141)；注意该按钮默认写入用户级 settings，相关讨论见 [issue #595](https://github.com/clangd/vscode-clangd/issues/595)，手工写入工作区级 `.vscode/settings.json` 亦可。）

## 四、编译数据库详解

### 4.1 clangd 如何发现 compile_commands.json

clangd 官方文档说明：clangd 在"你编辑的文件的所有父目录"以及"名为 `build/` 的子目录"中查找 `compile_commands.json`。官方给出的例子：编辑 `$SRC/gui/window.cpp` 时，依次搜索 `$SRC/gui/`、`$SRC/gui/build/`、`$SRC/`、`$SRC/build/`……。[clangd：Installation / Project setup](https://clangd.llvm.org/installation)

补充规则：

- 如果编译数据库不在源码树内，官方建议把它软链接或复制到源码树根目录。[clangd：Installation](https://clangd.llvm.org/installation)
- `compile_flags.txt`：当所有文件使用同一组编译选项时，可在源码根放一份每行一个 flag 的 `compile_flags.txt`，clangd 假定编译命令为 `clang $FLAGS some_file.cc`；`compile_commands.json` 存在时 `compile_flags.txt` 被忽略；且使用 `compile_flags.txt` 时后台索引不可用。[clangd：Installation](https://clangd.llvm.org/installation)
- 非 CMake 项目可用 Bear 等工具拦截构建生成 `compile_commands.json`（社区工具，clangd 文档以"build system, or tools integrated with the build system"概括）。[clangd：FAQ](https://clangd.llvm.org/faq)
- `.clangd` 配置文件中的 `CompileFlags: CompilationDatabase: <dir>` 可以指定编译数据库目录（clangd 12+），取值可以是单个路径（绝对或相对 fragment 所在位置）、`Ancestors`（默认，搜父目录）或 `None`。[clangd：Configuration](https://clangd.llvm.org/config)
- 命令行参数 `--compile-commands-dir=<dir>` 全局指定数据库目录；若路径无效，clangd 回退到"当前目录与源文件父目录"的默认搜索。注意 `--compile-commands-dir` 是进程级全局覆盖，会让各子项目的 `CompilationDatabase` 配置失效。[clangd(1) manpage](https://manpages.debian.org/bullseye/clangd/clangd.1.en.html)，[clangd discussion #2288](https://github.com/clangd/clangd/discussions/2288)

### 4.2 CMake 项目生成方式

在 CMake 配置命令中加 `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`，`compile_commands.json` 会写入构建目录；构建目录若是 `$SRC` 或 `$SRC/build` 则 clangd 直接能找到。[clangd：Installation](https://clangd.llvm.org/installation)，[CMake：CMAKE_EXPORT_COMPILE_COMMANDS](https://cmake.org/cmake/help/latest/variable/CMAKE_EXPORT_COMPILE_COMMANDS.html)

## 五、`.clangd` 配置文件语法

配置为 YAML 文件，机制自 clangd 11 引入。分两级：[clangd：Configuration](https://clangd.llvm.org/config)

- **项目级**：源码树中名为 `.clangd` 的文件（clangd 在活动文件的所有父目录中查找），适合入库共享。
- **用户级**：`config.yaml`，Linux 下通常在 `~/.config/clangd/config.yaml`（`$XDG_CONFIG_HOME/clangd/config.yaml`），macOS 为 `~/Library/Preferences/clangd/config.yaml`，Windows 为 `%LocalAppData%\clangd\config.yaml`。冲突时优先级：用户级 > 内层项目级 > 外层项目级。

常用片段示例（官方文档中的写法）：

```yaml
# .clangd（项目根）
CompileFlags:
  Add: [-Wall]                 # 追加编译选项
  Remove: [-W*]                # 移除选项；支持 -DFOO=* 前缀通配
  Compiler: clang++            # 替换编译命令的 argv[0]（clangd 14+）
  CompilationDatabase: build/  # 指定数据库目录（clangd 12+）

If:                            # 条件应用（按路径正则）
  PathMatch: .*\.h
  PathExclude: third_party/.*

Index:
  Background: Skip             # 对匹配文件关闭后台索引

Diagnostics:
  Suppress: [unused-includes]  # 抑制指定诊断码；'*' 关闭全部
  ClangTidy:
    Add: modernize*
    Remove: modernize-use-trailing-return-type
  UnusedIncludes: Strict

InlayHints:
  Enabled: true
  ParameterNames: true
  DeducedTypes: true
  BlockEnd: false
  TypeNameLimit: 24

Hover:
  ShowAKA: true                # 显示 desugared 类型 (aka ...)
```

要点：[clangd：Configuration](https://clangd.llvm.org/config)

- `Remove` 的语义：若值是已识别的 clang flag（如 `-I`），连同其参数一起移除；以 `*` 结尾则按前缀移除；否则精确匹配移除。
- `Compiler` 字段（clangd 14+）替换编译命令的可执行名，控制 flag 解析方式与 target 推断；若该名字匹配 `--query-driver` 的 glob，clangd 会调用它提取 include 路径。
- `BuiltinHeaders: QueryDriver`（clangd 21+）可让 clangd 使用 query-driver 提取到的编译器内置头文件，而非 clangd 自带的。
- 单个标量值可代替数组，如 `Remove: -mabi` 等价于 `Remove: [-mabi]`。
- 多个 fragment 用 `---` 分隔，主要配合不同 `If` 条件使用。
- **不要**把 `--query-driver` 写进 `CompileFlags: Add`——它是 clangd 进程自身的参数，不是编译选项。[clangd issue #2512](https://github.com/clangd/clangd/issues/2512)

## 六、VSCode settings.json 常用参数表

扩展自身的设置项（官方扩展文档）：[vscode-clangd docs/settings.md](https://github.com/clangd/vscode-clangd/blob/master/docs/settings.md)，[vscode-clangd README](https://github.com/clangd/vscode-clangd)

| 设置 | 默认值 | 说明 |
| --- | --- | --- |
| `clangd.path` | `"clangd"` | clangd 可执行文件路径 |
| `clangd.arguments` | `[]` | 传给 clangd 服务器的命令行参数数组 |
| `clangd.onConfigChanged` | `"prompt"` | 配置文件变化时行为：`prompt` / `restart` / `ignore`（clangd 12+ 支持自动重载时该特性被旁路） |
| `clangd.checkUpdates` | `false` | 启动时检查语言服务器更新 |
| `clangd.restartAfterCrash` | `true` | 崩溃后自动重启（最多 4 次） |

注意：在工作区级 settings 中配置 `clangd.path` / `clangd.arguments` 时，扩展出于安全考虑会弹窗询问是否采用，选择会按 workspace 记住。[vscode-clangd docs/settings.md](https://github.com/clangd/vscode-clangd/blob/master/docs/settings.md)

`clangd.arguments` 中常用的 clangd 命令行参数（官方 manpage）：[clangd(1) manpage](https://manpages.ubuntu.com/manpages/jammy/man1/clangd.1.html)

| 参数 | 作用 |
| --- | --- |
| `--compile-commands-dir=<dir>` | 全局指定查找 `compile_commands.json` 的目录；路径无效时回退默认搜索 |
| `--query-driver=<globs>` | 逗号分隔的 glob 白名单，匹配到的 GCC 兼容驱动会被 clangd 执行以提取系统 include 路径，如 `/usr/bin/**/clang-*,/path/to/repo/**/g++-*` |
| `--background-index` | 后台构建项目级索引（跳转/引用依赖它；现代版本默认开启） |
| `-j=<n>` | 后台索引等工作线程数，可用于限制 CPU/内存占用 |
| `--header-insertion=<iwyu\|never>` | 补全符号时是否自动插入 `#include` |
| `--all-scopes-completion` | 补全包含当前不可见作用域的符号 |
| `--completion-style=<detailed\|bundled>` | 补全项的粒度风格 |
| `--log=<level>` | 日志级别，排查时用 `--log=verbose` |
| `--check=<file>` | 不启动服务器，直接对单个文件做配置健康检查并输出诊断 |

最小可用 settings.json 示例：

```json
{
  "C_Cpp.intelliSenseEngine": "disabled",
  "clangd.arguments": [
    "--background-index",
    "--completion-style=detailed",
    "--header-insertion=never",
    "-j=8"
  ]
}
```

## 七、ROS 2 工作流（重点）

### 7.1 生成 compile_commands.json

colcon 官方文档"How to"一章明确：开启 CMake 选项 `CMAKE_EXPORT_COMPILE_COMMANDS` 后，每个包的构建目录会生成包含精确编译器调用的 `compile_commands.json`：[colcon：How to](https://colcon.readthedocs.io/en/released/user/how-to.html)

```bash
colcon build --cmake-args -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

并且 `colcon-cmake` 会在 `build/` 目录额外生成一个**工作区级** `compile_commands.json`，汇总所有包的数据。（该能力由 [colcon-cmake PR #69](https://github.com/colcon/colcon-cmake/pull/69) 实现（引入 `CompileCommandsEventHandler`），自 colcon-cmake 0.2.24 起可用（issue #81 的 milestone 为 0.2.24）；`apt` 源的 colcon 可能偏旧，可用 `pip install -U colcon-common-extensions` 升级，见 [colcon-cmake issue #61](https://github.com/colcon/colcon-cmake/issues/61)。）

### 7.2 clangd 如何发现它

VSCode 直接打开 colcon workspace 根目录（即含 `src/`、`build/` 的那层）时，对 `src/<pkg>/foo.cpp` 的父目录搜索会在 `$WS/build/` 命中工作区级汇总文件——因为 clangd 的搜索规则包含"父目录下名为 `build/` 的子目录"。**这是官方规则的直接推论，不需要任何软链接。**[clangd：Installation](https://clangd.llvm.org/installation)

把 `build/compile_commands.json` 软链接到 workspace 根是常见的社区实践（例如 colcon-cmake PR #62 讨论中用户的用法），用于编辑器不以 workspace 根为工作目录或工具链不识别 `build/` 子目录规则的场景；它不是 colcon 或 clangd 的官方要求，这里明确标注为社区实践。[colcon-cmake PR #62](https://github.com/colcon/colcon-cmake/pull/62)

如果希望显式指定（例如打开的是 `src/` 子目录），也可以用 `--compile-commands-dir`：

```json
{
  "clangd.arguments": [
    "--compile-commands-dir=${workspaceFolder}/build",
    "--background-index"
  ]
}
```

### 7.3 include 路径与 ament 的关系

不需要手工向 clangd 添加 `/opt/ros/<distro>/include`：ament_cmake 通过 CMake 把依赖的 include 目录传给编译器，这些 `-I` / `-isystem` 参数被原样记录进 `compile_commands.json`，clangd 按条目重建编译命令。手工维护 includePath 是微软 C/C++ 扩展 `c_cpp_properties.json` 方案的做法（如社区教程所示），与 clangd 方案互斥。（推断，依据 [colcon：How to](https://colcon.readthedocs.io/en/released/user/how-to.html) 与 [clangd：design/compile-commands](https://clangd.llvm.org/design/compile-commands)）

### 7.4 完整配置示例

`.vscode/settings.json`：

```json
{
  "C_Cpp.intelliSenseEngine": "disabled",
  "clangd.arguments": [
    "--compile-commands-dir=${workspaceFolder}/build",
    "--background-index",
    "--completion-style=detailed",
    "--header-insertion=never",
    "--query-driver=/usr/bin/g++*,/usr/bin/gcc*",
    "-j=8"
  ],
  "clangd.onConfigChanged": "restart"
}
```

`.vscode/tasks.json`（构建并生成编译数据库）：

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "colcon: build",
      "type": "shell",
      "command": "colcon build --symlink-install --cmake-args -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCMAKE_BUILD_TYPE=RelWithDebInfo",
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "problemMatcher": []
    }
  ]
}
```

（`--symlink-install` 为社区常用实践；tasks.json 的字段格式参考 [VSCode：Tasks](https://code.visualstudio.com/docs/debugtest/tasks)。ROS 扩展与 Robotics Developer Environment 扩展包的用法属于另一套方案，此处不展开。）

项目根 `.clangd`（可选，用于把构建目录从索引/诊断中排除等）：

```yaml
# 例：关闭 build/ 下生成代码的后台索引，减少内存占用
If:
  PathMatch: build/.*
Index:
  Background: Skip
```

### 7.5 多 workspace（underlay / overlay）

- overlay workspace 用 colcon 构建时，其 `compile_commands.json` 中的编译命令已经携带指向 underlay（如 `/opt/ros/<distro>/include` 或另一个 source 过的 install 目录）的 include 参数，clangd 解析 overlay 源码不需要额外配置。（推断，同 7.3 依据）
- 如果希望跳转到 underlay 源码并获得完整索引，常见社区做法是同时打开多个 workspace 文件夹（multi-root workspace）或分别构建 underlay 并让它也有自己的 `compile_commands.json`；clangd 对每个 workspace 文件夹独立运行/搜索数据库。此场景官方没有专门文档，标注为社区实践。

## 八、ROS 1 工作流

### 8.1 catkin_make

```bash
catkin_make -DCMAKE_EXPORT_COMPILE_COMMANDS=1
```

会在 `catkin_ws/build/` 生成**单个、覆盖整个 workspace** 的 `compile_commands.json`（catkin_make 把所有包合并在一个 CMake 上下文中构建）。clangd 打开 workspace 根即可通过 `build/` 子目录规则自动发现。[catkin_tools issue #551](https://github.com/catkin/catkin_tools/issues/551)（issue 中确认的行为，属于官方仓库记录）

### 8.2 catkin_tools（catkin build）

- 每个包在隔离的构建目录 `build/<pkg>/` 下各自生成一份 `compile_commands.json`，catkin_tools 官方不提供汇总；维护者在 issue #551 中明确表示"合并编译命令会破坏多项目之间的隔离，这不应由 catkin_tools 来做"。[catkin_tools issue #551](https://github.com/catkin/catkin_tools/issues/551)，[catkin_tools：Build Packages](https://catkin-tools.readthedocs.io/en/latest/verbs/catkin_build.html)
- 推荐持久化开启：

```bash
catkin config --cmake-args -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
catkin build
```

（该写法见 [betwo/vscode-catkin-tools README](https://github.com/betwo/vscode-catkin-tools)；也可直接写进 `.catkin_tools/profiles/<profile>/config.yaml` 的 `cmake_args`。）

- **坑**：若包的构建目录已存在且 CMake 未重跑，新加的 `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` 不会生效，需要 `catkin build --force-cmake` 或删掉对应构建目录。[catkin_tools issue #227](https://github.com/catkin/catkin_tools/issues/227)

### 8.3 让 clangd 找到分散的数据库（社区实践）

由于 clangd 默认只搜"父目录 + 父目录下的 `build/` 子目录"，编辑 `src/<pkg>/foo.cpp` 时默认找不到 `build/<pkg>/compile_commands.json`（它不在任何父目录的 `build/` 子目录里——`$WS/build/compile_commands.json` 不存在时才会继续向上）。常用社区做法：

1. **合并**（issue #551 中的方案，用 `jq`）：

   ```bash
   cd ~/catkin_ws
   jq -s 'map(.[])' build/*/compile_commands.json > build/compile_commands.json
   ```

   合并后 clangd 通过 `$WS/build/` 规则自动发现。注意每次构建后需要重新合并。

2. **软链接**：`ln -s build/<pkg>/compile_commands.json` 到源码树根或包根（社区实践，见 issue #551 与 colcon-cmake PR #62 讨论）。

3. **`.clangd` 指向**：在每个包根放 `.clangd`：

   ```yaml
   CompileFlags:
     CompilationDatabase: ../../build/<pkg>
   ```

   （`CompilationDatabase` 字段为官方机制，相对路径相对于 fragment 所在位置，见 [clangd：Configuration](https://clangd.llvm.org/config)；把它用于 catkin 包目录属于社区实践。）

4. **`--compile-commands-dir`**：只适合单包工作区，因为它是进程级全局覆盖。[clangd discussion #2288](https://github.com/clangd/clangd/discussions/2288)

ROS 消息生成头文件（`devel/include/<pkg>/Foo.h`）：编译命令中的 `-I<ws>/devel/include` 由 catkin 注入并记录进数据库，因此 clangd 能找到——前提是相关包至少构建过一次。这与 ROS 2 的 `install/`、`build/<pkg>/rosidl_generator_*` 同理。（推断）

## 九、交叉编译与 `--query-driver`

官方文档要点：[clangd：guides/system-headers](https://clangd.llvm.org/guides/system-headers)，[clangd：Troubleshooting](https://clangd.llvm.org/troubleshooting)，[clangd：design/compile-commands](https://clangd.llvm.org/design/compile-commands)

1. clangd 只自带编译器内置头文件（`<stddef.h>` 等，与内嵌 clang 版本绑定），C++ 标准库等其余系统头文件必须由系统提供。
2. clang 查找标准库的启发式依赖"驱动程序所在目录"和"target triple"。使用交叉编译器或自定义工具链时，这些启发式常常失败。
3. `--query-driver=<globs>` 是 GCC 兼容驱动的白名单：编译命令 argv[0] 匹配任一 glob 时，clangd 会执行该驱动来提取系统 include 路径与 target triple。必须显式开启，因为这会执行二进制文件。例如：

   ```json
   {
     "clangd.arguments": [
       "--query-driver=/usr/bin/arm-linux-gnueabihf-*,/opt/toolchains/**/bin/*g++*"
     ]
   }
   ```

4. 官方明确**推荐 `--query-driver` 优先于手工 `-isystem` 加路径**（手工加容易因 include 顺序错误出问题），且 `--query-driver` 的值应与 `compile_commands.json` 中的编译器命令匹配（如 `/usr/bin/c++`）。[clangd：Troubleshooting](https://clangd.llvm.org/troubleshooting)
5. `.clangd` 的 `CompileFlags: Compiler:` 字段（clangd 14+）可替换编译命令的可执行名，配合 `--query-driver` 使用：替换后的名字匹配 glob 时同样会被调用以提取 include 路径。[clangd：Configuration](https://clangd.llvm.org/config)
6. 再次强调：`--query-driver` 不能写在 `.clangd` 的 `CompileFlags: Add` 里（任何版本都不行），它是 clangd 进程参数。[clangd issue #2512](https://github.com/clangd/clangd/issues/2512)
7. clangd 21+ 提供 `CompileFlags: BuiltinHeaders: QueryDriver`，让内置头文件也来自被查询的驱动；官方同时警告：驱动不是 clang 时可能引入误报。[clangd：Configuration](https://clangd.llvm.org/config)

## 十、常见问题排查表

| 症状 | 原因与处理 | 来源 |
| --- | --- | --- |
| include 报错但编译通过 | 先用 `clangd --check=/path/to/file.cc` 看实际编译命令；若日志出现 "Generic fallback command"，说明 clangd 没找到编译数据库，先修数据库发现路径；若数据库命令本身能编译文件，检查驱动是否为相对路径（改为绝对路径） | [clangd：guides/system-headers](https://clangd.llvm.org/guides/system-headers) |
| 找不到标准库头文件（`iostream`、`stdio.h`） | 确认系统装有标准库开发包；编译器路径用绝对路径；交叉编译器/MinGW 场景加 `--query-driver`；用 `--log=verbose` 对比 `clang -###` 的搜索路径 | [clangd：Troubleshooting](https://clangd.llvm.org/troubleshooting) |
| 找不到编译器内置头文件（`stddef.h`） | 这些头文件随 clangd 安装在 `../lib/clang/<version>/include`，移动过二进制要放回去 | [clangd：Troubleshooting](https://clangd.llvm.org/troubleshooting) |
| 打开项目目录之外的头文件（第三方库）报一堆错 | 该头文件找不到项目的编译数据库，回退到默认命令；可用 `--compile-commands-dir=<dir>` 强制所有文件使用项目数据库 | [clangd：FAQ](https://clangd.llvm.org/faq) |
| ROS 消息 / 服务头文件找不到 | 对应包尚未构建过，生成代码（ROS 1 在 `devel/include`，ROS 2 在 `build/<pkg>/rosidl_generator_*` 与 `install/`）还不存在；先完整构建一次，确认数据库条目中有对应 `-I`（推断） | [colcon：How to](https://colcon.readthedocs.io/en/released/user/how-to.html) |
| 多个 compile_commands.json 冲突 | 父目录中更近的数据库优先；`--compile-commands-dir` 会全局覆盖并让 `.clangd` 的 `CompilationDatabase` 失效，按需取舍 | [clangd discussion #2288](https://github.com/clangd/clangd/discussions/2288)，[clangd(1) manpage](https://manpages.ubuntu.com/manpages/jammy/man1/clangd.1.html) |
| colcon 构建提示 "Manually-specified variables were not used: CMAKE_EXPORT_COMPILE_COMMANDS" | 非 CMake 包（ament_python 等）忽略该变量；官方建议加 `--no-warn-unused-cli`，或只对 CMake 包传参 | [colcon-cmake issue #76](https://github.com/colcon/colcon-cmake/issues/76)，[CMake manual](https://cmake.org/cmake/help/latest/manual/cmake.1.html) |
| `catkin build` 不生成 compile_commands.json | 构建目录已存在导致 CMake 未重跑；用 `catkin build --force-cmake` 或删除对应包构建目录 | [catkin_tools issue #227](https://github.com/catkin/catkin_tools/issues/227) |
| 索引慢 / 内存占用高 | 用 `-j=<n>` 限制后台索引线程；对 build/ 等目录用 `.clangd` 的 `Index: Background: Skip`；跳转/引用不全时先确认后台索引已完成 | [clangd(1) manpage](https://manpages.ubuntu.com/manpages/jammy/man1/clangd.1.html)，[clangd：Configuration](https://clangd.llvm.org/config)，[clangd：FAQ](https://clangd.llvm.org/faq) |
| 修改 `.clangd` 后没生效 | clangd 12+ 支持自动重载；老版本由扩展按 `clangd.onConfigChanged` 提示重启；也可用命令面板 "clangd: Restart language server" | [vscode-clangd docs/settings.md](https://github.com/clangd/vscode-clangd/blob/master/docs/settings.md) |
| 头文件的补全/诊断用错编译命令 | 数据库没有头文件条目，clangd 按文件名启发式"插值"选取近似源文件的命令，偶尔选错；可用 CompDB 等工具为头文件生成条目 | [clangd：design/compile-commands](https://clangd.llvm.org/design/compile-commands)，[clangd：FAQ](https://clangd.llvm.org/faq) |

## 十一、待确认 / 可能变动事项

1. **`.clangd` 配置项的版本门槛**：`CompilationDatabase` 需 clangd 12+、`Compiler` 需 14+、`InlayHints`/`Hover` 需 14+、`Index.StandardLibrary` 需 15+、`BuiltinHeaders` 需 21+；旧发行版 apt 仓库的 clangd 版本可能不满足，使用前用 `clangd --version` 确认。[clangd：Configuration](https://clangd.llvm.org/config)
2. **ROS 2 官方 IDE 文档缺位**：docs.ros.org 没有针对 clangd 的官方教程，colcon 官方文档只覆盖 compile_commands.json 的生成；settings.json / tasks.json 的完整示例由官方规则组合而来，标注为"推断/社区实践"的部分建议在目标 ROS 发行版上实测。
3. **underlay/overlay 场景**：官方没有专门文档；multi-root workspace 下 clangd 对每个文件夹独立搜索数据库的行为来自社区经验，未在官方文档中找到明确表述。
4. **catkin_tools 汇总数据库**：issue #551 截至最后更新（2022 年）仍为 open，维护者明确不打算支持官方合并；`jq` 合并、软链接、`.clangd` 指向三种做法均为社区实践，无官方背书。
5. **`--completion-style`、`--all-scopes-completion` 等参数**：来自 clangd(1) manpage；不同 clangd 版本的可选参数有差异，以本机 `clangd --help` 输出为准。
6. **新版 colcon 对汇总文件的行为**：colcon-cmake 的汇总实现历史上经过 issue #61 / #76 / #129 多次讨论，个别包类型（如纯 CMake 但无翻译单元的示例包）可能不出现在汇总文件中，如遇缺失请核对 [colcon-cmake issue #129](https://github.com/colcon/colcon-cmake/issues/129)。
