## 一、Java核心技术栈（并发、JVM、基础深挖）

---

### 1. JVM内存模型：程序计数器、虚拟机栈、本地方法栈、堆、方法区的作用及OOM场景？

#### 是什么

JVM 运行时数据区分为两大维度去记：**线程私有** 和 **线程共享**。这不是随意划分的，线程私有的区域天然线程安全，随线程生灭；线程共享的区域需要并发控制和 GC 管理。

**程序计数器（PC Register）**：它的本质是「当前线程执行位置的指针」。如果是 Java 方法，存的是正在执行的字节码指令地址；如果是 Native 方法，值为 Undefined。每个线程有自己的 PC，所以上下文切换时 JVM 通过 PC 知道切回来该从哪里继续执行。

**虚拟机栈（JVM Stack）**：栈帧的容器。一个方法就是一个栈帧，栈帧 = 局部变量表 + 操作数栈 + 动态链接 + 方法出口。局部变量表存的是编译期确定的基本类型值和引用指针，它的大小在 Class 文件的 Code 属性中已经写死了（max_locals）。操作数栈是字节码指令执行时的操作区——JVM 是栈式虚拟机，指令从操作数栈 pop 数据操作后 push 回去。

**本地方法栈**：与虚拟机栈功能相同，只是服务对象从 Java 方法换成了 C/C++ 写的 Native 方法。HotSpot 直接把它俩合并了。

**堆（Heap）**：JVM 管理的最大一块内存，存的是对象实例和数组。它为什么需要分代？因为大量对象朝生夕死（弱分代假说）——IBM 的研究表明 98% 的对象活不过第一次 GC。所以推出分代收集：新生代用复制算法（存活少，复制成本低），老年代用整理/清除算法。新生代内部又分 Eden:Survivor0:Survivor1 = 8:1:1，这是因为只有约 10% 的对象能活过第一次 Minor GC。

**方法区/元空间**：存的是类的元数据——类型信息（全限定名、父类、接口）、运行时常量池、字段/方法信息（方法字节码、返回值类型）、静态变量、JIT 编译后的 Native Code。静态变量在 Java 8+ 中被移到了堆中（Class 对象内部），所以方法区本身不再持有静态变量的实例引用。

#### 为什么这样设计

- **线程私有 vs 共享**：私有区域随线程销毁自动回收，避免了锁竞争，这是性能最优解。共享区域集中管理，避免数据冗余。
- **栈 vs 堆分离**：栈用连续内存、LIFO、无碎片；堆需要复杂的 GC 管理。调用栈帧创建销毁极快（指针移动），对象分配则需要空闲列表或指针碰撞算法。
- **永久代为什么被干掉**：永久代在 Java 堆中，大小受 `-XX:MaxPermSize` 限制但很难敲定合适值。设小了遇到大量动态代理会 OOM，设大了浪费堆空间。而且永久代回收效率极低（需要判断类加载器是否整个被回收）。元空间用本地内存后，理论上只受物理内存限制，而且元空间内存按类加载器分组管理，类加载器回收时整块释放。

#### 底层原理

- 栈帧创建：方法调用时，线程从当前栈顶压入新栈帧。字节码 `invokevirtual/ invokespecial/ invokestatic/ invokeinterface` 触发，`return` 系列指令弹栈。
- TLAB（Thread Local Allocation Buffer）：对象分配在堆上，但如果每个线程都去竞争堆的分配指针，性能极差。JVM 在 Eden 区给每个线程分配一小块私有空间（默认占 Eden 1%），线程在 TLAB 内分配对象不用加锁。TLAB 不够时才去 Eden 共享区通过 CAS 抢一块新的。
- 元空间内存模型：元空间由多个 VirtualSpace 组成，每个 ClassLoader 持有一个 Metaspace，内部由 Metablock 链表管理。类卸载时（ClassLoader 被 GC），整个 Metablock 释放回 VirtualSpace 的 chunk free list。

#### 坑点

1. **`-Xss` 设置不当**：线程数 × 栈大小 可能超过物理内存 + swap 上限。栈设太大会限制系统能创建的线程数，设太小会 StackOverflow（递归稍深就挂）。
2. **堆和元空间抢内存**：Java 8 堆 + 元空间共用本地内存，如果不设 `-XX:MaxMetaspaceSize`，元空间无上限增长，可能间接把堆挤爆（操作系统无内存可分给堆扩展）。
3. **直接内存 OOM 难以排查**：`-XX:MaxDirectMemorySize` 默认等于 `-Xmx`，但即使堆很空也可能 OOM——因为 NIO 分配堆外内存不经过 GC。排查需要看 `pmap -x` 和 `-XX:+PrintNMTStatistics`。
4. **StringTable 在堆中**：Java 7 后 `String.intern()` 产生的字符串常量从方法区移到堆中，频繁 intern 可能导致堆爆（很多新人以为还是 PermGen）。

#### 如何回答（面试模板）

"JVM 内存分五个区，我分两个维度讲。线程私有区包括 PC、栈、本地栈，线程共享区包括堆和方法区。堆主要存对象实例，用分代模型管理；方法区 Java 8 后用元空间替代永久代，存类元数据，好处是不再占堆空间、默认无上限但要记得设 MaxMetaspaceSize。OOM 我遇到过三种：堆溢出一般是内存泄漏或 Xmx 设太小；元空间溢出多因为 CGLIB/动态代理生成过多类；直接内存溢出常见于 NIO 没释放 ByteBuffer。"

---

### 2. GC调优：频繁Full GC的排查与解决，如何分析GC日志，如何选择垃圾收集器？

#### 是什么

GC 调优的本质不是调参数，而是三件事：**找出内存异常的对象 → 理解它们的生命周期 → 让它们在该回收的时候能被高效回收**。调参数只是最后一步。

#### 为什么需要调优

Full GC 频繁有两个根本原因：

1. **内存泄漏**：对象不该存活却被强引用持有，老年代只进不出 → 最终撑满 → 连续 Full GC
2. **对象过早晋升（Premature Promotion）**：新生代 Survivor 区太小或晋升阈值太低，导致本该死在年轻代的对象被搬到了老年代 → 老年代被"短寿"对象快速填充 → 频繁 Full GC

过早晋升尤其隐蔽：GC 日志里你会看到大量存活对象被复制到老年代，但这些对象其实很快就不用了，却又只能在下次老年代满时才能回收。这就像你把一天的垃圾都扔进一个要一周才清一次的桶。

#### 排查流程（完整版）

**第一步：宏观判断（jstat）**
```bash
jstat -gcutil <pid> 1000 100
```
关注列：
- **YGC / YGCT**：年轻代 GC 次数和耗时。如果 YGC 很频繁但耗时短（< 10ms），正常。
- **FGC / FGCT**：Full GC 次数和耗时。如果 FGC 占比 YGC 超过 1%，问题严重。
- **S0/S1**：Survivor 区使用率。如果一个始终为 0 或 100%，说明 Survivor 配置有问题或晋升过于激进。
- **OU**：老年代使用趋势。如果每次 YGC 后老年代都增加且不回落，说明有泄漏或过早晋升。

**第二步：GC 日志分析（开启日志）**
```bash
# Java 8
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/path/gc.log
# Java 9+
-Xlog:gc*:file=/path/gc.log:time,level,tags
```
关键信息解读：
- `[GC (Allocation Failure)` → 新生代分配失败触发的 Minor GC，正常
- `[Full GC (Allocation Failure)` → 老年代无法通过 Mixed/Concurrent GC 回收触发，严重
- `[Full GC (System.gc())` → 代码里有人调了 `System.gc()`，需要查出来干掉
- `Desired survivor size / Total survivor size` → 期望大小 vs 实际，差距大说明 Survivor 不够
- `[CMS-concurrent-mark: xxxK->yyyK(zzzK)` → CMS 并发标记的效率指标

用 GCEasy.io 或 GCViewer 可视化 GC 日志，比人眼看快得多。重点关注：GC 暂停时间 P99、吞吐量（GC 时间 / 总时间）、内存分配速率（Allocation Rate，这是根因指标！）。

**第三步：堆 dump 离线分析**
```bash
# 自动 dump (推荐)
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dump/
# 手动 dump (生产慎用,会触发 STW)
jmap -dump:format=b,file=/dump/heap.hprof <pid>
```
MAT 分析三板斧：
1. **Leak Suspects Report**（泄漏嫌疑报告）→ MAT 自动推测
2. **Dominator Tree**（支配树）→ 按 retained heap 排序，找占用最大的对象
3. **Path to GC Roots**（GC Root 路径）→ 追踪哪个 GC Root 引用了这个对象，这是找根因的关键

