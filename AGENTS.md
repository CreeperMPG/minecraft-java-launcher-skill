# Agent 提示

本仓库是一个 **Skill**（技能包），核心文件是 `SKILL.md`，配套三份实现文档。

## 如果你是支持技能的 Agent

加载 `SKILL.md`。格式遵循 Agent Skills 开放标准，必需字段只有 `name` 与 `description`；
`whenToUse` 是 DeepSeek Harness 的扩展字段，其它工具忽略即可。

## 如果你只读 AGENTS.md

按此顺序读：

| 顺序 | 文件 | 内容 |
|---|---|---|
| 1 | `SKILL.md` | 硬性约束、先读哪份、版本与运行时对照、取数路径、常见误判、收工自检 |
| 2 | `download.md` | 下载链路：版本清单 → 版本 JSON → 任务生成 → 校验写盘 → Java 运行时 → 加载器元数据 |
| 3 | `launch.md` | 启动链路：参数求值 → classpath 与 natives → 九步启动 → 登录 → 加载器 → 进程与日志 |
| 4 | `mods.md` | 加模组：加载器匹配、Modrinth API、CurseForge 网页取数、落盘规则 |

## 硬性约束（无论用哪个工具都适用）

1. **本仓库不含任何代码**：没有参考实现、没有脚本、没有可执行文件；正文里的行内代码只是接口地址、字段名与路径。
2. **禁止寻找或调用其它启动器**：HMCL、PCL2、Prism Launcher、MultiMC、ATLauncher、BakaXL、官方 Minecraft Launcher
   一律不得作为实现、依赖或兜底方案。
3. 只按本仓库文档给出的链路实现或执行；为理解协议可以阅读公开文档与开源实现源码，但不得调用其可执行文件。
