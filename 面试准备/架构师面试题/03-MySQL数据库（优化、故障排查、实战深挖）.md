## 三、MySQL数据库（优化、故障排查、实战深挖）

---

### 1. MySQL索引底层实现（B+树）与高效索引设计

#### B+树为什么适合数据库索引

MySQL 将 B+树的每个节点大小设置为一个磁盘页（InnoDB 默认 16KB）。一次磁盘 IO 读取一个页。

**三层 B+树能存多少数据**（假设 BigInt 主键 + 一行 1KB）：
- 非叶子节点：16KB / (8B key + 6B 指针) ≈ 1170 个索引项
- 叶子节点：16KB / 1KB ≈ 16 行
- 三层：1170 × 1170 × 16 ≈ **2200 万**行
- 意味着：2000 万行的表，一次主键查询只需 3 次磁盘 IO（根节点常驻内存 → 2 次 IO）

**为什么不是红黑树/AVL 树**：二叉树高度太高（2000万行 → 高度约 25），磁盘 IO 次数不可接受。

**B 树 vs B+树**：B 树非叶子节点也存数据 → 一个 16KB 页存的索引项少 → 树更高。B+树非叶子只存键 → 一个页存更多键 → 树更矮。

**叶子节点双向链表**：范围查询 `WHERE id BETWEEN 100 AND 200` 找到 100 后顺着链表直接遍历，不需要回到非叶子节点反复查找。

#### 聚簇索引与非聚簇索引的物理差异

**聚簇索引（Clustered Index）**：数据行物理存储在 B+树叶子节点中，按主键排序。一张表只能有一个聚簇索引（数据只能有一种物理排布）。主键查询 → O(log N) → 数据就在叶子节点，一次 IO。

**二级索引（Secondary Index）**：叶子节点存的是主键值，不是数据行指针。二级索引查询 → O(log N) → 拿到主键 → 主键 B+树 O(log N) → 拿到数据。这就是「回表」。所以覆盖索引（查询列全部在二级索引中）能避免回表，查询快一倍以上。

**为什么二级索引存主键而不是物理地址**：如果数据行移动了（页分裂、页合并），所有二级索引的指针都要更新，这不可接受。存主键只依赖主键不变，主键变化时才需要更新二级索引。

#### 索引设计的 10 条军规

1. **高选择性列优先**：`WHERE gender = 'M'` 区分度 50%，不加索引；`WHERE user_id = 123` 区分度 99.99%，必加
2. **联合索引的最左前缀**：`(a,b,c)` 能覆盖 `a`、`a,b`、`a,b,c` 的查询，不能覆盖 `b,c` 或 `c`
3. **索引列计算 = 全表扫描**：`WHERE YEAR(create_time) = 2024` → 改为 `WHERE create_time BETWEEN '2024-01-01' AND '2024-12-31'`
4. **隐式类型转换**：varchar 列 `phone`，写 `WHERE phone = 13800138000`（数字）→ MySQL 把列值全部转为数字再比较 → 全表扫描
5. **LIKE '%xxx'**：% 在前缀 → 无法使用 B+树前缀匹配 → 全表扫描；`LIKE 'xxx%'` 可以用
6. **OR 中有非索引列**：`WHERE id=1 OR name='xx'` → name 无索引 → 全表扫描
7. **覆盖索引**：`SELECT col1, col2 FROM t WHERE col1=1 AND col2=2` + 联合索引 `(col1,col2)` → Extra: Using index（最优）
8. **联合索引范围查询的列失效**：`(a,b,c)` 索引，`WHERE a=1 AND b>2 AND c=3` → c 不走索引（范围查询打断最左前缀）
9. **不等于/IS NULL**：不等于操作在索引中无法高效查找（需要扫描大量数据），优化器可能选择全表
10. **索引下推（ICP）**：MySQL 5.6+ `WHERE name LIKE '张%' AND age=20` → 存储引擎在索引层先过滤 age=20，再回表。索引拿到 name LIKE '张%' 的所有行，但只在 age=20 的行才回表取数据

#### 索引失效的核心原因（不只是"规则"，而是为什么）

索引的本质是**排序的数据结构**。B+树的叶子节点按索引列有序排列。如果查询条件破坏了这个"有序性"的利用方式（如函数计算、前缀模糊、类型转换），B+树的快速定位能力就失效了。

MySQL 优化器选择索引的唯一标准是**成本估算（Cost-Based Optimization）**。每个可能的执行计划都有一个 cost 值（基于 IO 和 CPU），选 cost 最小的。当 `rows` 很大时，用索引 + 回表的 cost > 全表扫描的 cost → 放弃索引。`FORCE INDEX` 强制使用索引可能导致更差的性能（因为绕过了优化器的 cost 估算）。

---

### 2. MySQL事务ACID与InnoDB保证机制

#### ACID 在 InnoDB 中的实现对应

**A（原子性）→ Undo Log**：每次修改前，把「旧值」写入 Undo Log。回滚时按 Undo Log 把数据改回去。Undo Log 本身也是数据，由 Redo Log 保护（Undo 的修改也写 Redo）。

**C（一致性）→ 约束 + MVCC**：主键/唯一键约束（数据层）、外键约束（业务可选择）、Check 约束、NOT NULL、数据类型都是数据库层面的强制一致性。应用层还要保证业务一致性。

**I（隔离性）→ MVCC + 锁**：快照读使用 ReadView（多版本），当前读使用行锁 + 间隙锁。

**D（持久性）→ Redo Log + Double Write**：
- Redo Log 是物理日志（记录 "在表空间 X 的偏移量 Y 处写了 Z 字节"），WAL 机制：先写 Redo Log（顺序写），再异步写数据页（随机写）
- Double Write Buffer：防止「页断裂」（16KB 页写到一半时宕机）。先写到 Double Write Buffer（连续 1MB 空间），再写到实际数据文件。崩溃恢复时从 Double Write 恢复完整页

#### 事务实现的完整链路

```
BEGIN;
UPDATE t SET c=2 WHERE id=1;
COMMIT;

执行流程：
1. 加锁（id=1 的行锁）
2. 写 Undo Log（记录 c 的旧值 = 1）
3. 修改 Buffer Pool 中的数据页（c=2，标记为脏页）
4. 写 Redo Log（记录修改的内容）
5. COMMIT → Redo Log 标记为 commit（两阶段提交 prepare → binlog → commit）
6. 释放锁
```

