# FlashSale 一期实现任务清单

本文件是实现阶段的唯一执行清单。任务按依赖顺序推进；每次只处理当前最早的未完成任务。完成任务后填写验证结果，再进入下一项。

状态：`TODO` 待处理，`DOING` 处理中，`DONE` 已完成，`BLOCKED` 被明确阻塞。

## 执行规则

1. 开始任务前先把状态改为 `DOING`。
2. 任务完成必须运行对应验证命令，并记录结果。
3. 发现设计缺口时先暂停当前任务，在本文件增加说明，不绕过核心一致性问题。
4. 所有跨服务业务写入遵循本地事务 + Outbox + 消费幂等 + 补偿对账。
5. 开发环境使用 Docker；Java 21、Maven、Spring Boot 3。

## A. 工程与本地基础设施

| ID | 任务 | 前置 | 涉及内容 | 完成标准 | 验证 | 状态 |
|---|---|---|---|---|---|---|
| A01 | 创建 Maven 多模块工程骨架 | 无 | 根 `pom.xml`、服务模块、公共模块 | 所有模块可被 Maven 识别 | `mvn validate -DskipTests`：通过（11 个模块） | DONE |
| A02 | 建立 Java 21 Docker 开发构建环境 | A01 | `Dockerfile.dev`、Maven 缓存、启动说明 | 容器内可执行 `mvn test` | `docker build`、容器内 `mvn -B test`：通过 | DONE |
| A03 | 编排本地基础设施 | A02 | Docker Compose、PostgreSQL、Redis、RocketMQ、Nacos、XXL-Job、MinIO | 依赖服务健康可用 | `docker compose config`：通过；PostgreSQL/Redis 健康 | DONE |
| A04 | 接入数据库迁移 | A01 | Flyway、`db/001_initial_schema.sql` | 空库可重复初始化，迁移版本可追踪 | Java 21 容器执行两次：V1 成功，第二次无迁移；33 张表 | DONE |
| A05 | 建立统一配置与环境模板 | A02 | `.env.example`、配置分层、服务端口约定 | 本地/测试配置可区分 | `docker compose --env-file .env.example config`：通过 | DONE |

## B. 公共基础组件

| ID | 任务 | 前置 | 完成标准 | 验证 | 状态 |
|---|---|---|---|---|---|
| B01 | 统一 API 响应、分页和错误码 | A01 | 响应结构与 `docs/api-design.md` 一致 | Docker Java 21 容器测试：2 tests passed | DONE |
| B02 | Trace ID、结构化日志和异常处理 | B01 | HTTP/MQ/任务链路可关联 Trace ID | Docker Java 21 容器测试：编译与 2 tests passed | DONE |
| B03 | JWT、角色和资源归属校验 | B01 | CUSTOMER/OPERATOR/ADMIN 隔离生效 | Docker Java 21 容器测试：3 tests passed | DONE |
| B04 | PostgreSQL、Redis、RocketMQ 客户端封装 | A03 | 连接、超时、重试、序列化配置统一 | Docker Java 21 容器编译通过，Redis/RocketMQ 客户端依赖已统一 | DONE |
| B05 | Outbox 通用模型与投递接口 | B04 | 支持租约、退避、PENDING/SENT/FAILED | Docker Java 21 容器测试：6 tests passed；租约与退避接口已完成 | DONE |
| B06 | 消费幂等通用模型 | B04 | PROCESSING/SUCCEEDED/FAILED 和超时恢复可复用 | Docker Java 21 容器测试：8 tests passed；抢占与超时恢复接口已完成 | DONE |

## C. 认证、网关与账户

