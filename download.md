# Minecraft Java 版下载实现文档

本文给出从 Mojang 公开接口获取 Java 版客户端全部文件的流程。按第 1～4 节顺序执行即可得到可启动的游戏目录；
第 5 节为模组加载器的附加内容。

接口与字段基于 2026-10 的正式版 `26.3`，历史版本差异在对应位置标注。

---

## 0. 目标目录结构

| 路径 | 内容 |
|---|---|
| `versions/<id>/<id>.json` | 版本 JSON（第 2 节） |
| `versions/<id>/<id>.jar` | 客户端 jar（第 3.1 节） |
| `libraries/<group>/<artifact>/<version>/<artifact>-<version>.jar` | 依赖库（第 3.2 节） |
| `assets/indexes/<indexId>.json` | 资源索引（第 3.3 节） |
| `assets/objects/<hash前2位>/<hash>` | 资源本体 |
| `assets/virtual/legacy/...` | 仅 1.6.x 需要 |
| `runtime/<component>/<平台>/...` | Java 运行时（第 3.5 节） |
| `log_configs/client-<ver>.xml` | 日志配置（第 3.4 节） |

`libraries/`、`versions/`、`assets/` 三个目录名不可更改，模组加载器与工具链依赖这些相对位置。

---

## 1. 获取版本清单

请求 `https://piston-meta.mojang.com/mc/game/version_manifest_v2.json`。

响应字段：

| 字段 | 说明 |
|---|---|
| `latest.release` / `latest.snapshot` | 最新正式版 / 最新快照的版本 id |
| `versions[]` | 全部版本，按时间倒序 |
| `versions[].id` | 版本号，如 `26.3` |
| `versions[].type` | `release` / `snapshot` / `old_beta` / `old_alpha` |
| `versions[].url` | 该版本 JSON 的地址（第 2 节） |
| `versions[].sha1` | 版本 JSON 的 SHA-1 |
| `versions[].time` | 最后修改时间（版本 JSON 可能被改写） |
| `versions[].releaseTime` | 首次发布时间 |
| `versions[].complianceLevel` | 玩家安全合规等级，`1.16.4-pre2` 之前为 0 |

步骤：按 `id` 选中目标版本，记录 `url` 与 `id`。

缓存：清单变动频繁，可设 ETag 条件请求，缓存不超过 15 分钟。

---

## 2. 获取版本 JSON

下载 `versions[].url`，保存为 `versions/<id>/<id>.json`。

### 2.1 必要字段

| 字段 | 用途 |
|---|---|
| `id` | 版本名，与目录名一致 |
| `mainClass` | 启动主类，启动时位于 classpath 之后（见启动文档第 4 节第 8 步） |
| `assets` / `assetIndex` | 资源索引 id 与下载信息（第 3.3 节） |
| `downloads.client` | 客户端 jar 的 `url` / `sha1` / `size`（第 3.1 节） |
| `libraries[]` | 依赖库（第 3.2 节） |
| `logging.client` | log4j 配置（第 3.4 节） |
| `javaVersion` | 所需 Java 大版本与运行时组件名（第 3.5 节） |
| `arguments` 或 `minecraftArguments` | 启动参数（启动文档第 2 节） |
| `minimumLauncherVersion` | 官方启动器兼容下限，第三方可忽略 |

### 2.2 参数形态判定

| 判定依据 | 版本范围 |
|---|---|
| 存在 `arguments`，含 `game` 与 `jvm` 两个数组 | 1.13 及以后 |
| 存在 `minecraftArguments`，为单个字符串 | 1.12.2 及以前 |

### 2.3 版本差异

| 版本 | `javaVersion` | `libraries[].natives` | `downloads.client_mappings` |
|---|---|---|---|
| 1.12.2 | `jre-legacy` / 8 | 有 | 有 |
| 1.16.5 | `jre-legacy` / 8 | 有 | 有 |
| 1.20.1 | `java-runtime-gamma` / 17 | 无 | 有 |
| 1.21.4 – 1.21.8 | `java-runtime-delta` / 21 | 无 | 有 |
| 26.3 | `java-runtime-epsilon` / 25 | 无 | 无（26.1 起客户端不再混淆） |

