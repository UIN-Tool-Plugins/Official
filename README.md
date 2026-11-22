# Official

UIN Tool 官方插件仓库。推送 `main` 后由 GitHub Actions 构建、签名，并把产物发布到 `dist` 分支。

## 插件列表

| 插件 | type | uiType | 后端 | 说明 |
|------|------|--------|------|------|
| `uin.official.hello` | 空 | web | — | 纯 Web 插件示例 |
| `uin.official.calculator` | 空 | web | Python | Web + Python 后端示例 |
| `uin.official.filemanager` | native | native | — | Native 插件（Java → DEX） |
| `uin.official.terminal` | 空 | cui | — | CUI 终端插件（shell 脚本） |
| `uin.official.sysinfo` | other | cui | C | C 语言后端 + CUI 插件 |

> 所有示例插件默认显示（`visible` 内容为 `1`），会构建并发布到 `dist`，同时出现在插件源列表中。把 `visible` 改成 `0` 即可隐藏：仍然构建发布，但不会写进 `source.json`。

## 添加到 UIN Tool

软件 → 插件源管理 → 添加 → 输入：

```
https://github.com/UIN-Tool-Plugins/Official
```

## 仓库结构

```
.
├── .github/workflows/build.yml    CI：构建 → 签名 → 发布 dist
├── assets/
│   ├── host-sdk.jar               原生插件编译用的宿主公开 API
│   └── host-sdk.sha256            固定版本，CI 强校验
├── scripts/
│   ├── build-plugin.sh            单个插件完整构建
│   ├── build-all.sh               本地批量构建
│   ├── changed-plugins.sh         CI 计算本次要构建的插件
│   ├── check-host-sdk.sh          校验 host-sdk.jar
│   ├── deploy-dist.sh             同步 dist、生成索引、推送
│   ├── generate-index.sh          生成 source.json 的入口
│   ├── lib.sh                     javac / d8 / 路径查找
│   ├── sign-tpk.sh                JAR v1 签名
│   ├── tpk.py                     打包 / 验签 / 索引 / 版本
│   └── update-host-sdk.sh         从 UIN Tool 工程同步 SDK
├── plugins/
│   └── <pluginId>/
│       ├── type                   构建方式（空 / native / other）
│       ├── visible                1 = 在源列表显示，0 = 隐藏
│       ├── plugin.json            插件元数据
│       ├── icon.png               PNG 图标
│       └── ...                    web/、scripts/、src/ 等
└── source.json                    源元数据（sourceId/name/description）
```

## 构建流程

每次推送 `main`：

1. `changed-plugins.sh` 根据 diff 算出要构建的插件；`scripts/`、`assets/` 或 workflow 有改动时全量构建。
2. 克隆 `dist` 分支，读取每个插件上次发布的 `.version`。
3. `build-plugin.sh`：
   - 把插件复制到 `build/work/<id>/`，**不修改源码目录**
   - 校验 `pluginId`、目录名、`version`、`uiType`、入口、图标
   - 计算生效版本（见下文）
   - `type=native` 用 `javac --release 11` + `d8 --min-api 23` 生成 `plugin.dex`
   - `type=other` 在暂存目录执行 `build.sh`
   - 打包成 TPK（固定顺序、固定时间戳、无目录条目）
   - 用 `PLUGIN_KEYSTORE*` 写 JAR v1 签名
   - 再次校验包结构、签名、证书指纹，写 `build/meta/<id>.json`
4. `deploy-dist.sh` 把 TPK、生效版 `plugin.json`、图标、`visible`、`.version`、`.sha256`、`.signer` 同步到 `dist/plugins/<id>/`，删除已移除的插件。
5. `tpk.py index` 生成 `dist/source.json`，提交并推送。推送冲突会自动 rebase 重试 3 次。

同一个仓库的发布有 `concurrency` 锁，两个 job 不会同时改 `dist`。在 Actions 页面手动 Run workflow 会全量构建（没有可比较的 before 提交）。

## 版本规则

`plugin.json` 的 `version`：

| 源仓库 version | 行为 |
|----------------|------|
| `0` | 每次构建在 dist 上次版本上 +1，通常保持 0 即可 |
| `>=1` | 使用固定版本；必须大于 dist 上次版本，否则构建失败 |

