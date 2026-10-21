# dsh-purge · DSH 0.2.1 适配版

装在官方 [DeepSeek Harness](https://www.deepseek.com/harness/) 上的红队评测插件，本仓库是针对 **DSH 0.2.1-rc.2** 的适配版本。

Harness 不只是把问题丢给模型 —— 它运行时还会改提示词、卡审批门、限沙箱、改会话格式。只看「模型答不答」，会把宿主拦截误判成模型拒绝。这个插件在你自己的安装里把这两层拆开，用来做受控评测，也用来看默认策略该不该收紧。

| 场景 | 它做什么 |
|---|---|
| 侧栏 | 会话标题旁的按钮拉出右侧栏，两页：**清洗**、**演练台** |
| 清洗 | 分组查看补丁，应用 / 还原 / 卸载；编辑注入提示词和多套规则 |
| 演练台 | 正式版内置。授权后查看资产、技能与本机环境 |
| 宿主策略 | 调整默认文案、权限策略、工具上限。官方能力保留，不另起一套身份 |
| 启动时 | 自动再检查并应用。npm 升级盖掉 `node_modules` 之后不用手改文件 |

不写死盘符：依次按 `$DSH_HOME` → dsh 启动器旁边的 `.dsh` → `~/.dsh` 找。**不碰 Harness 源码仓库**，只有在「清洗」里点「应用」才写入磁盘。身份来自插件内置的加密提示词（提示词框里默认显示的那份），不读宿主磁盘上的提示词文件。

本插件只处理使用者本机已安装的官方 `@deepseek-ai` 包和本机配置。它不是公网扫描器，也不含针对第三方站点的攻击载荷。

---

## 兼容性

| DSH 版本 | 状态 |
|---|---|
| 0.2.0-rc.2 | 上游版本直接可用 |
| **0.2.1-rc.2** | **需要本仓库的适配**（见下） |

上游版本声明的 `dshTarget` 是 `0.2.0-rc.2`，直接装到 0.2.1 上会出现三类断裂：

1. **每轮对话直接失败** —— 0.2.1 取消了 `{{cwd}}` 提示词变量（只注册 `provider` / `model`），宿主报
   `unknown prompt variable "{{cwd}}" in section "deployment:persona-suffix"`
2. **身份提示词打空** —— 0.2.1 把 Web 面身份提示词从 `dsh-web-app` 搬到 `dsh-harness-dev-context`，旧的目标文件映射落空
3. **侧栏按钮挂不上** —— 0.2.1 的会话标签栏换成 DockKit 结构（`_tabStrip_*` / `data-dockkit-strip`），旧选择器找不到锚点

## 本仓库做了什么

### 一、上游 0.2.1 适配（7 处）

| # | 文件 | 改动 |
|---|---|---|
| 1 | `lib/core.js` `targetFiles()` | 新增 `dev-context` → `dsh-harness-dev-context/lib/index.js` 映射 |
| 2 | `lib/core.js` patch #32 | `file` 增加 `dev-context`；补 3 条 0.2.1 替换（新身份句 / LEGACY 前缀 / `Advertised URL` 描述）与新 marker |
| 3 | `lib/core.js` patch #53 | `file` 增加 `dev-context`；补 0.2.1 收缩句；移除 `Do not start a replacement server...` |
| 4 | `lib/core.js` patch #77 | 新增 0.2.1 `append()` 变体（`if (this.revised.size > 0) this.revised.clear()`） |
| 5 | `client.js` `TabRowPurgeMount` | 锚点增加 `[data-dockkit-strip-tabs]`、`[data-dockkit-strip]` |
| 6 | `cordis.patch.yml` | `personaSuffix` 去掉 `{{cwd}}`，改为 `Use pwd to determine the current working directory.` |
| 7 | `lib/identity.js` | `CWD_ONLY_SUFFIX` 及两处 strip 替换去掉 `{{cwd}}`（真正报错的是这处 `deployment:persona-suffix`） |

### 二、把只存在于产物的修复回填源码（3 处）

上游的构建产物 `client.js` 里有 3 处修复是**手工改在产物上**的，源码里没有 —— 执行一次
`npm run build:client` 就会静默洗掉它们。本仓库把它们全部回填进 `lib/redteam/client.js`：

| 位置 | 修复 | 不修的症状 |
|---|---|---|
| `api()` | `res.json()` → 先取 text 再 `JSON.parse`，失败返回结构化 error | 后端回非 JSON（502 / HTML 错误页）时整个面板白屏 |
| `download()` | `URL.revokeObjectURL` 延后 1s | 部分浏览器里 click 启动的下载会断流，报告文件是空的 |
| `VersionBar` | 倒计时 interval 用 ref 持有，卸载 / 关闭弹窗时清理 | 组件卸载后定时器泄漏，一直跑 |

回填之后 `node scripts/build-client.mjs` 的产物与原 adapted 的 `client.js` **逐字节一致** ——
源码能重现产物，适配不再依赖构建产物。

### 三、profile 层的两项配置

```yaml
# ~/.dsh/profiles/web/cordis.patch.yml
- id: dsh-purge
  config:
    autoApplyOnStart: true      # npm 升级 DSH 后重启会自动重打补丁
```

```bash
# ~/.dsh/net-scope-allow.txt —— 每行一个 IP 或主机名
<本机局域网 IP>
<服务器 IP>
```

---

## 安装

以 profile `web`、DSH 0.2.1-rc.2 为例：

```bash
# 1) 拿到插件源码（link 依赖，改源码即生效）
mkdir -p ~/plugins && tar xzf dsh-purge-1.1.72-adapted-0.2.1.tar.gz -C ~/plugins/
#    或 git clone 本仓库到 ~/plugins/dsh-purge

# 2) 挂进 profile
cd ~/.dsh/profiles/web
#    package.json → dependencies 加 "dsh-purge": "link:/绝对路径/plugins/dsh-purge"
#    package.json → dsh.profile.bundles 数组加 "dsh-purge"
pnpm install --offline

# 3) 出网白名单
printf '%s\n' <本机局域网 IP> <服务器 IP> > ~/.dsh/net-scope-allow.txt

# 4) 打补丁（幂等）
cd ~/plugins/dsh-purge && node bin/dsh-purge.js --apply

# 5) 重启宿主
launchctl kickstart -k gui/$(id -u)/com.<用户名>.dshweb   # macOS
systemctl restart dsh-web                                    # Linux
```

> **桌面端（EXE）**：`dshTarget` 仍写 `0.2.0-rc.2`，桌面端用 profile `desktop`，与 web 分开安装、分开应用。

## 使用

```sh
# CLI
dsh-purge --status      # 看补丁状态与目标文件
dsh-purge --apply       # 应用（幂等）
dsh-purge --revert      # 还原
dsh-purge --uninstall   # 卸载
dsh-purge --edit        # 编辑注入的提示词
```

聊天里：

```
/purge status | apply | revert | uninstall | edit | help
/rules list | use <id> | create <id> | delete <id> | reset | help
/skills list | import <压缩包或文件夹> | create <id> [说明] | delete <id> | help
/rewind
```

模型可调用的工具：`purge_status`、`purge_apply`、`purge_revert`。

输入框旁的「回退一次」「回退上一轮」走官方 `session.fork({ atSeq })`：撤回已进行的那一轮，
新会话接上之前的上下文，旧会话归档，那句回到输入框，再发送会重新注入。`/rewind` 同效。

## 宿主策略覆盖

profile 覆盖层在内置配置之后加载，插件应用时保留它。想让工具结果更快回来，
在 `~/.dsh/profiles/web/cordis.patch.yml` 里加（已有同 id 就改原来的，别重复加）：

```yaml
- id: bash-sandbox
  config:
    timeoutMs: 10000
```

新会话默认用低推理：

```yaml
- id: agent-default-model
  config:
    provider: deepseek-official
    model: deepseek-flash
    reasoningEffort: low
```

## 重建前端

改过 `lib/redteam/client.js` 之后：

```bash
cd ~/plugins/dsh-purge && node scripts/build-client.mjs
```

产物是 `client.js`，DSH 直接从磁盘读它，改完刷新页面即可（不必重启宿主）。

## 校验

```bash
cd ~/plugins/dsh-purge
node bin/dsh-purge.js --status | head -8     # 插件根应指向本机 dsh 安装目录
node bin/dsh-purge.js --status | grep 已清洗  # 71/86 已清洗, 0 待清洗
```

端点（需带登录票据）：

```
/dsh-purge/status  /dsh-purge/rewind  /dsh-purge/rules
/dsh-purge/override  /dsh-purge/own-servers  /dsh-purge/last-user
```

## 还原与回滚

| 动作 | 命令 |
|---|---|
| 还原补丁 | `node bin/dsh-purge.js --revert`（备份在 `~/.dsh/dsh-purge/host-backups/`） |
| 移除插件 | `dsh plugin --profile web remove dsh-purge`（先 revert） |
| 删掉 profile 里的条目 | 手改 `package.json` 的 dependency + bundle，再 `pnpm install --offline` |

## 常见问题

**同一任务在标准模式能跑、在极简或 PTC 被拦** —— 通常是 preset 里的 `run_code` 或 plan
拦截句没洗净，或 minimal 缺 `agent-instructions`。完整退出宿主 → 清洗里应用 → 自动重启 →
新开一轮对话。只换 preset 不重应用，旧进程里的补丁不会更新。

**Bash 调用串行等待** —— Harness 按顺序执行同一批 Bash 调用，插件已把带标记的旧等待值
恢复为官方 60 秒（补丁 #21）。无标记的 10 分钟等待无法区分是插件旧值还是你自己的设置，
保留不动。

---

## 来源与许可

本仓库基于 [**YuJunZhiXue/dsh-purge**](https://github.com/YuJunZhiXue/dsh-purge)（MIT，
Copyright (c) 2026 小杨）适配而来，遵循其 [署名要求](https://github.com/YuJunZhiXue/dsh-purge#署名要求)；
原仓库的版权与许可声明完整保留在本仓库的 [LICENSE](LICENSE) 中。
本仓库的 0.2.1 适配与源码回填改动见 [ADAPT-0.2.1.md](ADAPT-0.2.1.md)。

上游项目声明为免费非盈利。本仓库同样不作商业售卖、付费倒卖或黑灰产牟利用途。
