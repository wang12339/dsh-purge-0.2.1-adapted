# dsh-purge 0.2.1-rc.2 适配说明（Mac 本机）

- 日期：2026-10-21
- 上游版本：dsh-purge 1.1.72（package.json 的 `dshTarget` 仍写 0.2.0-rc.2）
- 宿主：DSH 0.2.1-rc.2，`~/.local/lib/node_modules/@deepseek-ai/dsh`，profile `web`
- 安装方式：link 依赖（`link:~/plugins/dsh-purge`），改源码即生效

## 本机目录布局

| 路径 | 作用 |
|---|---|
| `~/10/dsh-build/dsh-purge` | **规范源码**（唯一维护点），link 的真实目标 |
| `~/plugins/dsh-purge` | → 上一行的软链（profile 依赖指向这里） |
| `~/.dsh/dsh-purge/` | 插件运行时状态（override-state、host-backups、installed-rev） |
| `~/.dsh/prompt-inject.md` | 注入的提示词正文 |
| `~/.dsh/net-scope-allow.txt` | 出网白名单（本机 + 服务器 IP） |
| `~/.dsh/profiles/web/` | profile：`package.json`（依赖 + bundle）、`cordis.patch.yml`（autoApplyOnStart） |

> **全机只保留这一份**（2026-10-21 清理）：上游 master 快照、`purge.tgz`、
> adapted 存档包都已删除，合计释放 8.3MB。
> 上游目前**不可达**（npm 无 `dsh-purge` 包、GitHub 仓库 404），所以这份就是唯一源，
> 改动务必同步进本文档。
>
> 另外 `~/.hermes/backups/dsh-mac-targets-pre-purge-20261021-191654/` 保留着
> 打补丁前的 53 个目标文件原件（不是插件副本，是 `--revert` 的兜底）。

> **上游原版别直接拿去装**：`dsh-purge.upstream-*` 的 `cordis.patch.yml` 里
> `personaSuffix: Your working directory is {{cwd}}.`，而 0.2.1 只注册
> `provider`/`model` 两个变量 → **每轮对话直接失败**：
> `unknown prompt variable "{{cwd}}" in section "deployment:persona-suffix"`

## 改了什么

### 一、上游 → 0.2.1 的适配（7 处，随 adapted 包带过来）

| # | 文件 | 改动 | 原因 |
|---|------|------|------|
| 1 | `lib/core.js` `targetFiles()` | 新增 `dev-context` 映射 → `dsh-harness-dev-context/lib/index.js` | 0.2.1 把 Web 面身份提示词从 `dsh-web-app` 搬到 `dsh-harness-dev-context`（`WEB_SURFACE_PROMPT` / `LEGACY_WEB_SURFACE`），旧映射打空 |
| 2 | `lib/core.js` patch #32 | `file` 增加 `dev-context`；新增 3 条 0.2.1 替换（新身份句、LEGACY 前缀、`Advertised URL` 描述）；markers 增加新标记 | 同上 |
| 3 | `lib/core.js` patch #53 | `file` 增加 `dev-context`；新增 0.2.1 收缩句；`Do not start a replacement server...` 一并删除 | 同上 |
| 4 | `lib/core.js` patch #77 | 新增 0.2.1 `append()` 变体（`if (this.revised.size > 0) this.revised.clear()`） | 0.2.1 客户端多了一层 size 判断，旧 pattern 打不中 |
| 5 | `client.js` `TabRowPurgeMount` | 锚点增加 `[data-dockkit-strip-tabs]`、`[data-dockkit-strip]` | 0.2.1 会话标签栏改用 DockKit 结构（`_tabStrip_*` / `data-dockkit-strip`），旧选择器找不到，侧栏标签按钮挂不上 |
| 6 | `cordis.patch.yml` | `personaSuffix` 去掉 `{{cwd}}`，改为 `Use pwd to determine the current working directory.` | 0.2.1 只注册 `provider`/`model` 变量，`{{cwd}}` 让整轮对话直接失败 |
| 7 | `lib/identity.js` | `CWD_ONLY_SUFFIX` 及两处 strip 替换去掉 `{{cwd}}` | 同上；这个才是实际运行时报错的那一处（`deployment:persona-suffix`） |