| ID | 任务 | 前置 | 完成标准 | 验证 | 状态 |
|---|---|---|---|---|---|
| C01 | Auth 服务注册/登录 | B01-B03,A04 | 注册只能创建 CUSTOMER；登录签发 JWT | Docker Java 21 容器测试：10 tests passed（公共 8 + Auth 2） | DONE |
| C02 | ADMIN 账户管理 | C01 | 角色、启停用、最后 ADMIN 保护和审计完整 | Docker Java 21 `mvn -B test -pl flashsale-auth -am`：通过 | DONE |
| C03 | Gateway 路由、鉴权与限流 | B02-B03,A03 | 路由、JWT、Sentinel、Trace 生效 | 公共运行时接线已完成；Docker Java 21 `mvn -B test`（11 模块）通过，待完成真实路由层验收 | DOING |

## D. 商品与优惠券

| ID | 任务 | 前置 | 完成标准 | 验证 | 状态 |
|---|---|---|---|---|---|
| D01 | Product 服务 CRUD 与上下架 | B01-B05,C01 | 商品规则、审计、缓存失效正确 | API/事务测试 | DONE |
| D02 | 商品库存与直接订单库存流水 | D01,B05 | 条件扣减、释放、不可超卖 | 并发测试 | DONE |
| D03 | Coupon 模板管理 | B01-B05,C01 | 创建、修改、暂停、恢复、审计正确 | API/事务测试 | DONE |
| D04 | 优惠券领取 Lua、幂等和 Outbox | D03,B05-B06 | 不超发、不重复领取、失败可补偿 | 并发/故障测试 | DONE |
| D05 | 用户券状态机与订单锁券/恢复 | D04 | AVAILABLE/RESERVED/CONSUMED/EXPIRED 合法迁移 | 状态机测试 | DONE |

## E. 活动与活动库存

| ID | 任务 | 前置 | 完成标准 | 验证 | 状态 |
|---|---|---|---|---|---|
| E01 | 活动创建、预热、开始、结束 | B01-B05,C01 | 活动基础状态和缓存正确 | Docker Java 21 `mvn -B -pl flashsale-activity -am test`：通过（19 tests） | DONE |
| E02 | 活动库存事件序号与账本 | E01,B05 | RESERVE/RELEASE 连续序号、Outbox 同事务提交 | Docker Java 21 `mvn -B -pl flashsale-activity -am test`：通过（19 tests） | DONE |
| E03 | 活动库存消费、流水和 checkpoint | E02,B06 | 重复消息不重复变更，checkpoint 不跨洞 | checkpoint 测试 | DONE |
| E04 | 活动 PAUSING/暂停屏障 | E02-E03 | 关闭新预扣、清算在途、建立 pause barrier；崩溃可恢复 | Docker Java 21 `mvn -B -pl flashsale-activity -am test`：通过（19 tests） | DONE |
| E05 | 异步恢复任务与 Redis 校验重建 | E04,B05-B06 | Outbox、checkpoint、账本、Redis 校验后才 ACTIVE；不一致可重建 | Docker Java 21 `mvn -B -pl flashsale-activity -am test`：通过（19 tests） | DONE |
| E06 | 活动查询、指标和运营接口 | E01-E05 | 对外查询不泄露内部处理中状态，指标可追踪 | Docker Java 21 `mvn -B -pl flashsale-activity -am test`：通过（19 tests） | DONE |

## F. 订单、支付与履约

| ID | 任务 | 前置 | 完成标准 | 验证 | 状态 |
|---|---|---|---|---|---|
| F01 | 订单预览与金额快照 | D01,D05,E01 | 只校验不占资源，金额快照明确 | 单元/API 测试 | DONE |
| F02 | 普通订单创建与幂等 | D02,B05-B06,F01 | 多商品全量预留或整体失败 | 并发/幂等测试 | DONE |
| F03 | 活动订单创建与限购 | E02,E03,F01 | 单活动单商品，Redis 预扣与 DB 账本一致 | 并发/故障测试 | DONE |
| F04 | 用户取消、超时释放 | D02,D05,E02,F02-F03 | CAS 只释放一次，活动 RELEASE 可追平 | 竞争/补偿测试 | DONE |
| F05 | Payment 支付幂等与回调 | F02-F04,B05-B06 | 重复支付/回调只生效一次 | 并发/回调测试 | DONE |
| F06 | 支付成功确认与履约 | F05,D05 | 库存确认、券核销、履约、支付事件同事务 | 事务/幂等测试 | DONE |
| F07 | 延时超时取消与 Job 兜底 | F04-F06,A03 | 支付与超时竞争得到唯一合法终态 | 延时/竞争测试 | DONE |

