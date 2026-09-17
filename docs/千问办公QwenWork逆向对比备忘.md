# 千问办公（QwenWork）逆向对比备忘

> 目的：对比两个开源参考项目的千问办公逆向原理是否一致，并评估与 wild-work 的契合度。
> 克隆位置（不入版本控制）：`ref/xrl-router-plugin-qwenwork`、`ref/Buddy2api`
> 编写日期：2026-09（对照 wild-work v2.1.x）

---

## 0. 两个参考项目速览

| 项 | xrl-router-plugin-qwenwork | Buddy2api |
|---|---|---|
| 作者 | 杏仁鹿（krkr@xrl.im） | LBJ_WICM |
| 语言/栈 | TypeScript + Express（xrl-router 插件，WS 注册） | Python + FastAPI + SQLite + 静态 Web 管理页 |
| 版本/最后提交 | 0.1.0 / 2026-08-06 | 2.1.9 / 2026-09-15 |
| 许可 | MIT | MIT |
| 渠道 | qwenwork（默认）+ wukong（钉钉悟空 DEAP） | workbuddy / qclaw / qwenwork / traework |
| 定位 | **纯协议桥接层**：不做密钥池/重试/路由（交给 xrl-router） | **完整网关**：账号库、API Key、额度、Codex `/v1/responses` |
| 逆向文档 | `docs/reverse/QWENWORKCN_REVERSE.md`（极详尽，含失败路径与绕过链） | `docs/design/multi-channel-v2.md` Appendix B（实现要点 + 冻结门闩） |

两者**互相独立**（无代码引用关系），却得出高度一致的结论，互证价值很高。

---

## 1. 逆向原理对比：核心算法字节级同源

### 1.1 完全一致的部分

| 环节 | 两项目实现（逐条一致） |
|---|---|
| 应用真身 | QwenWorkCN 桌面端（Electron），userData = `%APPDATA%\QwenWorkCN`，凭据文件 `auth-v2.dat` |
| 凭据加密 | Electron safeStorage `v10` 头。Windows：`Local State` 的 `os_crypt.encrypted_key`（`DPAPI\0` 前缀）→ DPAPI(CurrentUser, entropy=NULL) → 32B AES key → **AES-256-GCM**（`v10 + 12B nonce + ct + 16B tag`） |
| token 刷新 | `POST https://gateway.qwenwork.cn/api/v1/deviceToken/refresh`，body `{refresh_token, target:"c"}` → 返回 `device_token`(或 `token`) + **轮换的** `refresh_token` + `expires_at` |
| AES 会话密钥 | **16 个 ASCII hex 字符**（`uuid4().hex[:16]`），`key = iv = utf8 字节`；**不是** `os.urandom(16)` |
| info 明文 | `{uid, aid:"", name, email, security_oauth_token:<access>}` → AES-128-CBC(PKCS7) → base64 |
| Cosy-Key | `base64(RSA_PKCS1_v1_5(公钥, 16字符))`，**PKCS1 非 OAEP**（OAEP → 403） |
| authorization | `Bearer COSY.<b64(header)>.<md5>`，header = `{version:"v1", requestId, info, cosyVersion, ideVersion}` |
| 签名串 | `f"{o}\n{cosyKey}\n{ts}\n{body}\n{path}"`；`path` = URL pathname，**去 query + 去 `/algo` 前缀** |
| RSA 公钥 | **同一把 PEM**（1024-bit，modulus 头 `c0f223…`），且与 wild-work `internal/qoder/cosy.go` 的 `serverPubKeyPEM` 逐字节相同 |
| 推理端点 | `POST /algo/api/v2/service/pro/sse/agent_chat_generation?FetchKeys=llm_model_result&AgentId=agent_common` |
| 请求体 | **明文 JSON**（不带 `Encode=1`），必需 `request_id` / `session_id` |
| 响应格式 | 外层 envelope `data:{"headers":…,"body":"<内层 OpenAI chunk JSON>","statusCodeValue":200}`，需剥一层 |
| 模型档位 | `qwork-advanced` / `qwork-auto` / `qwork-lite` / `qmodel_latest`；禁止把 `glm-5.2` 当 `x-model-key` |
| 签到 | **无签到活动**（Buddy2api `checkin_supported=False`） |
| 额度 | 无稳定 COSY 额度 API；Buddy2api 用 `unit=unknown` 且拒绝填 0 参与加总 |
| 静态头族 | `Cosy-Business-Product: qoder_work`、`Cosy-Business-Type: agent`、`Cosy-ClientType: 6`、`Cosy-Scene: qwork`、`Login-Version: v2`、`x-model-source: system` |

