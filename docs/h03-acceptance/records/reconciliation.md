# H03 最终对账记录

执行时间：2026-11-02，在 500 QPS × 20 秒场景结束、活动售罄后执行。

执行方式：

```text
psql -U flashsale -d flashsale -f load/h03/verify.sql
```

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

## 说明

- 六个业务服务均注册 `outboxDispatch` executor，H03 临时 XXL-Job MySQL 幂等创建调度任务。
- 对账检查的是约束与不变量（负库存、重复有效订单、事件断号、Outbox 积压、消费失败、孤立预留），
  不检查库存投影的追平进度。
- 数据库库存投影为异步追平，对账时允许存在短暂滞后；滞后不影响上述 7 项结果。

## 执行工件

- [`load/h03/verify.sql`](../../../load/h03/verify.sql)