**两阶段提交为什么需要**：Redo Log 和 binlog 是两个独立的日志系统。Redo Log 用于崩溃恢复（InnoDB 层面），binlog 用于主从复制（MySQL 层面）。如果不使用两阶段提交，先写 Redo 后写 binlog 或不一致，主从切换后数据会错。

---

### 3. TB级数据分库分表落地

#### 分片键选择的黄金法则

1. **查询条件中最常出现的列**：SQL 中 WHERE/GROUP BY/JOIN ON 是否都带这个键？如果带键率 < 70%，方案不可行
2. **数据分布均匀**：取模 → `user_id % 64`，均匀但扩容难（一致性哈希解决）
3. **避免跨分片 JOIN**：如果有跨分片查询的刚需，考虑在分片键选择时让关联数据落在同一分片（如订单和订单明细用同一订单 ID 分片）

#### 分库分表后的问题与解决

**分页查询**：
- 传统 `LIMIT 10000, 20`：每个分片取 10020 条 → 合并 64 分片 → 64 × 10020 → 归并排序 → 取 20 条 → 中间件（Sharding-JDBC）的归并引擎处理这个。性能极差。
- 优化：二次查询法。先在所有分片 `LIMIT 10000, 20`，归并出这 20 条的时间范围，再回到各分片精确取这 20 条

**跨分片聚合**：如果业务必须横跨所有分片做 SUM/COUNT/AVG → 用 Elasticsearch 做宽表索引提前预聚合

**分布式 ID 生成**：
- 雪花算法：1bit 保留 + 41bit 毫秒时间戳 + 10bit 工作机器 ID + 12bit 序列号 → QPS 极高、趋势递增、没有中心节点。12bit 序列号每毫秒 4096 个 → 单机 QPS 上限约 400 万
- 号段模式（Leaf-Segment）：从数据库批量取号段（如 1000 个 ID 一次），缓存到本地消费，性能更高

#### 数据迁移方案（最关键）

**双写双读过渡**：
```
Phase 1: 全量迁移（凌晨导出老库 → 导入分片库）
Phase 2: 增量订阅（Canal 订阅老库 binlog → 同步分片库）
Phase 3: 数据校验（每行对比 MD5/CRC，发现不一致则补偿）
Phase 4: 灰度切读（1% 流量读分片库，验证 OK → 逐步 100%）
Phase 5: 双写（写老库 + 分片库）→ 观察 N 天 → 停写老库
```

---

### 4. 核心慢查询从3s优化至50ms

#### 完整优化案例的思维过程

**原始 SQL**：
```sql
SELECT * FROM orders WHERE status IN ('PAID','SHIPPING') 
ORDER BY create_time DESC LIMIT 20;
```

**Step 1：EXPLAIN 结果**
```
type: ALL  |  rows: 2,300,000  |  Extra: Using where; Using filesort
```
全表扫描 230 万行，文件排序。`status` 区分度低（两个值覆盖了 80% 的数据），不适合作为索引前置。

**Step 2：业务分析**
用户只看自己的订单。所以加上 `user_id` —— 高区分度列。

**Step 3：改写**
```sql
SELECT id, order_no, status, amount, create_time 
FROM orders 
WHERE user_id = ? AND status IN ('PAID','SHIPPING') 
ORDER BY id DESC LIMIT 20;
```

**Step 4：建联合索引**
```sql
CREATE INDEX idx_uid_status_id ON orders(user_id, status, id);
```
覆盖索引：`SELECT` 的 5 个字段全部在联合索引的叶子节点中（ID 和 status 在索引中，order_no + amount 没有 → 其实还需要另外的优化，这是面试时考官会追问的地方）。

最终：`CREATE INDEX idx_uid_status ON orders(user_id, status, id, order_no, amount, create_time);`  → `Extra: Using index`（覆盖索引，不回表，不需要 filesort）。

**Step 5：效果**
`rows: 2.3M → 10, Extra: Using index, type: ref`

**面试必问的追问**：为什么不把 user_id 拆成一个独立索引 + status 独立索引？答：联合索引有最左前缀，`(user_id)` 查询可以用，而且 user_id 高区分度在前，先缩小范围再精确匹配 status。如果两个独立索引 → 只能用其中一个（或 index_merge，但不常见），效率远不如一个联合索引。

---

### 5. 线上MySQL卡顿、死锁、主从延迟如何排查？

#### 是什么

MySQL 卡顿的排查不是「看慢查询日志」这么简单。卡顿可能来自多个层次：SQL 层（慢查询）、锁层（MDL 锁 / 行锁等待）、存储引擎层（脏页刷新）、系统层（磁盘 IO 满）、复制层（主从延迟导致读库数据旧）。

#### 5 层排查法（从外到内）

**第一层：连接和线程**
```sql
SHOW FULL PROCESSLIST;  -- 看当前执行的 SQL 状态
```
关键状态：
- `Sending data` → 正在查询数据（可能是全表扫描）
- `Creating sort index` → 在创建排序索引（ORDER BY 没走索引）
- `Waiting for table metadata lock` → **MDL 锁等待**（有人在做 DDL 或长事务没提交！）
- `Waiting for table level lock` → 表锁等待（MyISAM / `LOCK TABLES`）
- `System lock` → 内部锁

**第二层：锁诊断**
```sql
-- 查看当前事务
SELECT * FROM information_schema.INNODB_TRX\G
-- 查看锁等待
SELECT * FROM information_schema.INNODB_LOCK_WAITS\G
-- 查看当前持有的锁
SELECT * FROM information_schema.INNODB_LOCKS\G
-- MySQL 8.0+ 统一视图
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;

-- 用 sys schema(更易读)
SELECT * FROM sys.innodb_lock_waits;  -- 谁在等谁的锁
SELECT * FROM sys.schema_table_lock_waits;  -- MDL 等待
```

**第三层：InnoDB 引擎状态**
```sql
SHOW ENGINE INNODB STATUS\G
```
关键部分：
- `LATEST DETECTED DEADLOCK` → 死锁详情（事务 1 在等什么锁、事务 2 持有什么锁）
- `TRANSACTIONS` → 活跃事务列表 + 持有的锁数量
- `BUFFER POOL AND MEMORY` → Buffer Pool 命中率
- `SEMAPHORES` → OS 级别等待（`os WAIT ARRAY INFO` 多说明有争用）