#### 如何选择垃圾收集器（决策树）

选择 GC 不是越新越好，而是要匹配业务模型：

| 场景特征 | 推荐 GC | 理由 |
|---------|---------|------|
| 单机 < 2G 堆，客户端应用 | Serial | 单线程 STW 时间反而短，内存和 CPU 开销小 |
| 吞吐量优先，批处理，不在乎单次停顿时长 | Parallel | JDK 8 默认，吞吐量最高（CPU 全用于 GC） |
| 堆 4G-32G，要求响应时间可控 | G1 | JDK 9+ 默认，pause time 可预测 |
| 堆 > 16G，要求 < 10ms 停顿 | ZGC | 并发整理，不产生碎片，JDK 15+ 生产可用 |
| 堆 > 8G，要求 < 10ms | Shenandoah | 类似 ZGC，Red Hat 维护 |

**GC 选型的关键认知**：
- **不是堆越大越好**：堆越大，GC 需要追踪的对象越多，单次暂停越长。32G 是堆的一个心理线（超过后指针压缩失效，对象头变大）。
- **G1 不代表停顿小**，它只是**可预测**。设 `-XX:MaxGCPauseMillis=100` 如果满足不了，G1 会自行调整年轻代大小甚至退化为 Full GC 来"尝试满足"，而非一定能满足。

#### G1 核心参数调优

```
-XX:MaxGCPauseMillis=200      # 期望最大暂停时间，不宜设太小(<100ms)
-XX:G1HeapRegionSize=4m       # Region 大小，堆>16G 建议增大
-XX:InitiatingHeapOccupancyPercent=45   # Mixed GC 触发阈值
-XX:G1MixedGCCountTarget=8    # Mixed GC 分多少次完成
-XX:G1ReservePercent=10       # 预留空间防止晋升失败
```

调优核心：**不要乱调，先看日志里的 "To-space exhausted" / "Full GC (Allocation Failure)" 出现频率**，这些才是需要调的信号。

#### 坑点

1. **`System.gc()`**：有人为了"保险"在代码里调用，结果导致 Full GC 频繁。通过 `-XX:+DisableExplicitGC` 禁止。
2. **G1 的 "Humongous 对象" 致命问题**：超过 Region 大小 50% 的对象称为 Humongous Object，直接分配在老年代的连续 Region 中。如果 Humongous 对象频繁创建释放，会触发 G1 的并发标记周期提早开始，甚至导致 Full GC。解决方案：增大 Region 大小或优化代码减少大对象。
3. **CMS 的 Concurrent Mode Failure**：并发清理期间老年代被填满，退化为 Serial Old 做 Full GC，暂停时间几秒到几十秒。解决：降低 CMS 触发阈值（`-XX:CMSInitiatingOccupancyFraction=70`，给浮动垃圾留空间）。
4. **JHipster/Spring Boot DevTools**：热重启会导致 ClassLoader 泄漏，元空间持续增长。生产环境务必关闭 DevTools。
5. **G1 的 "to-space exhausted"**：晋升对象太多，Reserve Region 不够。增大 `-XX:G1ReservePercent` 或减小年轻代。

---

### 3. Java类加载机制：完整流程、双亲委派、打破场景？

#### 是什么

类加载不是「把一个 .class 文件读到内存」这么简单。它是一个**从字节流到 JVM 可使用的类型信息**的完整转换过程。五个阶段各有深意：

**1. 加载（Loading）的三个产出**：
- 通过类的全限定名找到字节流（来自文件系统、网络、ZIP、动态生成均可）
- 把字节流转成方法区能用的运行时数据结构
- 在堆中生成 `java.lang.Class` 对象作为这个类在堆中的「入口」

注意：`Class` 对象和「方法区中的类元数据」是两码事。元数据在方法区，Class 对象在堆中，Class 对象通过指针访问元数据。

**2. 验证（Verification）**：这是 JVM 安全的基石，四层校验：
- 文件格式验证（魔数 CAFEBABE、版本号、常量池类型）
- 元数据验证（是否有父类、是否继承了 final 类、抽象方法检查）
- 字节码验证（类型转换安全、操作数栈/局部变量表匹配）——最复杂的一步
- 符号引用验证（引用的类/字段/方法是否存在、访问权限是否够）

**3. 准备（Preparation）——最容易答错**：
只分配内存并赋**零值**！`public static int value = 123;` 在这个阶段 value = 0，不是 123。例外：被 `final static` 修饰且是静态常量（编译期确定），在准备阶段就赋予正确值（因为它是 ConstantValue 属性）。

**4. 解析（Resolution）**：
符号引用 → 直接引用。JVM 并不要求一定在「解析」阶段完成，可以在首次使用时解析（延迟解析）。符号引用是类文件中的字面量（如 `com/example/User`），直接引用是内存中的实际地址/偏移量。

**5. 初始化（Initialization）**：
执行 `<clinit>()` 方法——JVM 把类中所有的静态变量赋值语句和 static{} 块按代码顺序合并成一个方法。`<clinit>()` 是线程安全的（JVM 加锁保证），所以静态域初始化天然线程安全，但这也可能导致死锁（见下文坑点）。

#### 双亲委派模型原理

不是一个简单的「先问爸爸」——它的核心是**线性层级 + 逆向代理**：

1. 每个类加载器有固定的 parent（构造函数传入）
2. `loadClass(name)` 调用时先检查是否已加载 → 没有则调 parent.loadClass(name)
3. 一直向上到 Bootstrap ClassLoader（最顶层，C++ 实现，没有对应的 Java 对象，`getParent()` 返回 null）
4. Bootstrap 尝试从核心库（rt.jar / modules）加载 → 失败 → 下沉
5. Extension 尝试加载（Java 8）→ 失败 → 下沉
6. Application 尝试从 classpath 加载 → 失败 → 下沉
7. 自定义加载器 findClass → 失败 → ClassNotFoundException

**三层类加载器**（Java 8）：
```
Bootstrap ClassLoader (C++)       ← 加载 rt.jar, 核心 JDK 类
    ↑ parent
Extension ClassLoader (Java)      ← 加载 jre/lib/ext
    ↑ parent
Application ClassLoader (Java)    ← 加载 classpath
    ↑ parent
User-defined ClassLoader (Java)
```

Java 9+ 引入了模块化，三层变成了 PlatformClassLoader（替代 Extension）+ AppClassLoader + Bootstrap，但委派本质不变。

#### 为什么需要双亲委派（两个核心原因）

1. **安全性——保护核心 API**：没有双亲委派，我可以自己写一个 `java.lang.String` 放在 classpath，你的应用可能错误加载了我的假冒类。有了双亲委派，加载 `java.lang.String` 的请求必然到达 Bootstrap，Bootstrap 加载的是 JDK 的正版 String。
2. **避免重复加载**：同一个类如果被不同的加载器加载，JVM 认为它们是两个不同的类（JVM 用「类的全限定名 + 类加载器」唯一标识一个类）。两个 String 同时存在会让 instanceof、类型转换全乱套。

#### 实际需要打破的场景

**场景一：SPI（Service Provider Interface）**

问题本质：接口在核心 API（Bootstrap 加载），实现在 classpath（Application 加载）。按双亲委派，Bootstrap 加载的类访问不到 Application 加载的类。

以 JDBC 为例：
- `java.sql.DriverManager` 在 rt.jar，由 Bootstrap 加载
- MySQL 的驱动 `com.mysql.cj.jdbc.Driver` 在 classpath，由 Application 加载
- DriverManager 需要加载 MySQL 驱动实现，但它不能向下委派（parent 不理解 child 的类空间）

解法：`Thread.currentThread().getContextClassLoader()`——DriverManager 通过当前线程的上下文类加载器（默认是 Application ClassLoader）去加载驱动实现类。这是 Java 标准库自己打破双亲委派的经典案例。

**场景二：Tomcat 类加载隔离**

Tomcat 为什么不用简单的双亲委派？因为多个 Web 应用部署在同一 Tomcat：
- 每个应用有自己的类隔离（应用 A 用 Spring 5，应用 B 用 Spring 6，互不干扰）
- 应用之间共享通用库（Servlet API）
- 支持热部署（替换 war 包时只重建该应用的类加载器）

Tomcat 的 WebappClassLoader 打破了双亲委派，优先自己加载，只有加载不到时才交给上层（反向委派）。核心 API（java.*）例外，永远不自己加载。

**场景三：模块热替换（如开发环境的 JRebel、生产环境不停机更新）**

