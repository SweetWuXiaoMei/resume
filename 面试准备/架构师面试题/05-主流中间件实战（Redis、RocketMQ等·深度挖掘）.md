## 五、主流中间件实战（Redis、RocketMQ等·深度挖掘）

---

### 1. Redis核心数据结构底层实现

#### 每种数据结构的编码演进

```
String:  SDS（简单动态字符串）—— 记录长度 + 预分配空间 + 二进制安全
        - O(1) strlen（元数据中有 len）
        - 预分配空间（len < 1MB → 翻倍；> 1MB → +1MB），减少 realloc
        - 与 C 字符串的最大区别：不依赖 \0 结尾，二进制安全（可存图片）

Hash:   ziplist（键值对少、值短时）→ hashtable（键值对多或值长时）
        阈值：hash-max-ziplist-entries=512, hash-max-ziplist-value=64

ZSet:   ziplist（元素少、值短时）→ skiplist + dict
        跳表节点包含 level[] 数组（概率生成层数）、backward 指针、score、成员
        dict 是 O(1) 按成员查分数
        skiplist 是 O(logN) 按分数范围查

Geo:    内部是 ZSet，用 Geohash 编码经纬度为 score
        附近的人：georadius 底层是 ZSet 的 zrangebyscore

BitMap: 内部是 String（SDS），操作位
        SETBIT key 10000000 1 → 1.25MB 内存
        布隆过滤器可以用 BitMap 实现
```

#### 跳表（Skip List）为什么不用平衡树

1. 实现简单：跳表只需维护前后指针 + 随机生成层数，红黑树需要复杂的旋转和着色
2. 范围查询：跳表找到最小边界后顺着 level[0] 直接遍历，天然支持范围查询
3. 并发友好：跳表可以局部加锁（只锁修改节点附近的节点），平衡树需要全局锁
4. Redis 选择跳表正是因为它的简洁性和范围查询的天然性

---

### 2. Redis脑裂：原因、危害、预防

#### 脑裂的完整成因

```
1. Sentinel 集群和 Master 之间网络断开
2. Sentinel 发现 Master 失联 → 选举新 Master（Slave 提升）
3. 旧 Master 的网络恢复 → 两个 Master 同时存在
4. 客户端可能连接到旧 Master 写入数据
5. 旧 Master 重新被 Sentinel 感知 → 降级为 Slave
6. 旧 Master 上的新数据被全量同步覆盖 → 数据丢失！
```

#### 解决方案的底层原理

```conf
# redis.conf
min-replicas-to-write 1     # 至少有 1 个 Slave 可同步
min-replicas-max-lag 10     # 这 1 个 Slave 的延迟 < 10 秒
```

**为什么这两个配置能防脑裂**：网络隔离后，旧 Master 无法与任何 Slave 同步 → 不满足 min-replicas-to-write=1 → 拒绝所有写操作 → 没有新数据写入 → 降级为 Slave 后无需舍弃任何数据。

**这本质上是 CAP 的取舍**：如果 Master 孤立但仍然接受写入 → AP（可用但数据不一致）。如果 Master 孤立后拒绝写入 → CP（牺牲部分可用性保一致性）。

---

### 3. 缓存一致性：穿透、击穿、雪崩

#### 为什么先更新 DB 后删缓存

```
方案 A：先删缓存 → 再更新 DB
  线程1 删缓存 → 线程2 读（缓存无 → 查 DB 旧值 → 写缓存旧值）→ 线程1 更新 DB 新值
  → 缓存中是旧值！DB 中新值！不一致！

方案 B：先更新 DB → 再删缓存
  线程1 读缓存（miss）→ 查 DB 旧值 → 线程2 更新 DB 新值 + 删缓存 → 线程1 写缓存旧值
  → 缓存中是旧值！DB 中新值！还是会不一致，但概率极低！
  因为：读 DB → 写缓存 的时间（~1ms）vs 更新 DB + 删缓存 的时间
  读操作必须刚好卡在这个窗口内才能中招
```

