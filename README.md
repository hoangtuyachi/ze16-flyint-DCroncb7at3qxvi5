**大家好，我是小富～**

面试都背过道八股题：Redis 的内存淘汰策略 `LRU` 和 `LFU` 是什么？怎么选好？

很多同学对这两个算法的理解，只停留在都是缓存淘汰，但说不清它们具体区别，概念混淆，更不知道实际场景该怎么选？

而且 Redis 的 key 淘汰算法其实还不是**正统**的 `LRU` 和 `LFU` 算法，而是基于 LRU/LFU 的一个变种。所以我们先了解下 LRU/LFU 基础算法，再看看它和变种的 Redis LRU 算法有何不同。

## 一、什么是 LRU 算法？

LRU 的全称是 Least Recently Used（最近最少使用），核心逻辑特别好记：**最近没被用过的，下次也大概率用不上**。

举个例子：你电脑桌面上放着常用的软件图标（微信、IDE），这些是最近常用的；而几个月没打开过的压缩工具，会被你拖到文件夹里。这就是 LRU 的思路：保留`最近使用`的，淘汰`最近最少使用`的。

> 假设缓存容量只有 3，依次存入 A、B、C，此时缓存是 [A,B,C]；
>
> 若此时访问 A（A 变成最近使用），缓存顺序变为 [B,C,A]
>
> 若再存入D（缓存满了），需要淘汰最近最少使用的 B，最终缓存是 [C,A,D]

### LRU 的优缺点

#### 优点：

* 逻辑简单：只关注使用时间，实现成本低，容易理解
* 响应快：插入、删除、查询的时间复杂度可以做到 O (1)（用链表 + 哈希表实现）
* 贴合短期局部性：很多业务场景中，最近用的数据，确实接下来更可能用（比如你刚打开的文档，接下来大概率会继续编辑）。

#### 缺点：

* **突发访问误淘汰**：如果突然有大量`一次性数据`访问，会把原本常用的缓存挤掉。

  比如缓存容量 3，原本缓存 [A,B,C]（A 是高频使用），突然连续访问 D、E、F，此时会淘汰 A、B、C，缓存变成 [D,E,F]；但后续再访问 A 时，A 已经被淘汰，需要重新从数据库加载，导致`缓存命中率骤降`。
* **不考虑使用频率**：如果 A 每天用 100 次，B 每天用 1 次，但 B 是 1 分钟前刚用，A 是 2 分钟前用，LRU 会优先淘汰 A，这显然不合理（高频使用的 A 不该被淘汰）。

### 实现 LRU 两种方案

#### 方案 1：基于 LinkedHashMap

Java 的`LinkedHashMap`自带按访问顺序排序的功能，只需重写`removeEldestEntry`方法，就能实现 LRU 缓存。这是最简洁的实现方式，适合业务中不需要极致性能的场景，快速开发。

```
|  |  |
| --- | --- |
|  | import java.util.LinkedHashMap; |
|  | import java.util.Map; |
|  |  |
|  | public class LRUCache extends LinkedHashMap { |
|  | // 缓存最大容量 |
|  | private final int maxCapacity; |
|  |  |
|  | // 构造函数：accessOrder=true 表示“按访问顺序排序”（核心） |
|  | public LRUCache(int maxCapacity) { |
|  | super(maxCapacity, 0.75f, true); |
|  | this.maxCapacity = maxCapacity; |
|  | } |
|  |  |
|  | // 核心：当缓存大小超过maxCapacity时，自动删除“最老的 entry”（最近最少使用的） |
|  | @Override |
|  | protected boolean removeEldestEntry(Map.Entry eldest) { |
|  | return size() > maxCapacity; |
|  | } |
|  |  |
|  | // 测试 |
|  | public static void main(String[] args) { |
|  | LRUCache cache = new LRUCache<>(3); |
|  | cache.put(1, "A"); |
|  | cache.put(2, "B"); |
|  | cache.put(3, "C"); |
|  | System.out.println(cache); // 输出：{1=A, 2=B, 3=C}（插入顺序） |
|  |  |
|  | cache.get(1); // 访问1，1变成最近使用 |
|  | System.out.println(cache); // 输出：{2=B, 3=C, 1=A}（按访问顺序排序） |
|  |  |
|  | cache.put(4, "D"); // 缓存满了，淘汰最老的2 |
|  | System.out.println(cache); // 输出：{3=C, 1=A, 4=D} |
|  | } |
|  | } |
```

#### 方案 2：基于双向链表 + 哈希表

