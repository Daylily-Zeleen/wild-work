# wild-work v2.2.1 — 积分显示修正：可用/不可用拆分 + 有效期 + 明细翻页

> 本版聚焦「积分到底有多少能用」这一件事，修正 TraeWork 可用余额虚高与面板数字误导。

## 修复

### 本地 token 看似有效、上游已拒绝 → 永久卡在 401（重要）

**现象**：TraeWork 账号积分恒为 0，hover 不弹明细（接口 400）。

**根因**：刷新时上游会**作废旧 access token**（refresh token 同时轮换）。若新 token 未落盘，
或同一账号在另一实例/客户端上被刷新过，本地文件里就是**已被作废、但 `expiresAt` 仍在未来**的 token：

```
本地 expiresAt = 2026-09-30 22:33（看起来还有 12 天）
NeedsRefresh(10min) → false  ← 永远不会去刷新
上游实际返回     → 401      ← 但 token 早已被作废
```

于是 `NeedsRefresh` 永远为假、永不重试，积分/明细永久为空。

**修复**：不再只信任本地过期时间，对 401 本身做一次「refresh + 重试」，成功后写回磁盘。
覆盖三条路径：积分刷新（`creditTotals`）、单账号刷新（`RefreshCredits`）、
明细查询（`ResourceDetail`）、批量刷新（`RefreshAll`）、费率拉取（`RefreshPricing`，
原先连 refresh 结果都没落盘）。

**验证**（把 `accessToken` 改坏、保留有效 `refreshToken` 后启动）：

```
session dead, refreshing platform=traework uid=1096660468371514
traework refresh success uid=1096660468371514 refresh_rotated=true expires_at=1790917166
→ 积分 2710/不可用 2600，明细 29 条正常，新 token 已写回磁盘
```

> 另外：`RefreshPricing` 原先调 `RefreshToken` 后**没有** `SaveAtomic`——
> refresh token 已轮换却不落盘，是造成上述「本地看似有效」的典型场景，已一并补上。

### TraeWork 可消耗余额虚高（重要）

早期实现用 `group_type != 1` 判定可消耗额度，把 `available_endpoint=1`
（官方客户端专用池）的「用户福利」「签到奖励」也计入了可消耗余额。

**实测证据**（2026-09-18，三账号各做一次 `glm-5.2` 对话后对比用量）：

```
账号 3066985700146732（对话前 → 对话后）
  gt=1 ep=0 每日签到   used 420.7852 → 422.1424  (+1.3568)  ← 本工具只扣这里
  gt=1 ep=1 每日签到   used   0.0000 →   0.0000  (不动)
  gt=4 ep=1 用户福利   used   0.0000 →   0.0000  (不动)
```

判据已修正为 **`available_endpoint == 0`**。影响：

- `pool.Pick()` 不再按虚高余额选号（原先可能选中一个实际可用余额已耗尽的账号）
- 签到后解冻判定（`ReenableIfCredits`）同样只看可用池，不再被专用池额度误放行

### 总分与分项「对不上」的澄清

不是计算错误。分项合计与 `usage_summary` 严格自洽：

| 账号 | Σlimit | total_amount | Σused | consumed_amount |
|------|--------|--------------|-------|-----------------|
| 1096660468371514 | 5350 | 5350 | 40.4864 | 40.49 |
| 1342951362670730 | 9750 | 9750 | 3675.0232 | 3675.02 |
| 3066985700146732 | 9150 | 9150 | 2920.7852 | 2920.79 |

`used` 的小数尾差来自上游 `consumed_amount` 只保留两位小数。真正的坑是
**`total_amount` 是含 ep=1 专用池的总量**，把它当可用余额就会得出「账号有 9150 积分」的错觉
（该账号实际可用仅 1828）。现已把两者分开显示。

### 到期时间

各渠道字段不同，统一解析为 `YYYY-MM-DD` 并按 UTC+8 墙钟处理：

| 渠道 | 字段 | 说明 |
|------|------|------|
| WorkBuddy / 国际版 | `CycleEndTime` | **上游从不下发 `PackageEndTime`**（旧判据恒 miss） |
| TraeWork | `expire_time` | Unix 秒 |
| Qoder | 无 | 不显示有效期列 |

上游未下发时该列整体隐藏，不会出现一列空白或用零值冒充「永不过期」。

## 改进

### 面板：积分数字拆成「可用 / 不可用」

- 账号卡片：`1828可用积分/4400不可用`（不可用额度降级为次要色）
- 明细 tooltip 底部：`可用 1828　不可用 4400`（无不可用额度时只显示一个数字，不制造无意义的 0）
- 明细表中不可用额度整行淡显 + 「不可用」角标

### 面板：明细分页与有效期列

- 每页 8 条，底部 `‹ 1 / 4 ›` 翻页；翻页只重绘不重新请求（响应已缓存）
- 新增「有效期」列（仅当上游确实下发到期时间时出现）
- 明细缓存随 `loadState()` 失效，避免刷新后 tooltip 仍显示旧余额
- 限额为 0 的包（如 TraeWork 的免费 0 限额包）不再出现在明细里

## 内部改动

- `provider.ResourceItem` 增加 `ExpireAt` / `Usable`；新增 `provider.Summarize()` 统一汇总小计
- `pool.Status` / `AccountView` 增加 `UnusableCredits`，随 `state-*.json` 持久化（新增字段，向后兼容）
- `pool.ReenableIfCredits(uid, remain, unusable)` 签名变更
- 积分刷新路径改走 `UserResourceDetail` 单次请求，同时拿到可用余额与小计（原先只调 `UserResource`）
- 三渠道 `UserResourceDetail` 增加 `Usable: true`（国内版/国际版/Qoder 无端点分区）

## 升级说明

无需迁移。旧 `state-*.json` 缺 `unusable` 字段时按 0 处理；启动后首次刷新积分即会填上真实值。
