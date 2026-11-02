# H03 高并发验收记录

## 实测结论（2026-11-02，本机）

- **500 QPS × 20 秒**：10,001 次请求，dropped 0，P50 4.0 ms，P95 81 ms；300 件库存成交 300 单，活动 Redis 库存归零。
- **幂等重放**：82,453 次同键重复提交只生成 100 笔订单，重复有效订单 0。
- **暂停 / 恢复屏障**：暂停期新预扣全部拒绝，恢复后继续成交；12 笔订单下 Redis 288 = PostgreSQL 288，事件序号 1–12 连续。
- **七项 SQL 对账**：负库存、重复有效订单、事件断号、Outbox 积压、消费幂等未完成、无订单的预留，全部 0 违规。

## 证据索引

- [500 QPS × 20 秒记录](records/500qps-x20s.md)
- [幂等重放记录](records/idempotency.md)
- [最终对账记录](records/reconciliation.md)
- [活动暂停 / 恢复屏障记录](records/pause-resume.md)
- [k6 原始 summary 输出](evidence/k6/)
- 执行工件：[`load/h03/run.sh`](../../load/h03/run.sh)、[`load/k6/h03.js`](../../load/k6/h03.js)、[`load/h03/verify.sql`](../../load/h03/verify.sql)

## 原始证据

`evidence/k6/` 保留实际 k6 生成的 `summary-k6.json`，包含 iterations、http requests、dropped iterations
和延迟分位数，可用相同脚本在本地复现。

## 结论边界

- 本机为 16 GB 内存、25 个容器的单机环境，结论只覆盖上述场景。
- 500 QPS × 20 秒是持续速率下的 10,001 次请求，不等同于“1 秒内 10,000 次瞬时突发”。
- 数据库库存投影为异步追平，压力结束后存在短暂滞后；500 QPS 场景的结论不包含“数据库与 Redis 实时强一致”。
- 暂停 / 恢复场景使用串行请求，只验证状态机与账本，不代表吞吐能力。