**第四层：磁盘 IO**
```sql
-- 查看脏页比例
SHOW GLOBAL STATUS LIKE '%dirty%';
-- Innodb_buffer_pool_pages_dirty / Innodb_buffer_pool_pages_total > 10% → 刷脏页频繁
```
配合系统命令：`iostat -x 1`（`util%` 100% → IO 瓶颈）

**第五层：和业务核时间点**：什么时候开始卡的？和发版时间、大促时间、定时任务时间重合吗？

#### 死锁排查实战

**死锁日志解读**（`SHOW ENGINE INNODB STATUS\G`）：
```
------------------------
LATEST DETECTED DEADLOCK
------------------------
*** (1) TRANSACTION:
UPDATE accounts SET balance = balance - 100 WHERE id = 1
*** (1) HOLDS THE LOCK(S):
RECORD LOCKS ... index `PRIMARY` ... lock_mode X locks rec but not gap
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS ... index `PRIMARY` of table `accounts` ... lock_mode X waiting

*** (2) TRANSACTION:
UPDATE accounts SET balance = balance + 100 WHERE id = 2
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS ... index `PRIMARY` ... lock_mode X locks rec but not gap
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS ... index `PRIMARY` of table `accounts` ... lock_mode X waiting
```
解读：事务 1 持有 id=1 的 X 锁，在等 id=2 的 X 锁。事务 2 持有 id=2 的 X 锁，在等 id=1 的 X 锁 → 循环等待 → 死锁。InnoDB 回滚了事务 2。

**转账死锁的根因及解决**：
A 转给 B 和 B 转给 A（或 A 转给 B 的同时 C 转给 A）同时执行。加锁顺序不一致 → 破坏死锁第三个条件「循环等待」。
解决：所有转账按账户 ID 升序加锁：
```java
List<Long> ids = Arrays.asList(fromId, toId);
Collections.sort(ids);  // 按 ID 排序
for (Long id : ids) {
    accountMapper.selectForUpdate(id);  // 按相同顺序加锁
}
```

**工具**：`pt-deadlock-logger`（Percona Toolkit）→ 持续监控死锁日志并告警。

#### 主从延迟排查

**查看延迟**：
```sql
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 延迟秒数（0 表示追上）
-- Last_IO_Error: IO 线程报错（网络问题、binlog 损坏）
-- Last_SQL_Error: SQL 线程报错（主键冲突、表不存在）
```

**延迟原因分类**：
1. **大事务**：主库一个大事务执行 30s → binlog 到从库后 SQL 线程也要执行 ~30s → 延迟 30s。解决：拆分大事务
2. **从库硬件差**：主库 SSD，从库 HDD → SQL 线程回放慢。解决：从库规格不低于主库
3. **无主键表**：InnoDB 用隐藏 ROW_ID 作为主键 → 全表扫描的 DELETE 极慢。解决：所有表必须有主键
4. **并行复制配置不当**：`slave_parallel_workers` = 1 → 单线程回放 → 慢。MySQL 5.7+ 开启并行复制（`slave_parallel_type=LOGICAL_CLOCK`）
5. **网络**：跨机房同步带宽不够 → 数据量大时延迟。解决：专线 / 压缩 binlog

#### 坑点

1. **`Seconds_Behind_Master` = 0 不代表零延迟**：它只是 SQL 线程已经执行到的 binlog 位置和当前时间戳的差。如果 IO 线程追不上主库 binlog，SQL 线程追的是旧的 binlog → 延迟已经很大了但 SBM 还是 0。要看 `Master_Log_File` 和 `Read_Master_Log_Pos` 对比 `SHOW MASTER STATUS`。
2. **大事务导致从库卡顿**：主库一条 `DELETE FROM logs WHERE create_time < '2020-01-01'` → 几千万行 → SQL 线程在从库执行了 30 分钟 → 从库整个卡住。

---

### 6. MySQL的锁机制（行锁、表锁、间隙锁），在什么场景下会触发不同的锁？如何避免死锁？

#### 是什么

InnoDB 实现了四种锁粒度，但实际上只有两种基础锁：**Record Lock（行锁）** 和 **Gap Lock（间隙锁）**，Next-Key Lock 是它们的组合。

**核心原则（必须记住）**：**InnoDB 锁的是索引记录，不是数据行。** 没有索引 → 全表扫描 → 锁住所有行乃至间隙 → 等于表锁。

#### 四种锁的详细机制

**① Record Lock（记录锁 / 行锁）**

锁住**索引记录**，不锁间隙。当使用唯一索引进行精确匹配时（`WHERE id = 1`），加 Record Lock。行锁是共享（S Lock）或排他（X Lock）。

**② Gap Lock（间隙锁）**

锁住索引记录之间的**间隙**（开区间）。目的：**阻止其他事务在间隙中插入新记录**（防止幻读）。只在 REPEATABLE READ 隔离级别下存在，READ COMMITTED 下自动禁用间隙锁。

示例：`SELECT * FROM t WHERE id BETWEEN 10 AND 20 FOR UPDATE` → 在 id=10 和 id=20 之间的间隙（10, 20）加 Gap Lock → 其他事务无法 `INSERT INTO t VALUES(15, ...)`。

**③ Next-Key Lock（临键锁）**

**Record Lock + Gap Lock 的组合**，锁住索引记录 + 前面的间隙。InnoDB 默认使用 Next-Key Lock。

为什么用 Next-Key Lock 而不是单独的行锁 + 间隙锁？因为这样可以在一次加锁操作中同时防止：修改已有记录 + 插入幻影记录。

**④ 插入意向锁（Insert Intention Lock）**：特殊的 Gap Lock，两个 INSERT 在同一个间隙中互不阻塞（除非插入相同主键/唯一键）。

#### 什么场景触发不同锁（关键判断表）

| SQL 写法 | 索引情况 | 加的锁 | 说明 |
|---------|---------|--------|------|
| `SELECT ... WHERE 主键=1 FOR UPDATE` | 主键精确查找 | Record Lock | 只锁 id=1 这行 |
| `SELECT ... WHERE 主键 BETWEEN 10 AND 20 FOR UPDATE` | 主键范围查找 | Next-Key Lock | 锁 id=10 的行 + 10~20 间的间隙 + 锁 20 行 + 20 后的间隙 |
| `SELECT ... WHERE 普通索引 col=5 FOR UPDATE` | 非唯一索引精确查找 | Next-Key Lock（多行） | 所有 col=5 的记录锁 + 相邻间隙 |
| `UPDATE t SET ... WHERE name='test' FOR UPDATE` | **无索引** | **全表所有行锁 + 所有间隙锁** | **等于是表锁！** |
| `INSERT INTO t VALUES(15)` | - | 插入意向锁 | 不阻塞其他 INSERT |