**结论：两者对「千问办公逆向原理」的认知完全一致，且与 wild-work 现有 Qoder COSY 实现同构。**

### 1.2 存在差异的部分（版本漂移 + 工程取舍）

| 维度 | xrl-router-plugin | Buddy2api |
|---|---|---|
| 逆向观测版本 | macOS `QwenWorkCN.app 0.1.3` → Windows 实机验证 | Windows `0.1.8-26081406` |
| `Cosy-Version` | `1.0.47` | `1.1.18`（取自 qoderclicn 常量 `l0A`，并设 `COSY_VERSION_FROZEN` 门闩，未冻结拒绝出站） |
| header `cosyVersion`/`ideVersion` | `"1.0.0"` / `"1.0.0"` | `1.1.18` / `0.1.8` |
| `Cosy-MachineOS` / UA | `aarch64_darwin` / `node` | `x86_64_win32` / `qoderwork/0.1.8` |
| `Cosy-MachineId` | 固定 `"unknown"` | 取 `auth-v2.dat` 的 `loginDeviceId`（更贴近官方） |
| body 策略 | **原样透传** OpenAI chat.completions + 补 `request_id`/`session_id` | **重构造** qoder 原生结构（`chat_context.text` 为字符串、`session_type:"qoder_work"`、`model_config`…） |
| token 生命周期 | 三源 fallback（内存 → `auth-v2.dat` → `.env QWEN_KEYS`）+ `fs.watch` 文件监听 + **双向写回**（auth-v2.dat + .env），按需刷新（5min 缓冲，单飞防并发） | 账号入库 SQLite，`pick_account_with_fallback` 按需刷新，刷新成功后写回 `auth-v2.dat`（原子 replace + 首次 `.bak` + 保留未知字段） |
| 平台覆盖 | macOS（Keychain + PBKDF2 1003/saltysalt/AES-128-CBC IV=0x20）+ Windows（DPAPI + GCM） | **仅 Windows**（DPAPI），Linux Docker 明确不支持 |
| 额度查询 | 无 | `GET /api/v1/adapter/user/account-context?include=user,plan,quota`，递归抠 `remaining/balance/credits` 等键，抠不到就 `unsupported` |
| 模型列表 | 静态 4 个 | 静态 + `GET /api/v2/model/list` 动态拉取 |

### 1.3 关键发现

