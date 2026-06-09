# RustDesk Guide

## Project Layout

### Directory Structure
* `src/` Rust app
* `src/server/` audio / clipboard / input / video / network
* `src/platform/` platform-specific code
* `src/ui/` legacy Sciter UI (deprecated)
* `flutter/` current UI
* `libs/hbb_common/` config / proto / shared utils
* `libs/scrap/` screen capture
* `libs/enigo/` input control
* `libs/clipboard/` clipboard
* `libs/hbb_common/src/config.rs` all options

### Key Components
- **Remote Desktop Protocol**: Custom protocol implemented in `src/rendezvous_mediator.rs` for communicating with rustdesk-server
- **Screen Capture**: Platform-specific screen capture in `libs/scrap/`
- **Input Handling**: Cross-platform input simulation in `libs/enigo/`
- **Audio/Video Services**: Real-time audio/video streaming in `src/server/`
- **File Transfer**: Secure file transfer implementation in `libs/hbb_common/`

### UI Architecture
- **Legacy UI**: Sciter-based (deprecated) - files in `src/ui/`
- **Modern UI**: Flutter-based - files in `flutter/`
  - Desktop: `flutter/lib/desktop/`
  - Mobile: `flutter/lib/mobile/`
  - Shared: `flutter/lib/common/` and `flutter/lib/models/`

## Rust Rules

* Avoid `unwrap()` / `expect()` in production code.
* Exceptions:

  * tests;
  * lock acquisition where failure means poisoning, not normal control flow.
* Otherwise prefer `Result` + `?` or explicit handling.
* Do not ignore errors silently.
* Avoid unnecessary `.clone()`.
* Prefer borrowing when practical.
* Do not add dependencies unless needed.
* Keep code simple and idiomatic.

## Tokio Rules

* Assume a Tokio runtime already exists.
* Never create nested runtimes.
* Never call `Runtime::block_on()` inside Tokio / async code.
* Do not hide runtime creation inside helpers or libraries.
* Do not hold locks across `.await`.
* Prefer `.await`, `tokio::spawn`, channels.
* Use `spawn_blocking` or dedicated threads for blocking work.
* Do not use `std::thread::sleep()` in async code.

## Editing Hygiene

* Change only what is required.
* Prefer the smallest valid diff.
* Do not refactor unrelated code.
* Do not make formatting-only changes.
* Keep naming/style consistent with nearby code.

## Localization (`src/lang/*.rs`)

Each file is a `HashMap<key, translation>`. Layout:

* `template.rs` is the master list of every key. **Never edit it** as part of translation work.
* `en.rs` holds only the keys whose English display text differs from the key itself.
* Every other file (`de.rs`, `fr.rs`, …) carries the full key set; an untranslated entry has an empty value: `("key", "")`.

### Finding the English source for a key

When filling an empty entry, determine the source English text with this rule:

* If `key` exists in `en.rs` **with a non-empty value**, that value is the source text (look it up in `en.rs`).
* Otherwise the **key string itself is the source text** (the key is already plain English).

Then translate that source into the file's target language (infer the language from the file's existing non-empty entries / filename).

### Translation hygiene

* Only fill empty values. Never change keys, and never touch existing non-empty translations.
* Preserve placeholders (`{}`) and escape sequences (`\n`, `\"`) exactly as in the source.
* Do not translate brand or technical tokens: `RustDesk`, `Socks5`, `TLS`, `UAC`, `Wayland`, `X11`, `TCP`, `UDP`, `2FA`, `RDP`, `D3D`, etc.
* Copy URL values (e.g. `doc_*` keys) verbatim from `en.rs`.

---

## 🚀 私有化 Fork — GitHub Actions 编译指南

### 仓库关系

```
hbb_common (你的私有库) ──┬── hwx_rustdesk (客户端, 公开fork, .gitmodules 子模块引用)
                         └── rustdesk-server (服务端, 公开克隆, .gitmodules 子模块引用)
```

`hbb_common` 是一个 git 子模块，被两个上层项目共用。所有品牌标识（应用名、服务器地址、公钥）
都通过编译时环境变量 `option_env!("VAR")` 注入，无需修改源代码。

### 步骤一：设置 GitHub Secrets

