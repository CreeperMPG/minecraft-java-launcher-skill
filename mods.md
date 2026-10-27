# Minecraft Java 版添加模组（Agent 操作文档）

适用场景：已知目标实例目录，要把一个或多个模组装进该实例的 `mods/`。

数据源只有两个：**Modrinth API**（无需鉴权）与 **CurseForge 网页**（用浏览器访问）。
接口与页面结构基于 2026-10 实测。

---

## 1. 先确认三件事

| 项 | 从哪里取 | 为什么必须先确定 |
|---|---|---|
| 游戏版本 | 实例目录名，或 `versions/<id>/<id>.json` 的 `id` | 模组按游戏版本发布，差一个小版本就可能崩 |
| 加载器类型 | 同上，看 `mainClass` 与 `libraries` | 决定哪些模组可用，装错类型游戏起不来 |
| `mods/` 目录 | `<实例目录>/mods/` | 实例隔离时不能装到公共 `.minecraft/mods` |

加载器判定（读 `versions/<id>/<id>.json`）：

| 判据 | 结论 |
|---|---|
| `mainClass` 为 `net.fabricmc.loader.impl.launch.knot.KnotClient` | Fabric |
| `mainClass` 为 `cpw.mods.bootstraplauncher.BootstrapLauncher`，`libraries` 含 `net.minecraftforge` | Forge |
| `mainClass` 为 `cpw.mods.bootstraplauncher.BootstrapLauncher`，`libraries` 含 `net.neoforged` | NeoForge |
| `mainClass` 为 `net.neoforged.fml.startup.Client` | NeoForge（26.x） |
| `mainClass` 为 `net.minecraft.client.main.Main` | 原版，需先装加载器才能加模组 |

---

## 2. 加载器类型匹配（最关键的一步）

| 加载器 | Modrinth 的 `loaders` 值 | CurseForge 页面 "Mod Loaders" 列 |
|---|---|---|
| Fabric | `fabric` | `Fabric` |
| Forge | `forge` | `Forge` |
| NeoForge | `neoforge` | `NeoForge` |
| Quilt | `quilt` | `Quilt` |
| LiteLoader | `liteloader` | `LiteLoader` |

匹配规则：

1. 实例的加载器必须出现在模组声明的加载器列表里，否则不要装。
2. **Forge 与 NeoForge 互不通用**。1.20.2 起两者分家，仅少数桥接模组例外；不确定时按严格匹配处理。
3. Quilt 可以加载多数 Fabric 模组，但项目若只声明 `quilt`，Fabric 实例不能装。
4. 游戏版本必须精确匹配：`1.21.1` 不等于 `1.21`；`26.3` 不等于 `26.3-rc-1`。
5. 同一项目常有多个构建（如 `xxx-fabric`、`xxx-neoforge`），**按接口或页面给出的加载器字段选，不要按文件名猜**。
6. 找不到与实例精确匹配的版本时，报告"该模组不支持当前版本/加载器"，不要拿最近版本硬塞。

---

## 3. 源的选择与顺序

| 顺序 | 源 | 鉴权 | 说明 |
|---|---|---|---|
| 1 | Modrinth API | 无需 key，仅需可识别的 User-Agent | 字段完整（含 sha1/sha512、依赖、客户端/服务端侧），优先使用 |
| 2 | CurseForge 网页 | 无 | 用浏览器访问；**直连 HTTP 一律 403**（Cloudflare） |

策略：先在 Modrinth 找（按 slug 或关键词）；找不到再走 CurseForge 网页。
两者都不存在时告知用户，不要臆造下载地址。

约束：Modrinth 限流约 300 次/分钟（响应头 `X-Ratelimit-*`），请求需带可识别的 User-Agent；
CurseForge 网页按正常浏览速度访问，不要并发抓取，不要绕过其验证。

---

## 4. Modrinth 流程

### 4.1 搜索项目

请求 `https://api.modrinth.com/v2/search`，参数：

| 参数 | 说明 |
|---|---|
| `query` | 关键词或 slug |
| `limit` | 返回条数，建议 ≤ 20 |
| `facets` | 结构化过滤条件，**需 URL 编码** |

`facets` 是"数组的数组"：**外层为与（AND），内层为或（OR）**。常用片段：

| 片段 | 含义 |
|---|---|
| `["project_type:mod"]` | 只要模组（排除整合包、资源包、光影、数据包） |
| `["categories:fabric"]` | 加载器为 Fabric（换成 `forge` / `neoforge` / `quilt` 同理） |
| `["versions:1.21.1"]` | 支持该游戏版本 |

