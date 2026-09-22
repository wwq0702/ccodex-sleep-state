# 部署流程：获取程序、接入 Codex、验证与恢复

本工具应与需要接入的 Codex 运行在同一台电脑、同一用户环境中。程序同时提供请求转发和网页管理面板，网页已嵌入二进制，不需要单独部署前端、数据库或 Docker。

默认连接路径：`Codex → 本机 127.0.0.1:17841 → 你配置的直连或代理 → 当前账号对应的上游`。当前配置只允许回环 IP，不支持直接把监听地址改成 `0.0.0.0`。以下步骤针对 Windows 和 macOS 本机使用；仓库在 Linux 上运行 CI 不等于已提供 Linux 桌面安装包。

## 1. 开始前准备什么

1. 先确认 Codex 在不使用本工具时能够正常回复。官方账号完成登录；API key 或中转用户先配好现有 provider。
2. 准备自己的连接方式：可直连的网络、本地 HTTP / SOCKS5 代理、订阅链接或订阅文件，选一种即可。
3. 使用 CCS / CC Switch 时，先选择本次使用的配置，再退出 Codex；接入期间暂时不要切换 provider。
4. 停止其他正在接管同一份 Codex 配置的本工具实例。默认端口是 `17841`，不同数据目录也不能同时占用同一个端口。

下载可执行安装包不需要 Go。只有从源码编译时才需要 Git 和 Go：当前 `go.mod` 要求 Go 1.26.0 或以上，仓库 CI 配置使用 Go 1.27.x。

## 2. 获取程序：两种方式选一种

### 方式一：下载发布包

