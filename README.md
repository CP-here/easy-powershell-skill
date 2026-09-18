# easy-powershell-skill

**语言切换：** [English](README.en.md) · [中文](README.md)

> 面向 AI 代理的 Windows PowerShell 命令执行规则与速查。一次就选对原生命令，终结「先试 Linux 命令、报错再返工」的循环。

在 Windows 上工作时，AI 代理习惯先敲 `grep`、`ls -la`、`rm -rf`，报错后才改用 `Select-String`、`Get-ChildItem`、`Remove-Item`。本技能把「先选对命令」定为硬性规则，并将高频对照、语法陷阱、调用模式与编码问题整理成按需加载的参考文件。

## 特性

- **规则驱动**：5 条硬规则——禁用 Unix 命令、不用别名、失败不原样重试——从源头切断试错循环。
- **主体极简**：`SKILL.md` 一页读完，只保留核心规则、高频速查与破坏性操作的强制规范。
- **按需加载**：`references/` 下 6 份文件覆盖对照表、陷阱、外部调用、编码、安全与系统管理，用到才读，不浪费上下文。
- **实测陷阱**：括号规则、`foreach` 管道解析、`ConvertTo-Json -Depth`、`$?` 与 `$LASTEXITCODE` 的区别、紧凑参数被外部程序拆散——都是 Agent 高频踩坑点。
- **安全护栏**：高影响操作确认表、递归删除预检、执行策略一次性绕过、失败分类与重试纪律。
- **中文友好**：面向中文输出与中文路径的编码方案（UTF-8 with BOM、`PYTHONIOENCODING`、`$OutputEncoding`）。

## 目录结构

```
easy-powershell-skill/
├── SKILL.md                        # 主体：硬规则 + 高频速查 + 破坏性操作必守项
├── README.md                       # 中文 README（主 README，本文件）
├── README.en.md                    # 英文 README
└── references/
    ├── bash-to-powershell.md       # bash → PowerShell → CMD 对照表 + 报错排查
    ├── pitfalls.md                 # 14 个语法陷阱（括号、JSON 深度、空值、退出码等）
    ├── native-invocation.md        # 外部 exe 调用、传参、临时 .ps1、Start-Process
    ├── encoding.md                 # 中文输出、BOM、5.1/7 版本差异
    ├── safety.md                   # 高影响确认、递归删除预检、失败重试纪律
    └── windows-admin.md            # 注册表/服务/网络/用户/计划任务/事件日志速查
```

## 安装

### 方式一：`npx skills`（推荐）

```bash
npx skills add <本仓库地址> --skill easy-powershell-skill
```

### 方式二：复制到项目内的技能目录

若要让 Agent 在**某个具体的 Windows 项目里**遵守这些规则，将整个 `easy-powershell-skill/` 目录复制到该项目的 Agent 技能目录：

| Agent | 项目级路径 |
|---|---|
| Claude Code | `<项目>/.claude/skills/easy-powershell-skill/` |
| Codex | `<项目>/.codex/skills/easy-powershell-skill/` |
| Cursor | `<项目>/.cursor/skills/easy-powershell-skill/` |
| 其他框架 | 放在 Agent 能发现 `SKILL.md` 的任意位置 |

项目级安装让技能随仓库分发：提交后，任何打开该项目的 Agent 都会自动遵循这些规则。

### 方式三：复制到用户级技能目录

| Agent | 用户级路径 |
|---|---|
| Claude Code | `~/.claude/skills/easy-powershell-skill/` |
| Codex | `~/.codex/skills/easy-powershell-skill/` |

安装后重启 Agent 会话，以便刷新技能索引。

## 使用

技能会在 `SKILL.md` 的 `description` 字段所描述的场景中自动触发：

- 在 Windows 终端执行命令，或编写、调试 PowerShell 脚本
- 把 bash/Linux 命令翻译成 PowerShell
- 排查在 Windows 上报错的命令
- 删除、覆盖、停止服务、修改注册表等破坏性操作

日常命令（列目录、`git status`、版本查看）直接执行，不触发本技能。

## 设计原则

1. **主体即纪律**：`SKILL.md` 只保留每次都必须遵守的内容，一页读完。
2. **扩展即手册**：细节全部外置到 `references/`，每份都有明确的触发场景，按需读取。
3. **个人可裁剪**：本技能为个人自用而设计——删掉用不到的参考文件不影响主体；需要更细的管理命令时，直接在 `windows-admin.md` 中追加即可。

## 参考与致谢

本技能综合参考了以下开源项目（均为 MIT License）：

- [jsrgjcy/powershell-windows-skill](https://github.com/jsrgjcy/powershell-windows-skill) — 高影响操作确认流程、12 领域系统管理参考
- [UncertaintyDeterminesYou4ndMe/powershell-windows-cli-agent-skill](https://github.com/UncertaintyDeterminesYou4ndMe/powershell-windows-cli-agent-skill) — bash 到 PowerShell 的翻译、`-WhatIf` 安全模式
- [agent-shells/powershell-skills](https://github.com/agent-shells/powershell-skills) — 模式目录与真实失败案例库
- [Misaka-Mikoto-Tech/agent-skills](https://github.com/Misaka-Mikoto-Tech/agent-skills) — `powershell-safe-invocation` 原生调用模式
- [sickn33/antigravity-awesome-skills](https://github.com/sickn33/antigravity-awesome-skills) — 操作符括号、纯 ASCII 输出、空值检查陷阱
- [GuanKr/pwsh-pitfalls](https://github.com/GuanKr/pwsh-pitfalls) — 两套退出码体系、`find` 陷阱、编码边界
- [dfinke/powershell-ai-skills](https://github.com/dfinke/powershell-ai-skills) — PowerShell 维护实践

## License

MIT