生效版本会写进 TPK 内的 `plugin.json` 和 `dist/source.json`。源仓库里的 `plugin.json` 不会被改写。

需要重发同一个固定版本时，本地执行：

```bash
bash scripts/build-plugin.sh plugins/<id> build/<id>.tpk <上次版本> --force
```

CI 不提供 `--force`，避免误操作。

## 签名

**官方仓库的 CI 必须配置签名密钥，否则构建直接失败**（`REQUIRE_SIGNING=1`）。这里用的证书必须和宿主 App 的 release 证书是**同一张**，否则客户端不会把插件识别为官方。

在仓库 Settings → Secrets and variables → Actions 添加：

| Secret | 说明 |
|--------|------|
| `PLUGIN_KEYSTORE_BASE64` | keystore 整份文件的 base64，`base64 -w0 your.jks` |
| `PLUGIN_STORE_PASSWORD` | keystore 口令 |
| `PLUGIN_KEY_PASSWORD` | 密钥口令；PKCS12 通常与库口令相同，可留空 |
| `PLUGIN_KEY_ALIAS` | 密钥别名 |
| `PLUGIN_CERT_SHA256` | 期望的证书 SHA-256；填了会强校验，防止签错 key |

相关约定：

- TPK 使用 **JAR v1 签名**（`META-INF/MANIFEST.MF` + `.SF` + `.RSA/.EC`），不是 APK v2/v3。
- 客户端只按**证书指纹**查信任表，不读 TPK 自报字段。官方证书内置在宿主里。
- 已签名的插件不能被无签名包更新；换签名会强警告后允许。
- 换签名密钥后，老 App 里的官方插件会变成「未知签名」，需要重新签名并发布。
- **不要把 keystore、base64、口令提交到仓库或贴进 issue。** `.gitignore` 已忽略常见密钥文件。

## 本地构建

需要 JDK 17、Android SDK platform + build-tools、Python 3、bash。原生插件需要 `d8`；C 示例需要 aarch64 静态交叉编译器（Ubuntu/Debian 安装 `gcc-aarch64-linux-gnu`，在 aarch64 Linux 上可直接用本机 gcc）。

```bash
# 全部插件（没有 dist/ 时版本从 1 开始）
bash scripts/build-all.sh

# 指定插件
bash scripts/build-all.sh uin.official.hello uin.official.terminal

# 带签名
export PLUGIN_KEYSTORE=/secure/uin-tool.p12
export PLUGIN_STORE_PASSWORD='...'
export PLUGIN_KEY_ALIAS='uin-tool'
bash scripts/build-all.sh uin.official.hello

# 本地预览 dist（不提交）
DIST_PUSH=0 bash scripts/deploy-dist.sh /tmp/dist-preview build source.json
```

产物在 `build/`，元数据在 `build/meta/`。`KEEP_WORK=1` 可保留 `build/work/<id>/` 便于排查。

## 插件目录约定

### type

| `type` 内容 | 构建方式 | 需要的文件 |
|-------------|----------|-----------|
| 空文件 | 直接打包 | `plugin.json`、图标、入口 |
| `native` | Java → DEX | `src/**/*.java`，或预编译 `plugin.dex` |
| `other` | 执行 `build.sh` | `build.sh` |

`type=native` 时 `plugin.json` 的 `uiType` 必须是 `native`。

### visible

| 内容 | 效果 |
|------|------|
| 无文件 / `1` / `true` | 显示在插件源列表 |
| `0` / `false` | 构建发布，但不写入 `source.json` |

### build.sh（type=other）

约定：

```bash
build.sh <暂存目录> <输出.tpk>
```

- 构建产物写到**暂存目录**内，例如 `"$1/sysinfo_server"`。
- 不要自己打包 TPK；统一由 `build-plugin.sh` 打包、签名。
- 为了兼容旧脚本，如果仍然生成了 `$2`，会直接使用它并做同样的校验。
- 打包时会自动排除 `type`、`visible`、`build.sh`、`src/`、`libs/`、`data/`、`*.tpk`、`*.class` 等构建输入。

