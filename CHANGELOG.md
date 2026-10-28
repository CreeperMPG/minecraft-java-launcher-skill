# Changelog

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，
版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

## [1.0.0] - 2026-10-28

### 新增

- `SKILL.md`：技能入口，含硬性约束、先读哪份、五条不能做错的、版本与运行时对照、取数路径、常见误判、收工自检。
- `SKILL.md` 的「0. 硬性约束」：禁止寻找其它启动器、禁止调用其它启动器、只按本技能链路实现；
  允许阅读公开文档与源码作参考，但不得调用其可执行文件；技能本身不含代码。
- `download.md`：下载链路实现文档——版本清单、版本 JSON、下载任务生成、校验与写盘、Java 运行时、加载器元数据、检查清单。
- `launch.md`：启动链路实现文档——参数求值与占位符、classpath 与 natives、九步启动流程、微软账号登录、离线登录、加载器装配、进程与日志、错误对照。
- `mods.md`：加模组实现文档——实例加载器判定、加载器类型匹配、Modrinth API 流程、CurseForge 网页（浏览器）流程、落盘规则、错误对照。
- `README.md`：安装方式、内容清单、兼容性、触发时机、依据与免责；标题下方以 `[!IMPORTANT]` 提示块标注两条硬性约束。
- `AGENTS.md`：给只读 AGENTS.md、不认技能的 Agent 的入口，指向 `SKILL.md` 与三份文档，并复述硬性约束。

### 变更

- 仓库定名 `minecraft-java-launcher-skill`，不再带工具前缀。
- 项目描述为 **Agent Skill**，不绑定单一工具；格式遵循 Agent Skills 开放标准（必需字段只有 `name` 与 `description`）。
- `SKILL.md` 的 `description` 写入了触发条件，只读标准字段的工具也能判断何时加载；新增 `license: MIT` 字段。
- `README.md` 新增「兼容性」一节，列出各家工具的技能目录与用法；文档内出现的 `~/.dsh/skills/` 等属于各工具自身的路径。

### 说明

- 技能正文不含任何代码：无参考实现、无脚本、无可执行文件，行内代码仅为接口地址、字段名与路径。
- 依据 2026-10 实测：游戏版本 `26.3`（Java 25）；Forge `1.20.1-47.4.28`、NeoForge `21.1.265` 与 `26.3.0.99`、Fabric loader `0.20.1`。