#### 如何避免死锁

**死锁四条件 + 破坏方法**：
1. **互斥**（无法改变，内存中必要的）
2. **占有且等待** → 破坏：`tryLock(timeout)` 超时释放已持有的锁 → 数据库层不能用（MySQL 不支持定时行锁），应用层用 `tryLock`
3. **不可剥夺** → 破坏：数据库设置了 `innodb_lock_wait_timeout`（默认 50s），超时回滚
4. **循环等待** → **破坏（最有效！）**：所有事务按相同顺序加锁

**实际避免死锁的 5 条规则**：
1. **按固定顺序加锁**：转账按账户 ID 升序，订单加锁按 order_id 升序
2. **短事务**：事务越小越短，死锁概率越低。不要在事务中调 RPC、发邮件、写文件
3. **索引优化**：确保 WHERE 条件走索引 → 行锁 → 锁范围小 → 死锁概率低。不走索引 → 表锁 → 锁范围大 → 死锁概率高
4. **RR 下减少不必要的间隙锁**：如果业务允许 RC 隔离级别 → 间隙锁不会产生 → 死锁场景显著减少
5. **应用层重试**：捕获 `DeadlockLoserDataAccessException` → 重试。InnoDB 已经帮你回滚了一个事务，你只要重试就行了

#### 坑点

1. **RR 下的 `LOCK IN SHARE MODE` 也加间隙锁**：即使是 S 锁，也会加 Next-Key Lock → 可能和 INSERT 冲突
2. **隐式锁**：INSERT 完成后，对此记录加「隐式锁」——不显式加锁，但当其他事务尝试对这行加锁时，隐式锁转为显式锁（X Lock）
3. **唯一索引的锁降级**：`WHERE unique_key = 5 FOR UPDATE` → 记录存在 → 退化 Record Lock（不加间隙锁）。记录不存在 → 加 Gap Lock（防止插入）

---

### 7. 执行计划如何解读？通过执行计划判断SQL性能问题？

#### 是什么

`EXPLAIN` 模拟优化器的执行计划（不是真实执行）。它告诉你：MySQL 打算怎么执行这条 SQL → 走了什么索引、大概扫描多少行、有没有临时表/文件排序。

#### 10 个字段逐字段详解

**① id**：SELECT 的序号，越大越先执行。id 相同 → 从上往下执行。子查询和派生表的 id 会递增。

**② select_type**：
- `SIMPLE`：简单 SELECT（没有子查询和 UNION）
- `PRIMARY`：最外层的 SELECT
- `SUBQUERY`：SELECT/WHERE 中的子查询
- `DERIVED`：FROM 中的子查询（派生表）
- `UNION`：UNION 中的第二个及之后的 SELECT
- `DEPENDENT SUBQUERY`：依赖外部查询的子查询（每行外部行都执行一次子查询 → **性能杀手**）

**③ type（最重要的字段！访问效率从优到差）**：
```
system  >  const  >  eq_ref  >  ref  >  range  >  index  >  ALL
只有1行   主键等值  唯一键关联  非唯一索引  索引范围  全索引扫描  全表扫描
```
- `const`：主键或唯一索引的等值查询，最多返回 1 行。最优。
- `eq_ref`：连接查询中驱动表的每一行在另一表中匹配唯一索引。如 `LEFT JOIN ... ON a.id = b.id`（b.id 是主键）。
- `ref`：非唯一索引的等值查询。如 `WHERE status = 'PAID'`。
- `range`：索引范围查询。`BETWEEN`、`IN`、`>`、`<` 等。
- `index`：全索引扫描（扫描整个索引，但只读索引不回表）。比 ALL 好（索引通常比数据小），但仍是全扫描。
- `ALL`：全表扫描，性能最差，必须优化。

**目标**：至少达到 `range`，争取 `ref`，最好 `const`/`eq_ref`。

**④ possible_keys**：可能用到的索引。这个字段参考价值有限（优化器考虑了但可能不用）。

**⑤ key**：实际使用的索引。**如果 key = NULL 但 possible_keys 有值 → 优化器觉得走索引不如全表扫描**（通常因为区分度太低）。

**⑥ key_len**：使用的索引的字节数。用来判断**联合索引用了前面几列**：
```
key_len = 4  → 只用了 user_id (INT, 4 bytes)
key_len = 8  → 用了 user_id + status (INT 4B + TINYINT 1B + 3B 对齐 = 8B)
key_len = 30 → 用了全部三列
```

**⑦ ref**：与索引比较的列或常量。`const` = 常量比较，`db.t.id` = 关联列比较。

**⑧ rows**：预估需要扫描的行数（采样统计，不是真实行数）。关注突变趋势——同一个 SQL 的 rows 突然从 10 涨到 10000 → 说明索引选择变了。

**⑨ filtered**：返回行数占扫描行数的百分比。rows=1000, filtered=10% → 估计最终返回 100 行 → 其余 900 行被 WHERE 过滤掉了。

**⑩ Extra（额外信息）**：
- `Using index`（最好）：覆盖索引，不回表，只读索引
- `Using index condition`（好）：索引下推（ICP），在引擎层就过滤
- `Using where`（正常）：服务层过滤（引擎返回了不需要的行）
- `Using temporary`（**需要优化**）：用了临时表（GROUP BY 的列没索引、DISTINCT + ORDER BY）
- `Using filesort`（**需要优化**）：排序没走索引，需要文件排序
- `Using join buffer`（需要关注）：JOIN 缓存被使用（被驱动表没走索引）

#### 实战：从 EXPLAIN 到优化方案

**场景**：
```sql
EXPLAIN SELECT * FROM orders WHERE user_id=1001 ORDER BY create_time DESC LIMIT 10;
-- type: ref, key: idx_user_id, Extra: Using where; Using filesort
```
分析：
- `type=ref` → 走了索引
- `Extra=Using filesort` → ORDER BY 没走索引，需要文件排序
- 原因：`ORDER BY create_time` 和索引 `(user_id)` 不匹配 → 只用了 user_id 过滤，create_time 需要额外排序