`downloads.client_mappings` 仅供开发使用，启动不需要下载。

---

## 3. 生成下载任务

每条下载任务包含四项：`url`、`sha1`、`size`、本地目标路径。

### 3.1 客户端 jar

| 来源 | 本地路径 |
|---|---|
| `downloads.client` 的 `url` / `sha1` / `size` | `versions/<id>/<id>.jar` |

文件名必须为 `<id>.jar`，不能保留 `client.jar`。

### 3.2 依赖库

`libraries[]` 的条目有三种形态，按是否存在 `downloads` 区分：

| 形态 | 判定 | 处理 |
|---|---|---|
| A．现代 | 有 `downloads.artifact` | 用该对象的 `url` / `sha1`，落点 `libraries/<artifact.path>` |
| B．含平台 natives（1.18 及以前） | 除 `downloads` 外还有 `natives` 对象 | `artifact` 同 A；另按当前平台从 `natives` 取 classifier 名，再取 `downloads.classifiers.<classifier>`，落点同为 `libraries/<path>` |
| C．简化 | 无 `downloads`，仅有 `name` 与可选 `url` | 按 maven 坐标拼路径与地址，见下 |

形态 C 的路径推导：`name` 为 `group:artifact:version[:classifier]`，对应
`group/artifact/version/artifact-version[-classifier].jar`；下载地址为 `url`（缺省 `https://libraries.minecraft.net/`）拼接该路径。

所有条目先按 §3.2.1 过滤，未通过的不下载、也不进入 classpath。

#### 3.2.1 规则过滤

`rules[]` 为顺序覆盖，非布尔与：

1. `rules` 为空或不存时，条目通过。
2. 否则初始为不通过，逐条检查：命中的规则用其 `action` 覆盖当前结果（`allow` 通过、`disallow` 不通过）。
3. 未命中的规则跳过。

单条规则命中，需同时满足其出现的全部条件：

| 条件 | 判据 |
|---|---|
| `os.name` | `windows` / `osx` / `linux` |
| `os.arch` | `x86` / `x86_64` / `arm64` |
| `os.version` | 正则，作用于 `os.version` 系统属性 |
| `os.versionRange` | `min` / `max` 数值比较（26.x 起出现） |
| `features.*` | 由启动器提供的布尔值，仅用于启动参数，不用于库 |

### 3.3 资源

下载 `assetIndex.url`，保存为 `assets/indexes/<assetIndex.id>.json`。

索引内容为 `路径 → 对象`，对象含 `hash` 与 `size` 两项。对每个对象：

| 项 | 值 |
|---|---|
| 下载地址 | `https://resources.download.minecraft.net/<hash 前 2 位>/<hash>` |
| 本地路径 | `assets/objects/<hash 前 2 位>/<hash>` |
| 校验 | `sha1` 等于 `hash` |

`assetIndex.totalSize` 为全部资源的总字节数，可用于进度计算。

旧版兼容：索引中若出现下列标志，需在下载后额外铺开一份拷贝（或硬链接）：

| 标志 | 适用版本 | 铺开位置 |
|---|---|---|
| `virtual: true` | 1.6.x（索引名 `legacy`） | `assets/virtual/legacy/<对象路径>` |
| `map_to_resources: true` | ≤ 1.5.2（索引名 `pre-1.6`） | `assets/<对象路径>` |

### 3.4 日志配置

`logging.client` 含三项：

| 字段 | 说明 |
|---|---|
| `argument` | JVM 参数模板，值为 `-Dlog4j.configurationFile=${path}` |
| `file` | 配置文件的 `url` / `sha1` / `size` |
| `type` | 固定为 `log4j2-xml` |

下载 `file`，保存到任意固定位置（建议 `log_configs/client.xml`）；启动时把该文件的绝对路径填入
`argument` 的 `${path}`，作为 JVM 参数传入。