### 二、本次（2026-10-21）补的回填：把适配从「产物」搬回「源码」

**问题**：adapted 包里的 `client.js` 有 3 处修复是**手工改在构建产物上**的，
源码 `lib/redteam/client.js` 里没有。任何人执行 `npm run build:client`
（= `node scripts/build-client.mjs`）都会**静默把这 3 处改动洗掉**。

| # | 位置 | 修复内容 | 症状 |
|---|------|----------|------|
| A | `lib/redteam/client.js` `api()` | `res.json()` → 先 `res.text()` 再 `JSON.parse`，失败时返回结构化 error 而非抛异常 | 后端返回非 JSON（502/HTML 错误页）时整个面板白屏 |
| B | `lib/redteam/client.js` `download()` | `URL.revokeObjectURL(url)` → 延后 1s 再 revoke | 部分浏览器里立刻 revoke 会让 click 启动的下载断流，报告下载为空 |
| C | `lib/redteam/client.js` `VersionBar` | 倒计时 interval 用 `countdownRef` 持有，卸载/关闭弹窗时 `clearCountdown()`，改用 `closeModal` 关闭 | 组件卸载后 interval 泄漏，定时器持续跑 |

**验证**：回填后重跑 `node scripts/build-client.mjs`，产物与 adapted 的 `client.js`
**逐字节一致**（diff 为空）→ 源码现在可重现产物，适配不再依赖构建产物。

### 三、与服务器版本的差异（有意为之）

服务器 `/home/dsh/plugins/dsh-purge/lib/core.js` 比 adapted 包多 3 行：
把 `~/.npm-global-next` 提到候选路径前面（服务器并行装了 0.2.0 与 0.2.1 两份 DSH，
不放前面会去洗那份死树，状态全 pending）。
**本机只有单份安装**（`~/.local/lib/node_modules/@deepseek-ai/dsh`），不需要该改动。

## 四、仓库层面的改动（本仓库自有）

上游原版指向 `YuJunZhiXue/dsh-purge`，本仓库把「仓库指向」全部改到自己：

| 位置 | 改动 | 原因 |
|------|------|------|
| `lib/update.js` | `REPO` → 本仓库，`STABLE_REF` → `main` | 原版指向上游 master（未适配）。上游一旦恢复，「一键更新」会把 0.2.1 适配覆盖回去 —— 必须改 |
| `lib/redteam/ui.js` | `githubRepoOf()` 的兜底仓库 → 本仓库 | 侧栏「查看仓库」链接 |
| `package.json` | `repository` / `homepage` / `bugs` / `dsh.install.{tarball,zip,git}` → 本仓库 | `dsh plugin add` 的安装源必须是适配版，否则装到的是上游未适配包 |
| `client.js` | 首次安装「用户须知」文案与仓库链接改写为本仓库自己的措辞 | 原文以原作者口吻叙述（「作者将依法追究法律责任」「隐瞒作者的营利性再分发」） |
| `.github/workflows/release.yml` | 触发分支 → `main`，仓库守卫 → 本仓库，发布地址 → 本仓库 | 原守卫写死上游仓库名，在本仓库恒不触发 |
| `release-notes.md` | 顶部补 `1.1.72` 条目 | 发布流程要求首行等于版本号 |
| `README.md` / `README.en.md` | 整份重写为本仓库自己的文档 | 原 README 含上游的推广位、赞助与违规商用公示章节 |
| `README.zh-CN.md`、`docs/notice/` | 删除 | 仅服务于上游 README 的章节，且与本仓库 README 重复 |