**优化**：
```sql
CREATE INDEX idx_uid_ctime ON orders(user_id, create_time);
-- 再 EXPLAIN:
-- key: idx_uid_ctime, Extra: Using index condition (or Backward index scan)
-- Using filesort 消失！
```

---

### 8. MySQL分区表原理？分区类型？适用场景与注意事项？

#### 是什么

分区表把一张逻辑表在物理层面分成多个独立的数据文件（`t#P#p0.ibd`, `t#P#p1.ibd`），但在 SQL 层面仍然是一张表，你写 `SELECT * FROM t` 正常查询。MySQL 根据分区键和分区函数自动路由到具体分区。

#### 底层原理

分区不是「把一张表切成 n 张小表」。分区是把**数据按行分配到不同的表空间文件**，但元数据仍共享（相同的表结构、索引定义）。分区裁剪（Partition Pruning）是核心优化：优化器根据 WHERE 条件中的分区键，在查询计划生成阶段就排除不相关的分区，只扫描需要的分区。

**分区的几种实现层次**：
- `PARTITION BY RANGE` → 优化器范围裁剪
- `PARTITION BY LIST` → 优化器枚举裁剪
- `PARTITION BY HASH` → 所有分区都可能被扫描（无法裁剪），但每个分区更小

#### 四种分区类型

**① RANGE 分区**：按范围分区，如按年/月分区日志表。最常用的分区类型。
```sql
CREATE TABLE logs (
    id BIGINT, created_at DATE, msg TEXT
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

**② LIST 分区**：按枚举值分区，如按地区分用户表。
```sql
PARTITION BY LIST (region_id) (
    PARTITION p_east  VALUES IN (1, 2, 3),
    PARTITION p_west  VALUES IN (4, 5, 6),
    PARTITION p_south VALUES IN (7, 8, 9)
);
```

**③ HASH 分区**：按哈希值均匀分布，`PARTITION BY HASH(user_id) PARTITIONS 8`。MySQL 内部用 `user_id % 8` 分配数据。无法裁剪特定分区（所有分区都要扫）。

**④ KEY 分区**：和 HASH 类似，但哈希函数由 MySQL 内部提供（`PASSWORD()` 等）。用于字符串分区的场景。

#### 适用场景

1. **日志归档删除**：日志按月分区 → 删除 3 年前的日志 → `ALTER TABLE logs DROP PARTITION p2019`（秒级，不需要生成 Undo/Redo），对比 `DELETE FROM logs WHERE YEAR(created_at)=2019`（可能需要几小时 + 大量 Undo + 锁）
2. **时间范围查询**：`WHERE created_at BETWEEN '2024-01-01' AND '2024-03-31'` → 分区裁剪只扫描对应的 3 个分区
3. **冷热数据分离**：热数据（最近 3 月）放 SSD，冷数据（3 个月前的）放 HDD

#### 注意事项（坑点）

1. **分区键必须是主键/唯一索引的一部分**：`A PRIMARY KEY must include all columns in the table's partitioning function`。如果你按 `created_at` 分区但主键是 `id`（不包含 `created_at`），分区建表会失败。
2. **最大分区数 8192**：每个分区本身有内存开销（打开的表描述符）
3. **分区列不能是表达式（MySQL 5.5）**：`PARTITION BY RANGE (YEAR(created_at))` → 5.7+ 可用函数，但推荐直接用 `TO_DAYS(created_at)` 范围
4. **跨分区的唯一约束弱化**：如果主键是 `(id, created_at)`，分区键是 `created_at`，那么 id 的唯一性只在单个分区内保证 —— 两个不同分区可以有相同的 id 值。
5. **现在很多场景被分库分表替代**：分区表仍是单机，不能水平扩展写入。数据量大到需要分区时，往往也需要分库了。但归档删除场景（`DROP PARTITION`）分区仍是最佳方案。

---

### 9. MySQL主从复制原理（binlog、relaylog、IO线程、SQL线程）与复制模式？

#### 是什么

MySQL 主从复制 = Master 把所有数据变更写入 binlog → Slave 拉取 binlog 到 relay log → Slave 重放 relay log 中的事件。涉及三个线程：主库 Binlog Dump Thread、从库 IO Thread、从库 SQL Thread。

#### 为什么需要主从复制

1. **读写分离**：主库写入、从库查询，扩展读能力
2. **数据灾备**：从库是主库的实时备份
3. **高可用切换**：主库宕机 → 从库提升为主
4. **报表/分析**：在从库跑分析 SQL，不影响主库业务

#### 复制原理（3 个线程的协作）

**① 主库 Binlog Dump Thread**：
- 主库有数据变更 → 写入 binlog
- 当 Slave 连接后，Dump Thread 启动
- 把 binlog 事件推送给 Slave 的 IO Thread
- 生命周期：Slave 连接 → 创建 Dump Thread → Slave 断开 → 销毁

**② 从库 IO Thread**：
- `START SLAVE` → IO Thread 启动
- 连接 Master → 发请求："给我从 binlog.000003:120 位置的数据"
- 接收 Master 推送的 binlog 事件 → 写入 relay log（`relay-bin.000001`）
- 写到 relay log 后 → 更新 `master.info` 文件（记录已读到的 binlog 位置，崩溃恢复用）

**③ 从库 SQL Thread**：
- 读取 relay log 中的事件 → 逐个回放
- 回放完成后 → 更新 `relay-log.info`（记录已回放到 relay log 的哪个位置）
- 如果回放遇到错误（主键冲突/Duplicate entry）→ 根据 `slave_exec_mode`（STRICT/IDEMPOTENT）决定行为

**两阶段提交在主从中的体现**：
1. Master：Redo Log prepare → 写 binlog → Redo Log commit（两阶段提交保证一致性）
2. Slave：IO Thread 写 relay log → SQL Thread 回放到 Redo Log + binlog（Slave 自身也写 binlog，如果开启了 log_slave_updates）

#### 三种复制模式对比

**① 异步复制（默认）**：
- Master 提交事务后不等待 Slave 确认 → 直接返回给客户端
- 优点：Master 性能好（不受 Slave 速度影响）
- 缺点：Master 宕机后，Slave 可能还没有收到 binlog → 数据可能丢失
- 适用：非核心业务、允许少量数据丢失