核心思路：每个模块独立的 ClassLoader，更新模块时丢弃旧的 ClassLoader 创建新的。对象无法跨 ClassLoader 交互（类型不兼容），需要用接口隔离。

#### 坑点

1. **`Class.forName()` vs `ClassLoader.loadClass()`**：
   - `Class.forName(name)` 默认会执行初始化（`<clinit>`），即触发静态代码块！这在加载 JDBC 驱动时是需要的（Driver 的 static 块里注册了自己）。
   - `ClassLoader.loadClass(name)` 只走到加载+验证+准备+解析，不初始化。
   - 经典 bug：用 `loadClass` 加载 JDBC 驱动，驱动没有注册自己，报错"找不到合适的驱动"。
   - Spring 中用 `ClassUtils.forName()` 往往用的是 `ClassLoader.loadClass` 行为。

2. **静态初始化块死锁**：`<clinit>()` 方法是线程安全锁的，但两个类如果相互依赖且在不同线程中加载，可能导致死锁。例如类 A 的 static 块里 new 类 B 的对象，类 B 的 static 块里 new 类 A 的对象。

3. **同一个 jar 被两个 ClassLoader 加载**：出现 `ClassCastException: com.example.User cannot be cast to com.example.User`。因为 JVM 认为它们是两个不同的类（`类 = 全限定名 + ClassLoader`）。

4. **jar 冲突/包冲突**：Maven 传递依赖导致同一个包名+类名出现在两个 jar，ClassLoader 先找到哪个就用哪个，可能行为不一致。`mvn dependency:tree` 排查。

---

### 4. Java并发三大核心问题：可见性、原子性、有序性；synchronized和volatile的区别与底层实现？

#### 可见性（Visibility）

**是什么**：线程 A 修改了变量 x，线程 B 在某个时刻后能否读到这个修改。

**根本原因**：现代 CPU 的缓存架构。每个 CPU 核心有独立的 L1/L2 Cache，线程运行在不同的核上，对共享变量的修改可能停留在各自核的 Cache 中没有刷回主存（或另一个核的 Cache 没有失效）。这就是「缓存一致性问题」。

**为什么需要**：没有可见性保证，多线程之间的状态同步完全不可靠。比如一个线程设置 `flag = true` 通知另一个线程退出循环，如果没有 volatile，第二个线程可能永远循环下去。

**JMM 的解决**：JVM 通过内存屏障（Memory Barrier/Fence）提供可见性保证。volatile 的写操作后插入 StoreLoad 屏障——作用是把 Store Buffer 里的数据**强制刷入**主存（通过 x86 的 `lock` 前缀指令）。

**底层细节（x86 的 MESI 协议）**：CPU Cache 之间的数据一致通过 MESI（Modified/Exclusive/Shared/Invalid）协议保证。一个核心写某缓存行，其他核心的该行标记为 Invalid，读时必须从主存或写者的 Cache 拿最新数据。volatile 就是利用了这套硬件协议。

#### 原子性（Atomicity）

**是什么**：一组操作要么全部执行，要么全不执行，中间不允许被插队。

**根本原因**：线程切换。`i++` 在 Java 层面是一行，但在 JVM 层面是三条字节码指令（`getfield → iadd → putfield`），线程可能在任意一条执行完后失去 CPU。两个线程同时 `i++` 100 次，最终结果可能小于 200。

**为什么 volatile 不能保证原子性**：volatile 只保证「读可见」、「写可见」，不保证「读-改-写」中间不被打断。`count++` 的问题在于，当线程 A 读了 volatile 变量准备+1时，线程 B 也读了同一个值，两个线程都基于旧值计算，最终各写回 1 而不是 2。这叫「读-改-写竞态」。

**解决方案**：
- JDK 1.5+ 的 `AtomicInteger/AtomicLong`：基于 CAS + 自旋。`compareAndSwap(expect, update)`，CPU 级别原子指令（`cmpxchg` + `lock` 前缀）。
- CAS 的 ABA 问题：值从 A 变成 B 再变成 A，CAS 认为没变，实际已经不是原来的 A。`AtomicStampedReference` 加版本号解决。

#### 有序性（Ordering）

**是什么**：代码编写的顺序与实际执行的顺序一致。

**根本原因**：重排序。两个层面：
1. **编译器重排序**：JIT 编译器在不影响单线程最终结果的前提下，可能重新安排指令顺序（数据依赖是底线）。
2. **处理器重排序**：CPU 的 Store Buffer / Load Buffer 会打乱内存访问的实际顺序（store-load 重排）。

**为什么需要禁止**：在单线程中重排序是完美的优化，但在多线程中它就是 bug 的源头。经典的 DCL 单例问题就是重排序导致的——先赋值 reference 再初始化对象，另一个线程看到的 reference 不为 null 但对象还没初始化好。

**happens-before 规则（JMM 的核心）**：
JMM 不保证对所有线程一致的「时间」顺序，而是靠 happens-before 关系：
- 程序次序规则：同一线程中，前面的操作 happens-before 后面的
- volatile 规则：volatile 写 happens-before 后续的 volatile 读
- 锁规则：unlock happens-before 后续的 lock
- 传递性：A happens-before B，B happens-before C → A happens-before C

#### synchronized 底层实现（完整版）

**字节码层**：
- 方法级同步：方法描述符中 `ACC_SYNCHRONIZED` 标志
- 代码块同步：`monitorenter` 和 `monitorexit` 指令（`monitorexit` 会出现两次——正常退出 + 异常退出）

**对象头（HotSpot 64bit）**：
```
Mark Word (64 bits):
| 锁标志位(2bits) | 锁信息(62bits) |
  01 → 无锁 / 偏向锁
  00 → 轻量级锁（存指向栈中 Lock Record 的指针）
  10 → 重量级锁（存指向 Monitor 的指针）
  11 → GC 标记
```

**锁升级（膨胀）流程——这就是为什么"重量级锁重"**：

**偏向锁（Biased Locking）**：Mark Word 存「偏向线程 ID」。同一个线程再次进入同步块时，检查偏向 ID 是否是自己 → 是，则无需任何 CAS 即可进入（零开销）。有线程竞争时，撤销偏向锁需要等待全局安全点（SafePoint），STW！这就是为什么 JDK 15 默认关闭偏向锁——在大多数现代应用（高并发、锁竞争常见）中，偏向锁的撤销成本反而高于收益。

**轻量级锁**：没有竞争或竞争轻微（线程交替执行）。线程在栈帧中创建 Lock Record → 通过 CAS 把对象头的 Mark Word 替换为指向 Lock Record 的指针。CAS 成功 → 获取锁成功。CAS 失败 → 自适应自旋（在 CPU 上空转等持有者释放）。自旋无果 → 膨胀为重量级锁。

**重量级锁**：依赖于操作系统的互斥量（Mutex Lock + Condition Variable）。ObjectMonitor 对象（C++）管理线程的阻塞和唤醒。阻塞/唤醒本质是系统调用（`futex` on Linux），上下文切换代价很高（数万 CPU 周期）。

**synchronized 的三次进化**：
- JDK 1.0：只有重量级锁 → 性能很差
- JDK 1.6：引入偏向锁、轻量级锁、适应性自旋 → 性能大幅提升
- JDK 15：默认关闭偏向锁 → 考虑现代应用高并发的现实

**Mark Word 的坑**：`hashCode()` 方法和偏向锁冲突。偏向锁状态下没有地方存 identity hash code。如果对象曾经计算过 hashCode（Mark Word 中有 hash），就无法进入偏向锁状态；正在偏向锁状态的对象如果调了 `hashCode()`，偏向锁撤销→升级为重量级锁。

#### volatile 底层实现（深入）

**JMM 定义的屏障类型**：
| 屏障 | 含义 |
|------|------|
| LoadLoad | Load1; LoadLoad; Load2 → Load2 读取前 Load1 一定已读 |
| StoreStore | Store1; StoreStore; Store2 → Store2 写入前 Store1 一定全局可见 |
| LoadStore | Load1; LoadStore; Store2 → Store2 写入前 Load1 一定已完成 |
| StoreLoad | Store1; StoreLoad; Load2 → Load2 读取前 Store1 一定全局可见 |

**volatile 插入屏障策略**：
- volatile 写：前面插 StoreStore → 保证之前的写都不被重排到后面；后面插 StoreLoad → 保证本次的写对后续的读可见
- volatile 读：后面插 LoadLoad + LoadStore → 保证后续的读写不被重排到读前面

