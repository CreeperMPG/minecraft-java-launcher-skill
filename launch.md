# Minecraft Java 版启动实现文档

前置条件：已按《Minecraft Java 版下载实现文档》准备好 `versions/`、`libraries/`、`assets/`、`runtime/`，
并已取得账号令牌（第 5、6 节）。

接口与参数基于 2026-10 的正式版 `26.3`，历史版本差异在对应位置标注。

---

## 1. 命令行结构

启动即执行一条命令：`<java> [JVM 参数] -cp <classpath> <mainClass> [游戏参数]`。

- 主类之前的参数一律交给 JVM，之后的交给游戏。
- 同名 JVM 参数重复时后者生效，可用启动器自身的 `-Xmx` 覆盖默认值。
- 参数以参数数组传递，不拼接字符串，不手动添加引号。

---

## 2. 参数求值

### 2.1 参数形态

| 版本 | 形式 |
|---|---|
| 1.13 及以后 | `arguments.jvm` 与 `arguments.game`，均为数组 |
| ≤ 1.12.2 | `minecraftArguments`，单个字符串，按空格切分；JVM 参数需启动器自备 |
| Fabric 的 profile JSON | `arguments.jvm` / `arguments.game` 为字符串，按空格切分 |

数组元素有两种：纯字符串，或带 `rules` 与 `value` 的条件对象。条件对象按 §3.2.1（下载文档）求值，
不通过则整条跳过；`value` 可能为字符串或数组，一律展开为若干参数。

### 2.2 features

由启动器提供、仅用于游戏参数求值的布尔集合：

| feature | 对应的参数 |
|---|---|
| `is_demo_user` | `--demo` |
| `has_custom_resolution` | `--width` / `--height` |
| `has_quick_plays_support` | `--quickPlayPath` |
| `is_quick_play_singleplayer` | `--quickPlaySingleplayer` |
| `is_quick_play_multiplayer` | `--quickPlayMultiplayer` |
| `is_quick_play_realms` | `--quickPlayRealms` |

未由启动器置真的 feature 视为 `false`，对应参数不加入命令行。

### 2.3 占位符

参数中出现的 `${...}` 按下表替换；未知占位符保留原样并记录日志。

| 占位符 | 取值 |
|---|---|
| `${auth_player_name}` | 玩家名 |
| `${version_name}` | 版本 id（实例名） |
| `${game_directory}` | 实例目录绝对路径 |
| `${assets_root}` | `assets/` 绝对路径 |
| `${assets_index_name}` | 版本 JSON 的 `assets` 字段 |
| `${auth_uuid}` | 32 位无连字符 UUID |
| `${auth_access_token}` | 访问令牌 |
| `${clientid}` | OAuth 应用 client_id |
| `${auth_xuid}` | Xbox 用户 id |
| `${user_type}` | `msa` / `legacy`（1.21.8 及以前需要） |
| `${version_type}` | `release` / `snapshot` / 自定义 |
| `${resolution_width}` / `${resolution_height}` | 窗口尺寸 |
| `${natives_directory}` | natives 目录绝对路径 |
| `${launcher_name}` / `${launcher_version}` | 启动器标识 |
| `${classpath}` | 拼好的 classpath |
| `${library_directory}` | `libraries/` 绝对路径（模组加载器使用） |
| `${classpath_separator}` | `;`（Windows）/ `:`（其他） |
| `${quickPlayPath}` | 快速进入日志文件路径 |
| `${game_assets}` | `assets/virtual/legacy`（1.6 及以前） |

### 2.4 默认 JVM 参数（可选）

26.x 的 `arguments` 含 `default-user-jvm`，给出官方推荐的堆与 GC 参数（Windows 10.0.17134 起为 ZGC），
求值方式与 `arguments.jvm` 相同，可作为启动器"JVM 参数"输入框的默认值。

---

## 3. classpath 与 natives

### 3.1 classpath

由规则通过的 `libraries[].artifact` 与客户端 jar 依次拼接，分隔符 Windows 为 `;`、其他平台为 `:`。

| 要求 | 说明 |
|---|---|
| 绝对路径 | 工作目录为实例目录，相对路径会解析错误 |
| 顺序 | 有意义：同名类以先出现者为准，模组依赖冲突由此处理 |
| natives jar | 1.18 及以前的 natives jar 不进入 classpath |
| `inheritsFrom` | 版本 JSON 含该字段时，由合并后的库列表构造（§7.1） |

