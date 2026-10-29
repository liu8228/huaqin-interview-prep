# 05 · Java 与后端技术问答（JD：精通 Java、掌握高级特性）

> [返回目录](README.md) · Java 10 年是你的主场，答案要答得快、准、有案例收口。

## 1. 集合与基础

**Q1：HashMap 的底层实现？JDK8 改了什么？**
> 数组+链表+红黑树；JDK8 起链表长度 ≥8 且数组长度 ≥64 才树化，查询从 O(n) 到 O(logn)；扩容阈值=容量×负载因子(0.75)；1.8 扩容用高低位拆分减少 rehash。并发场景用 ConcurrentHashMap（1.8 改 CAS+synchronized 锁桶头），不并发也别用 Hashtable。
> 追问预防：为什么容量是 2 的幂——位运算取模更快；负载因子为什么 0.75——空间与冲突的折中。

**Q2：ConcurrentHashMap 怎么保证并发安全？**
> JDK7 分段锁（Segment），JDK8 改为 CAS 插入空桶 + synchronized 锁桶头节点，粒度更细；size 用 CounterCell 分散计数；迭代器是弱一致的，不抛 ConcurrentModificationException。

**Q3：ArrayList 与 LinkedList 区别？**
> ArrayList 动态数组、随机访问 O(1)、中间插删 O(n)；LinkedList 双向链表、头尾插删 O(1)、按需遍历 O(n)。工程上绝大多数场景选 ArrayList（CPU 缓存友好），LinkedList 只在明确头尾队列操作时用——**答出"工程判断"而不是背概念，是加分点。**

**Q4：equals 与 hashCode 契约？**
> equals 相等则 hashCode 必须相等；只覆写 equals 不覆写 hashCode 会导致 HashMap/HashSet 行为异常。String 缓存了 hash；实际开发中先想清楚"相等"的业务含义（如 SN、工单号一致），再决定要不要重写。

## 2. JVM 与 GC

**Q5：JVM 内存结构？常见 OOM？**
> 线程共享：堆（新生代 Eden+2 Survivor / 老年代）、方法区（JDK8 起元空间，本体在本地内存）；线程私有：虚拟机栈、本地方法栈、程序计数器。
> 常见 OOM：堆溢出（大对象/泄漏，用 MAT 分析 dump）、元空间溢出（动态类生成，CGLib）、栈溢出（递归深度，StackOverflow）。排查：jmap+jstat+jstack，内网常用 Arthas 在线诊断。

**Q6：GC 算法与收集器？怎么选？**
> 算法：标记-清除、标记-整理、复制。收集器：CMS 已废；JDK8 默认 Parallel（吞吐优先）；G1 大堆可预测停顿（Region 分区、优先回收价值高的 Region）；低延迟选 ZGC。
> 调优思路（答题顺序）：先明确目标（吞吐 or 延迟）→ 调堆大小与代比例 → 避免大对象直接进老年代、避免 System.gc → 用 GC 日志验证，不盲调。

**Q7：线上 CPU 飙高怎么排查？**
> top 找进程 → top -Hp 找线程 → jstack 把线程号（16 进制）对应到栈 → 定位代码（多为死循环、锁竞争、频繁 GC——先看 GC 情况排除回收风暴）。工具链答得顺手即可，重点展示**排查路径有条理**。

## 3. 并发

**Q8：synchronized 与 ReentrantLock 区别？**
> synchronized：JVM 级互斥，锁升级（偏向→轻量级→重量级），自动释放；ReentrantLock：API 级，支持公平锁、tryLock 超时、多条件队列 Condition。若面试官问新版本特性（虚拟线程等），如实按我实际用过的版本来谈，不抢新词。

**Q9：volatile 的语义？**
> 可见性（写立即刷主存、读强制拉主存）+ 禁止指令重排，**不保证原子性**（i++ 仍会丢更新）；典型用途：状态标志位、双检锁的实例引用。 CAS 与 ABA：原子类基于 CAS；ABA 用版本号（如 AtomicStampedReference）解决。