**x86 实现**：x86 是强内存模型，只存在 StoreLoad 重排。所以 **volatile 的成本主要在 StoreLoad 屏障**——x86 上实现为 `lock addl $0, (%rsp)`（往栈上写 0 并 lock 前缀），这会清空 Store Buffer，保证它之前的 store 全局可见。

**volatile 为什么"轻量"**：它不阻塞线程（没有内核态切换），代价只是 StoreLoad 屏障清除 Store Buffer 的操作（几十到几百 CPU 周期）。对比重量级锁的系统调用（几万周期），确实一个量级的差距。

#### 如何组织这个问题的面试回答

"并发编程的三个核心问题是可见性、原子性、有序性。可见性来自 CPU 缓存，volatile 通过内存屏障解决；原子性来自线程切换，synchronized 和悲观锁/CAS 乐观锁解决；有序性来自编译器+CPU 重排序，volatile 和锁都能保证。synchronized 底层有锁升级过程：偏向锁→轻量级锁→重量级锁，首次进入偏向后无需 CAS 进入，有竞争后升级到轻量级锁自旋等待，再不行升级到重量级锁调用系统 mutex。volatile 底层是内存屏障，x86 用 lock 前缀实现，通常被当做轻量级锁用。但 volatile 不能保证原子性，比如 count++，因为读-改-写是复合操作。"

---

### 5. 线上JVM卡顿、OOM故障排查：工具、步骤、根因分析？

#### 为什么 OOM 排查是架构师核心能力

OOM 的排查不是「dump 文件出来用 MAT 看一下」这么简单。真正的难点在于：
- 线上 OOM 往往不是一次性的，而是**渐进式泄漏**——监控里 JVM 内存曲线缓缓上升，直到某天触发 OOM
- OOM 不是 root cause，OOM 只是「堆被占满了的表现」，根因可能是内存泄漏、大对象、GC 策略不对、元空间超限、直接内存未释放等
- 生产环境不能随便 `jmap -dump:live`（会触发 Full GC！）

#### 完整排查流程（操作系统层 → JVM层 → 应用层）

**第一步：保留现场**

如果没有开启 `-XX:+HeapDumpOnOutOfMemoryError`，这是第一要改的事。但这里有个坑：live 对象太多了，dump 过程可能 OOM 本身还没触发完进程就被 kill 了。可以在 OOM 前人工介入：

```bash
# 先别急着 dump，先采集轻量信息
jstat -gcutil <pid> 1000 10        # GC 统计
jstack <pid> > jstack.log           # 线程堆栈
top -Hp <pid>                       # 线程 CPU 占用
# 确定不是假性 OOM（比如实际上没有 OOM，只是响应慢）后再 dump
```

**第二步：按 OOM 类型分类排查**

堆 OOM：
```
java.lang.OutOfMemoryError: Java heap space
```
→ `jmap -histo:live <pid> | head -20` 看哪些类实例最多、占内存最大。这是什么概念呢？比如结果第一位是 `[B`（byte 数组），说明有人在大量分配字节数组（可能是上传文件、读网络流未释放）。

→ 拿 dump 到 MAT 分析。MAT 的核心是 **Leak Suspects Report**——它会自动找出嫌疑对象并给出一句话摘要。

→ 追踪 **Dominator Tree**（支配树）：找出实际 ref 住最多内存的对象。比如一个 HashMap 实际保留了 10G 堆内存，看 GC Root Path → 一步步溯源到是哪个线程、哪个框架注入的引用。

元空间 OOM：
```
java.lang.OutOfMemoryError: Metaspace
```
→ `jstat -gc <pid>` 看 MU/MC 是否持续增长

→ 场景一：CGLIB/动态代理 —— 每个代理类都占用元空间（MyBatis Mapper 代理、Spring AOP 代理、Hibernate 动态代理）

→ 场景二：Groovy 脚本动态执行 —— 每次编译生成类

→ 场景三：Lambda 表达式的 `LambdaMetaFactory` 生成匿名类

→ 用 `-XX:+TraceClassLoading -XX:+TraceClassUnloading` 观察哪些类在加载，哪些从未被卸载

直接内存 OOM：
```
java.lang.OutOfMemoryError: Direct buffer memory
```
→ 常见原因：NIO 的 `ByteBuffer.allocateDirect()` 分配堆外内存后，GC 只能通过虚引用（PhantomReference + Cleaner）间接回收。如果 YGC/Full GC 太不频繁，堆外内存一直不回收——**这就是为什么明明没设 MaxDirectMemorySize，堆内存也远没满，但报告 Direct buffer memory OOM**。

→ 排查：需要 NMT（Native Memory Tracking）：`-XX:NativeMemoryTracking=detail`，`jcmd <pid> VM.native_memory detail`

无法创建本地线程：
```
java.lang.OutOfMemoryError: unable to create new native thread
```
→ 不是堆满！是操作系统层面限制了线程数（ulimit -u / pid_max / vm.max_map_count）

→ `ps -eLf | wc -l` 看当前线程数，`ulimit -u` 看上限

#### 常用工具全解

| 工具 | 场景 | 核心用法 | 注意事项 |
|------|------|---------|---------|
| **jstat** | 实时监控 GC | `jstat -gcutil <pid> 1000` | 无入侵，首选 |
| **jstack** | 线程堆栈/死锁 | `jstack <pid>` | 关注 `BLOCKED` / `WAITING` 线程 |
| **jmap -histo** | 排查 OOM | `jmap -histo:live <pid>` | `:live` 触发 Full GC |
| **jmap -dump** | 离线分析 | `jmap -dump:live,...` | 生产慎用，会长时间 STW |
| **Arthas** | 线上实时诊断 | dashboard/thread/watch/trace/tt | 阿里开源，无侵入（attach） |
| **MAT** | dump 分析 | Dominator Tree / Leak Suspects | 需要大内存加载 dump |
| **jcmd** | NMT | `jcmd <pid> VM.native_memory summary` | 需要开启 NMT |
| **GC Easy** | GC 日志分析 | 上传 gc.log | 给出优化建议 |

#### 线上 Case 汇总

**Case 1：CompletableFuture 线程池泄漏**

现象：slowly growing heap → 偶尔 OOM。heap dump 发现大量 `ForkJoinWorkerThread`。

根因：使用了 `CompletableFuture.supplyAsync()` 而没有传自定义 Executor，导致所有异步任务跑到 ForkJoinPool.commonPool()。如果 commonPool 中的某个任务因为依赖资源挂起，大量请求快速创建新线程（ForkJoinPool 的补位机制），线程栈内存占用持续增长。

修复：所有 CompletableFuture 传自定义线程池 + 限制池大小。

**Case 2：Fastjson 的 ASM 类加载泄漏**

现象：元空间 OOM，`TraceClassLoading` 发现大量 `com.alibaba.fastjson.asm.*` 的类。

根因：Fastjson 1.x 每次序列化时用 ASM 动态生成序列化类，在某些版本中这些类没有被正确卸载（ClassLoader 没被回收，导致元空间一直增长）。

修复：升级 Fastjson 版本或改用 `JSON.parseObject` 而非 ASM-based 方式。

**Case 3：Arthas 定位死循环**

现象：CPU 100% 但 GC 正常，没有 OOM。

排查：`dashboard` → 发现 CPU 最高的线程 → `thread <id>` → 定位到某段 parse 代码。追溯代码发现输入数据中含有特殊字符触发了一个 while 循环的无限解析。

修复：加输入校验 + parse 超时机制。

---

### 6. 线程池核心参数、大小设计、异常排查？

#### ThreadPoolExecutor 的执行流程（为什么这个顺序）

```
新任务 → 当前线程数 < corePoolSize？→ 是 → 创建核心线程执行
                              → 否 → 队列满了没？→ 没满 → 入队
                                                  → 满了 → 当前线程数 < maxPoolSize？→ 是 → 创建新线程
                                                                                      → 否 → 拒绝策略
```

**为什么是这样的顺序**？先队列，后最大线程数。这个顺序意味着：**corePoolSize 的线程能处理多少就是多少，忙不过来的先进队列排队，只有队列满了才临时扩容**。这也意味着如果用无界队列（`LinkedBlockingQueue()` 无参），maxPoolSize 就是个摆设——队列永远不会满，永远不会扩容。

#### 核心参数辨析