`file.id` 随版本变化：`client-1.7.xml`（≤17w14a）、`client-1.12.xml`（≤24w36a）、`client-1.21.2.xml`（24w37a 起）。

### 3.5 Java 运行时

请求 `https://piston-meta.mojang.com/v1/products/java-runtime/2ec0cc96c44e5a76b9c8b7c39df7210883d12871/all.json`。

URL 中的 UUID 是固定常量（非法值返回 404），Mojang 未公开其文档，直接照抄；
若该常量失效，改用本机已安装的 JDK 即可跳过本节。

#### 3.5.1 清单结构

顶层为 `平台 → 组件 → 数组`，取数组首项：

| 字段 | 说明 |
|---|---|
| `manifest.url` / `manifest.sha1` | 该组件的文件清单（§3.5.2） |
| `version.name` | 运行时版本号，如 `25.0.1` |
| `availability.progress` | 发布进度，100 表示可用 |

平台键：`linux`、`linux-i386`、`mac-os`、`mac-os-arm64`、`windows-x64`、`windows-x86`、`windows-arm64`、`gamecore`。
组件名取自版本 JSON 的 `javaVersion.component`；`javaVersion.majorVersion` 用于校验所选 Java 的大版本。

#### 3.5.2 文件清单

`manifest.url` 指向的文件只有 `files` 一项，键为相对路径，值为：

| 字段 | 说明 |
|---|---|
| `type` | `directory`（建目录）/ `file`（下载）/ `link`（Unix 符号链接） |
| `downloads.raw` | 原始文件的 `url` / `sha1` / `size` |
| `downloads.lzma` | xz 压缩变体的 `url` / `sha1` / `size` |
| `executable` | 为 `true` 时在 Unix 上需赋予可执行权限 |
| `target` | `link` 类型的目标路径 |

处理：`file` 落点为 `runtime/<component>/<平台>/<路径>`；下载 `lzma` 变体时须解压，
**用 `raw.sha1` 校验解压后的文件**。

---

## 4. 下载与校验

### 4.1 就绪判定

文件存在、字节数等于 `size`、SHA-1 等于 `sha1`，三者同时满足才视为就绪。
仅比较大小不足以免于被截断或占位文件。

### 4.2 写入方式

1. 目标目录不存在则创建。
2. 下载到同目录临时文件。
3. 校验临时文件，失败则删除并计入重试。
4. 原子改名为目标文件名。

### 4.3 并发与重试

| 项 | 建议值 |
|---|---|
| 并发数 | 小文件（资源对象）8～16；大文件（jar、运行时）2～4 |
| 超时 | 连接 15 s，整体 300 s |
| 重试 | `429` / `5xx` / 超时重试 3 次，指数退避；`403` / `404` 不重试 |
| 断点续传 | 请求 `Range: bytes=<已下载>-`，仅当响应为 `206` 时续写 |

### 4.4 镜像（可选）

镜像仅替换 URL 前缀，哈希始终取自官方 JSON；任一项失败应回落官方源。

| 官方前缀 | 镜像前缀 |
|---|---|
| `https://piston-meta.mojang.com/` | `https://bmclapi2.bangbang93.com/` |
| `https://piston-data.mojang.com/` | `https://bmclapi2.bangbang93.com/` |
| `https://libraries.minecraft.net/` | `https://bmclapi2.bangbang93.com/maven/` |
| `https://resources.download.minecraft.net/` | `https://bmclapi2.bangbang93.com/assets/` |

---

## 5. 模组加载器（可选）

加载器元数据独立于原版，需额外获取。

### 5.1 Fabric

| 用途 | 地址 |
|---|---|
| 可用 loader 列表 | `https://meta.fabricmc.net/v2/versions/loader/<mc>` |
| 完整版本 JSON | `https://meta.fabricmc.net/v2/versions/loader/<mc>/<loader>/profile/json` |
| 安装器版本（可选） | `https://meta.fabricmc.net/v2/versions/installer` |

`profile/json` 返回的即为可直接使用的版本 JSON，`inheritsFrom` 指向原版。处理方式：