注意：搜索结果里的 `categories` 混有加载器名与功能标签（如 `optimization`），**判定加载器要看项目的 `loaders` 字段**。

### 4.2 取版本列表

请求 `https://api.modrinth.com/v2/project/<slug 或 id>/version`，参数 `loaders` 与 `game_versions`
均为 JSON 数组并需 URL 编码（如 `["fabric"]`、`["1.21.1"]`）。版本对象字段：

| 字段 | 说明 |
|---|---|
| `version_number` | 版本号（含游戏版本与加载器，如 `mc1.21.1-0.8.13-fabric`） |
| `version_type` | `release` / `beta` / `alpha` |
| `loaders` | 该构建支持的加载器 |
| `game_versions` | 支持的游戏版本列表 |
| `date_published` | 发布时间 |
| `files[]` | 下载文件，见 4.4 |
| `dependencies[]` | 依赖，见 4.3 |

选择顺序：`version_type` 为 `release` 优先，同类取 `date_published` 最新；
用户指定版本时按 `version_number` 精确取。

### 4.3 解析依赖

`dependencies[]` 每项含 `project_id`、`version_id`（可能为空）、`dependency_type`：

| `dependency_type` | 处理 |
|---|---|
| `required` | 必须安装：按 `project_id` 取其可用版本并递归处理 |
| `optional` | 可装可不装，默认不装并告知用户 |
| `incompatible` | 冲突：若已安装则提示用户 |
| `embedded` | 已内嵌在主 jar 中，**不要**再单独安装 |

批量取项目信息：`https://api.modrinth.com/v2/projects`，参数 `ids` 为项目 id 的 JSON 数组。

### 4.4 下载与校验

从 `files[]` 中取 `primary` 为真的项：

| 字段 | 用途 |
|---|---|
| `url` | 直接下载地址 |
| `filename` | 落盘文件名，沿用 |
| `hashes.sha1` / `hashes.sha512` | 校验 |
| `size` | 字节数 |

已存在判定：`https://api.modrinth.com/v2/version_file/<sha1>`，参数 `algorithm=sha1`，
能返回版本对象说明本地已有同一文件。

---

## 5. CurseForge 网页流程（浏览器）

CurseForge 全站由 Cloudflare 保护：脚本直连项目页、文件页与站点内部接口均返回 403
（"Just a moment" 挑战页）。**必须用真实浏览器内核**（无头或有头皆可）完成取数与下载。

### 5.1 启动浏览器

| 项 | 要求 |
|---|---|
| 程序 | `msedge.exe` 或 `chrome.exe` |
| 模式 | `--headless=new` 可用；若被反复挑战，改用有头模式 |
| 关键参数 | 独立 `--user-data-dir=<固定目录>`、`--disable-blink-features=AutomationControlled`、`--user-agent=<与程序匹配的真实 UA>` |
| 远程控制 | `--remote-debugging-port=<端口>`，通过 CDP 执行脚本、设置下载目录 |
| 首次访问 | 出现 Cloudflare 挑战页时**等待 30～90 秒**，直到标题变为项目名 |
| 后续访问 | 复用同一 `user-data-dir`（持有 clearance cookie），一般数秒通过 |
| 通过判据 | `document.title` 含项目名，页面出现文件表格或下载链接 |

### 5.2 定位文件页

| 页面 | 地址 |
|---|---|
| 项目页 | `https://www.curseforge.com/minecraft/mc-mods/<slug>` |
| 文件列表 | `https://www.curseforge.com/minecraft/mc-mods/<slug>/files/all?page=1&pageSize=50` |
| 单个文件页 | `https://www.curseforge.com/minecraft/mc-mods/<slug>/files/<fileId>` |
| 下载入口 | `https://www.curseforge.com/minecraft/mc-mods/<slug>/download/<fileId>` |

### 5.3 读取文件列表

在已通过挑战的页面上下文里请求文件页，解析表格。表头为：

| Type | Name | Game Versions | Mod Loaders | Size | Downloads | Uploaded |

| 列 | 取值示例 | 用途 |
|---|---|---|
| Name | `31.17.0.95 for NeoForge 26.3` | 人类可读名，含加载器与游戏版本 |
| Game Versions | `26.3` | **与实例游戏版本精确比对** |
| Mod Loaders | `NeoForge` | **与实例加载器精确比对** |
| Size | `4.5 MB` | 下载后校验大小 |
| 行内链接 | `/minecraft/mc-mods/<slug>/download/<fileId>` | 取出 `fileId` |