## G. 投递、补偿与运维

| ID | 任务 | 前置 | 完成标准 | 验证 | 状态 |
|---|---|---|---|---|---|
| G01 | 各服务 Outbox 投递器 | B05,D01,D04,E02,F04,F06 | 批量、租约、退避、重复投递可用 | 集成/故障测试 | DONE |
| G02 | 各消费者幂等处理器 | B06,D04,E03,F05-F07 | 重复、失败、PROCESSING 超时可恢复 | MQ 集成测试 | DONE |
| G03 | XXL-Job 对账与补偿 | E03-E05,D04-D05,F04,F07 | Redis/DB/账本/Outbox/订单状态可对账补偿 | 故障演练 | DONE |
| G04 | 监控、指标、日志和告警 | B02,G01-G03 | 覆盖延迟、积压、库存不一致、死信、补偿失败 | 指标检查 | DONE |
| G05 | Docker 一键启动与初始化 | A03,A04,G04 | 新环境可一键启动并完成健康检查 | `docker compose up` | DONE |

## H. 验收测试

| ID | 任务 | 前置 | 完成标准 | 验证 | 状态 |
|---|---|---|---|---|---|
| H01 | 全量 HTTP 接口契约与权限验收 | C01-C03,D01-D05,E01-E06,F01-F07,G04 | Auth、商品、优惠券、活动、订单、支付、库存和运营接口逐项符合 `api-design.md`：路径/方法、请求响应字段、分页、错误码、JWT 角色、资源归属、幂等键和 Trace ID 一致；未授权、越权、重复请求和非法参数均返回规定结果 | 2026-09-25～2026-09-26：按 `docker-compose.yml` + `docker-compose.acceptance.yml --profile r21` 分层执行验收，测试完成后已停止本轮启动服务，仅保留预先存在的独立 `redis` 容器。已验证且当前无异常：Compose/Flyway；Auth 注册、登录、JWT、Trace；Product CRUD/上下架/查询；Coupon 创建、规则限制、暂停/恢复、领取、重复领取幂等；Activity 创建、取消/重复取消、指标、预热、开始、暂停/重复暂停、恢复请求/查询；Order 预览、创建、列表、详情、明细、重复与冲突幂等、取消/重复取消；Payment 成功支付、重复支付幂等、回调及回调幂等、金额/币种异常；Inventory 运营 JWT、未授权、非法数量、资源不存在，以及 acceptance seed `INVENTORY_ACCEPTANCE_SEED=9001=10` 下预占/重复预占、确认/重复确认、已确认释放错误、释放/重复释放；CUSTOMER/OPERATOR/ADMIN 权限隔离和 Trace ID。已验证库存活动链路：Redis 库存从 9 减至 8 后经取消释放恢复至 8；活动库存事件 payload、Outbox、消费者幂等和 checkpoint 推进测试通过；活动 5 暂停后屏障为 `3`，延迟启用 Outbox/Activity Consumer 后 3 条事件连续消费至 checkpoint `3`，幂等记录均为 `SUCCEEDED`，未发现库存流水、账本、Redis 不一致。相关修复已完成。不能严格保证/尚未完成：没有完成一次“存在未消费库存事件且 barrier>0 → 暂停 → 追平 → 恢复成功 → 最终库存对账”的真实 HTTP 闭环；活动 5 后续恢复因时间窗口过期返回 `ACTIVITY_NOT_READY`，不是库存校验失败；活动 7 的恢复任务 `id=4` 虽为 `SUCCEEDED` 且活动回到 `ACTIVE`，但 `barrier=0`，只证明无待处理事件时恢复成功。该边缘场景暂不继续处理，保留为 H01 未完成项；详细补做方案见末尾“H01 活动恢复边缘场景补做方案”。 | DOING |
| H02 | 核心状态机、数据库不变量与跨服务一致性验收 | A01-G05,H01 | 订单、支付、用户券、活动、库存、Outbox、消费幂等和补偿记录只能发生设计规定的状态迁移；库存不得为负，库存流水/活动事件序号连续，订单和支付终态唯一，用户券不得重复核销，Outbox/消费记录与业务变更满足同事务约束；检查 Redis、PostgreSQL、RocketMQ 最终状态一致 | 2026-09-26：已完成设计定向 review；Docker Java 21 串行分层测试通过：common、inventory、coupon、order、payment、activity、job；临时 PostgreSQL（V1→V3）验证订单创建/取消、活动事件序号、支付幂等与履约、Outbox/消费幂等；临时 Redis 验证活动库存预扣/暂停门闸、优惠券并发限购与补偿；全量 `mvn -B test -q` 和 Compose config 通过。首次并行 Maven 触发公共模块 Surefire discovery 失败，串行重跑通过；首次优惠券 Redis 集成因既有容器未发布端口失败，改用临时端口 Redis 重跑通过。测试容器及临时 PostgreSQL/Redis 已关闭，仅保留预先存在的独立 `redis` 容器。H01 的 `barrier > 0` 暂停→追平→恢复真实 HTTP 闭环仍未完成，故 H02 保持 `DOING`，不标记完成。 | DOING |
| H03 | 高并发、幂等与容量压测 | D04,E02-E03,F02-F04,G01-G04,H02 | 覆盖活动热点库存、同用户限购、优惠券领取、重复提交、支付/取消竞争和 MQ 消费；核心正确性 100% 满足 AC01-AC08，零负库存、零重复有效订单/券/支付；报告记录并发量、持续时间、吞吐、P95/P99、错误率、Outbox 积压和消费延迟，并与发布基线比较 | DOING（2026-09-26 更新）：活动热点、限购、重复提交、Outbox 调度和基础优惠券已验证；`500/s × 20s` 完成 10,001 请求且 0 dropped iterations；仍缺多用户优惠券容量、支付最终状态明细、完整资源/MQ/Redis/数据库报告，以及 1 秒 10,000 请求瞬时突发的证据 | DOING |
| H04 | 故障恢复、补偿与告警演练 | G01-G04,H02 | 覆盖服务/数据库重启、Outbox 投递中断、消费者崩溃、重复/乱序/延迟消息、RocketMQ 重投、Redis 预扣丢失、XXL-Job 重复触发和补偿失败；恢复后无数据不变量破坏，任务可重试且幂等，Redis 可由 PostgreSQL/事件账本安全重建，失败进入告警和人工处理记录 | Docker Java 21 按演练脚本注入故障并记录故障时间、恢复时间、重试次数、告警、补偿记录和最终对账结果；每个场景完成后关闭测试服务 | TODO |
| H05 | 从零部署、全链路冒烟与发布验收 | A03-A05,G05,H01-H04 | 使用干净 Docker 环境完成镜像构建、Flyway 初始化、Compose 健康检查和全部服务启动；完成注册/登录、浏览、领券、下单、支付、履约、取消/超时和运营查询冒烟；Prometheus、日志、Trace、告警可查询；重启后数据和任务可恢复；验收结束清理容器和临时数据 | Docker Java 21 执行一键部署及发布验收脚本；保存版本、配置、健康检查、端到端结果、监控证据和清理记录 | TODO |

