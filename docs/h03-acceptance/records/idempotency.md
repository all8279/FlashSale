# 幂等重放

## 场景

- 场景：`duplicate_order`，100 个 CUSTOMER 并发重复提交**同一个幂等键**，持续 30 秒
- 幂等键：`ORDER_SUBMIT_duplicate_shared`（所有 VU 共用）
- 请求体：`kind=DIRECT`，商品 `product_id=6`（普通购买，不受活动库存限制）
- 执行时间：2026-11-01，本机 Docker 环境
- 原始结果：`docs/h03-acceptance/evidence/k6/duplicate-order-summary-k6.json`

## k6 结果

| 指标 | 数值 |
|---|---:|
| 实际请求 | 82,453 |
| 实际速率 | 2,747.5 req/s |
| dropped iterations | 0 |
| P50 | 31.9 ms |
| P95 | 71.1 ms |
| max | 515.0 ms |

## 数据库结果

| 检查 | 数值 |
|---|---:|
| `order_submission_idempotency` 行数 | 100 |
| 不同 `user_id` | 100 |
| 不同 `order_id` | 100 |
| `status=SUCCEEDED` | 100 |
| `duplicate_successful_orders` | 0 |

## 结论

82,453 次重复提交最终只生成 100 笔订单（每个用户 1 笔），重复请求返回首次结果，
未产生重复扣减；七项对账中 `duplicate_successful_orders` 为 0。

## 执行工件

- [`load/k6/h03.js`](../../../load/k6/h03.js)（`duplicateOrder()`）
- [`load/h03/verify.sql`](../../../load/h03/verify.sql)