客户端 jar 的选取：

| 情况 | jar 路径 |
|---|---|
| 普通版本 | `versions/<id>/<id>.jar` |
| 存在 `jar` 字段 | `versions/<jar>/<jar>.jar` |
| 有 `inheritsFrom` 且无 `jar` | 不加入原版 jar，类由加载器提供的库给出 |

### 3.2 natives

| 版本 | JVM 参数 | 需要的操作 |
|---|---|---|
| 1.18 及以前（JSON 含 `natives`） | `-Djava.library.path=${natives_directory}` | 将各 natives jar 解压到该目录，跳过 `META-INF/` 与 `extract.exclude` 中的路径 |
| 1.19 及以后（无 `natives`） | `-Djava.library.path`、`-Djna.tmpdir`、`-Dorg.lwjgl.system.SharedLibraryExtractPath`、`-Dio.netty.native.workdir` | 仅创建目录，解压由 LWJGL / JNA / Netty 在运行时完成 |

26.x 的四个路径均带子目录：`${natives_directory}/java`、`/jna`、`/lwjgl`、`/netty`。

---

## 4. 启动步骤

1. 定位版本 JSON：`versions/<id>/<id>.json`；含 `inheritsFrom` 时按 §7.1 合并。
2. 求值 `libraries[].rules`，构造 classpath（§3.1）。
3. 按 §3.2 准备 natives 目录。
4. 确定 Java 可执行文件：取 `javaVersion.component` 对应的运行时
   （`runtime/<component>/<平台>/bin/java.exe`）或用户指定的 JDK；
   所选 Java 的大版本须与 `javaVersion.majorVersion` 一致。
5. 求值 JVM 参数：启动器自身参数与 `arguments.jvm`（其中 `-cp` 取第 2 步的 classpath）。
6. 取得账号信息（第 5 或第 6 节），填入 `--username` / `--uuid` / `--accessToken` / `--clientId` / `--xuid`。
7. 求值游戏参数：`arguments.game` 展开并替换占位符，按 features 取舍。
8. 执行 `<java> [JVM 参数] <mainClass> [游戏参数]`：`mainClass` 取合并后版本 JSON 的 `mainClass` 字段，
   工作目录设为实例目录。
9. 等待进程结束；退出码非 0 时读取 `crash-reports/` 与 `logs/latest.log`。

---

## 5. 正版登录（微软账号）

需自行注册 Azure 应用取得 `client_id`，并通过 Mojang 的第三方应用审批。

### 5.1 设备代码流

| 步骤 | 请求 |
|---|---|
| 申请设备码 | `POST https://login.microsoftonline.com/consumers/oauth2/v2.0/devicecode`；表单 `client_id`、`scope=XboxLive.signin offline_access` |
| 轮询授权 | `POST https://login.microsoftonline.com/consumers/oauth2/v2.0/token`；表单 `grant_type=urn:ietf:params:oauth:grant-type:device_code`、`client_id`、`device_code` |

申请设备码返回 `device_code`、`user_code`、`verification_uri`、`expires_in`、`interval`。
`user_code` 与 `verification_uri` 展示给用户，其余用于轮询。

轮询以 `interval` 秒为间隔：返回 `authorization_pending` 或 `slow_down` 时继续，其余错误终止。
成功时返回 `access_token` 与 `refresh_token`。

令牌续期使用同一 token 接口，表单为 `grant_type=refresh_token`、`client_id`、`refresh_token`、
`scope=XboxLive.signin offline_access`；Web 应用另需 `client_secret`。

### 5.2 换取 Minecraft 令牌