**② 半同步复制（Semi-Sync，MySQL 5.5+）**：
- Master 提交事务后**等待至少一个** Slave 确认收到 binlog 才返回给客户端
- `rpl_semi_sync_master_wait_for_slave_count = 1` → 等 1 个 Slave 确认
- 优点：至少有一个 Slave 有完整数据，切换时不会丢数据
- 缺点：Master 等待 Slave 确认 → 有延迟（Slave 慢 / 网络抖动）
- 降级机制：等待超时（`rpl_semi_sync_master_timeout`，默认 10000ms）→ 退化为异步复制
- 适用：金融、支付等高一致性要求

**③ 全同步复制（MySQL Group Replication / MGR）**：
- Master 等待**所有** Slave 都执行完才返回
- 优点：数据 100% 不丢
- 缺点：性能极差（受最慢 Slave 制约），基本不用
- 替代：MGR 用 Paxos 协议实现多主/单主模式

#### 三种 binlog 格式

- **STATEMENT**：记录 SQL 语句。节省空间但可能不确定（`NOW()` 在主从产生不同值）
- **ROW（推荐）**：记录每行被修改的前后镜像。体积大但精确（主从结果绝对一致）
- **MIXED**：默认语句模式 + 特殊情况下 ROW 模式。推荐直接设置 ROW 模式

#### 坑点

1. **`sync_binlog=0` 导致 binlog 丢失**：MySQL 不主动刷 binlog 到磁盘（交给 OS）。Master 宕机 → 未刷盘的 binlog 丢失 → Slave 没收到 → 主从数据不一致
2. **ROW 模式下大事务的 relay log 暴涨**：`UPDATE 全表` → ROW 模式记录 1 亿行的前后镜像 → relay log 几百 GB
3. **Slave SQL 线程单线程回放**：`slave_parallel_workers=0` → 回放慢 → 主从延迟大。启用并行复制

---

### 10. MySQL读写分离如何实现？用哪些中间件？如何解决读写分离后的一致性问题？

#### 是什么

读写分离 = 写操作（INSERT/UPDATE/DELETE）走主库，读操作（SELECT）走从库。这是提升 MySQL 读并发能力最直接的方式。但引入了主从延迟导致的「写后读」一致性问题。

#### 实现方案对比

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| Sharding-JDBC | JDBC 层拦截，根据 SQL 类型自动路由 | 轻量（无代理层）、性能好 | 语言绑定（Java） |
| MyCat | 独立代理层，MySQL 协议代理 | 跨语言、运维简单 | 多一跳网络、单点风险 |
| ProxySQL | 代理层，支持查询缓存 + 读写分离 | 性能好、查询重写 | 运维成本 |
| 应用层手动 | `@Master` / `@Slave` 注解 | 最灵活 | 侵入性强、维护成本高 |

#### Sharding-JDBC 读写分离配置

```yaml
spring:
  shardingsphere:
    datasource:
      names: master, slave0, slave1
      master:
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://master:3306/order_db
      slave0:
        url: jdbc:mysql://slave0:3306/order_db
      slave1:
        url: jdbc:mysql://slave1:3306/order_db
    rules:
      readwrite-splitting:
        data-sources:
          order_rw:
            type: Static
            props:
              write-data-source-name: master
              read-data-source-names: slave0, slave1
            load-balancer-name: round_robin
```

**强制走主库**：
```java
// 写操作后马上读 → 强制走主库
HintManager hintManager = HintManager.getInstance();
hintManager.setWriteRouteOnly();  // 此线程的所有操作走主库
try {
    orderService.createOrder();
    Order order = orderService.findById(orderId);  // 走主库
} finally {
    hintManager.close();
}
```

#### 一致性问题解决方案（主从延迟导致读到旧数据）

**问题**：写主库 → 立即读从库 → 从库还没同步 → 读到旧数据（如「订单不存在」）

**方案一：强制走主库（最直接）**
写操作之后的关键读（如支付后查状态）用 `HintManager` 强制走主库。

**方案二：延迟阈值**
从库延迟超过 N 秒 → 自动切到主库。
```yaml
props:
  slave-latency-threshold: 1  # 从库延迟 > 1s 自动走主库
```

**方案三：缓存双写**
写入时更新 Redis 缓存（和 DB 写在同一事务或补偿同步）。读优先走 Redis → 绕过主从延迟。

**方案四：等待从库同步（不推荐但能应急）**
```java
// 写完后 sleep 一下等待同步
orderService.createOrder(order);
Thread.sleep(100);  // 等 100ms（主从延迟 < 50ms）
Order order = orderService.findById(orderId);
```
不优雅，但在没有中间件的场景下能解决问题。

**方案五：应用层过滤**
读取时检查数据版本号/更新时间。如果从库数据版本 < 最近写入的版本 → 换从库或走主库。

---

### 11. MySQL慢查询日志开启与分析？除了慢查询日志还有什么监控工具？

前面第 4 题已详细解答，这里强调补充内容：

**慢查询日志配置最佳实践**：
```cnf
slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=0.05          # 50ms（不是默认的 10s！要设小）
log_queries_not_using_indexes=1   # 记录不走索引的查询
log_slow_admin_statements=1       # 记录 DDL 的慢查询
log_slow_slave_statements=1       # 记录 Slave 回放的慢 SQL
min_examined_row_limit=1000       # 至少扫描 1000 行才记录（过滤微小查询）
```

**分析工具链**：
- `mysqldumpslow`（自带）：统计频率 + 平均耗时。`mysqldumpslow -s t -t 10 slow.log`
- `pt-query-digest`（Percona Toolkit）：分析报告，含响应时间分布、典型 EXPLAIN、查询指纹
- 可视化：PMM（Percona Monitoring and Management）= Prometheus + Grafana 的 MySQL 定制版

**Performance Schema 使用方法**：
```sql
-- 找执行最慢的 SQL 模板
SELECT DIGEST_TEXT, COUNT_STAR, 
       AVG_TIMER_WAIT/1000000000 AS avg_ms,
       SUM_ROWS_EXAMINED/COUNT_STAR AS avg_rows
FROM performance_schema.events_statements_summary_by_digest
ORDER BY AVG_TIMER_WAIT DESC LIMIT 10;
```

---

### 12. MySQL参数调优有哪些？如何根据业务场景调整？

#### 分层参数体系

**第一层：连接**
```cnf
max_connections=1000            # 最大连接数（不是越大越好，每个连接占内存）
wait_timeout=600                # 非交互连接空闲 10 分钟关闭
interactive_timeout=28800       # 交互连接（mysql 客户端）8h
```
连接数不是越大越好——每个连接在 InnoDB 中占 ~2MB（线程栈 + 缓冲），1000 连接 = 2GB 内存。