先查看[本 Fork 的 Releases](https://github.com/wwq0702/ccodex-sleep-state/releases)。**如果没有发布版本，直接使用下面的源码构建方式。** Fork 不会复制上游已经上传的二进制包。

也可以选择[上游 Releases](https://github.com/gylive/ccodex-sleep-state/releases)中的现成版本，但上游包不包含本 Fork 的修改。教程描述当前源码；旧包的命令、界面和功能以对应版本说明为准。

| 设备 | 发布包文件名 | 解压后的程序与启动脚本 |
| --- | --- | --- |
| Windows x64 | `ccodex-sleep-state-windows-amd64.zip` | `ccodex-sleep-state.exe`、`start.cmd` |
| Windows ARM64 | `ccodex-sleep-state-windows-arm64.zip` | `ccodex-sleep-state.exe`、`start.cmd` |
| Apple 芯片 Mac | `ccodex-sleep-state-darwin-arm64.tar.gz` | `ccodex-sleep-state`、`start.command` |
| Intel Mac | `ccodex-sleep-state-darwin-amd64.tar.gz` | `ccodex-sleep-state`、`start.command` |

Windows 可在「设置 → 系统 → 系统信息」查看系统类型。macOS 可在「关于本机」查看芯片；M 系列选择 `arm64`，Intel 选择 `amd64`。

同时下载该版本的 `SHA256SUMS`。在下载目录执行下面对应的命令，将输出的 SHA256 与 `SHA256SUMS` 中**同名文件**的值逐字核对，字母大小写不影响比较。不一致时不要继续使用该文件。

Windows PowerShell（x64 示例）：

```powershell
Get-FileHash .\ccodex-sleep-state-windows-amd64.zip -Algorithm SHA256
Get-Content .\SHA256SUMS
```

macOS 终端（Apple 芯片示例）：

```sh
shasum -a 256 ccodex-sleep-state-darwin-arm64.tar.gz
cat SHA256SUMS
```

完整解压后打开包含程序和启动脚本的目录，不要直接在压缩包里运行，也不要只拖出程序。建议使用 Windows 的 `%LOCALAPPDATA%\Programs\ccodex-sleep-state` 或 macOS 的 `~/Applications/ccodex-sleep-state`。

**Code → Download ZIP、Release 的 Source code (zip/tar.gz) 和 `ccodex-sleep-state-source.tar.gz` 都是源码。** 源码包需要编译，不能当作对应平台的可执行包；其中项目附带的 `source.tar.gz` 还包含构建时锁定的依赖。

### 方式二：从本 Fork 源码构建

在准备保存项目的目录打开终端，先检查工具版本：

```sh
git --version
go version
git clone https://github.com/wwq0702/ccodex-sleep-state.git
cd ccodex-sleep-state
```

macOS：

```sh
go build -trimpath -o ccodex-sleep-state ./cmd/ccodex-sleep-state
./ccodex-sleep-state version
```

Windows PowerShell：

```powershell
go build -trimpath -o ccodex-sleep-state.exe ./cmd/ccodex-sleep-state
.\ccodex-sleep-state.exe version
```

第一次编译会自动下载依赖，需要能够访问 Go 模块下载源，并可能耗时较长。以上构建没有注入发布版本号，`version` 输出 `dev` 是正常情况。构建失败时先解决错误，不要继续运行上一次留下的程序。

源码构建后，直接执行下一节的 `setup` 命令即可。仓库里的双击脚本位于 `scripts/`，发布包才会把脚本复制到程序同目录。需要打包四个平台或发布自己的 Release，见[构建安装包和发布版本](development.md#构建安装包和发布版本)。

## 3. 启动并接入 Codex

发布包用户可双击 Windows 的 `start.cmd` 或 macOS 的 `start.command`。源码构建用户、或需要查看完整错误信息的用户，在程序所在目录执行：

Windows PowerShell：

```powershell
.\ccodex-sleep-state.exe setup
```

macOS：

```sh
./ccodex-sleep-state setup
```

`setup` 会创建缺失的服务配置、备份并尝试接管 Codex 配置，然后打开管理面板。已有配置会继续使用，不需要每次执行 `init`。保留终端窗口；命令持续运行表示服务正在工作。

正常情况下浏览器会自动进入面板。如果没有打开，访问终端输出的管理地址，并填写本次管理口令；默认地址是 [http://127.0.0.1:17841/admin/](http://127.0.0.1:17841/admin/)。口令会随服务重启变化，不是 Codex 密码。

启动本身不发送模型请求。首次没有配置出口时会尝试发现本机常见 SOCKS5 端口；发现本地代理不代表已经验证上游模型可用。

### 连接方式怎么选

如果首页提示缺少连接，在「订阅与代理」选择：

| 现有条件 | 填写内容 |
| --- | --- |
| 已运行本地代理软件 | 该软件实际开放的 HTTP / SOCKS5 地址，例如 `socks5://127.0.0.1:7897`；端口以你的软件配置为准 |
| 已有订阅链接 | 完整订阅 URL |
| 已有 YAML / URI 列表 | 本地订阅文件的完整路径 |
| 当前网络能直接访问上游 | 直连 |

点击「先测试能否读取」，确认格式无误后再「保存并应用」。读取成功只说明配置能解析，不代表模型能够回复。

按需在「连接设置」选择模型和账号规则。官方 ChatGPT 登录可使用 state 采集与注入；官方 API key 和 Responses 中转采用 API 通道转发，不采集官方 state。首次 `setup` 未明确配置兜底策略时采用普通转发兜底，已明确设置的严格策略会保留。

## 4. 怎么确认接入成功

1. **面板可访问**：本地服务已经启动。
2. **Codex 配置显示已接管**：配置入口已接到本工具；如果未接管，按首页提示处理，必要时使用「检查与修复配置」。
3. **重启 Codex、新建会话**：发送一条短消息，确认面板的请求计数或会话记录增加。
4. **Codex 收到回复**：本次真实请求的转发链路已完成。首条请求如需采集 state，可能等待较长时间并消耗实际额度。

这些结果要分开判断：面板能打开不等于已接管，已接管不等于请求通过，可用 state 不代表模型质量提升。普通转发兜底也可能正常回复而没有注入 state。

需要诊断时，保持服务运行，另开终端在程序目录执行 `doctor`：

```powershell
.\ccodex-sleep-state.exe doctor
```

```sh
./ccodex-sleep-state doctor
```

`doctor` 读取当前服务的脱敏状态，不发模型请求。界面操作细节见[面板教程](web-panel.md)。

## 5. 停止、升级、回退和卸载

**正常停止：** 等当前回复结束，在服务终端按 **Ctrl+C**，等待程序报告退出及配置恢复，再重启 Codex。关闭浏览器不会停止服务；关闭注入不会恢复原来的 Codex 配置。程序没有安装后台服务或开机自启，下次使用仍需启动。

**异常退出后的恢复：** 先确认服务进程已经退出，在程序目录执行：

```powershell
.\ccodex-sleep-state.exe restore
```

```sh
./ccodex-sleep-state restore
```

若提示配置冲突，不删除事务文件、不强行覆盖；重新运行 `setup`，通过面板「检查与修复配置」预览并处理。通常选择保留当前配置再接管；需要撤销后续修改时才选择恢复接管前备份。

**升级：** 正常停止旧服务并确认恢复，保留旧程序和整个服务数据目录的私有备份，将新包解压到新目录，再运行新版 `setup`。使用原来的服务数据目录可沿用配置；两版程序不要同时接管同一份 Codex 配置。

**回退：** 停止新版并完成恢复，恢复升级前的服务数据备份，再启动旧程序。不要把新版本生成的配置直接交给旧版读取，也不要覆盖升级期间另外修改过的 Codex 配置。详细兼容边界见[升级与回退指南](upgrade-0.4.0-preview.md)。

**卸载：** 先完成停止和配置恢复，再删除本工具程序目录；确认不再需要配置和备份后再自行处理服务数据目录。不要删除 Codex 的 `.codex` 或 `auth.json`。

## 6. 配置和日志在哪里

| 内容 | Windows 默认位置 | macOS 默认位置 |
| --- | --- | --- |
| 服务配置 | `%LOCALAPPDATA%\ccodex-sleep-state\config.json` | `~/Library/Application Support/ccodex-sleep-state/config.json` |
| 服务日志 | 服务数据目录内的 `logs` | 服务数据目录内的 `logs` |
| Codex 配置 | `%USERPROFILE%\.codex\config.toml` | `~/.codex/config.toml` |
| 修复归档 | 服务数据目录内的 `backups` | 服务数据目录内的 `backups` |

在程序目录执行 `ccodex-sleep-state paths` 可查看实际路径；Windows 使用 `.\ccodex-sleep-state.exe paths`，macOS 使用 `./ccodex-sleep-state paths`。

`CODEX_HOME` 指向 Codex 配置目录；`--data-dir` 指向本工具的数据目录；`--config` 指向本工具的 JSON 文件。它们不是同一个参数。使用自定义数据目录时，`setup`、`doctor`、`status`、`restore` 应使用同一个 `--data-dir`；使用自定义 JSON 时，`setup` 和 `paths` 等读取配置的命令也应带上相同的 `--config`。

只想查看面板而暂不接管 Codex，可以在其他实例停止后执行：

```powershell
.\ccodex-sleep-state.exe setup --data-dir "$env:LOCALAPPDATA\ccodex-trial" --no-config
```

```sh
./ccodex-sleep-state setup --data-dir "$HOME/ccodex-trial" --no-config
```

此时不会接管 Codex。需要接入时停止服务，去掉 `--no-config` 后再启动；该选项不禁止订阅下载或手动连接测试，也不会自动换端口。

## 7. 常见卡点

| 现象 | 下一步 |
| --- | --- |
| 下载后找不到 exe 或可执行程序 | 检查是否下载成源码包，或是否进入了真正包含程序的解压子目录 |
| 双击 exe 只显示帮助后退出 | 改用 `start.cmd`，或在 PowerShell 中带 `setup` 参数运行 |
| 端口被占用 | 先检查旧服务是否仍在运行，正常停止；不要同时启动多个默认实例 |
| 只有面板，没有请求记录 | 重启 Codex 并新建会话，检查 `CODEX_HOME`、profile、项目或任务覆盖是否与接管对象一致 |
| CCS 切换后出现 409 | 使用配置修复流程；后续按「停止本工具 → CCS 切换 → 启动本工具 → 重启 Codex」操作 |
| `state_unavailable` | 查看采集原因与冷却；按需要明确选择普通转发兜底或关闭注入 |
| 401 / 403 / 429 | 分别处理登录、权限或等待限流恢复，不能把本地服务启动成功当作上游权限已解决 |

平台拦截、文件权限和其他问题请继续查看 [Windows 教程](windows.md)、[macOS 教程](macos.md)及[问题与验收清单](issues-and-verification.md)。