### H02 不变量—实现—测试—证据矩阵（2026-09-26）

| 不变量 | 实现依据 | 测试/证据 | 结果 |
|---|---|---|---|
| 合法状态迁移且终态唯一 | 订单/支付/用户券/库存状态机、数据库 CAS/触发器 | `flashsale-order`、`flashsale-payment`、`flashsale-coupon`、`flashsale-inventory` 模块测试；订单/支付 PostgreSQL 集成测试 | 通过 |
| Outbox 与业务写入同事务 | 各服务 JDBC Store 与私有 Outbox | 订单创建/取消、支付集成测试；`JdbcOutboxPortIntegrationTest` | 通过 |
| 消费幂等与业务变更同事务 | 消费幂等记录 `PROCESSING/SUCCEEDED/FAILED` | `JdbcConsumerMessageHandlerIntegrationTest`、活动消费者测试 | 通过 |
| 活动事件序号连续、checkpoint 不跨洞 | sequence 锁、事件账本、checkpoint 推进 | 活动模块测试；订单活动创建/取消 PostgreSQL 集成测试 | 通过 |
| Redis 只作为实时入口，恢复不覆盖已有键 | Lua 门闸/预扣、恢复互斥与缺失键重建 | 活动 Redis 测试、订单 Redis 集成测试、恢复验证测试 | 通过（真实 H01 闭环仍待补做） |
| 失败保持安全状态并留痕 | 补偿记录、失败状态、告警路径 | 任务补偿测试、Outbox 重试测试、活动恢复失败测试 | 通过 |
| Redis/PostgreSQL/MQ 最终对账 | 对账任务、库存流水/账本/Outbox/checkpoint | 临时 PostgreSQL/Redis 集成验证；全量 Maven 回归 | 通过（RocketMQ 真实 HTTP 闭环依赖 H01） |