**第二层：InnoDB Buffer Pool（最核心）**
```cnf
innodb_buffer_pool_size=24G       # 物理内存的 50-80%（专用数据库可到 80%）
innodb_buffer_pool_instances=8    # 池分 8 个实例（>1G 时建议 4-8，减少竞争）
```
Buffer Pool 命中率：`SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%'` → `read_requests / (reads + read_requests)` > 99% 正常。

**第三层：Redo Log**
```cnf
innodb_log_file_size=4G           # 单个 redo log 文件大小
innodb_log_files_in_group=2       # redo log 文件数量
innodb_flush_log_at_trx_commit=1  # 1=每次提交刷盘 2=每秒刷盘
```
`flush_log_at_trx_commit=1` 最安全，但高频小事务场景（如日志系统）可用 `=2`（每秒刷盘，丢 1 秒数据）。

**第四层：IO**
```cnf
innodb_io_capacity=2000           # SSD 设 2000-5000，HDD 设 200
innodb_read_io_threads=8          # 读 IO 线程
innodb_write_io_threads=8         # 写 IO 线程
innodb_flush_method=O_DIRECT      # 绕过 OS 缓存（SSD 性能 + 避免双份缓存）
```

**第五层：临时表**
```cnf
tmp_table_size=64M
max_heap_table_size=64M
```
超过此大小 → 磁盘临时表（慢）。调大内存临时表。

**第六层：其他**
```cnf
innodb_thread_concurrency=0       # 0=不限制（InnoDB 自己调度）
innodb_flush_neighbors=0          # SSD 下关闭（不刷新相邻页）
innodb_adaptive_hash_index=ON     # 自适应哈希索引（提升热点查询）
```

#### 调优策略

- **大内存场景**（128G+）：`innodb_buffer_pool_size=100G`，`innodb_buffer_pool_instances=16`
- **高频写入场景**：`innodb_flush_log_at_trx_commit=2` + `sync_binlog=100`（性能优先）
- **高频查询场景**：重点调大 Buffer Pool

---

### 13. InnoDB和MyISAM的区别？各自的适用场景？为什么InnoDB现在是默认存储引擎？

#### 核心差异

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务** | 支持（ACID） | 不支持 |
| **锁粒度** | 行锁 | 表锁 |
| **外键** | 支持 | 不支持 |
| **崩溃恢复** | Redo Log 保证安全 | 崩溃后表可能损坏（需 `REPAIR TABLE`） |
| **MVCC** | 支持（高并发读写） | 不支持（读写互斥） |
| **全文索引** | MySQL 5.6+ 支持 | 原生支持 |
| **COUNT(*)** | 全索引扫描 | 表级计数器（极快，但有 WHERE 也一样慢） |
| **数据存储** | 聚簇索引（ibd 文件） | 堆表（MYD 数据 + MYI 索引 + frm 表结构） |
| **压缩** | 页级压缩 | 静态表压缩 |
| **地理索引** | GIS 类型 | 不支持 |

#### 为什么 InnoDB 是默认引擎

1. **事务是现代社会的基本需求**：没有事务 → 扣库存失败但订单创建了 → 数据不一致
2. **行锁支持高并发写入**：MyISAM 的表锁 → 一次只能一个写操作 → 写入 QPS 最高几百。InnoDB 行锁 → 并发写入 QPS 过万
3. **崩溃恢复**：MyISAM 异常关闭后可能表损坏 → 修复时锁表 → 线上不可用
4. **从 MySQL 5.5 到 8.0，InnoDB 不断进化**：全文索引、GIS、Memcached 插件、原生分区

MyISAM 残留场景（非常少）：只有极少数纯只读的系统表、或者对`COUNT(*)` 性能有极致要求且不需要 WHERE 的极特殊场景。

---

### 14. MySQL的MVCC原理是什么？如何实现？解决了什么问题？

（前面 3.9 已部分覆盖，这里扩展更底层细节）

#### MVCC 的物理实现

**每行记录的 3 个隐藏列**：
1. `DB_ROW_ID`（6 bytes）：行 ID（如果表没有主键，InnoDB 用此列作为聚簇索引键）
2. `DB_TRX_ID`（6 bytes）：最后修改这行的事务 ID
3. `DB_ROLL_PTR`（7 bytes）：回滚指针，指向 Undo Log 中此行的旧版本

**Undo Log 版本链**：
```
当前行（DB_TRX_ID=100, c=3）
  ← DB_ROLL_PTR 指向
Undo Log Record（trx_id=99, c=2）
  ← 上一个 Undo Record
Undo Log Record（trx_id=90, c=1）
  ← 上一个 Undo Record
Undo Log Record（trx_id=80, c=0）[初始值]
```

#### ReadView 可见性判断算法

```java
class ReadView {
    long creator_trx_id;   // 创建此 ReadView 的事务 ID
    Set<Long> m_ids;       // 生成 ReadView 时，活跃的（未提交的）事务 ID 集合
    long min_trx_id;       // m_ids 中的最小事务 ID
    long max_trx_id;       // 系统下一个要分配的事务 ID
}

boolean isVisible(DataRow row, ReadView view) {
    long rowTrxId = row.DB_TRX_ID;
    
    // 1. 如果这行是当前事务自己修改的 → 可见
    if (rowTrxId == view.creator_trx_id) return true;
    
    // 2. 如果修改该行的事务在 ReadView 创建前已提交 → 可见
    if (rowTrxId < view.min_trx_id) return true;
    
    // 3. 如果修改这行的事务在 ReadView 创建后才开始 → 不可见
    if (rowTrxId >= view.max_trx_id) return false;
    
    // 4. 如果修改这行的事务在活跃集合中（未提交）→ 不可见
    if (view.m_ids.contains(rowTrxId)) return false;
    
    // 5. 修改事务已提交（不在活跃集合中，且在 min 和 max 之间）→ 可见
    return true;
}
```

#### RC 和 RR 的 ReadView 生成策略

- **READ COMMITTED**：每次 `SELECT` 都创建新 ReadView → 每次都能看到最新提交的数据 → 其他事务提交的修改立即可见（但会出现不可重复读）
- **REPEATABLE READ**：只在第一次 `SELECT` 时创建 ReadView → 整个事务用同一个 ReadView → 其他事务提交的修改不可见（保证可重复读）

