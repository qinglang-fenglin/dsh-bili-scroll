# 本地改动说明（PATCH-NOTES）

本目录不是上游原版，而是 **dsh-bili-scroll 2.4.0 的本地修改版**。
上游仓库：`https://github.com/c1overzz/dsh-bili-scroll`（MIT）。

## 位置

- 工作副本 / 唯一源码：`d:\TraeWork\dsh-bili-scroll`
- 部署位置：`~/.dsh/profiles/desktop/node_modules/dsh-bili-scroll`
- 原版备份：`d:\TraeWork\dsh-bili-scroll-2.4.0-original.zip`
- 用户配置：`~/.dsh/bili-duty/config.json`（不跟着插件目录走）
- 运行日志：`~/.dsh/bili-duty/duty.log`
- 每轮动态 overlay：`~/.dsh/bili-duty/headless-overlay.yml`

## 改了什么（按功能找，不要按行号找）

行号会随上游升级变化，改动位置以函数名 / 现有注释标记为准。

### 1. 桌面端 dsh 检测
- 位置：`index.js` 的 `desktopBootstrap` / `findDshCli` / `resolveDsh`
- 目的：桌面版 DSH 优先使用自带的 `app.asar/.../dsh-desktop-host/lib/cli.js`，并加 `--expose-internals`。
- 原因：`~/.dsh/profiles/node_modules/@deepseek-ai/dsh` 是指向 npm 全局旧版本（0.1.5-rc.2）的 Junction，原逻辑会误用它，导致值班轮用错版本启动失败。
- 上游升级时：保留“桌面端优先于 createRequire”的顺序；如果桌面端目录结构变了，只需改 `desktopBootstrap` 里的相对路径。

### 2. 自定义模型 provider
- 位置：`index.js`
  - `Config` 里的 `modelProviders`
  - `DEFAULTS`、`normalize`、`normalizeProviders`、`clampPatch`
  - `customProviders`、`providerUsable`、`usableProviders`
  - `ensureOverlay` 里的 `llm-pi-ai` 段
  - `listModelCatalog` 的自定义 provider合并 + 缓存签名
- 目的：让值班轮能使用 DeepSeek 之外的 OpenAI 兼容网关（当前主要用商汤 TokenPlan）。
- 原理：把 `modelProviders` 写进每轮的一次性 overlay 的 `llm-pi-ai.providers`，同时让 provider 白名单和设置页列表接受这些 id。
- 局限：只对 **pi-ai 配置型 provider** 有效；由其他 bundle 注册的 adapter 不能靠 overlay 补出来（上游 issue #175 说的就是这一类）。
- 上游升级时：如果上游重构了 `providerUsable` 或 overlay 生成，按“配置式 provider 要能被白名单接受 + 要写进 overlay”这两个目标重新落一遍即可。

### 3. 设置页
- 位置：`client.js` 的模型下拉区域
- 改动：去掉前端硬编码的 `USABLE_PROVIDERS`，直接信任 `/models` 的筛选结果；新增「自定义模型提供方」JSON 文本框。
- 上游升级时：保留“前端不再写死 provider 名单”和“自定义 provider 可编辑”这两个行为。

### 4. 配置保存兼容
- `saveCfgHost` 的回读比较对 `modelProviders` 放宽：宿主会给嵌套对象补默认值，严格 JSON 比较会误报“没写进去”。
- `/config` 的空值保护里，`modelProviders` 和 `modelProvider` / `modelName` 一样视为“空是合法值”。

### 5. 值班章程
- 位置：`prompts/duty.md`（运行时的 `~/.dsh/bili-duty/duty.md` 也同步改了）
- 改动：删掉「先读 AGENTS.md / memories.md」那一段，改成“这一轮没有文件系统，人设由 `personaPrompt` 提供”。
- 原因：值班轮是最小权限 profile，没有文件工具；原段落会让模型先去找文件、浪费一轮，甚至直接收尾。
- 上游升级时：如果上游改回这段，按同样思路替换；不要让人设依赖 AGENTS.md。

## 升级上游时的思路

1. 不要直接覆盖现有部署目录；先把新版解压到临时目录。
2. 用原版备份（2.4.0）和新版做 diff，先看上游动了哪些函数。
3. 把本文件的 4 类改动重新落到新版上，优先按函数 / 注释标记定位，不要按行号。
4. 保持以下行为不变：
   - 桌面端 bootstrap 优先
   - `modelProviders` 能写进 overlay
   - provider 白名单接受配置里的自定义 id
   - 设置页不硬编码 provider
5. 验证顺序：
   - `node --check index.js` / `node --check client.js`
   - 在插件目录跑 `node selfcheck.mjs`
   - 用 `Config` 解析一份带 `modelProviders` 的样例
   - 用桌面端 `dsh --profile bili-duty --patch <overlay> --dump-config` 确认 `llm-pi-ai` 配置落在最终树里
   - 实跑一轮，看 `duty.log` 是否还有 `NO_ADAPTER` / `没有适配器` / 回退行
6. 验证通过后再整体覆盖到 `node_modules/dsh-bili-scroll`，然后重启 DSH。

## 注意

- `~/.dsh/profiles/bili-duty/cordis.patch.yml` 每次插件启动都会被模板覆盖；要改 profile 必须改 `profile-template/bili-duty/cordis.patch.yml`。
- 不要直接以 `node_modules` 里的文件为源码；改 `d:\TraeWork\dsh-bili-scroll`，再复制过去。
- 不要动 `vendor/` 和 `lib/asr.py`、`lib/favops.py` 的哈希，否则 `selfcheck` / 安装流程会认为文件被篡改。
- 商汤 key 来自 `~/.dsh/.credentials.yaml` 里的 `SENSENOVA_API_KEY`，overlay 只引用名字，不写明文。
- 当前用户配置把值班轮固定为 `sensenova / deepseek-flash`；设置页下拉可切换 `sensenova-6.8-flash` 等模型。

## 回滚

把 `dsh-bili-scroll-2.4.0-original.zip` 解压覆盖回 `node_modules/dsh-bili-scroll`，重启 DSH。