| 参数 | 作用 | 常见误用 |
|------|------|---------|
| corePoolSize | 常驻线程数（默认不回收） | 设太小→队列积压；设太大→CPU 浪费 |
| maxPoolSize | 最多「同时」运行的任务数 = max + 队列容量 | 不等于"线程上限"（线程上限受队列+拒绝策略影响） |
| keepAliveTime | 多余线程的存活时间 | `setAllowCoreThreadTimeOut(true)` 后核心线程也可回收 |
| workQueue | 任务排队区 | 三种队列区别极大：有界/无界/同步 |
| RejectedExecutionHandler | 拒绝策略 | CallerRunsPolicy 不丢任务，有反压效果 |

#### 线程数量设计（不只是公式）

**CPU 密集型**：
真的 CPU 密集型是极少的（密码学计算、图像处理）。多数看起来 CPU 密集的业务（如复杂查询）其实等待 IO/DB。所以公式 `CPU 核数 + 1` 只适用于几乎没有 IO 的场景。
- 关键：线程数过多导致频繁上下文切换，整体吞吐量下降

**IO 密集型**：
如果线程把 80% 时间花在等待 IO（数据库、RPC、文件），则 `线程数 = CPU 核数 × (1 + 0.8/0.2) = CPU 核数 × 5`。但前提是这些线程确实在等待不同的 IO（不共享同一瓶颈）。

**实际经验（比公式重要）**：
1. 压测决定：用 JMeter 从低并发开始逐步增加，找到吞吐量不再增长的点
2. 监控反馈：Micrometer 暴露 `pool.size` / `pool.queue.size` / `pool.queue.wait_time` 给 Prometheus，Grafana 可视化成趋势图
3. 动态调整：美团有动态线程池方案，业务高峰期动态扩核心线程、低峰期恢复，避免空闲浪费

#### 坑点

1. **Executors.newFixedThreadPool 和 newCachedThreadPool 的隐藏陷阱**：
   - `newFixedThreadPool(n)` 内部用的是 `LinkedBlockingQueue()`（无界！），任务无限排队、OOM
   - `newCachedThreadPool()` 内部用的是 `SynchronousQueue` + `Integer.MAX_VALUE` 的 maxPoolSize，线程无限增长
   - 阿里规约：**永远不用 Executors 工厂方法**，手动 `new ThreadPoolExecutor(...)` 并指定有界队列

2. **CallerRunsPolicy 的双刃剑**：
   - 好处：提供反压（拒绝的任务由调用线程执行，调用方自然会慢下来）
   - 坏处：如果调用线程是 Netty IO 线程或主业务线程，执行慢任务会阻塞其他请求——雪崩
   - 解决：只在非 IO 线程池中使用 CallerRunsPolicy

3. **线程池预热**：
   - 容器重启后前几秒，corePoolSize 有空闲但队列积压？`prestartAllCoreThreads()` / `prestartCoreThread()`

4. **ThreadLocal + 线程池 = 数据污染**：
   - 线程池中线程会复用，上一个任务留下的 ThreadLocal 没 remove，下一个任务在同一线程中能读到。这是非常难排查的 bug。
   - 解决：所有 ThreadLocal 必须在 finally 中 remove；或全局使用 `TransmittableThreadLocal`

5. **shutdown vs shutdownNow**：
   - `shutdown()`：优雅关闭，新任务拒绝，已有任务运行完
   - `shutdownNow()`：暴力打断了，返回未执行的任务列表；但不能保证一定能终止（`Thread.interrupt()` 只设置中断标志，需代码响应）

---

### 7. Java中的锁种类：偏向锁、轻量级锁、重量级锁、自旋锁、公平锁/非公平锁、可重入锁、读写锁、StampedLock？

#### synchronized 的锁升级详解

**偏向锁（JDK 15 之前默认开启，之后默认关闭）**：

原理：对象头的 Mark Word 中记录「偏向线程 ID」。目的：大多数场景下锁总是被同一个线程获取，那么每次 lock 时只需要检查一下 Mark Word 里的 thread ID 是不是自己——是的话就什么都不用做（零开销）。

偏向锁前提：对象没有计算过 identity hashCode（计算过 hashCode 后 Mark Word 存不下偏向线程 ID + epoch + age + 锁标志）。

撤销代价：偏向锁「撤销」必须等到全局安全点（所有线程停在 SafePoint），然后把对象头的锁改回无锁或升级为轻量级锁。这在高并发场景下是一个冷启动惩罚。

**轻量级锁**：

用于没有实际竞争、仅交替执行的情况。线程在栈帧创建 Lock Record，把 Mark Word 复制到 Lock Record 中（称为 Displaced Mark Word），然后 CAS 把 Mark Word 指向 Lock Record。CAS 失败意味着有竞争。

如果当前持锁线程还在运行：自旋等待（JVM 自适应自旋）。

**重量级锁**：

自旋无果 → 通过 `inflate`（膨胀）把轻量级锁升级为重量级锁。JVM 创建 ObjectMonitor（C++ 对象），内部有 `_owner`（当前持锁线程）、`_EntryList`（阻塞队列）、`_WaitSet`（wait 队列）。线程阻塞是调用 `pthread_mutex_lock` + `pthread_cond_wait`（或 Linux futex）。

**为什么要升这么多级**：因为绝大多数锁事实上一辈子都只有一个线程在获取；即使有竞争，大部分情况下也是一两个线程交替执行（不是真正的同时竞争）。真正的内核态阻塞只发生在极少数情况下。锁升级是自适应的——从最快的无锁/偏向，逐步叠加复杂性，只在必要时付出重量级代价。

#### 公平锁 vs 非公平锁

AQS 中实现的关键区别：

非公平锁（`ReentrantLock()` 默认）：
```java
final void lock() {
    if (compareAndSetState(0, 1))     // 先 CAS 抢一次
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);                    // 抢不到再排队
}
```

公平锁（`ReentrantLock(true)`）：
```java
final void lock() {
    acquire(1);   // 老老实实排队
}
// acquire → tryAcquire 中判断：
// hasQueuedPredecessors() → 队列中前面有人吗？有就让位
```

不公平策略的"偷跑"只在锁刚释放那一刻发生。如果队列中已经有人在排队，新来的线程 CAS 抢一次，抢不到就入队。所以非公平锁不是完全无序的，只是容许轻微插队。

**性能差异**：公平锁每次都要切换线程（从持有者 → 队首等待者），上下文切换成本高。非公平锁允许刚释放锁的线程再次获得锁——这个线程刚跑完，Cache 和 TLB 还热着，新请求可以直接复用。

#### StampedLock（JDK 8+）

解决 ReadWriteLock 的「写锁饥饿」问题。ReadWriteLock 的致命问题：大量读线程持续持有读锁，写锁永远获取不到（因为写锁需要等待所有读锁释放）。

StampedLock 的三种模式：
- **乐观读**（tryOptimisticRead）：返回版本号 stamp，然后不阻塞地去读。读完校验 stamp 是否有效（这期间有没有写操作）。如果校验失败回退为悲观读。绝大多数据场景下写不频繁，乐观读一次就成功了。
- **悲观读**（readLock）：类似传统读锁
- **写锁**（writeLock）：类似传统写锁

```java
// 乐观读范式
long stamp = sl.tryOptimisticRead();
int a = x; int b = y;   // 读数据
if (!sl.validate(stamp)) {
    stamp = sl.readLock();   // 回退悲观读
    try { a = x; b = y; }
    finally { sl.unlockRead(stamp); }
}
return a + b;
```

**StampedLock 的坑**：
1. 不可重入！持有锁时再 lock 会死锁（不是重入，是自己把自己锁住）
2. 不支持 Condition（线程间协调只能用 LockSupport）
3. 读锁和 writeLock 返回的 stamp 必须正确 unlock，传错会抛异常

#### 如何回答这道题

"锁分类可以从两个维度讲。一、synchronized 的锁升级：偏向锁（单线程场景，JDK15默认关闭）→轻量级锁（交替执行，CAS+自旋）→重量级锁（真正竞争，内核态互斥）。二、JUC 的锁：ReentrantLock 基于 AQS+volatile state+CAS，分公平和非公平两种——非公平吞吐高因为释放锁的线程刚用完可以立刻重入，缓存热；ReadWriteLock 读共享写排他但有写锁饥饿问题；StampedLock 通过乐观读缓解了这个问题，代价是不可重入。"

---

### 8. volatile内存语义、为什么不能原子性、DCL单例？

#### 内存语义（JMM 层面）

volatile 的核心是两个 happens-before 规则：
1. volatile 写 **happens-before** 后续的 volatile 读
2. volatile 写之前的操作，对 volatile 读之后的操作可见（通过传递性）

这意味着什么？如果一个线程写 volatile 变量 `flag = true`，在此之前写入的普通变量（如 `data = 123`）也会被刷新到主存。当另一个线程读到 `flag == true` 时，它也能读到 `data == 123`。这就是 volatile 作为「轻量级同步」的真正价值。