`LinkedHashMap`本质是哈希表 + 双向链表，但如果想深入理解 LRU 的实现原理，建议自己手写一个核心是用哈希表保证 O (1) 查询，用双向链表保证 O (1) 插入、删除（维护访问顺序）。适用于高并发场景和面试。

```
|  |  |
| --- | --- |
|  | import java.util.HashMap; |
|  | import java.util.Map; |
|  |  |
|  | public class LRUCache2 { |
|  | // 双向链表节点：存储key、value，以及前后指针 |
|  | static class Node { |
|  | K key; |
|  | V value; |
|  | Node prev; |
|  | Node next; |
|  |  |
|  | public Node(K key, V value) { |
|  | this.key = key; |
|  | this.value = value; |
|  | } |
|  | } |
|  |  |
|  | private final int maxCapacity; |
|  | private final Map> map; // 哈希表：key→Node，O(1)查询 |
|  | private final Node head; // 虚拟头节点（简化链表操作） |
|  | private final Node tail; // 虚拟尾节点 |
|  |  |
|  | public LRUCache2(int maxCapacity) { |
|  | this.maxCapacity = maxCapacity; |
|  | this.map = new HashMap<>(); |
|  | // 初始化虚拟头尾节点，避免处理null指针 |
|  | this.head = new Node<>(null, null); |
|  | this.tail = new Node<>(null, null); |
|  | head.next = tail; |
|  | tail.prev = head; |
|  | } |
|  |  |
|  | // 1. 获取缓存：命中则把节点移到链表头部（标记为最近使用） |
|  | public V get(K key) { |
|  | Node node = map.get(key); |
|  | if (node == null) { |
|  | return null; // 未命中 |
|  | } |
|  | // 移除当前节点 |
|  | removeNode(node); |
|  | // 移到头部 |
|  | addToHead(node); |
|  | return node.value; |
|  | } |
|  |  |
|  | // 2. 存入缓存：不存在则新增，存在则更新并移到头部；满了则删除尾部节点 |
|  | public void put(K key, V value) { |
|  | Node node = map.get(key); |
|  | if (node == null) { |
|  | // 新增节点 |
|  | Node newNode = new Node<>(key, value); |
|  | map.put(key, newNode); |
|  | addToHead(newNode); |
|  |  |
|  | // 缓存满了，删除尾部节点（最近最少使用） |
|  | if (map.size() > maxCapacity) { |
|  | Node tailNode = removeTail(); |
|  | map.remove(tailNode.key); // 同步删除哈希表中的key |
|  | } |
|  | } else { |
|  | // 更新节点值，并移到头部 |
|  | node.value = value; |
|  | removeNode(node); |
|  | addToHead(node); |
|  | } |
|  | } |
|  |  |
|  | // 辅助：把节点添加到链表头部 |
|  | private void addToHead(Node node) { |
|  | node.prev = head; |
|  | node.next = head.next; |
|  | head.next.prev = node; |
|  | head.next = node; |
|  | } |
|  |  |
|  | // 辅助：移除指定节点 |
|  | private void removeNode(Node node) { |
|  | node.prev.next = node.next; |
|  | node.next.prev = node.prev; |
|  | } |
|  |  |
|  | // 辅助：移除尾部节点（最近最少使用） |
|  | private Node removeTail() { |
|  | Node tailNode = tail.prev; |
|  | removeNode(tailNode); |
|  | return tailNode; |
|  | } |
|  |  |
|  | // 测试 |
|  | public static void main(String[] args) { |
|  | LRUCache2 cache = new LRUCache2<>(3); |
|  | cache.put(1, "A"); |
|  | cache.put(2, "B"); |
|  | cache.put(3, "C"); |
|  | System.out.println(cache.get(1)); // 输出A，此时1移到头部 |
|  | cache.put(4, "D"); // 淘汰3 |
|  | System.out.println(cache.get(3)); // 输出null（已被淘汰） |
|  | } |
|  | } |
```

### LRU 的使用场景

LRU 适合短期高频场景，比如:

* **浏览器缓存**：浏览器缓存网页时，会优先淘汰最近最少打开”的页面（比如你一周没打开的网页，会被优先清理）；
* **Redis 默认内存淘汰策略**：Redis 的`allkeys-lru`和`volatile-lru`是基于 LRU 的（实际是 “近似 LRU”，性能更高），适合 “缓存访问具有短期局部性” 的场景（比如电商商品详情页，用户打开后短时间内可能再次刷新）；
* **本地缓存基础版**：如果业务中没有突发大量一次性访问，用 LRU 实现本地缓存足够满足需求（比如用 LinkedHashMap 快速开发）。