**兜底方案**：如果删缓存失败 → Canal 监听 binlog → 补偿删除。最终一致性就是靠这种兜底保证的。

#### 缓存击穿的「互斥锁」实现

```java
public Product getProduct(Long id) {
    String cacheKey = "product:" + id;
    Product p = redis.get(cacheKey);
    if (p != null) return p;
    
    // 缓存没有 → 竞争重建锁
    String lockKey = "lock:product:" + id;
    if (redis.setIfAbsent(lockKey, "1", Duration.ofSeconds(10))) {
        try {
            p = db.query(id);
            redis.set(cacheKey, p, Duration.ofMinutes(30));
        } finally {
            redis.del(lockKey);
        }
    } else {
        Thread.sleep(50);  // 没拿到锁自旋等一会
        return getProduct(id);  // 重试读缓存
    }
    return p;
}
```

#### 布隆过滤器的原理

m 位数组 + k 个哈希函数。写入：k 个 bit 位 set 1。查询：k 个 bit 位全为 1 → "可能存在"（有假阳性）；任一为 0 → "一定不存在"（不会漏报）。误判率取决于 m 和 k 的选择。Guava 的 `BloomFilter.create()` API 可以直接用。

---

### 4. RocketMQ消息积压与丢失

#### 消息积压的紧急处理

```
1. 临时新增 Consumer Group + 新建 Topic
2. 原 Consumer 收到消息后不处理业务，直接转发到新 Topic
3. 新 Group 的 N 台 Consumer 消费新 Topic → 快速消化积压
4. 积压清空后恢复原架构
```

这个方案的精髓在于**原消费者只转发不处理（吞吐极高）**，把消化积压的工作交给独立的新集群。

#### 消息丢失的三个环节

**Producer 丢失**：
```java
// 异步发送无回调 → 丢消息
producer.send(msg);

// 同步发送 → 不丢（有异常可以重试）
SendResult result = producer.send(msg);

// 异步 + 失败重试 → 不丢
producer.send(msg, new SendCallback() {
    @Override public void onSuccess(SendResult result) {}
    @Override public void onException(Throwable e) { retry(); }
});
```

**Broker 丢失**：异步刷盘（ASYNC_FLUSH）→ 宕机丢 500ms 数据。同步刷盘（SYNC_FLUSH）→ 不丢但性能差。折衷：主从同步复制（Master 写完等 Slave 确认）。

**Consumer 丢失**：先 commit offset 再处理业务 → 处理失败消息丢失。正确做法：先处理业务成功 → 再 commit offset。消费失败返回 `RECONSUME_LATER`。

---

### 5. ZooKeeper的核心作用、ZAB协议原理？工作中如何使用ZK、遇到过什么问题？

#### 是什么

ZooKeeper 是分布式协调服务，本质是一个**高可用的分布式小文件系统 + 通知机制**。核心能力：存储元数据（树形 ZNode）+ Watch 通知 + 临时节点 + 顺序节点。

#### ZAB 协议（ZooKeeper Atomic Broadcast）

ZAB 是 ZK 专用的原子广播协议。两种模式：

**① 广播模式（正常）**：
- Leader 收到写请求 → 生成 Proposal（带单调递增的 ZXID）
- 发送给所有 Follower → Follower 写入事务日志并 ACK
- 超过半数 ACK → Leader 发送 Commit → 各节点提交

**② 恢复模式（Leader 选举）**：
- 各节点进入 LOOKING 状态 → 投票（投自己）
- 比较 ZXID：选 ZXID 最大的节点（数据最完整）
- 获得超过半数投票 → 成为新 Leader
- ZXID = epoch(高32位) + counter(低32位)，epoch 递增，counter 清零

