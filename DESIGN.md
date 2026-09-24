# codex-proxy 方向文档

> ## ⚠️ 最高优先级：本文档只供参考，不是权威依据
>
> **一切以最新源代码和最新官方文档为准。** 实际编码前必须核对权威信源：
>
> - **仓库代码**（文件路径 / 函数签名 / 行号）—— 上游几乎每天在动，本文档记录的位置都可能已过期；
> - **Anthropic 协议** —— 以官方文档（platform.claude.com）为准；
> - **上游接口行为** —— 以实际请求/响应为准，必要时直接读源码验证。
>
> 本文档的价值在"方向与决策记录"，不在"事实的准确性"。**动代码前先验证，不要照着文档写。**

> 性质：**方向性文档**，不是详细设计。原则是**小步走**——每个里程碑独立可验证，做完一个再做一个。

## 1. 目标

把 ChatGPT 订阅（Codex 登录态）的模型能力，用 **Anthropic Messages 协议**代理出来，
让 Claude Code 这类客户端可以直接使用；同时提供一个**我们自己的终端控制面板**，用来观察和配置这个代理。

```
Claude Code ──(Anthropic Messages 协议 + SSE)──▶ codex-proxy ──(Responses API)──▶ chatgpt.com/backend-api/codex
                                                      │
                                                      └── 控制面板（同进程 TUI，看 + 配）
```

## 2. 三条原则

1. **尽量只增不改，让合并上游省事**（是权衡，不是教条）：
   - 首选**新增**文件/目录 —— 永不冲突；
   - 其次**追加**式改动 —— 如 README 顶部追加说明、`codex-rs/Cargo.toml` 的 members 加一行；
   - 必要时**可以改上游文件**，但要满足：改动小、集中在一处、尽量避开高频文件（见 §7.1），
     并登记到"上游文件改动清单"（§7.2），这样合并时知道该检查哪里。
2. **能复用就复用**：见 §4 复用清单。不重复造轮子——尤其别自己写鉴权、SSE 解析、模型目录。
3. **小步走**：见 §6 里程碑。每步都能独立验证，不追求一次做完。

## 3. 整体形态

一个二进制（crate 名 `codex-anthropic-proxy`）：

- **后台**：HTTP 服务，接 Claude Code 的 Anthropic 协议请求，翻译成 Responses 协议打给订阅后端
- **前台**：终端控制面板（ratatui），总览 / 实时请求 / 设置 / 日志 四个页面
- **同进程**：打开就是面板，关掉 = 代理停止（以后需要再加后台模式）

配置文件：`~/.codex-proxy/config.toml`（面板里改，自动保存）。

## 4. 复用清单（已核实）

| 要做的事 | 直接拿现成的 |
| --- | --- |
| 鉴权 + token 自动刷新 | `codex-login` 的 `AuthManager` / `AuthHeaders` |
| 调后端 + SSE 解析 + 重试 + 限流 | `codex-api` 的 `ResponsesClient`，直接产出 `ResponseEvent` 事件流 |
| 上游地址常量、provider 装配 | `model-provider-info` 的 `CHATGPT_CODEX_BASE_URL` + `codex-client` 的 `Provider` |
| 额度（5 小时 / 周）数据 | `codex-api` 的 rate limits 解析 —— 面板进度条的数据源 |
| 模型目录（设置页选模型） | `models-manager`，从后端拉真实可用模型列表 |
| CLI 参数 + `-c` 覆盖 | `codex-utils-cli` 的 `CliConfigOverrides` |
| 进程加固 | `codex-process-hardening` |
| HTTP 服务框架 | `axum`（workspace 已有依赖） |
| 面板框架 | `ratatui` + `crossterm`（workspace 依赖，与 TUI 同一套） |
| 请求 dump 调试 | 抄 `codex-rs/responses-api-proxy/src/dump.rs` 的做法 |

**拿不到的**：Codex TUI 的面板组件（列表选择、设置页、状态栏）都是 `codex-tui` 的**私有模块**，
公开面只有 25 个条目（`run_main` / `Cli` / `Terminal` / markdown 渲染等）。
所以面板要自己写页面 —— 但轮子是 **ratatui**，不是 Codex 的 TUI，用同一套框架不算造轮子。
可以照 `codex-rs/tui/src/bottom_pane/*_view.rs` 的写法来（同样的模式与风格）。

**真正必须自己写的**：Anthropic 协议类型 + 双向翻译（本项目的核心价值，没有现成的）。

## 5. 目录结构

```
codex-rs/anthropic-proxy/          ← crate 名 codex-anthropic-proxy
├── Cargo.toml
├── src/
│   ├── main.rs                    ← 启动：读配置 → 起服务 + 起面板
│   ├── config.rs                  ← 配置读写（~/.codex-proxy/config.toml）
│   ├── auth.rs                    ← 鉴权装配（AuthManager → AuthProvider）
│   ├── upstream.rs                ← 调 Responses 后端
│   ├── translate.rs               ← 双向翻译（纯函数，可离线测试）
│   ├── server.rs                  ← axum 路由：/v1/messages、/v1/messages/count_tokens
│   ├── state.rs                   ← 运行状态（请求流、统计、额度）供面板读取
│   └── panel/                     ← 面板（ratatui）
│       ├── overview.rs · requests.rs · settings.rs · logs.rs
└── tests/                         ← 录制 SSE 样本回放
```

## 6. 里程碑（慢慢做，一次一个）