## 二、什么是 LFU 算法？

`LFU` 的全称是 **Least Frequently Used（最不经常使用）**，核心逻辑和 LRU 完全不同：**用得少的，下次也大概率用不上**，它关注的是`使用频率`，而不是`使用时间`。

还是用生活例子：你手机里的 APP，微信、抖音是高频使用的（每天打开几十次），而指南针是低频使用的（几个月打开一次）。当手机内存不足时，系统会优先卸载指南针这类低频 APP，这就是 LFU 的思路。

> 假设缓存容量 3，初始存入 A（使用 1 次）、B（使用 1 次）、C（使用 1 次），频率都是 1；
>
> 访问 A（A 频率变成 2），此时频率：A=2，B=1，C=1；
>
> 访问 A（A 频率变成 3），此时频率：A=3，B=1，C=1；
>
> 存入 D（缓存满了），需要淘汰 “频率最低” 的 B 或 C（若频率相同，淘汰 “最近最少使用” 的，即 LFU+LRU 结合），最终缓存是 A、C、D（频率分别为 3、1、1）。

### LFU 的优缺点

#### 优点：

* **更贴合长期高频场景**：能保留使用频率高的数据，即使它不是 最近使用的（比如 A 每天用 100 次，即使 10 分钟前没⽤，也不该被淘汰）；
* **避免突发访问误淘汰**：面对一次性突发访问（比如访问 D、E、F），因为这些数据的频率低，不会把高频的 A、B、C 挤掉，缓存命中率更稳定。

#### 缺点：

* **实现复杂**：需要维护使用频率，还需要处理频率相同的情况（通常结合 LRU），时间复杂度比 LRU 高（插入、删除约 O (log n)）；
* **冷启动问题**：新存入的缓存频率低，容易被淘汰。比如一个新功能的接口，刚开始访问频率低，会被 LFU 优先淘汰，但其实后续会变成高频接口；
* **频率老化问题**：一个数据过去高频，但现在不再使用（比如活动结束后的活动页面），它的高频率会一直占用缓存，导致新的高频数据无法进入。

### 实现 LFU：基于哈希表 + 优先级队列

LFU 的实现比 LRU 复杂，核心需要两个哈希表：

* `cache`：key→Node（存储 value 和使用频率、最后使用时间）；
* `freqMap`：频率→LinkedHashSet（存储该频率下的所有 key，LinkedHashSet 保证插入顺序，用于处理频率相同的情况）；

  再用一个变量`minFreq`记录当前最小频率，快速找到最该淘汰的 key。