| 步骤 | 请求 | 取用字段 |
|---|---|---|
| Xbox Live 认证 | `POST https://user.auth.xboxlive.com/user/authenticate`；body 为 `Properties.AuthMethod=RPS`、`Properties.SiteName=user.auth.xboxlive.com`、`Properties.RpsTicket=d=<微软 access_token>`、`RelyingParty=http://auth.xboxlive.com`、`TokenType=JWT` | `Token`、`DisplayClaims.xui[0].uhs` |
| XSTS 授权 | `POST https://xsts.auth.xboxlive.com/xsts/authorize`；body 为 `Properties.SandboxId=RETAIL`、`Properties.UserTokens=[<XBL Token>]`、`RelyingParty=rp://api.minecraftservices.com/`、`TokenType=JWT` | `Token` |
| 换取 MC 令牌 | `POST https://api.minecraftservices.com/authentication/login_with_xbox`；body 为 `identityToken`，值为 `XBL3.0 x=<uhs>;<XSTS Token>` | `access_token`（有效期 24 h） |
| 拥有检查 | `GET https://api.minecraftservices.com/entitlements/mcstore`，头 `Authorization: Bearer <MC token>` | `items[]` 为空表示未购买 |
| 玩家档案 | `GET https://api.minecraftservices.com/minecraft/profile`，同一鉴权头 | `id`（UUID）、`name`（玩家名） |

OAuth 部分为表单（`application/x-www-form-urlencoded`），Xbox 与 Minecraft 部分为 JSON。

### 5.3 参数落位

| 参数 | 来源 |
|---|---|
| `--username` | 档案的 `name` |
| `--uuid` | 档案的 `id` |
| `--accessToken` | `login_with_xbox` 返回的 `access_token` |
| `--clientId` | 自身 Azure 应用的 `client_id` |
| `--xuid` | `access_token`（JWT）中的 `xid` 声明 |
| `--userType` | 正版为 `msa` |

### 5.4 XSTS 常见错误

| XErr | 含义 |
|---|---|
| `2148916233` | 该微软账号没有 Xbox 账号 |
| `2148916235` | 所在地区不支持 Xbox Live |
| `2148916236` / `2148916237` | 需要成年人验证 |
| `2148916238` | 未成年账号需加入家庭组 |

---

## 6. 离线登录

UUID 取 `OfflinePlayer:<玩家名>` 的 MD5，第 7 字节低 4 位置为 `3`（版本 3），第 9 字节高 2 位置为 `10`
（变体 IETF），其余字节保留，最终输出 32 位小写十六进制。

| 参数 | 取值 |
|---|---|
| `--username` | 玩家名 |
| `--uuid` | 上式计算值 |
| `--accessToken` | 任意非空字符串 |
| `--userType` | `legacy` |

该算法须与客户端、服务端保持一致，否则同一玩家在不同端身份不同。

---

## 7. 模组加载器

### 7.1 `inheritsFrom` 合并

| 字段 | 合并规则 |
|---|---|
| `id` | 子版本 |
| `mainClass` | 子有则用子，否则用父 |
| `arguments.game` / `arguments.jvm` | 父在前、子在后拼接 |
| `libraries` | 父与子拼接，同名以子为准 |
| `downloads` / `assetIndex` / `assets` / `javaVersion` / `logging` | 子缺省时用父 |
| `jar` | 有则用指定版本 jar；无则不加原版 jar |

`inheritsFrom` 可链式，递归至不含该字段的版本。

### 7.2 Forge / NeoForge：处理器链

`install_profile.json` 的 `processors[]` 需按顺序逐条执行，每条等价于
`java -cp <展开后的 classpath> <jar 内主类> <展开后的 args>`。

展开规则：

| 展开对象 | 规则 |
|---|---|
| `data` 中的 `[group:artifact:version:classifier@ext]` | maven 坐标，解析为 `libraries/<path>` 下的文件路径 |
| `data` 中的 `/data/xxx.lzma` | 安装器 zip 内的相对路径，解到临时文件 |
| `data` 中的 `'literal'` | 字面量 |
| `args` 中的 `{KEY}` | 上述 `data` 的解析结果 |
| `{ROOT}` / `{INSTALLER}` / `{MINECRAFT_JAR}` / `{MINECRAFT_VERSION}` / `{LIBRARY_DIR}` / `{SIDE}` | 内置变量 |
| `outputs` 的键与值 | 产物路径及其 SHA-1，用于校验 |

`processors[].sides` 限定客户端或服务端，客户端安装只执行含 `client` 或未限定的项。
产物（`PATCHED` 等）必须写入 `libraries/`，供启动时加入 classpath。

