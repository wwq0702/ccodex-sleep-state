# 构建与维护

应用本身是 Go，发布包运行时不用装编译器。只想运行程序请先看[完整部署流程](deployment.md)。开发需要 Go 和 Git；代理核心带来较多依赖，第一次下载和编译会比之后慢。

## 从本 Fork 构建

当前 `go.mod` 的最低 Go 版本是 1.26.0，CI 使用 Go 1.27.x。先用 `git --version` 和 `go version` 确认工具可用。

```sh
git clone https://github.com/wwq0702/ccodex-sleep-state.git
cd ccodex-sleep-state
go test ./...
go build -trimpath -o ccodex-sleep-state ./cmd/ccodex-sleep-state
./ccodex-sleep-state version
```

Windows PowerShell 的最后两条命令改为：

```powershell
go build -trimpath -o ccodex-sleep-state.exe ./cmd/ccodex-sleep-state
.\ccodex-sleep-state.exe version
```

未通过 `-ldflags` 注入版本号时，输出 `dev` 正常。构建完成不自动启动服务；接入本机 Codex 需要另行执行 `setup`，详见[启动步骤](deployment.md#3-启动并接入-codex)。Go module 路径仍沿用上游，克隆本 Fork 不要求修改 `go.mod` 或全仓库的导入路径。

## 检查改动

```sh
gofmt -w cmd internal
go vet ./...
go test -race -count=1 ./...
go test ./internal/turnstate -fuzz=FuzzParse -fuzztime=10s
```

测试里的认证、代理密码、state 都是合成数据。测试只访问本地模拟服务器，不需要真实订阅或模型账号。配置文件和日志放在测试临时目录。`-race` 需要当前系统支持的竞态检测工具链；Windows 通常还需要受支持的 C 编译工具链。

直接运行 `setup` / `serve` 和运行测试不是一回事：前者默认接管当前用户的 Codex。想手工测试启动，显式提供隔离的 `--data-dir` 并用 `--no-config`；要测试配置修改，把服务 JSON 的 `codex_home` 指向另一个测试目录。

## 参数表

所有参数都在本程序的 JSON 配置里，未知字段会报错。

| 字段 | 默认值 | 作用 |
|---|---|---|
| `model` | `gpt-6-astra` | 默认模型；允许 Astra、`gpt-5.6-sol`、`gpt-5.6-terra`，不替上游开通权限 |
| `account_mode` | `auto` | 自动套餐提示 / `personal` 10 块 / `team` 12 块；提示不是认证证明 |
| `state_fallback` | 内核空值为 `strict`；首次 `setup` 无明确值时设为 `passthrough` | `setup` 保留已明确的 `strict`；首页再次接入确认后选择兜底，不解除拒绝或重放 |
| `upstream_mode` | 自动 | 跟随当前 Codex 上游；`manual` 要自行确认认证与目标 |
| `upstream_kind` | `official` | `relay` 为 API 通道，不采集官方 state |
| `codex_profile` | 空 | 显式选择 profile；客户端必须使用同一 profile |
| `injection_disabled` | `false` | 关闭采集与注入，但仍通过本服务转发 |
| `pinned_route` | 空 | 固定匿名出口 ID；空为自动选择 |
| `listen` | `127.0.0.1:17841` | 唯一回环监听地址 |
| `upstream` | `https://chatgpt.com/backend-api/codex` | 上游；修改它会改变认证与正文的接收方 |
| `codex_home` | 自动检测 | 优先显式路径，其次 `CODEX_HOME`，再用户 `.codex` |
| `direct` | `true` | 将直连放在出口池最前；代理专用部署改为 `false` |
| `proxy_urls` / `proxy_envs` | 空 | 直接代理 URI / 存放 URI 的环境变量名 |
| `subscriptions` | 空 | `{url}` 或 `{url_env}` 列表；每项可设置 `user_agent`、`include_protocols`、`exclude_keywords` |
| `subscription_proxy_env` | 空 | 仅订阅下载使用的代理变量 |
| `probe_timeout_seconds` | `20` | 单次探测超时，1–60 秒 |
| `max_probes_per_round` | `6` | 每轮最多尝试出口数，1–20；单轮不重复同一出口 |
| `probe_cooldown_seconds` | `180` | 两轮采集最小间隔，30–3600 秒 |
| `state_ttl_seconds` | `3600` | 本地估计 TTL，120–3600 秒 |
| `refresh_before_seconds` | `1200` | 估计过期前进入续补窗口，至少 30 秒且小于 TTL |
| `baseline_blocks` | `10` | 旧版自定义基线；自动模式保留非默认值，明确选择个人 / Team 可覆盖该旧规则；不是质量保证 |

不要为了“更积极”把这些限制一路调小。先看请求记录与实际效果，保留对照实验；没有对照数据，不给出质量提升百分比。

## 构建安装包和发布版本

Fork 只复制代码和工作流文件，不复制上游已发布的安装包。以下构建和发布操作针对 `wwq0702/ccodex-sleep-state`，不会向上游仓库发版本。

### 本地打包，不上传

先完成上面的检查并提交要打包的改动，确保 `git status --short` 没有输出。在 macOS 或具备 Bash、Go、Git、zip、tar、shasum 的环境中，于仓库根目录执行：

```sh
VERSION="local-$(git rev-parse --short HEAD)" bash scripts/build-release.sh
```

脚本会拒绝未提交的改动和已经存在的输出目录，避免程序与源码包不一致。再次构建时更换 `VERSION` 或指定一个尚不存在的 `OUT` 目录。

产物位于 `dist/<VERSION>/`：四个平台包、`ccodex-sleep-state-source.tar.gz` 和 `SHA256SUMS`。源码包包含锁定的依赖及其许可证；Windows 包附 `start.cmd`，macOS 包附可执行的 `start.command`，两者都附 README、教程与许可证。脚本会生成并核对 SHA256，但不会创建 Git 标签、上传文件或发布 Release。

### 通过 Actions 获取预览包

1. 打开[本仓库 Actions](https://github.com/wwq0702/ccodex-sleep-state/actions)。若 GitHub 提示 Fork 的工作流未启用，先在页面启用。
2. 选择 [Preview build](https://github.com/wwq0702/ccodex-sleep-state/actions/workflows/preview.yml)，点击 **Run workflow** 并选择要构建的分支。
3. 等待该次运行的静态检查、竞态测试和打包全部成功，确认运行记录的提交 SHA 与目标提交一致。
4. 在运行记录的 **Artifacts** 下载预览产物，先解开外层 artifact，再选择对应系统的包并核对 `SHA256SUMS`。网页下载通常需要登录 GitHub。

预览工作流也会在推送 `codex/pool-upgrade` 分支时自动运行；其他分支要用手动入口。产物保留 14 天，工作流当前使用 `v0.4.0-preview-<提交>` 命名，它不创建版本标签或 Release。

### 发布到本 Fork 的 Releases

仅在准备向用户发布时执行。先把目标提交推送到本仓库，确认该提交的 CI 通过，核对 LICENSE、变更说明、问题验收清单和敏感信息扫描结果，并在独立目录检查实际安装包。

以下以一个尚未使用的 `v0.4.1` 为例，实际发布时换成你选定的新版本号；不要覆盖已有标签：

```sh
git remote get-url origin
git status --short
git rev-parse HEAD
git tag -a v0.4.1 -m "Release v0.4.1"
git push origin v0.4.1
```

执行标签命令前，应确认 `origin` 指向 `wwq0702/ccodex-sleep-state`、工作区干净、当前 HEAD 就是已经通过验证的目标提交。**推送 `v*` 标签会自动启动发布流程**，单纯推送代码不会发布安装包。

[Release 工作流](https://github.com/wwq0702/ccodex-sleep-state/actions/workflows/release.yml)按以下顺序执行：

1. Windows、macOS、Ubuntu 分别运行 `go vet` 和竞态测试。
2. 全部通过后交叉编译四个平台的包，生成完整源码包及 `SHA256SUMS`。
3. 上传产物并创建 GitHub **Pre-release**；当前脚本固定带 `--prerelease`，不会自动标为正式稳定版。

到[本仓库 Releases](https://github.com/wwq0702/ccodex-sleep-state/releases)确认附件齐全、版本标签和提交正确，并下载对应平台包完成启动、接入、停止恢复检查。工作流已经触发或某个测试任务成功，都不等于发布完成；当前自动发布说明是模板，发布前应核对它与本次改动一致。

仓库只保留维护所需的源文件、测试和文档；构建缓存、运行目录、日志、私有配置、个人环境快照都不属于版本历史。