```
|  |  |
| --- | --- |
|  | import java.util.*; |
|  |  |
|  | public class LFUCache { |
|  | // 缓存节点：存储key、value、使用频率、最后使用时间（处理频率相同时的淘汰） |
|  | static class Node { |
|  | K key; |
|  | V value; |
|  | int freq; // 使用频率 |
|  | long lastUseTime; // 最后使用时间（毫秒） |
|  |  |
|  | public Node(K key, V value) { |
|  | this.key = key; |
|  | this.value = value; |
|  | this.freq = 1; // 初始频率1 |
|  | this.lastUseTime = System.currentTimeMillis(); |
|  | } |
|  | } |
|  |  |
|  | private final int maxCapacity; |
|  | private final Map> cache; // key→Node |
|  | private final Map> freqMap; // 频率→key集合（LinkedHashSet保证顺序） |
|  | private int minFreq; // 当前最小频率（快速定位要淘汰的key） |
|  |  |
|  | public LFUCache(int maxCapacity) { |
|  | this.maxCapacity = maxCapacity; |
|  | this.cache = new HashMap<>(); |
|  | this.freqMap = new HashMap<>(); |
|  | this.minFreq = 1; |
|  | } |
|  |  |
|  | // 1. 获取缓存：命中则更新频率和最后使用时间 |
|  | public V get(K key) { |
|  | Node node = cache.get(key); |
|  | if (node == null) { |
|  | return null; |
|  | } |
|  | // 更新节点频率 |
|  | updateNodeFreq(node); |
|  | return node.value; |
|  | } |
|  |  |
|  | // 2. 存入缓存：不存在则新增，存在则更新；满了则淘汰最小频率的key |
|  | public void put(K key, V value) { |
|  | if (maxCapacity <= 0) { |
|  | return; |
|  | } |
|  |  |
|  | Node node = cache.get(key); |
|  | if (node != null) { |
|  | // 存在：更新value、频率、最后使用时间 |
|  | node.value = value; |
|  | node.lastUseTime = System.currentTimeMillis(); |
|  | updateNodeFreq(node); |
|  | } else { |
|  | // 不存在：检查缓存是否满 |
|  | if (cache.size() >= maxCapacity) { |
|  | // 淘汰最小频率的key（频率相同则淘汰最早使用的） |
|  | evictMinFreqKey(); |
|  | } |
|  | // 新增节点 |
|  | Node newNode = new Node<>(key, value); |
|  | cache.put(key, newNode); |
|  | // 加入freqMap：频率1的集合 |
|  | freqMap.computeIfAbsent(1, k -> new LinkedHashSet<>()).add(key); |
|  | // 新节点频率是1，minFreq重置为1 |
|  | minFreq = 1; |
|  | } |
|  | } |
|  |  |
|  | // 辅助：更新节点频率 |
|  | private void updateNodeFreq(Node node) { |
|  | K key = node.key; |
|  | int oldFreq = node.freq; |
|  | int newFreq = oldFreq + 1; |
|  |  |
|  | // 1. 从旧频率的集合中移除key |
|  | LinkedHashSet oldFreqSet = freqMap.get(oldFreq); |
|  | oldFreqSet.remove(key); |
|  | // 如果旧频率是minFreq，且集合为空，minFreq+1 |
|  | if (oldFreq == minFreq && oldFreqSet.isEmpty()) { |
|  | minFreq = newFreq; |
|  | } |
|  |  |
|  | // 2. 加入新频率的集合 |
|  | freqMap.computeIfAbsent(newFreq, k -> new LinkedHashSet<>()).add(key); |
|  |  |
|  | // 3. 更新节点的频率和最后使用时间 |
|  | node.freq = newFreq; |
|  | node.lastUseTime = System.currentTimeMillis(); |
|  | } |
|  |  |
|  | // 辅助：淘汰最小频率的key（频率相同则淘汰最早使用的） |
|  | private void evictMinFreqKey() { |
|  | // 1. 获取最小频率的key集合 |
|  | LinkedHashSet minFreqSet = freqMap.get(minFreq); |
|  | // 2. 淘汰集合中第一个key（LinkedHashSet按插入顺序，即最早使用的） |
|  | K evictKey = minFreqSet.iterator().next(); |
|  | minFreqSet.remove(evictKey); |
|  |  |
|  | // 3. 同步删除cache和freqMap（如果集合为空） |
|  | cache.remove(evictKey); |
|  | if (minFreqSet.isEmpty()) { |
|  | freqMap.remove(minFreq); |
|  | } |
|  |  |
|  | System.out.println("淘汰key：" + evictKey); |
|  | } |
|  |  |
|  | // 测试 |
|  | public static void main(String[] args) { |
|  | LFUCache cache = new LFUCache<>(3); |
|  | cache.put(1, "A"); |
|  | cache.put(2, "B"); |
|  | cache.put(3, "C"); |
|  | System.out.println(cache.get(1)); // 输出A，频率变成2，minFreq还是1 |
|  | cache.put(4, "D"); // 缓存满，淘汰minFreq=1的2（B） |
|  | System.out.println(cache.get(2)); // 输出null（已淘汰） |
|  | cache.get(3); // 3频率变成2，minFreq变成2 |
|  | cache.get(4); // 4频率变成2 |
|  | cache.put(5, "E"); // 缓存满，淘汰minFreq=2的3（C，最早使用） |
|  | } |
```

## 三、Redis 中的 LRU 和 LFU：实现与区别

Redis 作为缓存数据库，当内存达到内存`maxmemory`限制时，会触发内存淘汰策略，其中`LRU`和`LFU`是最常用的两种。

**但 Redis 的实现并非 “严格版”，而是做了性能优化的 “近似版”，** 这一点尤其需要注意。

### Redis 的 LRU 实现

Redis 的 LRU 实现：近似 LRU（性能优先）

Redis 没有采用 “双向链表 + 哈希表” 的标准 LRU 实现，因为会增加内存开销和操作复杂度，而是用**随机采样 + 时间戳**实现近似 LRU：

* **核心原理**：每个 key 在内存中记录`lru`字段（最后一次访问的时间戳），当需要淘汰 key 时，Redis 会从所有候选 key 中**随机采样 N 个**（默认 5 个，可通过`maxmemory-samples`配置），然后淘汰这 N 个中`lru`值最小（最久未使用）的 key。
* **为什么近似**：严格 LRU 需要维护全量 key 的访问顺序，在 Redis 高并发场景下会成为性能瓶颈；近似 LRU 通过控制采样数（N 越大越接近严格 LRU，但性能稍差），在 “命中率” 和 “性能” 之间做了平衡。
* **相关配置**：