1. **RSA 公钥三方一致**（wild-work qoder / xrl / Buddy2api）→ 该 PEM 是「Qoder 系」共享公钥，可安全作为常量；也佐证千问办公与 QoderWork 同源（xrl 文档明确：千问办公内部代号即 **Qoder**，出品方 DingTalk，更新源 `static.qoder.com.cn/qwen-work-cn`）。
2. **明文 body 已双份独立验证**：xrl §6.8（HTTP 200 + glm-5.2）与 Buddy2api Appendix B（明文、不带 `Encode=1`）。这解除了 Buddy2api 设计文档中「QoderWork CN Encode 事实不足」对 **QwenWork** 的阻塞。
3. **`cosy-key` 语义已被修正**：xrl §6.1 曾推测为「RSA-1024 封装的会话对称密钥」，§6.8 自我修正为「RSA_PKCS1(asar 公钥, 16 字符 AES key)」，与另两方实现一致。
4. **`Cosy-Version` 校验宽松**：`1.0.47` / `1.1.18` / `0.1.43`（wild-work Qoder）三种取值均能通过网关 → 该头非强校验项。
5. **Buddy2api 把 QwenWork 与 QoderWork CN 明确分家**：QwenWork = `gateway.qwenwork.cn` / clienttype **6** / 明文；QoderWork CN = `gateway.qoder.com.cn` / clienttype **5** / `Encode=1` / `dt-`·`drt-`。其设计文档预期「两者 COSY 不字节兼容」——但**公钥相同、签名串格式相同**，实际是同一套 COSY 框架 + 不同常量/Encode 开关，wild-work 的 `internal/qoder` 因此具备直接改造为双渠道的基础。
6. **防重放机制**：签名 `md5` 绑定 `body + path + 时间戳`，故每请求必须新生成 `timestamp` + 新 `info`/`cosy-key`。xrl §6.5 记录的 `103 Duplicate request` 死锁，根因是复用抓包的 authorization；自造签名即绕过。

---

## 2. 与本项目（wild-work）的契合度

### 2.1 wild-work 现状

已有 **Qoder 渠道**（`internal/qoder`）：

| 项 | 现值 |
|---|---|
| 域名 | Base `openapi.qoder.com.cn` / Gateway `gateway.qoder.com.cn` |
| 鉴权 | `dt-` / `drt-`；COSY 签名（`cosy.go`：RSA_PKCS1 + AES-128-CBC + MD5） |
| RSA 公钥 | 与两个参考项目**逐字节相同** |
| 静态头 | `cosy-clienttype: 5`、`cosy-version: 0.1.43`、`cosy-data-policy: AGREE`、`user-agent: Go-http-client/2.0` |
| body | `buildAgentBody` 重构造（`session_type: "qodercli"`、`aliyun_user_type: personal_professional_trial`、`chat_context.text` 为 **对象**） |
| 编码 | `Encode=1` + `qoderEncode`（base64 + 自定义字母表 + 三段重排） |
| SSE | `parseNestedSSE` 剥 `body` 外层 |
| 签到 | `DailyCheckin` 返回错误（无活动） |
| 登录 | OAuth Device Flow（`client_id 1c5e33e1-…`、PKCE） |

**尚无 qwenwork 渠道**（`gateway.qwenwork.cn` / clienttype 6 / 明文 body / `auth-v2.dat` safeStorage / `ory_rt_` 刷新 / `qoder_work` 产品头）。

### 2.2 契合度评分

| 层 | 契合度 | 说明 |
|---|---|---|
| 签名算法层 | **95%** | `cosy.go` 结构可直接复用，仅需常量参数化 |
| 协议/传输层 | **90%** | 嵌套 SSE、错误分类、`provider.Upstream` 均已就位 |
| 凭据/存储层 | **50%** | `auth.Auth` 是「文件即凭据」，与 `auth-v2.dat`（safeStorage 加密）模型不同，需新增解密/写回 |
| 架构扩展点 | **90%** | `provider.Kind` + `Load*Dir` + `Runtime` 注册三步扩展点齐备 |
| 调度/保活 | **60%** | 有 `KeepaliveHours` 机制，但 QwenWork 的 refresh **轮换**特性与之冲突（见风险 1） |

综合：**约 85%**，算法与架构几乎白送，主要新增工作量在凭据解密与 token 轮换策略。

### 2.3 可直接复用的资产

1. **RSA 公钥**：`internal/qoder/cosy.go` 的 `serverPubKeyPEM` 原样可用。
2. **签名骨架**：`NewCosySession` / `AuthHeader` / `ApplyHeaders` 只需抽出「静态头集合 + 版本常量」为参数。
3. **SSE 剥壳**：`internal/qoder/sse.go` 的 `parseNestedSSE` 与两参考项目的 `unwrap_sse_payload` / `flushLine` 逻辑等价，QwenWork 响应 envelope 结构相同 → 直接复用。
4. **编码开关**：`internal/qoder/encoding.go` 的 `qoderEncode` 在 QwenWork 上**不需要**（明文即可），保留开关即可。
5. **渠道装配模板**：`internal/workbuddyai`（无签到、KeepaliveHours=nil、token 保活）是最贴近 QwenWork 的现成模板。

