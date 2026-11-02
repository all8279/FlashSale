# 活动暂停 / 恢复屏障

## 场景

- 场景：活动进行中暂停 → 暂停期间拒绝新预扣 → 请求异步恢复 → 恢复后继续下单
- 活动：`activity_id=2`，初始库存 300，`purchase_limit_per_user=2`
- 请求方式：串行 HTTP 调用（非 k6 并发），用于验证状态机与账本，不代表吞吐能力
- 执行工件：[`load/h03/verify.sql`](../../../load/h03/verify.sql)、[`load/h03/xxl-job-init.sql`](../../../load/h03/xxl-job-init.sql)

## 修复前后对比

| 项目 | 修复前 | 修复后 |
|---|---|---|
| 恢复任务调度 | 未注册 `activityRecovery`，`resume` 永久 `PENDING` | H03 初始化脚本注册 `activityRecovery`（5 秒调度） |
| 恢复后的 Redis 门闸 | `gate=CLOSED` 永久不重开，恢复后所有下单被拒 | `gate=RESERVE`、`status=ACTIVE`，恢复后下单成功 |

根因：`ENSURE_RECOVERY_PROJECTION` 在库存键存在时提前返回，只补详情键，未把门闸改回 `RESERVE`。修复点为 [`RedisActivityInventory`](../../../flashsale-activity/src/main/java/com/flashsale/activity/RedisActivityInventory.java)。

## 修复后实测（第二轮）

| 阶段 | 动作 | 结果 |
|---|---|---|
| baseline | — | DB 294 / Redis 294 / 订单 6 |
| 暂停前 | 3 个用户各下 1 单 | 3 单成功，Redis 294 → 291 |
| 暂停前 | 同键重放 | 返回同一订单号，库存不变 |
| 暂停 | `POST /pause` | `PAUSED`，`gate=CLOSED`，`pause_barrier_sequence=9` |
| 暂停中 | 3 个新用户下单 | 全部 `ACTIVITY_NOT_FOUND`，库存不变 |
| 恢复（第 1 次） | `POST /resume` | `FAILED: CHECKPOINT_GAP`（屏障 9 > 检查点 6），活动保持 `PAUSED` |
| 恢复（第 2 次） | 检查点追平后再次 `POST /resume` | `SUCCEEDED`，屏障 9，`gate=RESERVE` 自动重开 |
| 恢复后 | 同一 3 个用户再各下 1 单 | 3 单成功（每人 2 单，正好打满限购），Redis 291 → 288 |
| 恢复后 | 同键重放 | 返回同一订单号，库存不变 |
| 终态 | — | DB 288 = Redis 288，订单 12 笔 |

事件账本：`activity_inventory_event` 12 条，序号 1–12 连续无空洞；`activity_inventory_checkpoint.last_contiguous_sequence=12`，与最大序号一致。

## PostgreSQL 对账结果

| 检查项 | 结果 |
|---|---:|
| `negative_product_stock` | 0 |
| `negative_activity_stock` | 0 |
| `duplicate_successful_orders` | 0 |
| `activity_event_sequence_gaps` | 0 |
| `outbox_not_sent` | 0 |
| `consumer_not_succeeded` | 0 |
| `reservation_without_order` | 0 |

## 结论边界

- 恢复任务首次 `FAILED: CHECKPOINT_GAP` 是设计行为：屏障大于已追平检查点即失败保持 `PAUSED`，由下一次恢复任务重新截取并验证；本记录通过第二次 `resume` 完成恢复。
- 第一轮（修复前）依赖手工 `redis-cli set activity:2:gate RESERVE` 才能完成恢复后下单，其数据仅用于定位缺陷，不作为验收依据。
- 本轮为串行请求，不构成并发能力结论；并发结论见 [500 QPS × 20 秒](500qps-x20s.md) 与 [1000 QPS × 10 秒](1000qps-x10s.md)。
