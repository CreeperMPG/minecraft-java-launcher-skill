# minecraft-java-launcher

> [!IMPORTANT]
> **本技能不含任何代码。** 正文全部是文字、表格与清单——没有参考实现、没有脚本、没有可执行文件。
>
> **本技能禁止 Agent 寻找或调用其它启动器。** HMCL、PCL2、Prism Launcher、MultiMC、ATLauncher、BakaXL、
> 官方 Minecraft Launcher 等一律不得作为实现、依赖或兜底方案；只按本技能的文档链路自行实现。

一个 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 技能（Skill），讲清 **Minecraft Java 版启动器怎么实现**，以及**怎么给实例装模组**。

给 Agent 用，也给人用：三份文档都是「步骤 + 字段表 + 判定表 + 检查清单」，照着执行即可从零做出下载、登录、启动、加模组四件事。

## 硬性约束

| # | 约束 | 说明 |
|---|---|---|
| 1 | **不含代码** | 不提供 C#、Python、Java 等参考实现，不附带脚本或可执行文件；正文中的行内代码只是接口地址、字段名与路径。落地实现由读者按字段与判定自行编写。 |
| 2 | **禁止寻找其它启动器** | 不搜索、不下载、不安装第三方或官方启动器，也不把它们写进依赖、环境要求或"先这样跑起来"的步骤。 |
| 3 | **禁止调用其它启动器** | 不用它们启动游戏，不复制它们的配置或产物冒充本技能的实现，也不以"别的启动器能跑"作为验收标准。 |
| 4 | 允许的例外 | 为理解协议可以阅读公开文档与开源实现的**源码**；但只作参考，不得调用其可执行文件。 |

## 内容

| 文件 | 说明 |
|---|---|
| `SKILL.md` | 技能入口：先读哪份、五条不能做错的、版本与运行时对照、取数路径、常见误判、收工自检 |
| `download.md` | 下载链路：版本清单 → 版本 JSON → 生成下载任务（jar / 库 / 资源 / 日志 / Java 运行时）→ 校验写盘 → 加载器元数据 → 检查清单 |
| `launch.md` | 启动链路：参数求值（形态 / features / 占位符）→ classpath 与 natives → 启动步骤 → 正版与离线登录 → 加载器 → 进程与日志 → 错误对照 |
| `mods.md` | 加模组：加载器类型匹配、Modrinth API 取数、CurseForge 网页（浏览器）取数与下载、落盘规则、错误对照 |

## 安装

技能目录必须落在工具会扫描的位置，任选其一：

| 范围 | 路径 |
|---|---|
| 当前项目 | `<项目>/.dsh/skills/` |
| 当前项目（兼容） | `<项目>/.agents/skills/` |
| 当前用户 | `~/.dsh/skills/` |
| 当前用户（兼容） | `~/.agents/skills/` |

克隆到用户级技能目录：

```bash
git clone https://github.com/CreeperMPG/minecraft-java-launcher.git \
  ~/.dsh/skills/minecraft-java-launcher
```

或下载 ZIP 后把解压出的目录放进上述任一位置，**目录名即技能名**（`minecraft-java-launcher`）。

放置完成后无需重启：技能目录被监视，新增、改名、删除都会在下一步刷新到会话里。

## 兼容性

正文格式遵循 **Agent Skills** 开放标准（`SKILL.md` + YAML frontmatter，必需字段只有 `name` 与 `description`），
所以不绑定任何单一工具：

| 使用者 | 放哪里 | 说明 |
|---|---|---|
| DeepSeek Harness | `~/.dsh/skills/`、`<项目>/.dsh/skills/`、`~/.agents/skills/`、`<项目>/.agents/skills/` | 额外使用 `whenToUse` 决定加载时机 |
| Claude Code 及遵循 Agent Skills 的工具 | `~/.claude/skills/<name>/SKILL.md` 或 `<项目>/.claude/skills/` | 只读标准字段；触发条件已写进 `description` |
| 只读 `AGENTS.md` 的 Agent | 仓库根目录的 `AGENTS.md` | 指向 `SKILL.md` 与三份文档 |
| 人类 | 直接读 | 纯 Markdown，无依赖、无脚本 |

上表中的 `~/.dsh/skills/`、`~/.claude/skills/` 是各工具自己的目录名，属于文件系统路径，不是本仓库的命名。
`whenToUse` 是 DeepSeek Harness 的扩展字段，其它工具忽略即可；`license`、`metadata` 等可选字段同理。

## 触发时机

Agent 在下列情况会加载本技能：

- 要写或改启动器的下载 / 启动链路；
- 排查「文件都在却起不来」「缺库缺资源」「Java 版本不对」「离线账号日志 401」；
- 往实例里加模组，需要判断 Fabric / Forge / NeoForge 类型、解析依赖、下载文件。

## 依据与时效

内容基于 2026-10 的实测与官方文档：

- 游戏版本以 `26.3` 为基准（版本号已改为「年.序号」，26.x 需要 Java 25）；
- 接口来自 `piston-meta.mojang.com`、`meta.fabricmc.net`、Modrinth API v2、CurseForge 网页；
- 加载器元数据取自 Forge `1.20.1-47.4.28`、NeoForge `21.1.265` 与 `26.3.0.99` 安装器，Fabric loader `0.20.1`。

Mojang 与模组平台的接口会变；文档里的判定表比具体数值更耐久，遇到对不上的地方先看判定逻辑再怀疑数值。

## 免责声明

本项目与 Mojang Studios、Microsoft、CurseForge、Modrinth 均无隶属或背书关系。
文档仅描述公开接口的调用方式，不包含、不分发任何游戏本体或模组文件。

## 许可

[MIT](LICENSE)