> **仍然保留的部分**（有意）：`LICENSE`（MIT 要求保留版权声明）、
> README 末尾的「来源与许可」段、`release-notes.md` 里的上游 issue 链接（历史记录）、
> `docs/redteam-upstream-{DISCLOSURE,LICENSE}`（第三方 redteam 内容的合规声明）。

## 安装 / 重建流程

```bash
# 1) 装（link 依赖，改源码即生效）
cd ~/.dsh/profiles/web
#    package.json: dependencies 加 "dsh-purge": "link:~/plugins/dsh-purge"
#                  bundles 数组加 "dsh-purge"
pnpm install --offline

# 2) profile patch（DSH 升级后自动重打补丁）
#    ~/.dsh/profiles/web/cordis.patch.yml 追加：
#    - id: dsh-purge
#      config:
#        autoApplyOnStart: true

# 3) 出网白名单
printf '%s\n' <本机局域网 IP> <服务器 IP> > ~/.dsh/net-scope-allow.txt

# 4) 打补丁（幂等）
cd ~/plugins/dsh-purge && node bin/dsh-purge.js --apply    # 期望：71/86 已清洗, 0 待清洗

# 5) 重建前端（源码已含全部适配，可安全重跑）
cd ~/plugins/dsh-purge && node scripts/build-client.mjs

# 6) 重启本地 DSH + 重签 Caddy 票据（DSH 重启后 launch token 变了）
launchctl kickstart -k gui/$(id -u)/com.<用户名>.dshweb
python3 ~/.config/dsh-caddy/refresh-tickets.py
```

## 校验

```bash
cd ~/plugins/dsh-purge
node bin/dsh-purge.js --status | head -8        # 插件根应指向 ~/.local/lib/node_modules/...
node bin/dsh-purge.js --status | grep 已清洗     # 71/86 已清洗, 0 待清洗

# 端点（需带 dsh-auth 票据 + Host: 127.0.0.1:3080）
for P in status rewind rules override own-servers last-user; do
  curl -s -o /dev/null -w "$P -> %{http_code}\n" "http://127.0.0.1:3090/dsh-purge/$P" \
    -H "Host: 127.0.0.1:3080" -H "Cookie: $(grep -oE 'dsh-auth-[A-Za-z0-9_-]+=[A-Za-z0-9_.-]+' ~/.config/dsh-caddy/Caddyfile | head -1)"
done

# 产物可重现性（改完源码后跑，diff 应为空）
cp client.js /tmp/c.bak && node scripts/build-client.mjs && diff /tmp/c.bak client.js && echo OK
```

## 回滚

1. 还原补丁：`node bin/dsh-purge.js --revert`（备份在 `~/.dsh/dsh-purge/host-backups/`）
2. 移除插件：`dsh plugin --profile web remove dsh-purge`（或手删 package.json 的
   dependency + bundle 条目，再 `pnpm install --offline`）
3. 回到上游原版：上游不可达（npm/GitHub 都 404），本地也没有留副本。
   需要时按本文档「改了什么」反向回退这 10 处改动
4. 补丁前原件备份：`~/.hermes/backups/dsh-mac-targets-pre-purge-20261021-191654/`

## 注意

- **上游目前拿不到**：npm registry 无 `dsh-purge` 包（404），
  GitHub `YuJunZhiXue/dsh-purge` 也 404（私有或已删），master tarball 连不上。
  插件自带的「检查更新」在这台机器上恒定失败（`updateCheck` 返回 404）。
  → 这份就是唯一源，改坏了没有上游可对照，务必同步更新本文档。
- 上游若将来恢复，重新解包后需重新应用本文档的全部改动
- 补丁状态直接看 `node bin/dsh-purge.js --status`；`--apply` 幂等
- `autoApplyOnStart: true` 让 DSH 每次启动自动重打补丁，npm 升级 DSH 后不用手动补