筛选方式：逐行比对 `Game Versions` 与 `Mod Loaders` 两列，选中唯一匹配的行，记下 `fileId`。
Type 列的 `R` / `B` / `A` 分别表示 Release / Beta / Alpha，优先 `R`。

补充：站点内部接口 `https://www.curseforge.com/api/v1/mods/<modId>/files?pageIndex=0&pageSize=20&gameVersion=<mc>`
在浏览器上下文里可返回 JSON（含 `id`、`fileName`、`gameVersions`、`fileLength`、`releaseType`），
但它**需要数字 modId**，而 modId 无法从项目页或文件页直接获得（按 slug 查询返回 403）。
因此默认走 5.3 的表格解析；仅在已知 modId 时使用该接口。

### 5.4 下载

由浏览器完成下载：

1. 通过 CDP 把下载目录设为目标 `mods/` 目录（`Page.setDownloadBehavior`，`behavior: allow`）。
2. 导航到 `https://www.curseforge.com/minecraft/mc-mods/<slug>/download/<fileId>`。
3. 等待文件出现在下载目录，且不再有 `.crdownload` 临时文件。
4. 校验：文件大小与 5.3 中该行的 Size 一致，文件名与该行对应的 jar 名一致。

### 5.5 注意事项

- 同一 `user-data-dir` 不要被多个浏览器实例同时占用。
- 挑战未通过时不要重试轰炸，等待或改用有头模式。
- 页面为动态渲染，**不要用纯 HTTP 客户端抓 HTML**（403）；所有取数都在浏览器上下文完成。
- 页面结构与内部接口属于站点实现细节，可能变动；解析失败时回退到让用户手动下载。

---

## 6. 落盘规则

| 项 | 规则 |
|---|---|
| 位置 | `<实例目录>/mods/`，仅模组放这里（资源包、光影另放） |
| 文件名 | 沿用接口或页面给出的文件名，不要改名 |
| 同名项目 | 只保留一个版本；替换前先移走或删除旧文件，否则会同时加载两份 |
| 禁用 | 把文件移出 `mods/` 或改扩展名（加载器只认 `.jar`） |
| 校验 | Modrinth 用 `hashes.sha1`；CurseForge 网页路径用文件大小比对 |
| 依赖 | `required` 必须落地；`embedded` 不重复安装 |

---

## 7. 安装后验证

1. 启动实例，读取 `<实例目录>/logs/latest.log`，确认无模组加载错误。
2. 进入游戏主菜单的"模组"列表，核对目标模组与其依赖是否出现。
3. 列表为空但文件在 `mods/` 中，说明加载器不匹配或文件损坏，按第 8 节排查。

---

## 8. 错误对照

| 现象 | 原因 | 处理 |
|---|---|---|
| 脚本请求 CurseForge 返回 403 / "Just a moment" | Cloudflare 挑战，纯 HTTP 客户端无法通过 | 改用浏览器（5.1） |
| 模组列表里没有该模组 | 加载器类型不匹配，或文件不是 `.jar` | 按第 2 节重新选构建 |
| 启动即崩，日志出现 `ClassNotFoundException` / `NoSuchMethodError` | 游戏版本不匹配 | 换对应游戏版本的构建 |
| 日志出现 Mixin 或 ASM 异常 | 与其他模组冲突，或原版自带 ASM 抢先加载（1.21.2 起） | 二分排查；ASM 冲突见启动文档 §7.3 |
| 同一模组两份文件同时在 `mods/` | 重复安装 | 只保留与实例匹配的那一份 |
| 下载到的文件只有几 KB，打开是 HTML | 下的是页面而不是 jar | 走 5.4 的浏览器下载，或让用户手动下载 |
| 提示缺少前置模组 | `required` 依赖未安装 | 递归安装依赖 |

---

## 附录：地址速查

| 用途 | 地址 |
|---|---|
| Modrinth 搜索 | `https://api.modrinth.com/v2/search` |
| Modrinth 版本列表 | `https://api.modrinth.com/v2/project/<slug>/version` |
| Modrinth 批量项目 | `https://api.modrinth.com/v2/projects?ids=[...]` |
| Modrinth 哈希反查 | `https://api.modrinth.com/v2/version_file/<sha1>?algorithm=sha1` |
| CurseForge 项目页 | `https://www.curseforge.com/minecraft/mc-mods/<slug>` |
| CurseForge 文件列表 | `https://www.curseforge.com/minecraft/mc-mods/<slug>/files/all?page=1&pageSize=50` |
| CurseForge 下载入口 | `https://www.curseforge.com/minecraft/mc-mods/<slug>/download/<fileId>` |