| 项 | 处理 |
|---|---|
| 存放 | 保存为 `versions/<id>/<id>.json`，不下载 jar |
| `libraries[]` | 全部为形态 C（§3.2），`url` 为 `https://maven.fabricmc.net/` |
| `arguments` | 为字符串而非数组，按空格切分为参数列表 |
| `sha1` | `fabric-loader` 与 `intermediary` 两条缺失，需要校验时另取同目录 `.sha1` 文件 |
| `intermediary` | 26.x 的 profile 不含此项（1.21.1 及以前含） |

### 5.2 Forge / NeoForge

安装器 jar 即 zip 包，需取出下列文件：

| 文件 | 用途 |
|---|---|
| `install_profile.json` | 安装期依赖库与处理器链 |
| `version.json` | 运行期版本 JSON（含 `inheritsFrom`） |
| `data/client.lzma` | 二进制补丁（`install_profile.data.BINPATCH` 指向） |
| `data/*_args.txt` | 官方启动脚本参数（可选） |

`install_profile.json` 的主要字段：

| 字段 | 说明 |
|---|---|
| `spec` | 格式版本，当前为 1 |
| `version` / `minecraft` | 生成目录名 / 对应原版版本 |
| `libraries[]` | 安装期工具链，格式同原版库 |
| `data{}` | 处理器输入 |
| `processors[]` | 处理器链：`jar` / `classpath` / `args` / `sides` / `outputs` |
| `serverJarPath` | 服务端 jar 模板路径，客户端忽略 |

`data` 的值有三种写法：

| 写法 | 含义 |
|---|---|
| `[group:artifact:version:classifier@ext]` | maven 坐标，解析为 `libraries/<path>` 下的文件 |
| `/data/client.lzma` | 安装器 zip 内的相对路径，解到临时文件 |
| `'literal'` | 字面量字符串 |

处理器的执行方式与参数替换规则见启动文档 §7.2；产物（`PATCHED` 等）必须写入 `libraries/`，
否则启动时 classpath 缺项。

---

## 6. 检查清单

- [ ] `versions/<id>/<id>.json` 与 `<id>.jar` 同时存在
- [ ] 全部 `libraries[].downloads.artifact` 对应文件存在且 SHA-1 正确
- [ ] `assets/indexes/<indexId>.json` 存在，索引内每个对象均在 `assets/objects/` 命中
- [ ] 索引含 `virtual` / `map_to_resources` 时，相应目录已铺开
- [ ] 日志配置已落盘
- [ ] `runtime/<component>/<平台>/bin/java[.exe]` 存在且可执行（或已选定本机 JDK）
- [ ] 使用模组加载器时，其版本 JSON 与处理器产物均已就位

---

## 7. 注意事项

| 现象 | 说明 |
|---|---|
| 离线模式（`--accessToken` 为 `0`）下，日志出现 `401`、Realms 认证失败 | 离线令牌本就不代表正版会话，Realms 及需要正版鉴权的接口必然拒绝。这是预期现象，不影响单人游戏，**不可据此判定安装失败**。 |

安装是否成功，只以第 6 节检查清单为准，不以日志中是否存在上述报错为准。

---

## 附录：接口一览

| 用途 | 地址 |
|---|---|
| 版本清单 | `https://piston-meta.mojang.com/mc/game/version_manifest_v2.json` |
| 版本 JSON | `versions[].url` |
| 资源索引 | `assetIndex.url` |
| 资源对象 | `https://resources.download.minecraft.net/<hash前2位>/<hash>` |
| 依赖库 | `https://libraries.minecraft.net/<path>` |
| Java 运行时清单 | `https://piston-meta.mojang.com/v1/products/java-runtime/2ec0cc96c44e5a76b9c8b7c39df7210883d12871/all.json` |
| Fabric 版本 JSON | `https://meta.fabricmc.net/v2/versions/loader/<mc>/<loader>/profile/json` |
| Fabric maven | `https://maven.fabricmc.net/` |
| Forge maven | `https://maven.minecraftforge.net/` |
| NeoForge maven | `https://maven.neoforged.net/releases/` |