**Q10：线程池参数与执行顺序？拒绝策略？**
> corePoolSize、maximumPoolSize、keepAliveTime、workQueue、threadFactory、handler；执行顺序：核心线程 → 队列（要满）→ 扩到 max → 拒绝策略。策略：AbortPolicy 抛异常 / CallerRunsPolicy 调用者执行（天然反压，适合不能丢的任务）/ Discard 系列（慎用，会丢数据）。
> 设置思路：CPU 密集≈核数，IO 密集≈核数×(1+等待/计算)，**MES 报工场景是 IO 密集，队列要有界，避免 OOM**。

**Q11：ThreadLocal 原理与坑？**
> 每个 Thread 内有 ThreadLocalMap，key 是 ThreadLocal 的弱引用，value 是强引用——线程池复用线程时必须 remove()，否则脏数据/内存泄漏。

## 4. Spring 生态

**Q12：Spring Boot 自动装配原理？Bean 生命周期？**
> @EnableAutoConfiguration → 加载 spring.factories / AutoConfiguration.imports 中的配置类 → @Conditional 系列条件装配。
> 生命周期：实例化 → 属性填充 → Aware 回调 → BeanPostProcessor 前置 → @PostConstruct/init → BeanPostProcessor 后置（**AOP 代理在这里生成**）→ 使用 → 销毁。循环依赖：三级缓存（提前暴露半成品引用）。

**Q13：Spring 事务什么时候会失效？（高频）**
> 方法非 public；同类内 this 调用（绕过代理）；异常被 catch 吞掉；抛 checked 异常且没配 rollbackFor；多线程调用（事务绑定线程）；传播行为配错。**这道题答好直接证明"精通"不是虚的。**

**Q14：MyBatis 与 JPA 怎么选？# 和 $ 区别？**
> MyBatis SQL 可控，适合复杂查询与调优（MES 报表类查询首选）；JPA 开发效率高适合简单 CRUD。#{} 预编译防注入；${} 字符串拼接，只用于动态表名/排序字段且必须白名单校验。

**Q15：微服务你又做过多少？（Spring Cloud）**
> 如实：智算云平台模块间用 Spring Boot/Cloud 体系，注册中心+网关+配置中心+远程调用的套路清楚；拆分原则按业务边界（用户/组织/预算模块独立），**不鼓吹全微服务**——制造企业内部系统拆到合理粒度即可，避免分布式事务复杂化。

## 5. 中间件与接口质量（结合真实项目）

**Q16：Kafka 在项目里为什么用？怎么防丢防重？**
> 用途：指标刷新削峰、变更事件解耦。防丢：生产端 acks=all+重试；broker 副本；消费端先处理后提交 offset。防重：业务侧幂等（唯一键+去重表/乐观锁），**因为我承认重复和乱序永远可能发生，所以用业务幂等兜底**——这是生产答案不是背书。

**Q17：ElasticSearch 是什么？和数据库怎么分工？**
> 倒排索引+分词，全文检索与日志场景强项；分工：DB 保事务与强一致账务，ES 做检索/日志/报表查询，两者经异步同步对齐（允许秒级延时）。我项目里 ES 用于日志与检索场景。

**Q18：接口幂等怎么保证？（MES 高频：防重复报工/扫码）**
> 唯一业务键约束（SN+工序+单号）+ 去重表；客户端 token 预申请；乐观锁 version；接口侧"先查后写"不可靠，**必须落到数据库级约束**；重复请求返回首次结果。

**Q19：系统之间的接口超时/失败怎么设计？**
> 超时分级（连接/读）、有限重试+退避、失败进重试队列+死信告警、对账任务兜底、全链路 traceId 串日志。方法论来自真实交付（广药/明远/商飞多系统集成）。

**Q20：用过 Redis 吗？（提前打地基的诚实答法）**
> 生产项目主要用 Kafka/ElasticSearch/Doris 三件；Redis 我了解常用模式（缓存穿透/击穿/雪崩的对策：空值缓存/互斥重建/过期打散，分布式锁 setnx+过期），但没有大规模生产实战，如实说并表达 2-3 周可上手。**不夸大，如实最稳。**

## 6. 答题技巧

- 技术题先给**结论**，再给原理，最后绑一个真实场景："这在我们的场景里其实就是……（举工单/报工/接口案例）"。
- 不懂的概念：承认+给相关经验+给学习方法，三句走完，10 秒内不要沉默。
- 高频追问预判：HashMap→ConcurrentHashMap→CAS；线程池→队列满→OOM；事务→失效场景；索引→失效场景。每条主答后都预留一层追问的答案。