```
|  |  |
| --- | --- |
|  | # 淘汰所有key中最久未使用的 |
|  | maxmemory-policy allkeys-lru |
|  | # 只淘汰设置了过期时间的key中最久未使用的 |
|  | maxmemory-policy volatile-lru |
|  | # 采样数（默认5，范围3-10） |
|  | maxmemory-samples 5 |
```

### Redis 的 LFU 实现

Redis 的 LFU 实现，结合频率与时间衰减。

Redis 4.0 引入了 LFU 策略，解决了标准 LFU 的频率老化问题，实现更贴合实际业务：

**核心改进**：

* **频率计数**：用`lfu_count`记录访问频率（但不是简单累加，而是用对数计数：访问次数越多，`lfu_count`增长越慢，避免数值过大）；
* **时间衰减**：用`lfu_decay_time`控制频率衰减（单位分钟），如果 key 在`lfu_decay_time`内没有被访问，`lfu_count`会随时间减少，避免**过去高频但现在不用**的 key 长期占用缓存。
* **淘汰逻辑**：当需要淘汰时，同样随机采样 N 个 key，淘汰`lfu_count`最小的（频率最低）；若频率相同，淘汰**最久未使用**的（结合 LRU 逻辑）。

**相关配置**：

```
|  |  |
| --- | --- |
|  | # 淘汰所有key中访问频率最低的 |
|  | maxmemory-policy allkeys-lfu |
|  | # 只淘汰设置了过期时间的key中访问频率最低的 |
|  | maxmemory-policy volatile-lfu |
|  | # 频率衰减时间（默认1分钟，0表示不衰减） |
|  | lfu-decay-time 1 |
|  | # 初始频率（新key的lfu_count，默认5，避免新key被立即淘汰） |
|  | lfu-log-factor 10 |
```

### Redis 中 LRU 和 LFU 的核心区别

| 维度 | Redis LRU | Redis LFU |
| --- | --- | --- |
| 判断依据 | 最后访问时间（越久未用越可能被淘汰） | 访问频率（频率越低越可能被淘汰，结合时间衰减） |
| 适用场景 | 短期局部性访问（如用户会话、临时数据） | 长期高频访问（如热点商品、基础配置） |
| 应对突发访问 | 弱（易被一次性访问挤掉常用 key） | 强（低频的突发访问 key 不会淘汰高频 key） |
| 冷启动友好度 | 较好（新 key 只要最近访问就不会被淘汰） | 需配置`lfu-log-factor`（避免新 key 因初始频率低被淘汰） |
| 内存 overhead | 低（只存时间戳） | 稍高（存频率和时间衰减信息） |
| 典型业务案例 | 新闻 APP 的最近浏览列表 | 电商首页的热销商品缓存 |

### 怎么选？看业务访问模式

**选 LRU**：如果你的缓存访问具有**短期集中性**（比如用户打开一个页面后，短时间内会频繁刷新，但过几天可能再也不看），比如：

* 会话缓存（用户登录后 1 小时内频繁操作，之后可能下线）；
* 临时活动页面（活动期间访问集中，活动结束后很少访问）。

**选 LFU**：如果你的缓存访问具有 “长期高频性”（比如某些基础数据每天都被大量访问），比如：

* 商品类目、地区编码等基础配置数据；
* 首页 banner、热销榜单等高频访问数据。
* 排查技巧：可以先开启 Redis 的`INFO stats`监控`keyspace_hits`（缓存命中）和`keyspace_misses`（缓存未命中），如果切换策略后`keyspace_hits`提升，说明更适合当前业务。

## 总结：LRU 和 LFU 的本质区别与选择

从本质上看，LRU 和 LFU 的核心差异是**判断`价值`的维度不同**：

* LRU 用**最近是否被使用**衡量价值，适合短期热点；
* LFU 用**使用频率高低**衡量价值，适合长期热点。

在 Redis 中，两者都做了性能优化（近似实现），选择时不用纠结 理论完美性，先看业务访问模式：短期集中访问用 LRU，长期高频访问用 LFU。如果实在不确定，不妨先试 LRU（实现更简单，兼容性更好），再根据监控数据调整。

**看完等于学会，点个赞吧！！！**

本博客参考[nuts坚果](https://nutsvqn.com)。转载请注明出处！