| 加载器 / 版本 | `mainClass` | 处理器数 | 附加要求 |
|---|---|---|---|
| Forge 1.20.1 | `cpw.mods.bootstraplauncher.BootstrapLauncher` | 10 | `-p` 模块路径、`--launchTarget forgeclient` |
| NeoForge 1.21.1 | `cpw.mods.bootstraplauncher.BootstrapLauncher` | 10 | 同上 |
| NeoForge 26.3 | `net.neoforged.fml.startup.Client` | 2 | 无（客户端不再混淆） |

### 7.3 Fabric

向 `https://meta.fabricmc.net/v2/versions/loader/<mc>/<loader>/profile/json` 请求，
返回的即为可用的版本 JSON，保存为 `versions/<id>/<id>.json`。

| 项 | 处理 |
|---|---|
| `libraries[]` | 按简化形态处理（下载文档 §3.2 形态 C），`url` 为 `https://maven.fabricmc.net/` |
| `arguments` | 为字符串，按空格切分 |
| `mainClass` | `net.fabricmc.loader.impl.launch.knot.KnotClient` |
| 客户端 jar | 使用 `inheritsFrom` 指向的原版 jar |

注意事项：

- `-DFabricMcEmu= net.minecraft.client.main.Main ` 的值内含空格，须作为单个参数传递。
- `fabric-loader` 与 `intermediary` 条目缺少 `sha1`，需要校验时另取 maven 同目录的 `.sha1` 文件。
- 1.21.2 起原版自带 `org.ow2.asm`（1.21.2 / 1.21.3 为 9.3，1.21.4 起为 9.6），与 Fabric 的 9.10.1 冲突：
  须让 Fabric 的 ASM 排在原版之前，或从库列表中移除原版 asm。
- mod 置于 `<实例目录>/mods/` 下。

---

## 8. 进程与日志

### 8.1 进程

| 项 | 设置 |
|---|---|
| 可执行文件 | `<运行时>/bin/java.exe`；Windows 可用 `javaw.exe`（无控制台），需实时输出时用 `java.exe` |
| 工作目录 | 实例目录 |
| Shell | 不通过 `cmd /c` 启动 |
| 参数 | 参数数组，不拼接字符串 |
| 退出码 | `0` 为正常；非 0 见第 9 节 |

### 8.2 日志

`logging.client.argument`（`-Dlog4j.configurationFile=${path}`）中的 `${path}` 替换为已下载配置的绝对路径，
作为 JVM 参数传入。游戏自行写入 `<实例目录>/logs/latest.log` 并按天滚动，
启动器读取该文件即可显示日志，无需接管标准输出。

---

## 9. 错误对照

| 现象 | 原因 | 处理 |
|---|---|---|
| `Could not find or load main class` | classpath 缺客户端 jar，或 jar 未改名 | 检查 `versions/<id>/<id>.jar` |
| `ClassNotFoundException: org.lwjgl.*` | 库缺失或被规则误过滤 | 输出最终 classpath 与 JSON 对照 |
| `UnsatisfiedLinkError: no lwjgl in java.library.path` | 1.18 及以前未解压 natives | 检查 natives 目录内的 `.dll` / `.so` |
| `UnsupportedClassVersionError` | Java 版本低于 `javaVersion.majorVersion` | 更换对应大版本 Java |
| 界面完整但无字体、无音效 | `assets/objects` 不全，或 `--assetIndex` 错误 | 校验索引与对象哈希 |
| 1.6 及以前黑屏 | 未铺 `virtual/legacy` | 按下载文档 §3.3 处理 |
| 启动即退出且无输出 | 参数错误，或 `javaw` 丢失标准错误 | 改用 `java.exe` 复现 |
| Linux 下 `liblwjgl.so` 找不到 | natives jar 缺少执行权限 | 赋予可执行权限 |
| Forge 报缺少强制依赖 | 处理器链未执行完，`PATCHED` 库缺失 | 重跑 §7.2 |
| NeoForge 报 `libraryDirectory` 未定义 | 未实现 `${library_directory}` | 补全占位符 |
| 进服被踢 `Invalid session` | 令牌过期，或使用了离线 UUID | 重新鉴权或统一 UUID 算法 |
| Fabric 报 `Unrecognized option` 后跟原版主类名 | `-DFabricMcEmu=...` 被空格切开 | 作为单个参数传入 |
| Fabric 报 ASM 相关异常 | 原版 asm 抢先加载 | 调整 classpath 顺序或移除原版 asm |