**内存屏障详解**：

volatile 写编译后等价于：
```
[写操作之前的 StoreStore 屏障]
普通写操作...
StoreStore 屏障  ← 保证上面的写不被重排到下面
volatile 变量 = xxx;
StoreLoad 屏障   ← 最重的屏障，保证本次写对后续读可见
                ← x86: lock addl $0, (%rsp)
```

volatile 读编译后等价于：
```
LoadLoad 屏障
val = volatile 变量;
LoadStore 屏障
```

**x86 的优势**：x86 是强内存模型（TSO: Total Store Order），只允许 StoreLoad 一种重排。所以 x86 上 volatile 写只需要 StoreLoad 屏障，而 ARM 上需要全部 4 种屏障。这也是为什么 volatile 在 ARM 上的性能开销可能大于 x86。

#### DCL（Double-Checked Locking）深入

单例对象的创建 `instance = new Singleton()` 有三步：
1. 分配内存空间（JVM 内）
2. 调用构造器初始化对象
3. 将引用 instance 指向分配的内存

步骤 2 和 3 之间没有数据依赖！JIT 编译器可以重排为 1→3→2。如果线程 A 执行 3 后被切走（instance 已非 null 但对象未初始化），线程 B 看到 instance 不为 null，拿到一个未初始化完的对象。

**volatile 怎么解决**：volatile 的 StoreStore 屏障禁止了「写 instance 指针」被重排到「构造对象（写对象的成员变量）」之前。

**除了 volatile，还有什么办法**：
1. 静态内部类（Holder）：`Holder.INSTANCE`，JVM 的 `<clinit>` 锁天然线程安全 + 延迟加载
2. 枚举：`enum Singleton { INSTANCE; }`，防反射、防反序列化攻击、绝对安全的单例
3. 饿汉式：静态变量 = new，简单粗暴但能保证安全

**volatile 在 Singleton 中的另一个作用**：即使 volatile 解决了 new 的重排序，还有一个隐藏问题——CPU 缓存。没有 volatile，线程 B 可能因为缓存看不到 instance 已经被赋了新值。volatile 的可见性保证了 B 能看到最新的 instance。

---

### 9. JVM垃圾收集算法：标记-清除、标记-复制、标记-整理、分代收集？

#### 标记-清除（Mark-Sweep）

**原理**：可达性分析标记所有存活对象 → 遍历堆，清除未标记对象 → 加入空闲列表（free list）。

**为什么会有碎片**：因为只清除死对象，不移动活对象。被清除的对象散布在堆各处，形成不连续的空闲空间。后续分配大对象时，即使总空闲空间足够，找不到一块足够大的连续空间 → 触发 GC → 循环 → 最终 OOM。

**适用**：这是垃圾收集的萌芽，没有收集器直接用这个算法。CMS 用的是「标记-清除」但加了并发清除。

#### 标记-复制（Mark-Copy / Semispace）

**原理**：堆分为两半（From 和 To 空间）。只使用 From，GC 时标记 From 中存活对象 → 复制到 To → 清空 From → From/To 交换。

**内存浪费 vs 分配效率**：浪费 50% 内存换来极致分配效率（只靠指针碰撞 bump-the-pointer，不需空闲列表）。对象存活率低时，只复制少量活对象，复制成本低。

**Appel 式回收（HotSpot 新生代的方案）**：不搞 50-50，而是 8:1:1（Eden : S0 : S1），当 Survivor 不够用时借用老年代（分配担保）。为什么要 8:1:1？实际测试约 98% 对象朝生夕死，10% 的 Survivor 足够兜底。极少数情况不够用，由老年代分配担保（Handle Promotion）。

#### 标记-整理（Mark-Compact）

**原理**：标记存活对象 → 将所有存活对象滑动到内存一端 → 清理边界外的内存。

**为什么新生代不用这个**：新生代的对象死亡率太高了。如果存活 2%，你要复制那 2% 还是移动那 98% 再清理？复制 2% 显然比移动 98% 快得多。这就是为什么新生代用复制、老年代用整理。

**整理算法变种**：
- 顺序整理（Serial）：直接搬移，STW 期间所有事情做完
- 并发整理（ZGC/Shenandoah）：使用读屏障 + 着色指针，在应用线程运行的同时移动对象

#### 分代收集的理论基础

**弱分代假说（Weak Generational Hypothesis）**：绝大多数对象在分配后很快死去，活过多次 GC 的对象倾向于持续存活。

**跨代引用假说**：老年代引用新生代的情况很少（Intergenerational Reference）。这很重要！如果没有这个假说，每次 Minor GC 都需要检查整个老年代的所有引用。实际上，Minor GC 时只需扫描卡表（Card Table）记录的老年代 dirty page + GC Roots。

---

### 10. CMS和G1垃圾收集器对比、G1调优？

#### CMS：低停顿时代的先驱（但也是"带病上岗"）

CMS 的四个阶段：
1. **初始标记（Initial Mark）**——**STW**，标记 GC Roots 直接关联的对象。速度极快（几十到几百 ms）因为只扫描 GC Roots（线程栈、静态域、JNI 引用等）的直接引用。
2. **并发标记（Concurrent Mark）**——并发。GC 线程和应用线程同时运行，从 GC Roots 直接关联的对象开始遍历整个对象图（三色标记法）。这个过程时间最长但无 STW。
3. **重新标记（Remark）**——**STW**。修正并发标记阶段中因应用线程修改对象引用而漏标的对象（通过增量更新算法 + 卡表）。
4. **并发清除（Concurrent Sweep）**——并发。清除未标记对象，回收空间到空闲列表。

**CMS 的致命缺陷和演进坑**：

① **浮动垃圾（Floating Garbage）**：并发标记阶段，应用线程可能让一些对象变成垃圾（之前被标记为存活，之后无引用）。这些垃圾只能等到下次 GC 才能回收。为了给浮动垃圾留空间，CMS 在老年代占满 92-68%（默认）时就触发 GC，不能等到满了再收。

② **内存碎片**：CMS 用「标记-清除」而非「标记-整理」，老年代越来越碎片化。碎片太严重时找连续空间失败 → 触发 **Full GC (Serial Old)**，单线程串行收集，暂停时间可能几十秒。

③ **Concurrent Mode Failure**：并发清除阶段，应用线程分配老年代对象的速度 > CMS 清除的速度，老年代满了还没清完 → **退化为 Serial Old Full GC**。

④ **CMS 和 CPU 抢占**：CMS 并发阶段使用 CPU 核心，< 4 核时与应用线程争抢 CPU，反而拖慢应用。至少 4 核以上 CMS 才会发挥优势。

**CMS 参数军规**：
```
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=70   # 老年代占比 70% 触发 CMS
-XX:+UseCMSInitiatingOccupancyOnly      # 不要自动调整阈值
-XX:+CMSClassUnloadingEnabled           # 对永久代/元空间也做类卸载
-XX:+CMSScavengeBeforeRemark            # Remark 前先做一次 YGC
-XX:+ExplicitGCInvokesConcurrent        # System.gc() 也走 CMS 而不是 Full GC
```

#### G1：面向未来的分代收集器

G1 的核心思想：**把连续堆拆成大小为 2^n 的 Region（1MB-32MB），打破新生代/老年代在物理空间上是连续的假设**。每次 GC 只需要回收一部分 Region（Garbage First = 优先回收垃圾最多的 Region）。这就是「能控制停顿时间」的本质——不是让 GC 变快，而是让 GC 每次只做一部分工作。

**G1 内存布局**：
- Heap = 2048+ 个 Region（默认堆每 2GB 约 2048 个 Region）
- 每个 Region 在生命周期中扮演 Eden / Survivor / Old / Humongous 四种角色之一
- Region 之间不是固定分区的，而是在不同 GC 间动态变更角色

**Remembered Set（RSet）——G1 的空间代价**：
每个 Region 维护一个 RSet，记录「谁引用了我这个 Region 中的对象」。一个 Region 中的 RSet 可能记录几百个指向它的引用（Card Table 中的 dirty card），这要消耗大约 5%-20% 的堆空间。

**G1 的三类 GC**：

① **Young GC**：Eden 满了触发。只回收集中的 Eden Region → 回收到的存活对象复制到空的 Survivor/Old Region。触发时机由 Eden 区大小决定（可通过 `-XX:G1NewSizePercent` 调整）。