**为什么 ZK 集群节点数建议奇数**：3 节点可容忍 1 台故障（2 票 > 1.5），4 节点也只能容忍 1 台故障（3 票 > 2，但 4 节点的 2 票不过半）。4 节点不比 3 节点更可用，但多了一台机器成本。

#### 核心使用场景

1. **分布式锁**：临时顺序节点 + Watch 前一个节点。Curator 的 `InterProcessMutex` 封装了此逻辑
2. **服务注册发现**：Dubbo 的服务注册（临时节点，session 断开自动删除）
3. **选主**：多个实例抢着创建同一个临时节点，创建成功的成为 Leader
4. **配置管理**：持久节点存配置 + Watch 通知所有客户端

#### Watcher 机制的问题

- **一次性触发**：Watch 触发后自动删除，需要重新注册
- **非可靠**：网络抖动导致的短暂断连可能导致事件丢失
- 规避：Curator 框架的 `NodeCache` / `PathChildrenCache` 自动重注册 + 重连补偿

#### 坑点

1. **Session 超时**：网络抖动 → session 超时 → 临时节点全删 → 分布式锁全释放。解决办法：设长 session 超时（20s-60s）
2. **ZK 不适合做大容量存储**：每个 ZNode 默认最大 1MB，总数据量超过几个 GB 会严重影响性能（所有数据在内存 + 快照到磁盘）
3. **大量 Watcher 触发风暴**：`PathChildrenCache` 监听父节点 → 所有子节点同时变更 → 客户端被事件风暴淹没

---

### 6. Dubbo的核心架构（Provider、Consumer、Registry、Monitor、Container）与调用流程？如何解决超时、重试、负载均衡？

#### Dubbo 调用全链路

```
Consumer → Registry(订阅服务列表,缓存本地 + 变更通知)
Consumer → [负载均衡选一个Provider] → 发起 RPC 调用(TCP)
  → Provider 线程池处理 → 返回结果
Monitor: Consumer + Provider 异步上报调用统计到 Monitor
```

**Provider 线程模型**：Netty IO 线程（编解码 + 网络收发）→ 扔到业务线程池（默认固定 200 线程）→ 执行具体服务实现 → 返回。如果所有业务线程都在等慢调用 → 新请求排队超时。

#### 解决超时

- `timeout` 参数（默认 1000ms）：查询设 3s，写入设 5s，导出设 30s。按方法粒度设超时，不是全局统一
- 超时后：Consumer 默认重试（见下文重试），Provider 继续执行（不自动取消）

#### 重试策略

- `retries`（默认 2）：失败自动切换（Failover Cluster）
- **重试的致命陷阱**：非幂等接口绝对不能重试！`retries=0`（扣款、减库存）
- 其他容错策略：Failfast（快速失败）、Failsafe（记录日志忽略）、Failback（定时重发）、Forking（并行调多台）

#### 负载均衡策略

- Random（默认加权随机）：按权重随机，权重根据预热时间动态调整
- RoundRobin（加权轮询）：按权重顺序分配
- LeastActive（最少活跃数）：选正在处理请求最少的 Provider，适合长连接
- ConsistentHash：同一参数值打到同一 Provider，适合有状态场景

---

### 7. Redis持久化机制（RDB、AOF）：原理、优缺点、如何选择？

#### RDB（快照）

- 原理：fork 子进程 → 子进程把当前内存数据写入临时文件 → 写完后 rename 替换旧 RDB
- 触发：`save 900 1`（900s 内 1 次修改）、`bgsave` 后台执行、`save` 阻塞执行
- 优点：文件紧凑（压缩）、恢复快（直接加载数据）、fork 后主进程不参与 IO
- 缺点：两次快照之间的数据可能丢失；fork 时 Copy-on-Write 可能导致内存翻倍（写操作多时）

#### AOF（追加日志）

- 原理：记录每条写命令到 AOF 文件（Redis 协议文本），重启时重放命令
- `appendfsync`：always（每命令 fsync，最安全最慢）、everysec（每秒 fsync，折衷）、no（交给 OS）
- 优点：数据安全性高（最多丢失 1-2s 数据）、日志可读
- 缺点：文件大（比 RDB 大几倍）、恢复慢（重放命令 > 加载数据）

