# AOCI-CODE - Windows 兼容性分析

> 生成时间: 2026-10-13
> 分析对象: github.com/aoci-spec/aoci-code @ main (7586820)
> 评估环境参考: Windows 11 + PowerShell 5.1（项目明文契约的目标宿主）

## 总体结论

**Windows 是一等公民平台，产品级兼容。** 这不是"碰巧能编译"的跨平台，而是逐语义适配：原子发布用 Win32 `ReplaceFileW`、锁观察用显式 `FILE_SHARE_DELETE`、子进程抑制控制台闪窗、JSON 输出针对 PowerShell 5 代码页做 ASCII 安全转换，且有专门的中英双语文档（`docs/windows-host-agent.md` 为规范原文）与原生 windows-latest CI 门禁。普通用户在 Windows 上安装、使用、参与数据库认知**均无障碍**；仅**开发者参与源码构建/跑全量门禁**时需要类 Unix 环境（Git Bash/WSL）执行 6 个 .sh 脚本与 Makefile 目标。

| 维度 | 评级 | 说明 |
|------|------|------|
| 运行时（终端用户） | ★★★★★ | 原生 Win32 语义适配 + 发布 zip 包 + CI 原生验证 |
| 宿主集成（MCP） | ★★★★★ | Claude Code/Codex/Cursor/Qoder on Windows 有专门文档与测试 |
| PowerShell 5 契约 | ★★★★☆ | 有明文契约；中文输出已做 ASCII 安全 JSON，但控制台交互仍受代码页物理限制 |
| 开发者构建 | ★★★★☆ | `make build` 在 PowerShell 可用；部分门禁脚本是 bash-only |
| CI | ★★★★★ | 三条工作流覆盖 Windows（编译/一致性/场景/发布演练） |

## 1. 平台适配实现（源码证据）

`internal/fs` 采用 Go 惯例的按 OS 分文件（`_windows.go` + build tag），共 6 组 Windows 专属实现：

| 文件 | 机制 | 解决的 Windows 特有问题 |
|------|------|------------------------|
| `atomic_exchange_windows.go` | Win32 `ReplaceFileW`（kernel32，经 `golang.org/x/sys/windows` LazyDLL） | 保证 canonical 文件始终完整，被替换版本存为 `.exchange-backup` 恢复副本，再归一进公共 swap 状态机 |
| `atomic_create_windows.go` | 对应平台创建语义 | 原子创建不降级 |
| `lock_owner_open_windows.go` | `CreateFile` + 显式 `FILE_SHARE_READ\|WRITE\|DELETE` | Go 默认 `os.Open` 的共享语义不符合锁协议假设；观察者的短时只读句柄不能泄漏为 Release 阻断条件；并识别"名称已删除但句柄清理未完成"的短暂窗口（`ERROR_ACCESS_DENIED` / `ERROR_SHARING_VIOLATION` / `ERROR_DELETE_PENDING` → 重试而非误报） |
| `process_identity_windows.go` | Windows 进程身份 | 确认锁属主进程是否存活（配合最近的"锁属主在 9p 下消失"修复——Windows 文件共享/网络文件系统场景） |
| `git_command_windows.go` | `CREATE_NO_WINDOW`(0x08000000) + `HideWindow` 兜底 | aoci 常以无控制台形态运行（MCP stdio 由宿主拉起）；否则 exec 拉起 git.exe 会反复弹控制台窗口抢占焦点。只抑制控制台分配，不改 stdio 句柄继承 |
| `safe_inventory_platform_windows.go` | 平台清单差异 | Git 工作树清单的 Windows 路径/属性处理 |

其他平台适配点：

- `internal/ui/detach_windows.go`：面板 `--detach` 后台化的 Windows 实现（与 `detach_unix.go` 对应）；
- `internal/cli/json_ascii.go`：**普通 CLI 的合法 `--json` 结果在最终写出时转为 ASCII 安全 JSON**——直接针对"PowerShell 5 用本地代码页捕获 UTF-8 中文破坏 JSON"（`internal/cli/root.go` 注释与 `index_agent_host_contract_windows_json_test.go` 测试锁定）；
- `internal/cli/stdin_limit.go` 等：无平台分支的通用实现。

## 2. 构建与发布

- **CGO_ENABLED=0**（`.goreleaser.yml`）：纯 Go、无 cgo，天然可交叉编译、无运行时 DLL 依赖；
- 发布矩阵：linux / darwin / windows × amd64 / arm64，共 6 归档；**Windows 用 zip**（`format_overrides`），其余 tar.gz；
- 每归档附 SBOM；SHA256SUMS 由签名工作流在并入 SBOM 后重建；
- `-trimpath -s -w -buildid=` + `mod_timestamp` 固定：可重现构建；
- Makefile 的发布/验证目标用 `BUILD_BIN`（后缀取自 `go env GOEXE`）——Windows 下自动得到 `.exe` 后缀；
- 源码构建（PowerShell，摘自 README，已验证与 Makefile 一致）：

```powershell
git clone https://github.com/aoci-spec/aoci-code.git
Set-Location .\aoci-code
New-Item -ItemType Directory -Force .\build | Out-Null
make build          # 需要 make（Git Bash 自带 / w64devkit / MSYS2）
.\build\aoci.exe --version
```