② **Mixed GC**：老年代占用比例达到 IHOP 阈值（默认 45%）时触发。过程：
- 并发标记周期（初始标记 + 并发标记 + 重新标记 + 清理）
- 并发标记结束后，选一批老年代 Region（垃圾最多的）进行复制回收
- Mixed GC 会执行多次（`-XX:G1MixedGCCountTarget`），每次回收一部分 Region

③ **Full GC**：Mixed GC 时晋升对象太多，目标 Region 放不下 → 退化为 **Serial Old Full GC**！这就是要避免的痛苦场景。

**IHOP（Initiating Heap Occupancy Percent）自适应**：
G1 根据过往 Mixed GC 的速度自动调整该阈值（`-XX:G1AdaptiveIHOP`）。如果 Mixed GC 很慢，G1 会提前触发（降低阈值）；如果很快，会延后触发。

#### G1 调优实战

| 参数 | 说明 | 调优建议 |
|------|------|---------|
| `-XX:MaxGCPauseMillis=200` | 期望最大暂停时间 | **核心参数**，不宜<100（太激进导致频繁 GC），不宜>500（停顿太长） |
| `-XX:G1HeapRegionSize` | Region 大小 | 堆 4G-8G 用 4M，8G-16G 用 8M，16G+ 用 16M |
| `-XX:G1NewSizePercent=5` | 年轻代最小占堆% | 默认 5，调大可以减少 YGC 频率但增加单次暂停 |
| `-XX:G1MaxNewSizePercent=60` | 年轻代最大占堆% | 默认 60，G1 在占满 IHOP 前自动调整年轻代占比 |
| `-XX:InitiatingHeapOccupancyPercent=45` | Mixed GC 触发阈值 | 默认 45，调整需配合观察 Mixed GC 频率 |
| `-XX:G1MixedGCCountTarget=8` | Mixed GC 分几次完成 | 默认 8，减少次数=每次回收更多=单次暂停更长 |

**G1 最常见的 Full GC 根因及解决**：
1. **Humongous 对象过多** → 增大 Region 大小，拆分大对象
2. **晋升失败（To-space exhausted）** → 增大 `-XX:G1ReservePercent` 或堆本身
3. **Mixed GC 跟不上分配速率** → 降低 IHOP 阈值，更早触发

---

### 11. Java 8元空间：永久代→元空间、特点、OOM解决？

#### 为什么要替换永久代

永久代最初是为了 GC 简单（和堆一起管理），但这个「简单」最终暴露了严重缺陷：

1. **大小难确定**：类元数据占用空间变化极大（从几 MB 到几 GB 取决于框架）。`-XX:MaxPermSize` 设太大浪费堆空间，设太小 OOM。
2. **永久代回收极度困难**：永久代的 GC 策略和堆一样，但类的回收条件苛刻（类加载器被回收、所有实例已回收、Class 对象不再被引用）。这些条件同时满足的概率低，导致永久代几乎只增不减。
3. **与 JRockit 对齐**：Oracle 合并 Sun 和 BEA 后需要统一两套 JVM，JRockit 从来没有永久代。

#### 元空间的内存布局

元空间 = 元数据 Chunk + Klass 元空间 + 压缩类空间（Compressed Class Space）

- **元数据 Chunk** 在本地内存（C Heap）中，由操作系统管理
- 每个类加载器有自己的元空间（按 Chunk 分配）
- Chunk 内部的 Block 用于存储 Klass 对象（每个 Klass 约 1KB+）
- Compressed Class Space：当 `-XX:+UseCompressedClassPointers` 开启时（默认），Klass 指针从 64bit 压缩到 32bit，需要在固定地址空间分配（`-XX:CompressedClassSpaceSize`，默认 1G）

**类加载器回收时的行为**：加载的类所属的 Chunk 一次性返还给操作系统（整块释放），不会像堆对象那样逐个回收。这就是元空间相对永久代的一大优势——回收高效。

#### Metaspace OOM 的根因

**根因诊断步骤**：
```bash
jstat -gc <pid>        # MU/MC（元空间使用/容量）
# 如果 MU 持续增长接近 MC → OOM 在路上了

# 开启类加载记录
-XX:+TraceClassLoading -XX:+TraceClassUnloading  
# 看哪些类在加载、哪些从未被卸载

# 拿到 heap dump，MAT 中看
# Class Loader Explorer → 按 Retained Heap 排序 → 找最大的 ClassLoader
```

**坑点场景**：
- **Groovy 脚本**：每次 `GroovyShell.evaluate()` 编译一个新类，ClassLoader 引用被长期持有 → 无法回收
- **Lambda 大量创建**：虽然 Lambda 类由 `LambdaMetaFactory` 创建在 `java.lang.invoke` 下，但大量匿名类仍然占用元空间。每个 Lambda 约 1KB，100 万个 Lambda = 1GB 元空间
- **动态代理 + Spring AOP**：Spring 默认对每个 Bean 生成 CGLIB 代理（Spring Boot 2.x），大量 Bean = 大量代理类 = 元空间压力。解决方案：`spring.aop.proxy-target-class=false` 或合理设置 MaxMetaspaceSize

#### 元空间参数

```
-XX:MetaspaceSize=256m          # 元空间初始大小，到达后触发 Metadata GC
-XX:MaxMetaspaceSize=512m       # 元空间最大值（生产环境必须设！）
-XX:MinMetaspaceFreeRatio=40    # GC 后最少空闲比例，防止频繁 GC
-XX:MaxMetaspaceFreeRatio=70    # GC 后最多空闲比例，控制内存释放
```

**关键认知**：`MetaspaceSize` ≠ `-Xms`，它是「触发 Metadata GC」的水位线。元空间增长到 MetaspaceSize 时触发一次 GC（回收无用的类元数据），如果回收后还不够，继续增加到 MaxMetaspaceSize。

---

### 12. ThreadLocal原理、内存泄漏、使用问题？

#### ThreadLocal 的实现原理

Thread → ThreadLocalMap（Thread 的成员变量 `threadLocals`） → Entry[]（hash 数组） → Entry(WeakReference<ThreadLocal<?>> key, Object value)

**每个 Thread 持有自己的 ThreadLocalMap**，ThreadLocal 对象只是 Map 的 key。所以 `tl.set(value)` 的含义是：**在我的线程的 ThreadLocalMap 中，以当前 tl 对象为 key，存一个 value**。不同线程调用同一个 ThreadLocal 对象，读写的是不同线程各自的 ThreadLocalMap。

**Hash 冲突解决**：ThreadLocalMap 是定制哈希表，使用线性探测（Linear Probing）。ThreadLocal 的 `threadLocalHashCode` 是每创建一个 ThreadLocal 实例就累加 `HASH_INCREMENT = 0x61c88647`（黄金分割比），使得哈希值均匀分布。

#### 内存泄漏的本质

泄漏的核心在于**线程生命周期 > ThreadLocal 生命周期**。详细链路：

1. Thread → ThreadLocalMap → Entry(K=WeakRef<ThreadLocal>, V=强引用)
2. ThreadLocal 对象的外部强引用消失 → GC 回收 ThreadLocal 对象 → Entry 的 Key 变成 null
3. 但 Entry 的 Value 仍然是**强引用**！只要 Thread 还活着，Value 永远不会被回收
4. 线程池中的线程几乎永远不死 → Value 永远不回收 → 内存泄漏

**为什么 Key 用弱引用**：如果用强引用，ThreadLocal 对象本身也无法被回收（因为 ThreadLocalMap 持有它）。弱引用至少让 Key 可以被 GC 回收，ThreadLocalMap 的 `expungeStaleEntries()` 方法会在 get/set/rehash 时清理 Key=null 的 Entry。

**为什么 JDK 不自动清理**：因为无法知道外部何时放弃了对 ThreadLocal 的引用。如果每次 get/set 都去做一次全局扫描（处理所有 Key=null 的 Entry），性能代价太大。所以只在 get/set 到的那个 key 对应的槽位做探测清理。

#### 真实使用场景和坑点

**场景一：请求上下文传递（最常用）**

```java
public class RequestContext {
    private static final ThreadLocal<User> CURRENT_USER = new ThreadLocal<>();
    public static void setUser(User u) { CURRENT_USER.set(u); }
    public static User getUser() { return CURRENT_USER.get(); }
    public static void clear() { CURRENT_USER.remove(); }
}
```

**场景二：Spring 事务的 `TransactionSynchronizationManager`**  
将数据库连接绑定到当前线程，保证同一个事务中使用同一个 Connection。