#### 混合持久化（Redis 4.0+，生产推荐）

```conf
aof-use-rdb-preamble yes
```
AOF 文件 = RDB 快照（前半部分）+ AOF 增量命令（后半部分）。兼顾恢复速度（RDB 部分）和数据安全（AOF 增量）。

#### AOF 重写

- 为什么需要：AOF 会越写越大（如 `INCR key` 100 次 → 100 条命令）
- 原理：fork 子进程 → 读取内存当前状态 → 生成最小命令集 → 写入新 AOF → 替换旧 AOF
- 触发：`auto-aof-rewrite-percentage 100`（增长 100% 触发）+ `auto-aof-rewrite-min-size 64mb`

---

### 8. Redis集群模式（主从、哨兵、Redis Cluster）各自原理、优缺点、适用场景？

#### 主从集群

- 原理：一主多从，主写从读。手动切换
- 优点：简单（改配置即可）、读写分离提升读能力
- 缺点：手动故障切换（Master 挂了 Slave 只是只读的，需人工 `SLAVEOF NO ONE`）
- 适用：开发环境、小型项目

#### Sentinel（哨兵）

- 原理：Sentinel 进程独立部署（建议 3+ 奇数个），监控 Master + Slave 健康状态。Master 挂了 → Sentinel 选举新 Master → 通知客户端
- 投票机制：超过 `quorum` 个 Sentinel 认为 Master 挂了 → 主观下线 → 再确认 → 客观下线 → 选举新 Master
- 优点：自动故障转移、通知客户端
- 缺点：只有一个 Master 写（写性能上限单机）、扩容只能加 Slave 读

#### Redis Cluster

- 原理：16384 个 hash slot → 均匀分配到多个 Master 节点。每个 key 经过 `CRC16(key) % 16384` 映射到 slot
- 去中心化：节点间通过 Gossip 协议交换状态（谁负责哪些 slot），没有中心节点
- 优点：多 Master 共享写、水平扩展（加节点 → 重分配 slot → 数据自动迁移）、故障自动转移
- 缺点：客户端必须支持 Cluster 协议（JedisCluster/Lettuce Cluster）、跨 slot 操作有限制（mget/multi-key 操作所有 key 必须在同一 slot）
- 适用：大规模、高并发写入场景

#### 选型建议

- < 10G 数据，QPS 不高 → Sentinel
- > 100G 数据，高并发写入 → Cluster
- 极简场景 → 主从 + 应用层读写分离

---

### 9. Redis过期键删除策略（惰性删除、定期删除、内存淘汰）？

- **惰性删除**：访问 key 时检查是否过期 → 省 CPU 但耗内存
- **定期删除**：每 100ms 随机抽 20 个设置了过期的 key → 过期则删除 → 如果 >25% 过期则继续循环 → 限制了每次的执行时间（<25ms）→ 平衡 CPU 和内存
- **内存淘汰**：内存达到 `maxmemory` →
  - `noeviction`：拒绝写
  - `allkeys-lru`：所有 key 中 LRU 淘汰（最常用）
  - `volatile-lru`：有过期时间的 key 中 LRU
  - `allkeys-lfu`：LFU（访问频率最低的淘汰）
  - `volatile-ttl`：即将过期的淘汰

Redis 的 LRU 不是精确 LRU（维护双向链表代价高），而是**近似 LRU**：随机取 N 个 key（`maxmemory-samples`，默认 5），淘汰其中最不活跃的。

---

### 10. Redis Pipeline和事务的区别？适用场景？