| # | 内容 | 验收标准 |
| --- | --- | --- |
| **M0** | 骨架：新 crate + 空服务 | `cargo run` 起服务，`/healthz` 返回 ok |
| **M1** | 鉴权：读 `~/.codex/auth.json`，能刷新 token | 日志能打印出登录账号与套餐类型 |
| **M2** | 单轮非流式对话 | `curl` 打 `/v1/messages` 能拿到 Anthropic 格式回复 |
| **M3** | 流式（SSE） | Claude Code 连上后能看到逐字输出 |
| **M4** | 工具调用（id 往返 + 流式参数） | Claude Code 能读写文件、跑命令 |
| **M5** | 面板 v1：总览 / 实时请求 / 日志 | 打开就是面板，能看到请求流水 |
| **M6** | 设置页：看 + 改，自动保存 | 面板里能改模型映射、端口、开关 |
| **M7** | 打磨：额度显示、推理重放、缓存优化、图片、错误映射 | 长对话质量正常；面板能看到额度 |

每个里程碑 = 一个小提交，独立可验证。

## 7. 上游同步

### 7.1 高频文件参考（要改上游文件前先看这里）

| 文件 | 上游最近 500 次提交中被改 | 备注 |
| --- | --- | --- |
| `codex-rs/Cargo.toml` | 446 | 几乎每次提交都动；我们只加 members 一行 |
| `codex-rs/cli/src/main.rs` | 287 | 高频；要加子命令前先想清楚值不值得 |
| `README.md` | 97 | 我们只在顶部追加，正文不动 |

（数据可用 `git log -500 --oneline -- <文件> | wc -l` 随时重新测量。）

### 7.2 我们对上游文件的改动清单（保持更新）

| 文件 | 我们的改动 | 合并冲突时怎么办 |
| --- | --- | --- |
| `README.md` | 顶部追加本项目说明 | 两块都保留 |
| `codex-rs/Cargo.toml` | members 加一行（M0 时） | 两块都保留 |

**改动上游文件时必须同步更新这张表** —— 合并时照着它逐项检查就行。

### 7.3 同步命令

```bash
# 首次
git remote add upstream https://github.com/openai/codex.git

# 定期：合并上游 + 巡检协议相关改动
git fetch upstream
git merge upstream/main
git log --oneline <上次同步的 SHA>..upstream/main -- \
    codex-rs/login codex-rs/codex-api/src codex-rs/core/src/client.rs codex-rs/model-provider
```

因为改动以"只增不改"为主，合并通常无冲突；万一有冲突，先查 §7.2 的清单。
需要人工判断的只有"上游协议相关代码变了没"。

## 8. 关键事实速查（省得重新找）

> ⚠️ 下表是**当时**记录的位置，上游随时会变 —— 用之前先验证（见文首提示）。

| 事实 | 位置 |
| --- | --- |
| OAuth client_id = `app_EMoamEEZ73f0CkXaXp7hrann` | `codex-rs/login/src/auth/manager.rs:1717` |
| `auth.json` 结构（tokens / account_id / last_refresh） | `codex-rs/login/src/auth/storage.rs:41` |
| 上游基址 `https://chatgpt.com/backend-api/codex` | `CHATGPT_CODEX_BASE_URL`（`codex-rs/model-provider-info/src/lib.rs`） |
| 请求要带 `Authorization` + `ChatGPT-Account-ID` | `codex-rs/model-provider/src/auth.rs:106` |
| 请求体：`store: false` | `codex-rs/core/src/client.rs:999` |
| 请求体：`include: ["reasoning.encrypted_content"]` | `codex-rs/core/src/client.rs:961` |
| Responses 客户端与事件流类型 | `codex-rs/codex-api/src/endpoint/responses.rs`、`common.rs:412` |
| TUI 库入口（如需参考） | `codex-rs/tui/src/lib.rs:1084` |
| `x-oai-attestation` 目前可选（CLI 未设置） | `codex-rs/core/src/attestation.rs` |

Anthropic 协议侧（已核对官方文档）：`/v1/messages` 与 `/v1/messages/count_tokens`；
SSE 事件序列 `message_start → (content_block_start → delta* → content_block_stop)* → message_delta → message_stop`；
delta 类型 `text_delta` / `input_json_delta` / `thinking_delta` / `signature_delta`；
错误体 `{"type":"error","error":{"type","message"}}`。

## 9. 风险

1. **账号风险（最大）**：把订阅登录态用于非官方客户端属于灰色地带，可能被限流甚至封号，需自行权衡。
2. **未公开接口**：`chatgpt.com/backend-api/codex` 随时可能变 —— 这也是"复用 `codex-api`"的核心价值（上游会跟着改）。
3. **attestation**：`x-oai-attestation` 目前可选，若 OpenAI 将来强制要求，代理会整体失效。
4. **额度消耗**：订阅限额（5 小时 / 周）比 API 紧，Claude Code 的并发子代理是消耗大户。
5. **推理重放**：把加密推理藏进 thinking `signature` 是可行方案（官方文档保证 thinking 块原样回传），
   但代价是**该对话不可回迁到真正的 Anthropic API**（签名会校验失败）。

## 10. 暂不做的事

- 改 Codex TUI / 复用其面板组件（私有模块，拿不到）
- 代理侧会话表（用 signature 走私替代）
- apply_patch 工具翻译层（先透传 Claude Code 原生工具，实测后再定）
- WebSearch 服务端工具实现（先返回错误）
- 后台守护进程模式（M0–M7 都是单进程）
