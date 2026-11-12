# StarRocks 整体架构：FE、BE、CN 的职责

> 借 StarRocks 理解现代分布式 OLAP 数据库设计思想 · What 02/15 · 2026-11-12 · 审阅稿  
> 面向十年 Java/MySQL 经验工程师。英文官方文档 Latest（2026-11-12 查阅），实际部署版本待核实。  
> F：官方事实；H：条件性推断/待验证；O：真实观测（本篇暂无）。

## 1. 问题引入：谁真正执行订单日报？

物流订单业务库承担订单创建、状态修改和点查；StarRocks 中保存一份去重后的订单快照。运营人员查询：

~~~sql
SELECT order_day, site_id, COUNT(*) AS order_count
FROM orders
WHERE order_day >= '2026-11-06'
  AND order_day < '2026-11-07'
GROUP BY order_day, site_id;
~~~

上一期解释了为什么事务查询与大规模聚合是不同负载。本期要回答：**谁接收 SQL、谁决定数据在哪里、谁实际扫描和聚合、谁在故障时承担协调？**

StarRocks 的 FE 与 BE/CN 不是 MySQL 主从复制关系，也不是必须串联的三层组件。它们代表**元数据/规划/协调**与**数据存储/计算执行**之间的职责分工。

## 2. 概念定位：FE、BE、CN 分别负责什么？

