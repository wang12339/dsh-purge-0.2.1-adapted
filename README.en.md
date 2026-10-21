# dsh-purge · DSH 0.2.1 adaptation

A red-team evaluation plugin for the official [DeepSeek Harness](https://www.deepseek.com/harness/); this repository is the adaptation targeting **DSH 0.2.1-rc.2**.

The Harness does more than hand your prompt to a model — at runtime it rewrites prompts, gates approvals, constrains sandboxes and reshapes session formats. Judging only "did the model answer?" mistakes a host-level block for a model refusal. This plugin separates those two layers inside your own installation, so you can run controlled evaluations and see whether the default policy should be tightened.

| Surface | What it does |
|---|---|
| Side panel | A button next to the session title opens a right-hand panel with two pages: **Clean** and **Drill** |
| Clean | Browse patches by group, apply / revert / uninstall; edit the injected prompt and rule sets |
| Drill | Included in the official build. After authorization, inspect assets, skills and the local environment |
| Host policy | Adjust default wording, permission presets and tool limits. Official capabilities stay; no parallel identity system |
| On startup | Re-checks and re-applies automatically, so a `node_modules` overwrite from an npm upgrade needs no manual patching |

No hard-coded paths: the plugin looks at `$DSH_HOME`, then `.dsh` next to the dsh launcher, then `~/.dsh`. It never touches the Harness source tree — files are written only when you press **Apply** in the Clean page. The identity comes from the encrypted prompt shipped inside the plugin (the one pre-filled in the prompt box), not from prompt files on the host disk.

This plugin only operates on the official `@deepseek-ai` packages and local configuration already installed on your machine. It is not a public scanner and ships no attack payloads aimed at third-party sites.

---

## Compatibility

| DSH version | Status |
|---|---|
| 0.2.0-rc.2 | The upstream release works as-is |
| **0.2.1-rc.2** | **Requires the adaptation in this repository** (below) |

Upstream declares `dshTarget: 0.2.0-rc.2`. On 0.2.1 it breaks in three ways:

1. **Every conversation fails** — 0.2.1 dropped the `{{cwd}}` prompt variable (only `provider` / `model` are registered), so the host raises
   `unknown prompt variable "{{cwd}}" in section "deployment:persona-suffix"`
2. **The identity prompt lands nowhere** — 0.2.1 moved the web-surface identity prompt from `dsh-web-app` to `dsh-harness-dev-context`, so the old target-file mapping resolves to nothing
3. **The side-panel button does not mount** — 0.2.1 switched the session tab bar to a DockKit structure (`_tabStrip_*` / `data-dockkit-strip`), so the old selectors find no anchor

## What this repository changes

### 1. Upstream 0.2.1 adaptations (7)

| # | File | Change |
|---|---|---|
| 1 | `lib/core.js` `targetFiles()` | Adds the `dev-context` → `dsh-harness-dev-context/lib/index.js` mapping |
| 2 | `lib/core.js` patch #32 | `file` gains `dev-context`; adds three 0.2.1 replacements (new identity line, LEGACY prefix, `Advertised URL` wording) plus new markers |
| 3 | `lib/core.js` patch #53 | `file` gains `dev-context`; adds the 0.2.1 shrink line; drops `Do not start a replacement server...` |
| 4 | `lib/core.js` patch #77 | Adds the 0.2.1 `append()` variant (`if (this.revised.size > 0) this.revised.clear()`) |
| 5 | `client.js` `TabRowPurgeMount` | Anchors gain `[data-dockkit-strip-tabs]` and `[data-dockkit-strip]` |
| 6 | `cordis.patch.yml` | `personaSuffix` drops `{{cwd}}` and becomes `Use pwd to determine the current working directory.` |
| 7 | `lib/identity.js` | `CWD_ONLY_SUFFIX` and two strip replacements drop `{{cwd}}` (this is the one that actually threw, at `deployment:persona-suffix`) |

### 2. Backfilling artifact-only fixes into the source (3)

Three fixes in the upstream build artifact `client.js` were **hand-edited into the artifact** and never
landed in the source — a single `npm run build:client` silently drops them. This repository
backfills all three into `lib/redteam/client.js`:

| Location | Fix | Symptom without it |
|---|---|---|
| `api()` | `res.json()` → read text, then `JSON.parse`; return a structured error on failure | A non-JSON backend response (502 / HTML error page) blanks the whole panel |
| `download()` | `URL.revokeObjectURL` deferred by 1s | In some browsers the click-initiated download is cut off and the report file ends up empty |
| `VersionBar` | The countdown interval is held in a ref and cleared on unmount / modal close | The timer keeps running after the component is gone |

After the backfill, `node scripts/build-client.mjs` reproduces the adapted `client.js`
**byte for byte** — the source regenerates the artifact, so the adaptation no longer depends on a build output.

### 3. Two profile-level settings

```yaml
# ~/.dsh/profiles/web/cordis.patch.yml
- id: dsh-purge
  config:
    autoApplyOnStart: true      # re-apply automatically after an npm upgrade of DSH
```

```bash
# ~/.dsh/net-scope-allow.txt — one IP or hostname per line
<local LAN IP>
<server IP>
```

---

## Install

For profile `web` on DSH 0.2.1-rc.2:

```bash
# 1) Get the plugin source (link dependency — edits take effect immediately)
mkdir -p ~/plugins && tar xzf dsh-purge-1.1.72-adapted-0.2.1.tar.gz -C ~/plugins/
#    or clone this repository into ~/plugins/dsh-purge

# 2) Wire it into the profile
cd ~/.dsh/profiles/web
#    package.json → dependencies: add "dsh-purge": "link:/absolute/path/plugins/dsh-purge"
#    package.json → dsh.profile.bundles: add "dsh-purge"
pnpm install --offline

# 3) Outbound allowlist
printf '%s\n' <local LAN IP> <server IP> > ~/.dsh/net-scope-allow.txt

# 4) Apply the patches (idempotent)
cd ~/plugins/dsh-purge && node bin/dsh-purge.js --apply

# 5) Restart the host
launchctl kickstart -k gui/$(id -u)/com.<user>.dshweb   # macOS
systemctl restart dsh-web                                 # Linux
```

> **Desktop (EXE)**: `dshTarget` still reads `0.2.0-rc.2`. The desktop build uses profile `desktop`, installed and applied separately from web.

## Usage

```sh
# CLI
dsh-purge --status      # patch status and resolved target files
dsh-purge --apply       # apply (idempotent)
dsh-purge --revert      # revert
dsh-purge --uninstall   # uninstall
dsh-purge --edit        # edit the injected prompt
```

In chat:

```
/purge status | apply | revert | uninstall | edit | help
/rules list | use <id> | create <id> | delete <id> | reset | help
/skills list | import <archive or folder> | create <id> [note] | delete <id> | help
/rewind
```

Tools exposed to the model: `purge_status`, `purge_apply`, `purge_revert`.

"Rewind one turn" and "Rewind last turn" next to the input box use the official `session.fork({ atSeq })`: the finished turn is withdrawn, a new session continues with the previous context, the old session is archived, and the sentence returns to the input box — sending again re-injects it. `/rewind` does the same.

## Host policy overrides

The profile overlay loads after the built-in configuration and is preserved when the plugin applies its patches. To get tool results back faster, add to `~/.dsh/profiles/web/cordis.patch.yml` (edit the existing entry if the id is already there — do not duplicate it):

```yaml
- id: bash-sandbox
  config:
    timeoutMs: 10000
```

To default new sessions to low reasoning effort:

```yaml
- id: agent-default-model
  config:
    provider: deepseek-official
    model: deepseek-flash
    reasoningEffort: low
```

## Rebuilding the client

After editing `lib/redteam/client.js`:

```bash
cd ~/plugins/dsh-purge && node scripts/build-client.mjs
```

The output is `client.js`; the DSH reads it straight from disk, so a browser refresh is enough — no host restart needed.

## Verify

```bash
cd ~/plugins/dsh-purge
node bin/dsh-purge.js --status | head -8     # plugin root should point at the local dsh install
node bin/dsh-purge.js --status | grep 已清洗  # 71/86 已清洗, 0 待清洗
```

Endpoints (a login ticket is required):

```
/dsh-purge/status  /dsh-purge/rewind  /dsh-purge/rules
/dsh-purge/override  /dsh-purge/own-servers  /dsh-purge/last-user
```

## Revert and roll back

| Action | Command |
|---|---|
| Revert the patches | `node bin/dsh-purge.js --revert` (backups in `~/.dsh/dsh-purge/host-backups/`) |
| Remove the plugin | `dsh plugin --profile web remove dsh-purge` (revert first) |
| Drop it from the profile | Edit `package.json` (dependency + bundle), then `pnpm install --offline` |

## Common issues

**A task runs in standard mode but is blocked in minimal or PTC** — usually the preset's `run_code`
or plan-blocking lines were not cleaned, or minimal lacks `agent-instructions`. Fully quit the host →
apply in the Clean page → let it restart → start a new conversation. Switching presets alone does not
re-apply anything; patches in a still-running process are not refreshed.

**Bash calls queue up** — the Harness runs Bash calls in a batch sequentially. The plugin already
restores tagged old wait values to the official 60 seconds (patch #21). Untagged 10-minute waits are
indistinguishable from your own setting and are left alone.

---

## Source and license

This repository is an adaptation of [**YuJunZhiXue/dsh-purge**](https://github.com/YuJunZhiXue/dsh-purge)
(MIT, Copyright (c) 2026 小杨) and follows its
[attribution rules](https://github.com/YuJunZhiXue/dsh-purge#署名要求); the original copyright and
license notice are preserved verbatim in this repository's [LICENSE](LICENSE).
The 0.2.1 adaptation and the source backfill are documented in [ADAPT-0.2.1.md](ADAPT-0.2.1.md).

The upstream project declares itself free and non-commercial. This repository is not for sale,
resale or any black/grey-market monetisation either.