### 2.4 需要新增的工作

| # | 工作项 | 参考实现 | 备注 |
|---|---|---|---|
| 1 | `auth-v2.dat` 解密（Windows DPAPI + AES-256-GCM） | xrl `auth.ts` / Buddy2api `store.py` | wild-work 为 `CGO_ENABLED=0`，DPAPI 需用 `golang.org/x/sys/windows` 的 `CryptUnprotectData` 手写绑定（无需 cgo，可行） |
| 2 | macOS Keychain 解密（PBKDF2 1003 + saltysalt + AES-128-CBC IV=0x20） | xrl `auth.ts`（完整） | wild-work 跨平台，macOS 分支建议一并做；或先 Windows-only 并显式报错 |
| 3 | 写回 `auth-v2.dat`（对称加密 + 原子 replace + 首次 `.bak` + 保留未知字段） | Buddy2api `store.py::write_refreshed_auth` | 决定是否做「双向同步」，见风险 1 |
| 4 | 常量/静态头参数化 | 两项目 `constants.py` / `COSY_STATIC_HEADERS` | clienttype 6、`Cosy-Business-Product: qoder_work`、`Cosy-Scene: qwork`、`Cosy-MachineOS: x86_64_win32`、`User-Agent: qoderwork/0.1.8`、`Cosy-Version: 1.1.18` |
| 5 | body 策略定夺 | xrl 透传 / Buddy2api 重构造 | 建议**先照抄 xrl 透传**（最省事、已验证 200），失败再退到重构造 |
| 6 | 模型列表 | Buddy2api `models.py`（`GET /api/v2/model/list`，COSY GET，body 用空串 `""` 而非 `"{}"`，否则 403） | wild-work `models.go` 已有同构实现（Qoder 侧） |
| 7 | 额度 | Buddy2api `account-context` | 无稳定接口，建议 `UserResource` 返回 0 + `unit=unknown` 语义，避免污染面板加总 |
| 8 | 登录 | 可不做 | QwenWork 走 `auth-v2.dat` 导入比 OAuth 更省事（两参考项目均不做 OAuth，只做本机文件导入/监听） |
| 9 | 注册装配 | `cmd/wild-work/main.go` | `provider.QwenWork` + `LoadQwenWorkDir` + `Runtime` + scheduler（`KeepaliveHours` 见风险 1） |

预估新增代码：**约 400–600 行 Go**（含平台分支），加测试。

### 2.5 关键风险与决策点

1. **⚠️ refresh token 轮换互踩（最高优先级）**
   `deviceToken/refresh` 会**轮换** refresh_token。wild-work daemon 常驻 + `KeepaliveHours` 定时保活 → 与用户同时开着的千问办公 App 互相作废。
   - xrl 的解法：按需刷新（5min 缓冲，平均 1h 才刷 1 次）+ 写回 `auth-v2.dat` + `fs.watch` 监听 App 侧刷新 → 双向同步。
   - Buddy2api 的解法：按需刷新 + 刷新成功后写回 `auth-v2.dat`（带备份）。
   - **建议**：QwenWork 渠道设 `KeepaliveHours = nil`（对齐 workbuddyai），改为「每次 chat 前检查 `ExpiresAt`，临近过期才刷新」，并实现写回 `auth-v2.dat`。