#### MVCC 解决的核心问题

1. **读写不冲突**：SELECT（快照读）不加锁，UPDATE（当前读）加行锁。两者互不阻塞。没有 MVCC 的话，读要么加锁（阻塞写），要么读脏数据。
2. **一致性快照**：通过 Undo 版本链 + ReadView，实现了事务级别的数据快照，实现 RR 隔离级别。

#### MVCC 的局限

MVCC 只解决快照读的并发，当前读（`SELECT ... FOR UPDATE`、`UPDATE`、`DELETE`）还是要加锁。这就是为什么即使有 MVCC，仍然可能出现死锁。

---

### 15. MySQL主键设计原则？自增主键和UUID主键的区别？

#### 主键对存储引擎的影响

InnoDB 中，数据行按主键顺序存储在 B+树叶子节点。主键设计直接影响：
1. **插入性能**（页分裂频率）
2. **索引体积**（主键会出现在每个二级索引的叶子节点中）
3. **查询性能**（主键大小 = 每次 B+树查找的 IO 量）

#### 自增主键 vs UUID vs 雪花

| 特性 | BIGINT 自增 | UUID v4（随机） | 雪花算法（Snowflake） |
|------|-----------|---------------|---------------------|
| 长度 | 8 字节 | 36 字符 / 16 字节（binary） | 8 字节（Long） |
| 插入性能 | 极好（顺序追加） | 差（随机插入 → 页分裂频繁） | 好（趋势递增） |
| 全局唯一 | 单机唯一 | 全局唯一 | 全局唯一 |
| 分布式友好 | 不友好（冲突） | 友好 | 友好 |
| 可排序 | 是 | 否 | 是（按时间） |
| 页分裂 | 极少 | 频繁（树需要不停 rebalance） | 偶尔（同毫秒内可能少量分裂） |

#### UUID 导致页分裂的微观过程

B+树叶子页满了（如 16KB），新数据需要插入页中间 → 页分裂：
1. 分裂到 `16KB / 2 = 8KB` 的新旧两页
2. 调整叶子节点链表
3. 更新父页的指针
4. 整个过程需要写 Redo Log（~32KB）+ 可能递归分裂父页

自增主键：新数据总是插入最右边的叶子页 → 页满了也只需追加新页 → 不需要分裂已有页 → 极少量 Redo Log。

#### 主键设计的 5 条原则

1. **越短越好**：主键出现在每个二级索引的叶子节点。主键从 BIGINT(8B) 换成 UUID(36B) → 一个 4 字段的二级索引膨胀 35%+。
2. **趋势递增**：减少页分裂。雪花算法=趋势递增（时间戳在高位），完全满足。
3. **无业务含义**：永远不要用身份证号、手机号、订单号作为主键。业务规则会变（如「手机号作为主键」→ 用户绑定新手机 → 主键变更 → 所有二级索引更新）。
4. **避免联合主键**：联合主键让所有二级索引变大、让 JOIN 变复杂。
5. **分布式用雪花算法**：机器 ID 用 Redis 分配 or ZK 顺序节点 or K8s StatefulSet 序号。

---

### 16. 线上主从切换失败如何排查和解决？

#### 主从切换流程（MHA / Orchestrator）

1. 检测 Master 不可用（心跳超时 3 次）
2. 选最新 Slave（`Exec_Master_Log_Pos` 最大 / GTID 最新）
3. 补齐差异：选中的 Slave 和其他 Slave 对比 binlog 位置 → 如果还有日志没收到，从选中的 Slave 拉取差异 binlog 补齐（MHA 特有）
4. VIP 漂移到新 Master
5. 其他 Slave `CHANGE MASTER TO` 指向新 Master
6. 旧 Master 恢复后降级为 Slave

#### 切换失败的 5 大原因及排查

**① 数据不一致**
- 症状：切换后新 Master 少数据，或 `Last_SQL_Error: Could not execute ... Duplicate entry`
- 排查：`pt-table-checksum h=master,P=3306,u=root --databases=db_name`
- 解决：`pt-table-sync --print h=master,P=3306 --databases=db_name` → 生成修复 SQL

**② 主从延迟巨大**
- 症状：`Seconds_Behind_Master > 300`（5 分钟）→ 切换后大量数据丢失
- 排查：`SHOW SLAVE STATUS\G` → `Seconds_Behind_Master`
- 解决：等追上再切 / 如果有 MHA 的「差异补全」机制可以缩短窗口

**③ 中继日志损坏**
- 症状：`Last_SQL_Error: Slave SQL thread aborted. Relay log corruption`
- 排查：`SHOW SLAVE STATUS\G` → `Last_SQL_Error`
- 解决：
  ```sql
  STOP SLAVE;
  RESET SLAVE; -- 清空 relay log
  CHANGE MASTER TO MASTER_LOG_FILE='binlog.0000xx', MASTER_LOG_POS=xxxxxx;
  START SLAVE;
  ```

**④ GTID 不一致**
- 症状：MHA 选主时发现 GTID set 不一致（某个 Slave 有不属于当前 Master 的 GTID）
- 排查：对比各个 Slave 的 `SHOW GLOBAL VARIABLES LIKE 'gtid_executed'`
- 解决：
  ```sql
  -- 注入空事务同步 GTID
  SET GTID_NEXT='aaaa-bbbb-cccc-dddd:1-100';
  BEGIN; COMMIT;
  SET GTID_NEXT='AUTOMATIC';
  ```

**⑤ 网络分区 / 脑裂**
- 症状：切换后有两个 Master 同时存在，客户端写到两个 Master → 数据分歧
- 排查：`SHOW SLAVE HOSTS`（看谁注册了自己为 Master）+ 客户端看 VIP 落在哪
- 解决：`SET GLOBAL read_only=ON`（先停止写入），`STOP SLAVE`（断开旧 Master），数据修复后 `START SLAVE`

#### 安全切换的标准流程

```
1. 主库 SET GLOBAL read_only=ON                     -- 停止写入
2. 检查所有从库 SHOW SLAVE STATUS（Seconds_Behind_Master=0）
3. 选中的从库 STOP SLAVE; RESET SLAVE ALL; SET GLOBAL read_only=OFF  -- 提升为主库
4. 其他从库 CHANGE MASTER TO 新主库; START SLAVE
5. 验证（新主库上做一次插入+查询）
6. VIP 切换 / DNS 切换 / Nacos 配置变更
```

---