**[F]** StarRocks 官方文档定义两种典型部署：shared-nothing（存算一体）使用 FE + BE；shared-data（存算分离）使用 FE + CN，持久数据放在对象存储或 HDFS。[Architecture](https://docs.starrocks.io/docs/introduction/Architecture/)。

| 组件 | 主要职责 | 不是 | 典型资源/风险 |
| --- | --- | --- | --- |
| FE（Frontend） | 连接管理、元数据、SQL 解析/规划、任务协调与调度 | 扫描全部订单的 Worker | 元数据日志、选主、规划 CPU、客户端入口 |
| BE（Backend） | shared-nothing 的本地持久存储、导入处理、索引与查询执行 | 只负责备份的被动副本 | 本地磁盘、CPU、内存、Tablet、副本 |
| CN（Compute Node） | shared-data 的查询执行、远端数据访问、本地热数据缓存 | 完全不使用本地磁盘 | CPU、内存、缓存、网络、远端存储 |
| 对象存储/HDFS | shared-data 的持久数据层 | FE/CN 本身 | 延迟、吞吐、权限、可用性与费用 |

**[H]** 用控制平面与数据平面理解上述分工很有帮助，但不是绝对隔离：FE 仍参与结果协调，执行节点也有状态和管理职责。

## 3. 架构图：两种部署不要画成一条流水线

### 3.1 存算一体：FE + BE

~~~mermaid
flowchart TB
  API["报表 API / MySQL 客户端"] -->|"SQL"| FE["FE：元数据、规划、调度"]
  FE -.->|"执行任务"| BE1
  FE -.->|"执行任务"| BE2
  subgraph Data["Shared-nothing：执行与本地存储"]
    BE1["BE 1：扫描/聚合"] --- D1[("本地 Tablet 副本")]
    BE2["BE 2：扫描/聚合"] --- D2[("本地 Tablet 副本")]
    BE1 <-->|"必要时交换中间数据"| BE2
  end
  BE1 -.->|"结果/状态（简化）"| FE
  FE --> API
~~~

**[F]** FE 构造执行计划，BE 存储数据、生成索引并执行查询。本地持久数据有助于降低部分远程访问成本。数据副本提高可靠性，但副本健康必须实际核对。[Architecture](https://docs.starrocks.io/docs/introduction/Architecture/)。

**[H]** 图中的结果汇聚是概念简化，不能断言所有查询都由 BE 1 完成最终聚合，也不能断言每条 SQL 都发生跨节点交换。具体由物理执行计划决定。

### 3.2 存算分离：FE + CN + 共享存储

~~~mermaid
flowchart TB
  API["报表 API / MySQL 客户端"] -->|"SQL"| FE["FE：元数据、规划、调度"]
  FE -.->|"执行任务"| CN1
  FE -.->|"执行任务"| CN2
  subgraph Compute["Shared-data：计算与缓存"]
    CN1["CN 1：执行 + 本地缓存"]
    CN2["CN 2：执行 + 本地缓存"]
    CN1 <-->|"必要时交换数据"| CN2
  end
  CN1 <-->|"缓存未命中时远程读取"| Store[("对象存储 / HDFS：持久数据")]
  CN2 <-->|"缓存未命中时远程读取"| Store
  CN1 -.->|"结果/状态（简化）"| FE
  FE --> API
~~~

**[F]** CN 执行查询并缓存热数据，持久数据在共享存储；扩缩 CN 不需要像 BE 那样搬迁本地持久副本。[Architecture](https://docs.starrocks.io/docs/introduction/Architecture/)、[Deploy Shared-data](https://docs.starrocks.io/docs/deployment/deploy_shared_data_manually/)。

**[H]** 远端读取延迟、带宽、缓存命中率和对象存储费用可能成为瓶颈；热缓存表现与冷缓存不同。“shared-data 一定更慢/更便宜”都需要证据。

**错误画法**：Client → FE → BE → CN → 存储。正确画法是 **FE + BE** 或 **FE + CN + 共享存储** 两种部署边界，不能把两种模式的执行节点强行串联。

## 4. FE 内部：Leader、Follower、Observer 与元数据多数派

**[F]** 官方架构文档描述 FE 使用 BDB JE 管理元数据；Leader 处理元数据写入，Follower 同步日志并参与 Leader 选举，Observer 同步元数据但不参与选举。选主依赖 Follower 多数派。官方推荐典型高可用模式为三个 Follower（包括 Leader），必要时添加 Observer 扩展查询入口。[Architecture](https://docs.starrocks.io/docs/introduction/Architecture/)、[Scale in and out](https://docs.starrocks.io/docs/administration/management/Scale_up_down/)。

~~~mermaid
flowchart LR
  Client["SQL / DDL 客户端"] --> Leader["FE Leader：元数据写入"]
  Leader -->|"元数据日志"| F1["FE Follower：参与选举"]
  Leader -->|"元数据日志"| F2["FE Follower：参与选举"]
  Leader -->|"同步元数据"| Observer["FE Observer：不参与选举"]
  F1 -.->|"Leader 故障后参与新选举"| F2
~~~

这里的元数据复制与 BE 的订单数据副本是**不同层次**：即使 BE 的订单数据完整，FE 失去多数派也可能影响元数据操作、查询规划或调度。Observer 不计入选举多数派，增加 Observer 不能代替增加 Follower。

对 Java 工程师可类比：FE 像“元数据目录 + SQL 编译器 + 分布式任务协调者”，BE/CN 像执行任务的 Worker；但 FE 不是普通无状态网关，BE/CN 也不是简单线程池。

## 5. 订单 SQL 的控制路径与数据路径

~~~mermaid
sequenceDiagram
  participant API as 报表 API
  participant FE as FE
  participant A as BE/CN A
  participant B as BE/CN B
  participant Storage as 本地盘或共享存储
  API->>FE: 提交 GROUP BY SQL
  FE->>FE: 解析、优化、计划与调度
  FE-->>A: 下发执行任务
  FE-->>B: 下发执行任务
  A->>Storage: 读取负责的数据
  B->>Storage: 读取负责的数据
  Storage-->>A: 数据块
  Storage-->>B: 数据块
  A->>A: 过滤、局部聚合
  B->>B: 过滤、局部聚合
  opt 物理计划需要交换
    B-->>A: 中间聚合结果
  end
  A-->>FE: 结果/状态（简化）
  FE-->>API: 返回结果
~~~

这是**概念级**顺序图，不是精确到 RPC 的源码流程。四个要点：

1. **元数据在哪里？** FE 维护表、分区、节点等相关元数据，辅助计划与调度；Tablet、副本的细节留第 04、05 篇。
2. **谁执行算子？** BE/CN 执行扫描、过滤和聚合；Fragment/Pipeline/Exchange 留第 03、07 篇。
3. **从哪里读取字节？** BE 主要读取本地持久数据；CN 先利用缓存，必要时访问远端存储。
4. **结果是否正确且够新？** 订单去重、取消和导入可见水位属于业务/数据一致性问题，组件职责不能自动保证，留第 10、11 篇。

## 6. 两组设计取舍与决策边界

### 6.1 单 FE 与多 FE

| 选择 | 收益 | 代价 | 边界 |
| --- | --- | --- | --- |
| 单 FE | 简单、资源消耗低 | 元数据与查询入口有显著单点风险 | 教学、本地实验 |
| 多 Follower FE，按需 Observer | 选举多数派、冗余与读扩展 | 复制、运维、选主期间扰动 | 有明确 HA 要求的环境 |

**[H]** 即使 BE 数据副本完整，单 FE 故障仍可能让新 SQL 无法进入、规划或协调。**数据不丢失不等于查询服务可用**。多 FE 还必须检查客户端是否只连一个固定 FE 地址。

### 6.2 BE 本地存储与 CN + 共享存储

| 选择 | 收益 | 代价 | 边界 |
| --- | --- | --- | --- |
| FE + BE | 本地数据访问，热数据延迟有优势 | 存储与计算资源耦合，扩容可能涉及数据迁移 | 稳定、低延迟负载 |
| FE + CN + 共享存储 | 计算与存储独立扩展 | 远端 I/O、缓存管理、外部存储费用与故障域 | 峰谷明显、弹性需求强 |

**[F]** 官方 shared-data 功能矩阵说明不同版本支持的特性并不完全相同，且不支持两种模式混合部署或直接互相转换。[Feature Support](https://docs.starrocks.io/docs/deployment/feature-support-shared-data/)。

**[H]** 对订单日报，如果每天有突出的计算峰值，shared-data 值得评估；若反复读取相同热数据且极度延迟敏感，shared-nothing 值得优先比较。这里是**待验证选型假设**，完整成本分析留第 15 篇。

## 7. 可复现验证：从节点状态证明架构

### 7.1 记录环境与节点清单

准备 StarRocks 测试集群，记录：版本、部署模式、FE/BE/CN 数量、CPU/内存、本地盘、对象存储、缓存配置和客户端连接方式。使用具备 SYSTEM OPERATE 权限或 cluster_admin 角色的账号：

~~~sql
SHOW FRONTENDS;
SHOW BACKENDS;
SHOW COMPUTE NODES;
SHOW PROC '/frontends';
SHOW PROC '/backends';
SHOW PROC '/compute_nodes';
~~~

官方文档：[SHOW FRONTENDS](https://docs.starrocks.io/docs/sql-reference/sql-statements/cluster-management/nodes_processes/SHOW_FRONTENDS/)、[SHOW BACKENDS](https://docs.starrocks.io/docs/sql-reference/sql-statements/cluster-management/nodes_processes/SHOW_BACKENDS/)、[SHOW COMPUTE NODES](https://docs.starrocks.io/docs/sql-reference/sql-statements/cluster-management/nodes_processes/SHOW_COMPUTE_NODES/)。

**观察指标**：FE 的 Role、Alive、ReplayedJournalId、Version；BE 的 TabletNum、容量、Alive；CN 的 Alive、CPU、运行查询数和缓存状态。**判断标准**：shared-nothing 应能看到 BE，shared-data 应能看到 CN；FE 角色与拓扑应一致。字段可能因版本不同而变化，报错时先检查权限、版本与连接的集群。

### 7.2 订单 SQL 计划验证

准备已去重的订单快照：某日网点 10 有 5 单，网点 20 有 1 单，预期结果分别为 5、1。这是**人为构造的断言，不是实测**。

~~~sql
EXPLAIN
SELECT order_day, site_id, COUNT(*) AS order_count
FROM orders
WHERE order_day >= '2026-11-06'
  AND order_day < '2026-11-07'
GROUP BY order_day, site_id;
~~~

保存原始计划，实际运行 SQL 核对结果。可选获取 Query Profile，记录扫描字节、聚合耗时、网络交换和缓存相关指标。**EXPLAIN 的计划节点不是 FE/BE/CN 的进程清单**，不要混淆物理算子与部署组件。

### 7.3 故障路径与反证

| 测试环境故障注入 | 观察指标 | 判断标准 |
| --- | --- | --- |
| 停止单 FE | 新连接/查询失败率、恢复时间 | 是否形成入口和规划单点 |
| 多 Follower 中停止 Leader | 新 Leader、选举时长、查询失败率 | 多数派是否存活、元数据是否追赶 |
| 停止一个 BE | 副本健康、失败率、恢复耗时 | 不预设“有副本就零失败” |
| 停止一个 CN | 查询重调度、缓存预热、失败率 | 识别冷缓存恢复成本 |
| CN 冷热缓存对比 | 远端读取字节、p95 | 区分 CPU 与远端 I/O |
| 共享存储异常 | 超时、读取错误、影响范围 | 识别外部存储故障域 |

**[H]** 故障行为受版本、负载、超时、副本、节点数和外部存储影响；以上是实验设计而非已验证结果。切勿在生产集群随意注入故障。

记录模板：

~~~text
StarRocks version / deployment mode:
FE roles / Alive / ReplayedJournalId:
BE or CN count / health:
Storage type / cache:
Dataset / expected result:
SQL / EXPLAIN / Profile:
Fault / duration:
Observed success rate / p95 / recovery time:
Raw logs / alternative explanations:
~~~

**[O] 本篇暂无真实实验观察。** 没有编造生产事故、查询耗时或故障恢复数字。

## 8. 作者练习与关键审稿问题

**练习**：不查资料，分别画出自上而下的 Client → FE → BE（旁边是本地 Tablet 副本）和 Client → FE → CN（旁边是本地缓存、下方是对象存储）。在 FE 侧边补充 Leader/Follower/Observer，用不同箭头表示**元数据复制**、**任务调度**和**数据读取**。解释为什么 FE、BE、CN 不能画成同一条串行链路。

**本期最值得审阅的问题**：**如果 FE Leader 故障，但所有 BE 的订单数据副本仍然完整，为什么报表查询仍可能受到影响？请从元数据多数派、SQL 规划/调度和客户端连接入口三个层面解释，并说明为什么增加 Observer 不能替代 Follower 的选举多数派。**

## 9. 来源、版本与去重边界

英文官方来源（2026-11-12 查阅）：

1. [Architecture](https://docs.starrocks.io/docs/introduction/Architecture/)：FE/BE/CN、两种部署模式、FE 元数据角色。
2. [Scale in and out](https://docs.starrocks.io/docs/administration/management/Scale_up_down/)：Follower/Observer 与 HA。
3. [Deploy Shared-data Manually](https://docs.starrocks.io/docs/deployment/deploy_shared_data_manually/)：CN、远端存储与缓存。
4. [Feature Support: Shared-data](https://docs.starrocks.io/docs/deployment/feature-support-shared-data/)：版本差异和部署限制。
5. [SHOW FRONTENDS](https://docs.starrocks.io/docs/sql-reference/sql-statements/cluster-management/nodes_processes/SHOW_FRONTENDS/)。
6. [SHOW BACKENDS](https://docs.starrocks.io/docs/sql-reference/sql-statements/cluster-management/nodes_processes/SHOW_BACKENDS/)。
7. [SHOW COMPUTE NODES](https://docs.starrocks.io/docs/sql-reference/sql-statements/cluster-management/nodes_processes/SHOW_COMPUTE_NODES/)。

**版本限制**：上述架构原则以当前英文官方文档为依据，3.x/4.x 不同版本的 shared-data 特性与 SQL 返回字段可能不同；目标集群版本与实验数据待作者补充。

**知识边界**：第 01 篇讨论为何需要 OLAP；本篇只建立 FE/BE/CN 的架构全貌；既有《一条订单汇总 SQL 在 StarRocks 里是怎么执行的？》研究稿留第 03 篇深化，订单乱序与重放稿留第 10/11 篇；第 13 篇深入高可用，第 15 篇深入存算架构选型。不提前进入 How 系列。