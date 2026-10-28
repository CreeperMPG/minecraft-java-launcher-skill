---
name: minecraft-java-launcher
description: 实现或维护 Minecraft Java 版启动器（下载、安装、登录、启动），以及给实例装模组（Modrinth API / CurseForge 网页，含加载器类型匹配）。附三份实现文档。本技能不含代码，且禁止寻找或调用其它启动器。
whenToUse: 要写或改启动器的下载/启动链路、排查"文件都在却起不来""缺库缺资源""Java 版本不对""离线账号日志 401"，或往实例里加模组（Fabric / Forge / NeoForge 类型匹配、依赖、下载）时。
---

# Minecraft Java 版启动器与加模组

启动器只做两件事：**按契约把文件下载到位**，**拼一条 java 命令把它跑起来**。
正文是三份实现文档，本文只回答"先读哪份、什么不能做错"。

---

## 0. 硬性约束（先读，违反即未完成任务）

| # | 约束 | 具体做法 |
|---|---|---|
| 1 | **禁止寻找其它启动器** | 不搜索、不下载、不安装 HMCL、PCL2、Prism Launcher、MultiMC、ATLauncher、BakaXL、官方 Minecraft Launcher 等；不把它们写进依赖清单、环境要求或"先这样跑起来"的步骤。 |
| 2 | **禁止调用其它启动器** | 不用它们的可执行文件启动游戏；不复制它们的配置或产物冒充本技能的实现；不以"别的启动器能跑通"作为验收标准。 |
| 3 | **只按本文档的链路实现** | 下载与校验、参数求值、classpath、natives、登录、进程拉起，全部按 `download.md`、`launch.md`、`mods.md` 的步骤自行实现或执行。 |
| 4 | 允许的参考 | 为理解协议可以阅读公开文档与开源实现的**源码**；仅作参考，不得调用其可执行文件。 |
| 5 | **本技能不含代码** | 文档只给步骤、字段与判定；需要代码时由使用者按本文档自行编写。 |

遇到"用别人现成的启动器更快"的念头时，先回到第 3 条：本技能的交付物是**按文档实现的链路**，不是把游戏跑起来的结果。

---

## 1. 先读哪一份

| 任务 | 读 |
|---|---|
| 下载、安装、校验、Java 运行时 | `download.md` |
| 启动、参数、classpath、登录 | `launch.md` |
| 加模组、取数、加载器匹配 | `mods.md` |

三份文档均为「步骤 + 字段表 + 判定表 + 检查清单」，可直接照着执行。
本文以下各节是摘要；与文档冲突时，以文档为准。

---

## 2. 五条不能做错的

| # | 规则 | 依据 |
|---|---|---|
| 1 | 加载器类型由 `mainClass` 与 `libraries` 判定，不按版本号或文件名猜 | mods.md §1 |
| 2 | natives 的判据是「版本 JSON 里有没有 `natives` 键」，不是版本号区间 | launch.md §3.2 |
| 3 | 文件就绪 = 大小 + SHA-1 双校验；写入用临时文件后原子改名 | download.md §4 |
| 4 | classpath 一律绝对路径，且顺序有意义（同名类先出现者胜） | launch.md §3.1 |
| 5 | 离线账号（`--accessToken` 为 `0`）日志中的 401、Realms 认证失败属预期现象 | download.md §7 |

---

## 3. 版本与运行时对照

| 游戏版本 | Java 大版本 | 运行时组件 |
|---|---|---|
| 1.17 及以前 | 8 | `jre-legacy` |
| 1.18 – 1.20.4 | 17 | `java-runtime-gamma` |
| 1.20.5 – 1.21.11 | 21 | `java-runtime-delta` |
| 26.1 起（含 26.3） | 25 | `java-runtime-epsilon` |

实际以版本 JSON 的 `javaVersion` 为准；大版本不符会直接 `UnsupportedClassVersionError`。

---

## 4. 加载器与取数路径

| 加载器 | Modrinth 的 `loaders` | CurseForge 页面 "Mod Loaders" 列 |
|---|---|---|
| Fabric | `fabric` | `Fabric` |
| Forge | `forge` | `Forge` |
| NeoForge | `neoforge` | `NeoForge` |
| Quilt | `quilt` | `Quilt` |

Forge 与 NeoForge 互不通用（1.20.2 起分家）。

| 需求 | 取数路径 |
|---|---|
| 原版文件 | `piston-meta` 清单 → 版本 JSON → `piston-data` / `libraries.minecraft.net` / `resources.download.minecraft.net` |
| 模组（首选） | Modrinth API v2，无需 key |
| 模组（次选） | CurseForge 网页，**必须用浏览器**；纯 HTTP 客户端一律 403 |
| 加载器文件 | Fabric：`meta.fabricmc.net`；Forge / NeoForge：安装器 jar（`install_profile.json` + `version.json`） |

---

## 5. 常见误判

| 症状 | 容易误判为 | 实际原因 |
|---|---|---|
| 文件齐了但起不来 | 下载失败 | 1.19 起 natives 变普通 jar 进 classpath，老办法解压后没加进 classpath（launch.md §3.2） |
| 日志出现 401 / Realms 失败 | 安装或登录坏了 | 离线令牌本就不是正版会话（download.md §7） |
| 模组装了不生效 | 模组文件损坏 | 加载器类型不匹配，或游戏版本非精确匹配（mods.md §2） |
| 能进主菜单但无字体、无音效 | 资源没下全 | `assets/objects` 不全或 `--assetIndex` 错误（download.md §3.3） |
| `UnsupportedClassVersionError` | Java 安装有问题 | 所选 Java 与 `javaVersion.majorVersion` 不一致（第 3 节） |
| 1.6 及以前黑屏 | 下载中断 | 索引 `virtual: true` 但未铺 `assets/virtual/legacy`（download.md §3.3） |
| Forge 报缺少强制依赖 | 模组冲突 | 安装器 processor 链未跑完，`PATCHED` 库缺失（launch.md §7.2） |

---

## 6. 收工自检

- `versions/<id>/<id>.json` 与 `<id>.jar` 同名同目录，jar 文件名即版本 id
- 全部库文件 SHA-1 正确；`virtual` / `map_to_resources` 已铺开
- 日志配置已落盘，`${path}` 指向其绝对路径
- 所选 Java 大版本等于 `javaVersion.majorVersion`
- 装模组时按 `loaders` / "Mod Loaders" 列精确匹配，同项目只保留一份
- 进程退出码非 0 时，先读 `crash-reports/` 与 `logs/latest.log`，不要只看退出码