在 `hwx_rustdesk` 仓库的 **Settings > Secrets and variables > Actions** 中添加以下 3 个 Repository secrets：

| Secret 名 | 值 | 说明 |
|-----------|-----|------|
| `RS_PUB_KEY` | `你的 hbbs 服务器启动后生成的 id_ed25519.pub 内容 (base64)` | 客户端验证服务器身份 |
| `RENDEZVOUS_SERVER` | `你的 hbbs 服务器域名或 IP` | 如 `"hbbs.example.com"` |
| `API_SERVER` | `你的版本检查 API 地址` | 可选，默认用 rustdesk.com |

> 💡 **提示**：hbbs 首次启动时会在工作目录生成 `id_ed25519` 和 `id_ed25519.pub`。
> 将 `id_ed25519.pub` 的完整内容（一行 base64 字符串）设为 `RS_PUB_KEY`。

### 步骤二：确保子模块能通过 CI 认证

由于 `hbb_common` 是你自己的私有仓库，GitHub Actions 需要认证才能 `git submodule update`。

**方案 A — Personal Access Token (推荐)**：
1. 在 [GitHub Settings > Developer settings > Personal access tokens](https://github.com/settings/tokens)
   创建一个 Fine-grained token，权限只需 `Contents: read`（对私有 `hbb_common` 仓库）
2. 在 `hwx_rustdesk` 仓库添加一个名为 `GH_PAT` 的 Secret，贴上该 token
3. 在 `.github/workflows/*.yml` 的 `actions/checkout` 步骤中添加：
   ```yaml
   - uses: actions/checkout@v4
     with:
       submodules: recursive
       token: ${{ secrets.GH_PAT }}
   ```

**方案 B — SSH Deploy Key**：
1. 生成一对 SSH 密钥：`ssh-keygen -t ed25519 -C "ci@example.com" -f ./ci_key`
2. 在私有 `hbb_common` 仓库添加 Deploy Key（Settings > Deploy keys，勾选 Read-only）
3. 在 `hwx_rustdesk` 仓库添加名为 `SSH_PRIVATE_KEY` 的 Secret（私钥内容）
4. 在 workflow 的 checkout 步骤前添加：
   ```yaml
   - name: Setup SSH
     run: |
       mkdir -p ~/.ssh
       echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
       chmod 600 ~/.ssh/id_ed25519
   ```

### 步骤三：触发编译

- **Tag 推送**：推送 `v1.4.7` 类似 tag → 触发 `flutter-tag.yml` → 编译所有平台 + 发布 Release
- **每夜构建**：每天 UTC 00:00 自动触发 `flutter-nightly.yml`
- **手动触发**：在 Actions 页面选择 `playground.yml` → `Run workflow` → 填写参数

### 步骤四：自建 hbbs/hbbr 服务器

编译出的客户端默认会连接到你在 `RENDEZVOUS_SERVER` Secret 中指定的服务器地址。
确保你已部署 `rustdesk-server`：
- 使用 `rustdesk-server/.github/workflows/build.yaml` 编译 hbbs + hbbr
- 或使用 Docker：`docker run -d --restart unless-stopped --name hbbs rustdesk/rustdesk-server-s6 hbbs`
- 或使用 `docker-compose.yml`（参考 `rustdesk-server/docker-compose.yml`）

### 可选：消除外部依赖风险

以下依赖来自 `rustdesk-org` 的 GitHub fork。如果这些仓库变更权限，你的构建会失败。
建议 fork 到你自己名下，然后在 `libs/hbb_common/Cargo.toml` 中修改 URL：

| 依赖 | 当前来源 | 建议操作 |
|------|----------|----------|
| `confy` | `rustdesk-org/confy` | 再 fork 到你的 GitHub |
| `tokio-socks` | `rustdesk-org/tokio-socks` | 再 fork 到你的 GitHub |
| `sysinfo` | `rustdesk-org/sysinfo` | 再 fork 到你的 GitHub |
| `default_net` | `rustdesk-org/default_net` | 再 fork 到你的 GitHub |
| `machine-uid` | `rustdesk-org/machine-uid` | 再 fork 到你的 GitHub |

详情见 `libs/hbb_common/PRIVATE_FORK_TODO.md`。