H 阶段按 `H01 → H02 → H03 → H04 → H05` 顺序执行。每项开始前必须声明测试环境、数据集和通过阈值；每项完成后记录命令、结果、证据链接和环境清理情况。任何核心正确性、不变量、恢复或部署验收失败，都保持当前任务为 `DOING`，不得进入下一项。

## 当前执行位置

- 当前任务：`H01`
  - 状态：`DOING`
  - 备注：进入全量接口契约验收。

- 并行准备任务：`H03`
  - 状态：`DOING`
  - 备注：按已确认范围先实现 k6 压测工件；活动暂停/恢复不纳入 H03，本地实际压测和数据对账待 k6 镜像可用后执行。

### H03 最新验证状态（2026-11-02）

**已完成**：

- 活动预热并进入 `ACTIVE`；活动热点库存压测完成 `500/s × 20s`，10,001 请求、0 dropped iterations，P50 4.0ms、P95 81ms，活动库存 300、成交 300 单。
- 活动库存不变量、同用户限购、同键幂等和冲突幂等通过。
- 六个 `outboxDispatch` 每秒任务已自动初始化，六个 executor 注册成功；历史 Outbox 已从 `PENDING` 追平为 `SENT`。
- 基础优惠券领取通过；单用户限领只生成 1 张用户券，Coupon Outbox 为 `SENT`。
- `load/h03/verify.sql` 全部检查为 0。

**未完成**：

- 单机 `1s × 10,000` 瞬时突发未验证：本次只验证 500 QPS 持续 20 秒、累计 10,001 次请求。
- 数据库库存投影为异步追平，压力结束后存在短暂滞后（500 QPS 场景记录时 checkpoint 256/300），不作为实时强一致结论。
- 优惠券多用户容量、支付/取消最终状态明细、完整 MQ/Redis/数据库/资源报告证据仍需补齐。
  - 运行环境修复（2026-09-26）：H03 Compose 临时 XXL-Job MySQL 现在幂等预置六个业务服务的 `outboxDispatch` 每秒任务，并统一 Admin/executor 的 `default_token`；已实测六个 executor 注册成功，历史 Outbox 从 `PENDING` 追平为 `SENT`。随后补测 ACTIVE 优惠券领取：单用户限领只生成 1 张用户券，Coupon Outbox 为 `SENT`；补测支付/取消竞争场景并保留 k6 结果。H03 仍因优惠券多用户容量证据、支付最终状态明细、完整 MQ/Redis/数据库对账和目标突发窗口证据不足保持 `DOING`。