C 示例的后端会运行在插件实际使用的环境里：内置 proot 是 Debian (glibc)，实体 Termux 是 Android (bionic)。同时兼容两者的形态是**静态链接的 Linux aarch64 可执行文件**，所以 `build.sh` 优先使用 `aarch64-linux-gnu-gcc -static`（在 aarch64 Linux 上直接用本机 `gcc -static`），并用 `file` 检查产物必须是 `ARM aarch64` 且 `statically linked`。Android NDK 产出的是 bionic 动态可执行文件，在 Debian 容器里缺少 `/system/bin/linker64`，不能用作后端。

共享存储通常以 `noexec` 挂载，宿主解压 TPK 也不会保留可执行位。C 示例的 `scripts/start.sh` 会先把二进制复制到可执行目录再 `exec`：依次尝试 `$TMPDIR`、`/tmp`、`$HOME`（都位于 `/data`，允许执行），适配内置 proot 与实体 Termux。`backendStartCommand` 写 `sh scripts/start.sh`。

### .tpkignore

可选，每行一个 glob，匹配相对路径（`#` 开头为注释）。用来额外排除不希望进包的文件。不能反向包含被默认规则排除的目录。

### 图标

- 必须是 PNG，不能是 0 字节。
- 建议 256×256，不超过 500 KiB。
- `plugin.json.icon` 默认 `icon.png`；自定义路径也会同步到 `dist/plugins/<id>/icon.png`。

### native 插件

- 源码放在 `src/`，包名与 `mainClass` 对应。
- 编译目标 Java 11、`min-api 23`，与宿主插件 API 一致。
- 只接受单个 `plugin.dex`；如果 D8 产出 `classes2.dex`，构建会失败，需要拆分插件。
- 没有源码时可以直接提交预编译 `plugin.dex`，构建会校验 DEX 头。

## 更新 host-sdk

宿主插件 API 变更后，在 UIN Tool 工程重新生成 `host-sdk.jar`，然后同步：

```bash
bash scripts/update-host-sdk.sh /path/to/UIN_Tool
```

它会更新 `assets/host-sdk.jar` 和 `assets/host-sdk.sha256`。CI 会校验两者一致，并检查 `PluginInterface` / `PluginContext` 是否存在。

## dist 分支

```
dist/
├── source.json
└── plugins/<pluginId>/
    ├── <pluginId>.tpk
    ├── plugin.json        生效版本的元数据
    ├── icon.png
    ├── visible            显示标记（可选，0 = 隐藏）
    ├── .version
    ├── .sha256
    ├── .signer            签名证书 SHA-256
    └── .built             构建时间
```

`source.json` 中每个插件包含宿主需要的字段：

`pluginId`、`name`、`version`、`versionName`、`author`、`description`、`category`、`uiType`、`size`、`tpkPath`、`iconPath`、`iconSha256`、`lastUpdate`、`updateLog`、`repositoryUrl`，另外附带 `sha256`、`signerSha256`、`signed`、`minHostVersion` 供镜像和人工校验。`iconSha256` 是 `icon.png` 的内容摘要，客户端下载图标后按它校验，不一致就不显示也不写缓存。

## 常见问题

**构建提示未签名**

官方仓库 CI 必须在 Secrets 里配置 `PLUGIN_KEYSTORE_BASE64` 等，缺少会直接失败。本地未配置密钥时允许产出未签名包，但不要发布。

**版本没有变化**

源 `plugin.json` 的 `version=0` 时依赖 dist 里的 `.version`。如果 dist 分支被清空或换了仓库，版本会从 1 重新开始。固定版本必须大于已发布版本。

**native 插件构建失败**

检查 `ANDROID_HOME` / `ANDROID_JAR` / `d8` 是否可用，`src` 是否存在 Java 源文件，`mainClass` 是否与包名一致。

**C 示例产物的架构不对**

CI 会自动安装 `gcc-aarch64-linux-gnu`。本地需要 aarch64 交叉编译器，或直接在 aarch64 Linux 上构建。产物必须是静态链接，否则在 proot 容器里找不到动态解释器。

**删除插件后 dist 还有旧包**

部署阶段会对比 `plugins/` 目录并删除 dist 里多余的插件；如果刚删除，等这次 workflow 跑完。

## 开发自己的插件

参考 [Template](https://github.com/UIN-Tool-Plugins/Template) 仓库。Template 的构建脚本与本仓库一致，签名是可选的。