- **Pipeline**：打包发送命令 + 批量接收结果，只管网络优化（减少 RTT），不保证原子性。适合批量读写
- **事务（MULTI/EXEC）**：命令入队（MULTI）→ 批量执行（EXEC），保证不被插队（但不管回滚）。适合需要原子性的一组操作
- **Lua 脚本**：真正的原子操作（不会被中断 + 不会回滚部分结果），适合判断 + 操作原子化

---

### 11. RocketMQ架构（NameServer、Broker、Producer、Consumer）与消息发送消费流程？

- **NameServer**：路由中心，无状态，维护 Topic → Broker 的映射。每个 Broker 向所有 NameServer 注册
- **Broker**：消息存储，CommitLog（顺序写 + 所有 Topic 共享）+ ConsumeQueue（按 Topic 分区，只存 20 字节索引——offset + size + tag hash）
- **Producer**：从 NameServer 拉 Topic 路由 → 选 MessageQueue → 发送 → 支持同步/异步/单向
- **Consumer**：长轮询（Pull 模式）→ 拉取消息 → 处理 → 返回 CONSUME_SUCCESS / RECONSUME_LATER

---

### 12. RocketMQ事务消息原理？如何使用事务消息解决分布式事务？

**半消息流程**：
1. Producer 发半消息（存入 `RMQ_SYS_TRANS_HALF_TOPIC`，Consumer 不可见）
2. Broker 返回成功 → Producer 执行本地事务
3. 本地事务成功 → send Commit → 半消息移到真实 Topic → 对 Consumer 可见
4. 本地事务失败 → send Rollback → 半消息删除
5. 超时未确认 → Broker 回调 `checkLocalTransaction` 确认状态

**适用边界**：只解决「Producer 发消息 + 本地事务」的一致性，不是通用的分布式事务（不解决多 Producer 的事务）。

---

### 13. RocketMQ消息过滤机制（Tag过滤、SQL92过滤）？

- **Tag 过滤**：`consumer.subscribe("Topic", "TagA || TagB")`，Broker 端过滤（ConsumeQueue 中存了 tag hash），性能好
- **SQL92 过滤**：`consumer.subscribe("Topic", MessageSelector.bySql("age > 18 AND name IS NOT NULL"))`，Broker 需要解析属性过滤

---

### 14. ZooKeeper节点类型（持久/临时/顺序节点）的作用？

- 持久节点：配置管理，一直存在
- 临时节点：服务注册（session 断开自动删除）
- 持久顺序节点：分布式队列、选主
- 临时顺序节点：分布式锁（最小序号获得锁，Watch 前一个节点）

---

### 15. ZooKeeper Watcher机制原理、缺点与规避？

一次性触发 + 非可靠 + 网络断连丢失事件。Curator 的 `NodeCache` / `PathChildrenCache` / `TreeCache` 自动重注册 + session 重建后补偿。

---

### 16. Dubbo负载均衡策略？如何自定义？

Random（默认加权随机）、RoundRobin、LeastActive、ConsistentHash。自定义：实现 `LoadBalance` 接口 + `@Activate` 注解 → `dubbo.consumer.loadbalance=myLoadBalance`。

---

### 17. Dubbo序列化方式（Hessian、JSON、Protobuf）对比与选择？

| 方式 | 特点 | 性能 | 选用 |
|------|------|------|------|
| Hessian2（默认） | 二进制、跨语言、兼容性好 | 中 | 内部 Java 服务首选 |
| JSON（Fastjson/Jackson） | 文本、可读性好、调试友好 | 低 | 非性能敏感 / 跨语言调试 |
| Protobuf | 二进制、Schema 驱动、IDL 文件 | 极高 | 跨语言 + 高性能场景 |

---

### 18. Redis缓存预热、缓存降级实现方案？

**预热**：`@PostConstruct` + 异步线程 + 分批次加载热点数据到 Redis + ReadinessProbe 检查预热完成状态。
**降级**：Redis 不可用时 → 本地 Caffeine 缓存兜底（短 TTL）+ 返回默认数据/错误页。Sentinel + Cluster 降低降级概率。

---