2. **写回安全性**：必须保留 `auth-v2.dat` 中未知字段（`loginDeviceId`、`loginMethod`、`refreshStrategy` 等），先备份再原子替换；解密失败**禁止**写回（Buddy2api 明确约束）。
3. **`Cosy-MachineId` 取值**：xrl 用固定 `"unknown"`，Buddy2api 用 `loginDeviceId`。建议取 `loginDeviceId`（更贴近官方），缺失再退化。
4. **零 token 日志不变量**：新渠道需纳入 wild-work「日志/面板/消息框零 token」约束，`auth-v2.dat` 解密结果只允许落盘到受保护路径。
5. **平台限制**：DPAPI 绑定当前 Windows 用户，`auth-v2.dat` + `Local State` 拷贝到其他机器/账户无法解密 → 与 wild-work 跨平台定位有冲突，需在 UI/文档给出明确提示（Buddy2api README 就专门写了这条）。
6. **`x-model-key` 必须用 `qwork-*`**：Buddy2api 明令「禁止 `x-model-key: glm-5.2`」。wild-work 的 `channel/<model>` 前缀机制天然满足。
7. **`Encode` 开关别搞混**：Qoder 走 `qoderEncode`（base64 自定义字母表三段重排），QwenWork 走明文；两者不是同一套编码（xrl 观测的 0.1.3 是 ChaCha20+base91，后来被证明明文亦可）。
8. **签到**：两项目均确认无签到活动 → `DailyCheckin` 返回错误，并把 QwenWork 加入 `noExplicitCheckin` 白名单。

---

## 3. 结论

1. **逆向原理完全一致**：两个开源项目对千问办公的逆向结论（凭据加密、COSY 签名、明文 body、嵌套 SSE、模型档位、无签到）逐条吻合，RSA 公钥甚至与 wild-work 现有 Qoder 渠道**逐字节相同**，属于同一 COSY 框架的两个实例（QwenWork clienttype 6 / QoderWork CN clienttype 5）。
2. **差异只在版本常量与工程取舍**：`Cosy-Version`（1.0.47 vs 1.1.18）、body 构造策略（透传 vs 重构造）、token 生命周期管理（三源 fallback+文件监听 vs 入库+写回）、平台覆盖（mac+win vs win-only）。
3. **与 wild-work 契合度约 85%**：签名/协议/架构三层可直接复用，主要新增工作是 `auth-v2.dat` 的 DPAPI/Keychain 解密与写回、常量参数化、以及 token 轮换策略。
4. **最大风险不是算法，而是 refresh token 轮换互踩**：必须采用「按需刷新 + 写回 auth-v2.dat」而非 wild-work 现有的定时保活模式，否则会与千问办公 App 互相踢下线。
5. **推荐实现路径**：以 `internal/workbuddyai`（无签到、无定时保活）为模板新建 `internal/qwenwork`，签名复用 `internal/qoder/cosy.go` 骨架，body 先照抄 xrl 的「透传 + 补 request_id/session_id」，凭据解密先做 Windows DPAPI，macOS 分支可移植 xrl 的 Keychain 实现。

---

## 4. 参考资料

- `ref/xrl-router-plugin-qwenwork/docs/reverse/QWENWORKCN_REVERSE.md` — 千问办公逆向全文（含失败路径、`103 Duplicate` 死锁复盘、运行时内存提取公钥、safeStorage 解密）
- `ref/xrl-router-plugin-qwenwork/docs/specs/qwenwork-signing.md` / `qwenwork-token.md` / `qwenwork-forward.md` — 签名、token、转发规格与验收标准
- `ref/xrl-router-plugin-qwenwork/docs/DECISIONS.md` D-3 — 为什么改为自行管理 token（轮换互踩的原始记录）
- `ref/Buddy2api/docs/design/multi-channel-v2.md` Appendix A/B/C — 通道隔离设计 + QwenWork COSY 实现要点 + QoderWork CN 未决项
- `ref/Buddy2api/providers/qwenwork/{cosy,chat,token,store,models}.py` — Python 参考实现
- wild-work `internal/qoder/{cosy,client,sse,body,encoding}.go` — 现有 Qoder COSY 实现