**场景三：MyBatis PageHelper**  
分页插件通过 ThreadLocal 传递分页参数（当前页码、每页条数）。但是！如果在同一线程中先 set 了分页参数，但最终执行的 SQL 并非需要分页的那个，就会抛异常。解决：PageHelper.startPage 后紧跟需要分页的 Mapper 方法。

**坑一：线程池中的数据污染（最致命）**

```java
// 线程复用场景
executor.submit(() -> {
    try {
        RequestContext.setUser(u);
        // doSomething...
    } finally {
        RequestContext.remove();   // ← 必须 remove！
    }
});
```
为什么 finally remove 还不能完全安全？如果代码抛了 `Throwable`（非 Exception，如 `Error` 或 `ThreadDeath`），finally 也不一定执行。更好的做法：使用阿里 `TransmittableThreadLocal` + 线程池包装。

**坑二：InheritableThreadLocal 的局限性**

`new Thread()` 时会将父线程的 InheritableThreadLocal 拷贝到子线程，但**只在创建线程时拷贝一次**。线程池复用线程时，新的线程已经是旧的父线程的值了。这个问题 TransmittableThreadLocal 解决了，因为它在线程池任务提交时主动做值传递。

**坑三：内存泄漏的隐蔽性和排查**

泄漏症状是堆 dump 中看到大量 ThreadLocalMap$Entry 对象，但很难溯源是谁设的。最佳实践：
- 所有 ThreadLocal 设为 `static`（全局统一管理生命周期）
- 对每一个 ThreadLocal 的 set 加 try-finally + 日志（方便追踪来源）
- 如果 value 很大（如整个 Request），务必及时 remove

---

### 13. ConcurrentHashMap、CopyOnWriteArrayList 底层实现？

#### ConcurrentHashMap（JDK 8+）的精细化设计

JDK 7 `Segment` → JDK 8 `CAS + synchronized` 是并发容器的演进标志。

**JDK 7 问题**：每个 Segment 是一个独立的小 HashTable，锁粒度虽然是 Segment 级别，但默认 16 个 Segment → 最多 16 个线程并发写入。扩容也是按 Segment 为单位，Segment 内部是 ReentrantLock。

**JDK 8 的颠覆**：
- 抛弃 Segment，细化到**槽位（bin）级别**的并发
- `put(K,V)` 流程：
  1. 计算 hash，取模定位到 `tab[i]`
  2. `tab[i] == null` → `CAS tab[i] = new Node(hash, k, v, null)`，无锁！（Fast Path）
  3. `tab[i] != null` → `synchronized(tab[i])`，只锁这个槽位的头节点
  4. 锁住后判断链表 or 红黑树 → 插入或更新
  5. 检查链表长度 >= 8 → `treeifyBin` 树化（需要 tab.length >= 64）
  6. `addCount` → 计数值（LongAdder 思想 + CounterCell 多槽计数）

**关键设计点**：

① **扩容：多线程协作 `transfer`**：不阻塞读操作（扩容期间仍可读）
```java
// transfer 在多个 put 线程和帮助线程间分摊：
// 每个线程领取一个 stride（一批槽位），迁移该段的节点
// 迁移完成后 tab 引用指向新 table
```

② **计数：LongAdder 的 Cell 模型**：
```java
// 不直接用 volatile long（竞争太激烈）
// 而是 base + CounterCell[]，类似 LongAdder
// 每个槽位 = @Contended（填充 Cache Line 防伪共享）
// 读取时 sum = base + Σ Cell
```

③ **红黑树改成 TreeBin + TreeNode 双重结构**：
TreeBin 是树根节点（持有读写锁），TreeNode 是树节点。TreeBin 的读锁不是传统锁——CAS 增加一个读计数器，读线程可以并发读树。只有当写线程（树结构调整、插入删除）时才真正加锁。

#### CopyOnWriteArrayList：写时复制

**核心**：每次写操作（add/set/remove）复制整个数组 + 在副本上修改 + 旧引用指向新数组。锁是 `ReentrantLock`。

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);  // O(n) 复制
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

**读操作完全无锁**：迭代器拿到的是创建时快照（`snapshot`），过程中即使其他线程 add/remove，迭代器看到的是旧数组，不会 ConcurrentModificationException。

**致命性能问题**：数组复制是 O(n)！10 万元素，每次写 = 复制 10 万条数据。多写场景（如并发写、批量初始化）性能崩盘。只适合**白名单、监听器、配置等几乎不变的数据结构**。

---

### 14. CompletableFuture 使用场景与异步编程？

#### 为什么需要 CompletableFuture

Future 接口的痛点：
- `get()` 是阻塞的 → 主线程卡住
- 不能链式组合（thenApply/thenCompose）
- 不能异常处理（exceptionally）
- 不能组合多个 Future（allOf/anyOf）

CompletableFuture 继承了 Future + CompletionStage，提供**声明式的异步编排**。

**线程池**：如果不传自定义 Executor，默认用 `ForkJoinPool.commonPool()`。这是生产最大的坑——所有不传 Executor 的 CompletableFuture 任务都挤在 commonPool（默认 CPU 核数 - 1 个线程）。如果其中有慢任务或阻塞任务，会把 commonPool 吃满，影响 JVM 内所有使用 commonPool 的组件（parallelStream 等）。

#### 核心 API 分类

**执行**
- `supplyAsync(Supplier)` → 有返回值
- `runAsync(Runnable)` → 无返回值
- 永远传自定义线程池：`supplyAsync(supplier, executor)`

**链式转换**
- `thenApply(Function)` → 映射结果（同线程）
- `thenApplyAsync(Function)` → 映射结果（异步，重点：可能换线程）
- `thenAccept(Consumer)` → 消费（无返回值）
- `thenRun(Runnable)` → 忽略结果执行 Runnable

**组合**
- `thenCompose(Function<T, CompletableFuture<U>>)` → 扁平面包（避免 `CompletableFuture<CompletableFuture<U>>`）
- `thenCombine(other, BiFunction)` → 两个独立任务都完成后合并
- `allOf(futures)` → 等待所有完成
- `anyOf(futures)` → 只要一个完成即可

**异常处理**
- `exceptionally(Function<Throwable, T>)` → 异常时返回默认值
- `handle(BiFunction<T, Throwable, U>)` → 正常/异常都处理
- `whenComplete(BiConsumer<T, Throwable>)` → 回调不改变结果

**实际最佳范式**：
```java
CompletableFuture<Product> productF = 
    CompletableFuture.supplyAsync(() -> productService.get(id), executor);
CompletableFuture<Stock> stockF = 
    CompletableFuture.supplyAsync(() -> stockService.get(id), executor);

CompletableFuture<ProductDetail> result = productF
    .thenCombine(stockF, (product, stock) -> assemble(product, stock))
    .orTimeout(2, TimeUnit.SECONDS)  // 超时不抛错
    .exceptionally(ex -> fallback(id))  // 异常降级
    .whenComplete((detail, ex) -> {
        if (ex != null) log.error("error", ex);
    });
```

---

### 15. Java死锁：条件、排查、预防？

#### 四个必要条件

1. **互斥**——资源只能被一个线程持有（内存中无法改变）
2. **占有且等待**——持有一个锁的同时等待另一个锁
3. **不可剥夺**——持有的锁不能被强制释放
4. **循环等待**——T1 等 T2，T2 等 T3，T3 等 T1

破坏任意一个即可预防死锁。

#### 排查步骤

**jstack**（最快）：
```bash
jstack -l <pid> | grep -A 10 "Found one Java-level deadlock"
```

**Arthas**：
```bash
thread -b   # 一键找到阻塞其他线程的"元凶"
```

**如何阅读死锁日志**：
```
Found one Java-level deadlock:
============================
"Thread-1":
  waiting to lock Monitor(0x7f8e...) which is held by "Thread-0"
"Thread-0":
  waiting to lock Monitor(0x7f9a...) which is held by "Thread-1"
```
T1 持有 Monitor A 等 Monitor B，T0 持有 Monitor B 等 Monitor A → 循环等待。

#### 预防策略

1. **加锁顺序**（破坏循环等待）：对所有锁做全局排序，所有线程按相同顺序获取
2. **定时锁**（破坏占有且等待）：`tryLock(timeout, unit)`，超时释放已有锁
3. **一次性加锁**：用 `java.util.concurrent.locks.Lock` 一次性获取所有锁
4. **免锁设计**：用 CAS 乐观锁、不可变对象、Actor 模型替代互斥锁

**数据库死锁**往往是反向加锁（A 转账给 B，B 转账给 A），解决：按 ID 升序加锁 `SELECT ... FOR UPDATE`。

---