## 3. CI 的 Windows 覆盖

| 工作流 | Windows 内容 |
|--------|--------------|
| `ci.yml`（每个 PR） | `windows-compile`：ubuntu 上 `GOOS=windows GOARCH=amd64` 交叉编译冒烟；`windows-conformance`：**windows-latest 原生运行**纯标准库一致性套件（零 Docker，Windows 上成本最低、盲区收益最大——工作流注释原话） |
| `full-confidence.yml` | 三 OS 矩阵 `windows-latest / macos-15-intel / macos-15` + 专属 `windows-scenarios` job（windows-latest） |
| `release-rehearsal.yml` | 矩阵含 `windows-latest`（非发布发布演练） |
| `release.yml` | 打包（含 Windows zip）在 ubuntu；签名/attestation 与平台无关 |

## 4. 文档与契约

- `docs/windows-host-agent.md`（**中文为规范原文**，被跨层与 CLI 测试字节锁定）+ `docs/windows-host-agent.en.md`（英文渲染，先落后中文原文、增量跟随）；
- 覆盖：Windows 宿主集成、运行时身份、Guide 执行、**PowerShell 安全工作流**、Volumes 授权批次的 no-staging-files 规则；
- README 提供并行的 bash / PowerShell 命令块（克隆、构建、init、scan）。

## 5. 已知限制与注意事项

### 5.1 开发者侧（不影响终端用户）

| 项 | 现状 | 影响 |
|----|------|------|
| 6 个门禁脚本是 bash-only | `check-coverage-floors.sh`、`check-deps.sh`、`check-licenses.sh`、`check-opengauss-connector.sh`、`check-public-text.sh`、`release/clean-room-smoke.sh`（均为 `#!/usr/bin/env bash`，无 .ps1/.cmd 对应物） | 在 Windows 上跑 `make full` / `make verify` / 发布检查需要 Git Bash 或 WSL（`make` 本身 Git Bash 也自带）。日常 `make build` / `make fast` 的核心路径不受影响 |
| 黑盒套件是 Python | `scripts/blackbox/*.py`（含 stdio 捕获/死线库） | 需要 python3；Windows CI 的 conformance 子集已在原生跑，全量三套件照 AGENTS.md 在 Linux 门禁中运行 |
| Makefile 目标名混 bash 语法 | 部分目标内联 shell 命令 | Git Bash 下无问题；纯 PowerShell 直跑个别目标可能需适配 |

### 5.2 用户侧注意点

1. **路径**：所有示例要求"稳定绝对路径"存放 `aoci.exe`（MCP 配置指向该路径）；项目内操作已由 `--repo` + `safe_path.go` 处理正反斜杠与越界；
2. **PowerShell 5 的代码页**：中文 `--json` 输出已由二进制转 ASCII 安全；但人读输出（非 JSON）与控制台渲染仍受系统代码页物理限制，中文 Windows 默认 GBK——建议 `chcp 65001` 或使用 Windows Terminal + PowerShell 7 获得最佳体验；
3. **行尾**：仓库 `.gitattributes` 规范化行尾保证跨平台哈希确定性（`autocrlf` 改写会导致整树 stale——这是被索引条目点名的 S 级约束）；克隆本项目贡献代码时保持默认配置即可；
4. **MCP stdio**：宿主在 Windows 上拉起 `aoci mcp` 时 stdout 是 JSON-RPC，日志全走 stderr（已固定）；git.exe 子进程不会弹窗（§1 的 CREATE_NO_WINDOW）；
5. **网络文件系统/9p**：锁属主消失场景（Windows 文件共享、WSL 9p 挂载）有专门修复与测试（`lock_vanished_owner_test.go`），属已覆盖边界；
6. **Docker**：项目零 Docker 依赖（CI 数据库腿除外），Windows 上无需容器环境。

## 6. 与脚本次扫描结果的差异说明

自动化脚本（`05-Windows兼容性检查.md`）报告 ci.yml"包含 Windows"的初判有误导——实际是 ubuntu 交叉编译 job + windows-latest 原生 job 的组合（见 §3，已人工核实）。脚本亦未检出平台守卫/child_process 项，因为它是 Go 项目：对应物是 build tag 分文件与 `os/exec` 的 `SysProcAttr` 平台设置，本次已通过源码阅读补全。

## 7. 结论与建议

- **作为用户**：Windows 11 + PowerShell 环境可直接采用官方 zip 发布包（验证签名后）走标准 init → scan → 建索引流程，无需任何类 Unix 工具；
- **作为贡献者**：建议在 Git Bash（或 WSL2）中开发，`make build` / `make fast` 日常可用，提 PR 前的全量门禁（`make verify`、licenses、blackbox 套件）在 CI 的 ubuntu/windows 双端都有兜底；
- **可提的改进**（非必需）：为 6 个 bash 门禁脚本提供 .ps1 对应物会让纯 PowerShell 开发者闭环更完整——但项目显然选择了"CI 兜底 + 文档声明"的路线，且 `docs/windows-host-agent.md` 已把边界写清，属于自觉的取舍而非疏漏。
