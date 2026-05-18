> **适用场景**: 海量数据处理、高并发系统、内存受限环境下的算法设计
> **前置知识**: 哈希函数、位运算、堆数据结构、前缀树

---

## 1. 分治与文件散列

> **一句话概括**：当数据量超过内存容量时，通过哈希函数将大文件均匀拆分为小文件，使每个小文件能够独立载入内存处理。

### 1.1 原理

核心思想是将大文件通过哈希函数映射到 N 个小文件中。由于同一个数据的哈希值相同，重复数据必然落入同一个小文件，不同小文件之间不可能出现重复数据，这天然实现了问题的分解与并行处理。

**ASCII 原理图**：

```
         +---------------------+
         |     1TB 大文件       |
         +----------+----------+
                    |
          Hash(line) mod N
          +-----+--+--+-----+
          |     |     |     |
          v     v     v     v
        +--+  +--+  +--+  +--+
        | 0|  | 1|  |..|  |N-1|
        +--+  +--+  +--+  +--+
          |     |     |     |
          +-----+-----+-----+
                |
        每个小文件可独立载入内存处理
        （重复数据只出现在同一个小文件中）
```

**递归拆分**：如果某个小文件在首次拆分后仍然超过内存限制（例如某个极热门 URL 出现数亿次导致单文件过大），则对该小文件使用不同的哈希函数再次拆分，或使用 Boyer-Moore 投票算法找出众数后单独处理。

**关键保证**：
- 同一个数据经过相同的哈希函数计算，必然落入同一个小文件
- 不同小文件之间不存在重复数据
- 数据分布均匀（取决于哈希函数质量）

### 哈希函数选择

| 哈希函数 | 输出长度 | 速度 | 冲突率 | 适用场景 |
|----------|----------|------|--------|----------|
| MD5 | 128 bit | 慢 | 极低 | 安全场景、非均匀数据分布 |
| MurmurHash3 | 32/128 bit | 极快 | 低 | 通用文件散列（推荐） |
| FNV-1a | 32/64 bit | 快 | 中等 | 短字符串、简单场景 |

> **冲突处理建议**：MurmurHash3 发生不均匀分布时，可更换种子值再次尝试。一般而言，随机种子配合 MurmurHash3 即可满足绝大多数场景，无需切换到 MD5。

### 1.2 适用场景

| 场景 | 是否适用 | 原因 |
|------|----------|------|
| 大文件去重 | 是 | 重复数据落入同文件，逐个文件使用 HashSet 去重 |
| 大文件词频统计 | 是 | 同词落入同文件，逐个文件使用哈希表统计 |
| 两文件找交集 | 是 | 同哈希函数拆为等量小文件，对应编号比较即可 |
| 存在性判断 | 是 | 散列后配合位图或布隆过滤器处理单文件 |
| 在线实时查询 | 否 | 需要完整拆分过程，I/O 延迟较高 |
| 有序数据范围查询 | 否 | 散列打乱原始顺序，范围查询失效 |

### 1.3 复杂度分析

| 指标 | 值 | 说明 |
|------|-----|------|
| 时间复杂度（拆分） | O(N) | 逐行扫描，一次哈希计算 + 一次文件写入 |
| 时间复杂度（处理） | O(N) | 各小文件串行或并行处理，总数据量不变 |
| 空间复杂度 | O(M x B) | M 个小文件，每个文件分配 B 字节缓冲区 |
| 磁盘占用 | N + overhead | 原文件大小 + 少量文件系统元数据 |
| 内存占用 | M x B | 缓冲区大小与数据总量无关，可独立配置 |

### 1.4 伪代码

```text
// 文件散列分治伪代码 — 按哈希值均匀拆分大文件
function shardFile(inputFile, numShards, hashFunc):
    // 打开所有分片文件的写入流
    writers = array of BufferedWriter(numShards)
    for i in 0..numShards-1:
        writers[i] = openWriter("shard_" + i + ".txt")
    
    // 逐行读取输入文件，计算哈希并写入对应分片
    for each line in inputFile:
        hash = hashFunc(line)
        shardIndex = abs(hash) % numShards
        writers[shardIndex].write(line)
    
    close all writers

// MurmurHash3 32 位哈希（简化描述）
function murmurHash3(data, seed):
    h1 = seed
    for each 4-byte block in data:
        k1 = block as little-endian int32
        k1 = k1 * C1, rotateLeft(k1, 15), k1 = k1 * C2
        h1 ^= k1, rotateLeft(h1, 13), h1 = h1 * 5 + E1
    // 处理尾部 1-3 字节
    h1 ^= len(data)
    // 最终混合（finalization mix）
    h1 = finalizationMix(h1)
    return h1 & 0x7FFFFFFF  // 确保非负结果
```

### 1.5 实战例子

**问题一：100GB 日志文件去重**

问题描述：一个 100GB 的日志文件，每行一条日志记录，内存限制 512MB，要求去除完全重复的行。

分析过程：
- 直接加载到 HashSet 需要约 100GB 内存，不可能
- 拆分为 256 个小文件，每个约 400MB，可逐个载入内存
- 对每个小文件使用 HashSet 去重后写出

核心代码：

```text
// 100GB 日志去重伪代码
function deduplicateLog(inputPath, numShards=256):
    // 第一步：拆分大文件
    shardFile(inputPath, numShards, murmurHash3)

    // 第二步：逐个去重
    dedupWriter = openWriter("deduped.log")
    for i in 0..numShards-1:
        shardPath = "shard_" + i + ".txt"
        if not exists(shardPath): continue

        uniqueLines = new HashSet()
        for each line in readLines(shardPath):
            uniqueLines.add(line)

        for each line in uniqueLines:
            dedupWriter.write(line)

        deleteFile(shardPath)   // 及时释放磁盘

    close(dedupWriter)
    print("Deduplication complete.")
```

结果验证：输出文件中各行唯一，无重复。若某个分片文件仍然过大导致 OOM，可增加分片数（如 512）或对该分片进行二次散列。

**问题二：找两个大文件共同的 URL**

两文件找交集的场景同理，使用相同哈希函数拆分后对应编号比较即可。

***

## 2. 位图法 Bitmap

> **一句话概括**：用 bit 数组的每个二进制位映射一个整数的状态（存在性/词频），以极低的内存开销处理海量整数集合。

### 2.1 原理

位图法的本质是用一个 bit 数组来记录大量整数的状态。每个整数被直接用作 bit 数组的下标，对应位的值表示该整数是否存在或其出现频率。

**ASCII 原理图**：

```
索引（整数）:  0   1   2   3   4   5   6   7   8   9 ...
bit 数组:    [0] [1] [1] [0] [1] [0] [0] [1] [0] [0] ...
存在的整数:       1   2       4           7
```

**状态编码方式**：

| 模式 | 存储密度 | 最大容量 | 状态含义 | 典型应用 |
|------|----------|----------|----------|----------|
| 1bit 存在性 | 1 bit/整数 | 2^32 bit = 512MB | 0=不存在, 1=存在 | 判断 40 亿整数中是否存在某个数 |
| 2bit 词频 | 2 bit/整数 | 2^32 x 2bit = 1GB | 00=无, 01=1次, 10=2次, 11=3次+ | 找出现两次的数 |
| Nbit 扩展 | N bit/整数 | N x 2^32 bit | 自定义状态 | 多级状态统计 |

**关键公式**：位数组大小（字节）= (maxValue + 1) / 8 x bitsPerEntry。对于 int 范围（0 到 2^32 - 1），1bit 模式需要 2^32 / 8 = 512MB。

### 2.2 适用场景

| 场景 | 是否适用 | 原因 |
|------|----------|------|
| 判断整数是否存在 | 是 | 1bit/整数，512MB 覆盖全部 int 范围 |
| 找未出现的整数 | 是 | 遍历 bit 数组，值 0 的位置即为未出现 |
| 找出出现 N 次的整数 | 是 | 2bit 词频模式可精确区分 0/1/2/3+ 次 |
| 整数去重 | 是 | 1bit 模式天然去重 |
| 字符串/URL 存在性 | 否 | 字符串无法直接映射为下标，需配合哈希散列 |
| 浮点数存在性 | 否 | 浮点数范围不连续，无法直接映射 |

### 2.3 复杂度分析

| 指标 | 值 | 说明 |
|------|-----|------|
| 时间复杂度（构建） | O(N) | 遍历一遍数据，每次 O(1) 位运算 |
| 时间复杂度（查询） | O(1) | 位数组随机访问 |
| 空间复杂度（1bit） | 2^32 bit = 512MB | 覆盖全部无符号 int 范围 |
| 空间复杂度（2bit） | 2^32 x 2bit = 1GB | 覆盖全部 int 词频统计 |
| 空间/条目（存在性） | 1 bit | 理论最低存储密度 |
| 额外内存开销 | 无 | 位数组本身即数据结构 |

> **注意**：位图法空间占用固定，与数据规模无关，只与值域范围有关。对于 int 类型，无论有 1 个还是 40 亿个整数，位图始终占用 512MB（1bit 模式）。

### 2.4 伪代码

```text
// 位图伪代码 — 底层使用 long[] 存储，每个 long 容纳 64 位
class BitMap:
    // 构造函数：capacity = 最大索引值 + 1, bitsPerEntry = 每条目 bit 数（1 或 2）
    function init(capacity, bitsPerEntry=1):
        totalBits = capacity * bitsPerEntry
        wordLength = ceil(totalBits / 64)
        words = new long[wordLength]  // 全部初始化为 0

    // 将 index 位置置为 1
    function set(index):
        bitIndex = index * bitsPerEntry
        wordIndex = bitIndex / 64
        bitOffset = bitIndex % 64
        words[wordIndex] |= (1 << bitOffset)

    // 获取 index 位置的值（返回 0 或 1，2bit 模式返回 0-3）
    function get(index):
        bitIndex = index * bitsPerEntry
        wordIndex = bitIndex / 64
        bitOffset = bitIndex % 64
        return (words[wordIndex] >> bitOffset) & ((1 << bitsPerEntry) - 1)

    // 将 index 位置清零
    function clear(index):
        bitIndex = index * bitsPerEntry
        wordIndex = bitIndex / 64
        bitOffset = bitIndex % 64
        words[wordIndex] &= ~(1 << bitOffset)

    // 2bit 模式：设置指定位置的值（0-3）
    function set2Bit(index, value):
        bitIndex = index * 2
        wordIndex = bitIndex / 64
        bitOffset = bitIndex % 64
        words[wordIndex] &= ~(3 << bitOffset)   // 先清除两位
        words[wordIndex] |= (value << bitOffset) // 设置新值

    // 2bit 模式：递增指定位置的值（上限 3）
    function increment2Bit(index):
        current = get(index)
        if current < 3:
            set2Bit(index, current + 1)
        return current + 1
```

### 2.5 实战例子

**问题一：40 亿个非负整数中找到没有出现的数**

问题描述：给定 40 亿个无序非负整数（可能有重复），在 512MB 内存限制下找出一个没有出现的数。

分析过程：
- 非负 int 范围 0 到 2^32 - 1（约 43 亿），给定 40 亿个，至少有几亿个未出现
- 1bit 模式需要 512MB，完全在限制内
- 遍历 40 亿个数，将 bitArray[num] 置为 1
- 最后遍历 bit 数组，值为 0 的位置即为未出现的数

核心代码：

```text
// 40 亿整数中找缺失数伪代码
function findMissingInt(data):
    bitMap = new BitMap(capacity=MAX_INT)  // 512MB 覆盖全部 int 范围
    for each num in data:
        bitMap.set(num)                     // 标记存在

    for i in 0..MAX_INT:
        if bitMap.get(i) == 0:
            print("Missing number: " + i)
            return i

// 逐块读取大文件，避免一次性加载到内存
bitMap = new BitMap(capacity=MAX_INT)
for each batch in readFileInBatches("40亿ints.bin"):
    for each num in batch:
        bitMap.set(num)
```

**问题二：40 亿个无符号整数，1GB 内存，找到所有出现两次的数**

找出现两次的数的场景同理，使用 2bit 模式（00/01/10/11）标识词频状态即可。

***

## 3. 布隆过滤器 Bloom Filter

> **一句话概括**：用位数组和 K 个哈希函数近似判断元素是否在集合中，以可接受的误判率换取极低的内存消耗。

### 3.1 原理

布隆过滤器由一个长度为 m 的位数组和 K 个独立的哈希函数组成。添加元素时，用 K 个哈希函数计算出 K 个位位置并全部置为 1。查询时，检查这 K 个位是否全部为 1——只要有一位为 0，则元素一定不在集合中；全部为 1，则元素可能在集合中。

**ASCII 原理图**：

```
添加元素 x:                    查询元素 y:
  +---+---+---+---+---+---+     +---+---+---+---+---+---+
  | 0 | 1 | 0 | 1 | 1 | 0 |     | 0 | 1 | 0 | 1 | 1 | 0 |
  +---+---+---+---+---+---+     +---+---+---+---+---+---+
    ^       ^           ^           ^   ^       ^
    |       |           |           |   |       |
   h1(x)  h2(x)       hk(x)       h1(y) h2(y) hk(y)
                                所有位为 1 → y 可能∈集合
                                任一位为 0 → y 一定不在集合
```

**关键公式**：

设 n 为元素数量，m 为位数组长度（bit），k 为哈希函数个数：

- **误判率**：p = (1 - e^(-kn/m))^k
- **最优哈希函数个数**：k = (m / n) x ln 2
- **位数组大小**：m = -n x ln p / (ln 2)^2

**推导说明**：当 k = (m/n) x ln 2 时，误判率达到最小值。此时 p ≈ (1/2)^k。

**参数速查表**：

| 误判率 p | 位/元素 (m/n) | 最优 k | 10 亿元素所需内存 |
|----------|---------------|--------|-------------------|
| 10% | 4.8 | 3 | ~572 MB |
| 1% | 9.6 | 7 | ~1.14 GB |
| 0.1% | 14.4 | 10 | ~1.72 GB |
| 0.01% | 19.2 | 14 | ~2.29 GB |

### 3.2 适用场景

| 场景 | 是否适用 | 原因 |
|------|----------|------|
| URL 黑名单 | 是 | 容忍极低误判率，内存节省显著 |
| 爬虫 URL 去重 | 是 | 允许少量重复抓取，避免漏抓 |
| 缓存穿透防护 | 是 | 拦截一定不存在的 key，防止查库 |
| 垃圾邮件过滤 | 是 | 容忍部分误判，配合白名单纠偏 |
| 金融精确扣款 | 否 | 零容错，不能有任何误判 |
| 用户登录校验 | 否 | 需精确匹配，不能误判合法用户 |

### 3.3 复杂度分析

| 指标 | 值 | 说明 |
|------|-----|------|
| 时间复杂度（添加） | O(K) | K 次哈希计算 + K 次位写入 |
| 时间复杂度（查询） | O(K) | K 次哈希计算 + K 次位读取 |
| 空间复杂度 | O(m) bit = O(-n x ln p / (ln 2)^2) bit | m 由 n 和目标误判率 p 决定 |
| 每条目空间（p=0.1%） | ~14.4 bit | 比哈希表节省约 97% 空间 |
| 空间随 n 增长 | 线性 O(n) | m 与 n 成正比 |
| 删除操作 | 不支持 | 位清除会导致其他元素误判 |

### 3.4 伪代码

```text
// 布隆过滤器伪代码 — 位数组 + K 个哈希函数，自动计算最优参数
class BloomFilter:
    // 构造函数：expectedInsertions 预期插入数，falsePositiveRate 目标误判率
    function init(expectedInsertions, falsePositiveRate):
        bitLength = optimalBitLength(expectedInsertions, falsePositiveRate)
        numHashFunctions = optimalHashFunctions(expectedInsertions, bitLength)
        bits = new long[ceil(bitLength / 64)]  // 底层位数组

    // 添加元素：K 次哈希，对应位置 1
    function add(element):
        hashes = hashFunction.hashes(element, numHashFunctions)
        for each hash in hashes:
            index = abs(hash) % bitLength
            wordIndex = index / 64
            bitOffset = index % 64
            bits[wordIndex] |= (1 << bitOffset)

    // 判断元素是否可能在集合中
    function contains(element):
        hashes = hashFunction.hashes(element, numHashFunctions)
        for each hash in hashes:
            index = abs(hash) % bitLength
            wordIndex = index / 64
            bitOffset = index % 64
            if (bits[wordIndex] & (1 << bitOffset)) == 0:
                return false  // 任一位为 0，一定不在
        return true           // 所有位为 1，可能在

    // 计算最优位数组长度：m = -n * ln(p) / (ln 2)^2
    function optimalBitLength(n, p):
        return ceil(-n * ln(p) / (ln(2)^2))

    // 计算最优哈希函数个数：k = (m / n) * ln 2
    function optimalHashFunctions(n, m):
        return max(1, round((m / n) * ln(2)))

// 双重哈希策略：h_i = h1 + i * h2（i = 0..k-1）
function doubleHash(element, k):
    h1 = hash(element, seed=0)
    h2 = hash(element, seed=MAX_INT)
    result = array(k)
    for i in 0..k-1:
        result[i] = abs(h1 + i * h2)
    return result
```

### 3.5 实战例子

**问题：100 亿 URL 黑名单存储与查询**

问题描述：维护一个 100 亿条 URL 的黑名单，每个 URL 平均长度 64 字节。要求在内存受限条件下快速判断一个 URL 是否在黑名单中。

分析过程：

| 方案 | 内存需求 | 可行性 |
|------|----------|--------|
| 哈希表（K-V） | 100 亿 x ~64B = 640GB | 不可行 |
| 哈希表（String 对象开销） | 100 亿 x ~100B = 1TB | 不可行 |
| 布隆过滤器（p=0.1%） | ~14.4 bit/元素 x 100 亿 ≈ 17.2GB | 可行 |
| 布隆过滤器（p=1%） | ~9.6 bit/元素 x 100 亿 ≈ 11.4GB | 可行 |

可以看到，布隆过滤器将 640GB 的哈希表压缩到约 17GB。如果希望进一步降低内存，可将 p 放宽到 1%，只需约 11GB。

核心代码：

```text
// URL 黑名单伪代码 — 基于布隆过滤器
class UrlBlacklist:
    function init():
        // 100 亿 URL，误判率 0.1%
        filter = new BloomFilter(expectedInsertions=10_000_000_000,
                                 falsePositiveRate=0.001)

    // 批量加载黑名单 URL
    function loadBlacklist(blacklistFile):
        for each url in readLines(blacklistFile):
            filter.add(url)

    // 判断 URL 是否在黑名单中
    function isBlacklisted(url):
        return filter.contains(url)

// 演示
blacklist = new UrlBlacklist()
blacklist.filter.add("http://malware.com/virus.exe")
blacklist.filter.add("http://phishing.com/login")
print(blacklist.isBlacklisted("http://malware.com/virus.exe"))  // true
print(blacklist.isBlacklisted("http://safe.com"))               // false
```

结果验证：
- 黑名单 URL 一定会被拦截（零漏判）
- 非黑名单 URL 有 0.1% 的概率被误判，可通过白名单或二次校验纠正
- 内存消耗远低于哈希表方案，在大规模场景下优势尤为明显

> **生产建议**：布隆过滤器通常作为第一道快速过滤层，对于命中的 URL 再查询精确的数据库或缓存进行二次确认，兼顾性能与准确性。

***

## 4. 前缀树 Trie

> **一句话概括**：用树形结构存储字符串集合，共享相同前缀以节省空间，支持高效的插入、查询和前綵匹配。

### 4.1 原理

前缀树（也称字典树）是一棵多叉树。每个节点代表一个字符（不存储在节点中，而是通过边的指向隐式表示），从根节点到叶节点的路径拼接起来即构成一个完整的字符串。每个节点维护一个计数器，记录有多少个字符串经过或终止于该节点。

**ASCII 原理图**（包含 "app"、"apple"、"application"、"bee"）：

```
           (root)
         /       \
        a         b
       /          |
      p           e
     /            |
    p             e
   / \            |
 (e)  l           (e)
      |
      i
      |
      c
      |
      a
      |
      t
      |
      i
      |
      o
      |
      n
     (e)

(e) = 单词结束节点
```

**关键数据结构**：

```java
class TrieNode {
    TrieNode[] children;   // 子节点数组（大小 = 字母表大小）
    int count;             // 以当前节点结尾的单词数
    int prefixCount;       // 经过当前节点的单词数（前缀计数）
}
```

### 4.2 适用场景

| 场景 | 是否适用 | 原因 |
|------|----------|------|
| 词频统计（大量公共前缀） | 是 | 共享前缀节点，比哈希表节省空间 |
| 前缀匹配/自动补全 | 是 | 天然支持前缀查询，O(L) 时间复杂度 |
| URL 去重 | 是 | URL 具有大量公共前缀（协议、域名） |
| 搜索引擎关键词提示 | 是 | 输入前缀即可快速匹配所有候选词 |
| 单词拼写检查 | 是 | 精确匹配 O(L)，支持通配符搜索 |
| 字符串无序集合 | 否 | 哈希表更简单，无前缀共享时 Trie 更耗内存 |

### 4.3 复杂度分析

| 指标 | 值 | 说明 |
|------|-----|------|
| 时间复杂度（插入） | O(L) | L 为字符串长度 |
| 时间复杂度（搜索） | O(L) | L 为字符串长度 |
| 时间复杂度（前缀匹配） | O(L + M) | L 为前缀长度，M 为匹配结果数 |
| 空间复杂度（26 字母） | O(N x L x 26) | N 个字符串，平均长度 L，子节点数组大小 26 |
| 空间复杂度（ASCII） | O(N x L x 128) | 子节点数组大小 128，仅当需要支持全部 ASCII |
| 空间优化方案 | 哈希表/压缩 | 子节点数组改用 HashMap 或路径压缩 |

> **空间对比**：对于 1000 万个随机字符串，哈希表空间开销通常小于前缀树。但若字符串有大量公共前缀（如 URL、文件路径、DNA 序列），前缀树的空间效率显著优于哈希表。

### 4.4 伪代码

```text
// 前缀树（Trie）伪代码 — 26 小写字母，支持插入/搜索/前缀计数/词频统计
class TrieNode:
    children = array of TrieNode(26)  // 子节点数组，初始全为 null
    count = 0                         // 以当前节点结尾的单词数
    prefixCount = 0                   // 经过当前节点的单词数（前缀计数）

class Trie:
    root = new TrieNode()

    // 插入一个单词，更新词频与路径前缀计数
    function insert(word):
        node = root
        for each char c in word:
            index = charToIndex(c)
            if index < 0 or index >= 26: continue  // 跳过非字母
            if node.children[index] == null:
                node.children[index] = new TrieNode()
            node = node.children[index]
            node.prefixCount++
        node.count++  // 单词结束，词频加 1

    // 搜索单词是否完整存在
    function search(word):
        node = traverse(word)
        return node != null and node.count > 0

    // 统计单词出现次数
    function frequency(word):
        node = traverse(word)
        return node.count if node != null else 0

    // 统计以指定前缀开头的单词数量
    function countPrefix(prefix):
        node = traverse(prefix)
        return node.prefixCount if node != null else 0

    // 沿路径遍历，返回路径末端节点
    function traverse(s):
        node = root
        for each char c in s:
            index = charToIndex(c)
            if index < 0 or node.children[index] == null:
                return null
            node = node.children[index]
        return node

    // 获取所有单词及其频率（DFS 遍历收集）
    function allWordsWithFrequency():
        result = {}
        collectWords(root, "", result)
        return result

    function collectWords(node, prefix, result):
        if node == null: return
        if node.count > 0:
            result[prefix] = node.count
        for i in 0..25:
            if node.children[i] != null:
                collectWords(node.children[i], prefix + char('a' + i), result)
```

### 4.5 实战例子

**问题一：1000 万查询串中找出最热门的 10 个查询串**

问题描述：1000 万个搜索查询字符串，去重后不超过 300 万，内存限制 1GB，找出频率最高的 10 个。

分析过程：
- 使用哈希表统计：300 万 x (255 + 4) ≈ 777MB，1GB 内存够用
- 使用前缀树：当查询串具有大量公共前缀时更节省空间，且支持实时统计
- 遍历前缀树拿到所有词频，通过 TopK 堆结构选出 Top10

核心代码：

```text
// TopSearchQueries 伪代码 — 前缀树统计 + 小顶堆找 Top10
function findTopSearchQueries(queriesFile):
    trie = new Trie()

    // 逐行读取查询串并插入前缀树统计词频
    for each query in readLines(queriesFile):
        trie.insert(query.toLowerCase().trim())

    // 收集所有词频
    freqMap = trie.allWordsWithFrequency()

    // 使用小顶堆找 Top10
    minHeap = new MinHeap(size=10)
    for (word, freq) in freqMap:
        minHeap.add((word, freq))  // 自动淘汰堆顶最小值

    // 输出结果（从高到低）
    top10 = minHeap.topK()
    top10.sort(by freq descending)
    print("Top 10 search queries:")
    for i, (word, freq) in top10:
        print((i+1) + ". " + word + " (" + freq + ")")
```

**问题二：URL 去重统计**

URL 去重统计的场景同理，遍历文件插入前缀树并检查是否存在即可。

***

## 5. 堆结构 Heap

> **一句话概括**：利用堆（优先队列）在 O(N log K) 时间内从海量数据中高效筛选出 TopK 个最大或最小元素。

### 5.1 原理

堆是一种完全二叉树，分为小顶堆（Min-Heap）和大顶堆（Max-Heap）。堆的选型规则直接决定了 TopK 的过滤方向。

**选型规则**：

| 目标 | 使用堆类型 | 原因 |
|------|-----------|------|
| 找最大的 K 个元素 | 小顶堆（Min-Heap） | 比堆顶大的元素入堆，淘汰堆顶最小值 |
| 找最小的 K 个元素 | 大顶堆（Max-Heap） | 比堆顶小的元素入堆，淘汰堆顶最大值 |
| 多路归并求 TopK | 小顶堆 | 从各数组/文件的最小值开始合并 |

**小顶堆找最大 TopK 原理图**：

```
数据流: [3, 10, 7, 8, 2, 15, 1, 12]，K = 3

步骤 1: 堆未满，直接入堆        堆: [3]
步骤 2: 入堆                    堆: [3, 10] → 调整 → [3, 10]
步骤 3: 入堆                    堆: [3, 10, 7] → 调整 → [3, 10, 7]
步骤 4: 8 > 3(堆顶), 弹出 3    堆: [7, 10, 8]
步骤 5: 2 < 7(堆顶), 跳过      堆: [7, 10, 8]
步骤 6: 15 > 7(堆顶), 弹出 7   堆: [8, 10, 15]
步骤 7: 1 < 8(堆顶), 跳过      堆: [8, 10, 15]
步骤 8: 12 > 8(堆顶), 弹出 8   堆: [10, 12, 15]

最终 Top3 最大值: [10, 12, 15]
```

### 5.2 适用场景

| 场景 | 是否适用 | 原因 |
|------|----------|------|
| 海量词频 TopK | 是 | 哈希表/前缀树统计后，小顶堆筛选 |
| 多路归并 TopK | 是 | K 个已排序数组，堆维护当前最小值 |
| 中位数查找（数据流） | 是 | 两个堆（一大一小）维护中位数 |
| 任务调度（优先级队列） | 是 | 大顶堆保证高优先级任务先执行 |
| 精确 TopK 排序 | 是 | 堆排序 O(N log K)，优于全排序 O(N log N) |
| 范围 TopK 查询 | 否 | 需先过滤范围再使用堆 |

### 5.3 复杂度分析

| 指标 | 值 | 说明 |
|------|-----|------|
| 时间复杂度（插入） | O(log K) | 堆化（heapify）操作 |
| 时间复杂度（弹出堆顶） | O(log K) | 重新调整平衡 |
| 时间复杂度（整体 TopK） | O(N log K) | N 个元素依次入堆 |
| 空间复杂度 | O(K) | 只维护大小为 K 的堆 |
| 堆排序整体 | O(N log N) | N 个元素全部入堆并出堆 |
| 内存占用/条目 | O(K) | 与 N 无关，仅取决于 K |

> **对比全排序**：当 N 很大而 K 很小时（如 N = 10^9, K = 100），堆方案 O(N log K) 远超全排序 O(N log N)。例如 N=10^9, K=100，堆约需 10^9 x 7 ≈ 70 亿次操作，全排序约需 10^9 x 30 ≈ 300 亿次操作。

### 5.4 伪代码

```text
// TopK 追踪器伪代码 — 基于小顶堆/大顶堆
// 核心规则：找最大 K 个 → 用小顶堆（堆顶最小，淘汰堆顶）；找最小 K 个 → 用大顶堆（堆顶最大，淘汰堆顶）
class TopKTracker:
    // 构造函数
    // k: 目标大小, largest: true=找最大K个(小顶堆), false=找最小K个(大顶堆)
    function init(k, largest=true):
        heap = new MinHeap() if largest else new MaxHeap()
        maxSize = k

    // 添加一个元素
    function add(element):
        if heap.size() < maxSize:
            heap.push(element)
        else if element > heap.peek():  // 比堆顶（最差候选）更好
            heap.pop()                  // 淘汰堆顶
            heap.push(element)          // 入堆

    // 返回当前的 TopK 元素列表（从最优到最差排序）
    function topK():
        result = heap.toList()
        result.sort(descending)
        return result

// 词频条目包装类
class FrequencyEntry:
    word: String
    frequency: int
```

### 5.5 实战例子

**问题一：海量词频 TopK**

问题描述：对 10 亿个搜索词进行词频统计，找出出现次数最多的 100 个词。

分析过程：
- 先通过文件散列将大文件拆分为小文件（第 1 章方法）
- 对每个小文件用哈希表或前缀树统计词频
- 对每个小文件的词频结果使用大小为 100 的小顶堆收集 Top100
- 最终合并所有小文件的 Top100，再次使用小顶堆得到全局 Top100

核心代码：

```text
// 海量词频 TopK 伪代码 — 分治 + 小顶堆合并
function findGlobalTopK(k=100):
    globalTracker = new TopKTracker(k)  // 全局小顶堆

    // 处理每个小文件
    for each shard in 0..15:
        freqMap = countFrequency("shard_" + shard + ".txt")

        // 局部 TopK
        localTracker = new TopKTracker(k)
        for (word, freq) in freqMap:
            localTracker.add(FrequencyEntry(word, freq))

        // 合并到全局
        for entry in localTracker.topK():
            globalTracker.add(entry)

    // 输出全局 Top100
    result = globalTracker.topK()
    print("Global Top " + k + " words:")
    for i, entry in result:
        print((i+1) + ". " + entry.word + " (" + entry.frequency + ")")
```

**问题二：多个已排序数组的 TopK 归并**

多个已排序数组的 TopK 归并场景同理，使用大顶堆从各数组最大值开始逆向归并即可，时间复杂度 O(topK x log K)。

***

## 6. 哈希表 HashMap

> **一句话概括**：通过键值对映射实现 O(1) 平均查找、插入、删除，是海量数据频度统计的基石。

### 6.1 原理

核心思想是将键（Key）通过哈希函数映射为数组下标，从而直接定位到存储位置。理想情况下，每次访问只需 O(1) 时间。

**哈希函数**：将任意大小的输入映射为固定大小的输出（哈希值），再对数组长度取模得到桶索引。好的哈希函数应满足：
- 确定性：相同的输入必须产生相同的输出
- 均匀性：输出应均匀分布在整个值域
- 高效性：计算速度快

**冲突解决**：
- **链地址法（Separate Chaining）**：每个桶维护一个链表（或红黑树），冲突元素挂在链表上。JDK 8+ 的 HashMap 在链表长度超过 8 时转换为红黑树，优化最差情况。
- **开放地址法（Open Addressing）**：冲突时按探测序列寻找下一个空位，包括线性探测、二次探测、双重哈希等。

**负载因子与扩容**：
- 负载因子 = 已存储条目数 / 桶数组长度
- 当超过阈值（默认 0.75）时触发扩容：创建 2 倍大小的新数组，将所有旧条目重新哈希到新位置
- 扩容是 O(N) 操作，均摊后仍保持 O(1) 插入成本

**ASCII 原理图**：

```
                 +-----------------------+
                 |     哈希数组（桶）      |
                 +--+--+--+--+--+--+--+--+
                 |0 |1 | 2|..|..|..|N-1|
                 +--+--+--+--+--+--+--+--+
                    |        |
                    v        v
                  +---+    +---+
                  | K1|    | K3|--->...
                  +---+    +---+
                    |
                    v
                  +---+
                  | K2|
                  +---+
  插入过程:
  key -> hash(key) -> index = hash % N -> table[index] -> 链入链表
```

### 6.2 适用场景

| 场景 | 是否适用 | 原因 |
|------|----------|------|
| 词频统计 | 是 | 词→频度映射，O(1) 更新 |
| 去重判断 | 是 | 利用 key 唯一性 |
| 缓存 KV 存储 | 是 | O(1) 读写 |
| 索引映射 | 是 | ID→对象快速查找 |
| 范围查询 | 否 | 无序结构，无法范围扫描 |
| 有序遍历 | 否 | 遍历顺序不确定 |

### 6.3 复杂度分析

| 指标 | 值 | 说明 |
|------|-----|------|
| 平均查找/插入/删除 | O(1) | 均匀哈希假设下 |
| 最差查找 | O(log N) | 红黑树化后（JDK 8+）；纯链表时为 O(N) |
| 空间复杂度 | O(N) | N 个条目，每个含 key 引用 + value 引用 + 额外开销 |
| 扩容成本（均摊） | O(1) | 均摊到每次插入 |
| 扩容单次成本 | O(N) | 需重新哈希所有条目 |

**HashMap 每条记录内存占用（近似，64 位 JVM，指针压缩开启）**：

| 组件 | 大小 | 说明 |
|------|------|------|
| Node 对象头 | 12 B | mark word 8B + class pointer 4B（压缩） |
| hash 字段 | 4 B | int |
| key 引用 | 4 B | 压缩指针 |
| value 引用 | 4 B | 压缩指针 |
| next 引用 | 4 B | 压缩指针 |
| 内对齐 | 4 B | 对齐到 8 的倍数 |
| **Node 小计** | **32 B** | 不含实际 key/value 对象 |
| String key（约） | 40+ B | char[] + hash + coder + 对象头 |
| Integer value | 16 B | 对象头 + int 值 |
| **合计/条目** | **~88 B** | 含 key/value 对象 |
| **1GB 可容纳** | **~1200 万** | 1024^3 / 88 ≈ 1219 万 |

### 6.4 伪代码

```text
// 频度统计器伪代码 — HashMap 词频统计，超阈值自动溢出到磁盘后合并
class FrequencyCounter:
    // 构造函数：memoryThresholdBytes 为内存阈值，超限触发磁盘溢写
    function init(memoryThresholdBytes, tempDir):
        map = new HashMap()        // 内存中的词频映射
        spillFiles = []            // 溢写文件列表

    // 统计一个单词（增加频次）
    function add(word):
        map[word] = map.get(word, 0) + 1
        if usedMemory() > memoryThresholdBytes:
            spillToDisk()          // 溢出到磁盘

    // 返回最终统计结果
    function result():
        if spillFiles not empty:
            spillToDisk()
            mergeSpillFiles()      // 合并所有溢写文件
        return map

    // 溢写：将当前内存中的结果序列化到临时文件并清空内存
    function spillToDisk():
        spillFile = createTempFile(tempDir, "freq-")
        serialize(map, spillFile)
        spillFiles.add(spillFile)
        map.clear()

    // 合并：反序列化所有溢写文件，合并到内存
    function mergeSpillFiles():
        merged = new HashMap()
        for each spill in spillFiles:
            partial = deserialize(spill)
            for (key, value) in partial:
                merged[key] = merged.get(key, 0) + value
        map = merged
        deleteAll(spillFiles)   // 清理临时文件
```

### 6.5 实战例子

**问题：2GB 内存下统计 20 亿整数的最高频数**

问题描述：给定一个包含 20 亿个整数的文件（约 8GB，每个 int 4 字节），可用内存仅 2GB，找出出现频率最高的整数及其出现次数。

分析过程：
- 2GB 内存可以容纳的 HashMap 条目：2GB / ~88B ≈ 2400 万条（含 key/value 对象开销）
- 但输入数据有 20 亿条，远超过 HashMap 容量
- 需要分治策略：
  1. 将 20 亿个整数通过哈希函数拆分为 N 个小文件（第 1 章方法），N 的选择使每个小文件处理时的 HashMap 不超内存
  2. 每个小文件独立使用 FrequencyCounter 统计
  3. 合并所有小文件的统计结果，使用 TopK 堆找出全局最高频数

具体方案：
- 设 N = 200，期望每个小文件包含约 1000 万个整数
- 每个小文件内，不同整数的数量不会超过 1000 万（因为有重复），HashMap 可容纳
- 使用 MurmurHash3 拆分，确保均匀分布
- 对每个小文件统计，最后合并

核心代码：

```text
// 2GB 内存下 20 亿整数高频统计伪代码
function findHighestFrequency(inputFile):
    NUM_SHARDS = 200
    tempDir = "./freq_shards"

    // 第一阶段：哈希拆分
    splitByHash(inputFile, tempDir, NUM_SHARDS)

    // 第二阶段：逐个处理小文件
    globalFreq = {}
    for i in 0..NUM_SHARDS-1:
        shard = tempDir + "/shard-" + i + ".dat"
        if not exists(shard): continue

        counter = new FrequencyCounter(memoryThreshold=512MB, tempDir)
        for each line in readLines(shard):
            if line not empty:
                counter.add(line)

        for (key, value) in counter.result():
            globalFreq[key] = globalFreq.get(key, 0) + value
        deleteFile(shard)

    // 找出最高频数
    maxKey = argmax(globalFreq, by=value)
    print("最高频数: " + maxKey + " (出现 " + globalFreq[maxKey] + " 次)")

// 按哈希拆分大文件
function splitByHash(input, tempDir, numShards):
    writers = [openWriter(tempDir + "/shard-" + i + ".dat") for i in 0..numShards-1]
    for each line in readLines(input):
        if line not empty:
            num = parseLong(line)
            shard = abs(num XOR (num >> 16)) % numShards
            writers[shard].write(line)
    closeAll(writers)
```

结果验证：通过哈希拆分确保均匀分布，每个小文件独立统计，最后合并。在 2GB 内存限制下，200 路拆分后每个小文件约包含 1000 万个整数，HashMap 统计绰绰有余。

***

## 7. 外排序 External Sort

> **一句话概括**：当待排序数据远超内存容量时，通过分段排序 + 多路归并实现外部存储上的排序。

### 7.1 原理

外排序采用两阶段算法：

**第一阶段：分段排序（Sort Phase）**
- 将大文件分成若干个大小为 M 的 chunk（M <= 可用内存）
- 每个 chunk 读入内存，使用快速排序或 TimSort 排序
- 排序后的 chunk 写回磁盘作为临时文件

**第二阶段：多路归并（Merge Phase）**
- 将 K 个有序的临时文件同时打开
- 每次从 K 个文件中取出最小值输出到结果文件
- 使用败者树（Loser Tree）优化，将比较复杂度从 O(K) 降低到 O(log K)

**ASCII 原理图**：

```
第一阶段：分段排序
  +------------+     +--------+     +--------+
  |  100GB 文件 | --> | Chunk1 | --> | Sort1  | --> temp_1.dat
  |            |     +--------+     +--------+
  |            |     +--------+     +--------+
  |            | --> | Chunk2 | --> | Sort2  | --> temp_2.dat
  |            |     +--------+     +--------+
  |            |     ...            ...
  +------------+     +--------+     +--------+
                     | ChunkM | --> | SortM  | --> temp_M.dat
                     +--------+     +--------+

第二阶段：多路归并
  temp_1.dat  ----+
  temp_2.dat  ----+-->  [败者树] -->  result.dat
  ...             |    K 路归并
  temp_M.dat  ----+
```

**败者树 vs 胜者树的归并路数选择**：

| 指标 | 败者树 | 胜者树 |
|------|--------|--------|
| 每次调整比较次数 | log K | log K |
| 数据结构特点 | 父节点记录败者，根节点为胜者 | 父节点记录胜者，根节点为最小值 |
| 实现复杂度 | 稍复杂 | 较简单 |
| 实际性能 | 略优（减少一次比较） | 相当 |

**归并路数 K 与 I/O 次数的关系**：
- 总数据量 N，内存缓冲区大小 M
- 第一阶段产生约 N/M 个临时文件
- 第二阶段归并趟数 = log_K(N/M)
- I/O 总次数 = N/M + 2 * N * log_K(N/M)（一次读 + 两次写 × 趟数）
- K 越大，趟数越少，I/O 总量越少，但每路分配的缓冲区越小

### 7.2 适用场景

| 场景 | 是否适用 | 原因 |
|------|----------|------|
| 超大文件排序 | 是 | 外排序的核心场景 |
| 数据库外部排序 | 是 | 数据库实现 ORDER BY 的标准方案 |
| 日志按时间排序 | 是 | 日志文件常远超内存 |
| 大型数据集去重后排序 | 是 | 先去重再排序 |
| 小数据量排序 | 否 | 内存足够时直接内排序更高效 |
| 实时快速排序 | 否 | 外排序 I/O 延迟高，不适合在线场景 |

### 7.3 复杂度分析

| 指标 | 值 | 说明 |
|------|-----|------|
| 时间复杂度（I/O） | O(N log N) | 内部排序 + 归并 |
| I/O 次数 | O((N/M) * log_K(N/M)) | 分段写 + 多趟归并读写 |
| 空间复杂度 | O(M + K * B) | M 为排序缓冲区，K 为归并路数，B 为每路缓冲区 |
| 磁盘临时空间 | 2N | 原文件 + 临时文件（可原地归并优化） |
| 内部排序 | O(M log M) | 每 chunk 排序 |

### 7.4 伪代码

```text
// 外排序器伪代码 — 分段排序 + 败者树多路归并
class ExternalSorter:
    // 构造函数
    // chunkSize: 每个 chunk 最大元素数（内存限制）
    // mergeDegree: 归并路数 K（建议 16-64）
    function init(chunkSize, mergeDegree, tempDir):
        tempFiles = []   // 临时文件列表

    // 添加一批元素
    function addAll(elements):
        buffer = []
        for each elem in elements:
            buffer.add(elem)
            if buffer.size() >= chunkSize:
                flushChunk(buffer)
                buffer.clear()
        if buffer not empty:
            flushChunk(buffer)

    // 将内存中的一个 chunk 排序后写入磁盘
    function flushChunk(buffer):
        buffer.sort()    // 内排序（快排/TimSort）
        tempFile = createTempFile("chunk-")
        serialize(buffer, tempFile)
        tempFiles.add(tempFile)

    // 执行排序并返回结果文件路径
    function sort():
        if tempFiles.size() == 1:
            return tempFiles[0]
        return mergeAll(tempFiles)

    // 多趟归并：若文件数超过 K，分批归并
    function mergeAll(files):
        if files.size() <= mergeDegree:
            return mergeOnce(files)
        merged = []
        for batch in groupBy(files, mergeDegree):
            merged.add(mergeOnce(batch))
        return mergeAll(merged)

    // 一次 K 路归并
    function mergeOnce(files):
        result = createTempFile("merged-")
        k = files.size()
        // 打开所有输入流，读取第一个元素
        streams = [openInputStream(f) for f in files]
        tree = new LoserTree(k)
        for i in 0..k-1:
            tree.initialize(i, readNext(streams[i]))

        // 循环取出最小值，写入结果文件
        while true:
            entry = tree.popMin()   // 获取当前最小值及其来源
            if entry == null: break
            write(result, entry.value)
            tree.replace(entry.source, readNext(streams[entry.source]))

        closeAll(streams)
        deleteAll(files)  // 清理临时文件
        return result

// 败者树 — 父节点记录比较中的败者，根节点为最终胜者（最小值）
class LoserTree:
    function init(k):
        loser = new int[k]    // 败者索引
        values = new T[k]     // 当前值

    // 初始化指定位置的元素
    function initialize(idx, value):
        values[idx] = value

    // 替换指定位置的元素
    function replace(idx, value):
        values[idx] = value

    // 弹出当前最小值
    function popMin():
        // 找到当前最小值的索引
        minIdx = index of min(values)  // 沿树路径比较
        if values[minIdx] == null:
            return null
        val = values[minIdx]
        adjust(minIdx)  // 从该位置向上调整败者树
        return (value=val, source=minIdx)
```

### 7.5 实战例子

**问题：100GB 文件按频度排序归并**

问题描述：有一个 100GB 的日志文件，每行一个搜索词，要求按词的出现频度从高到低排序（频度相同按字典序）。可用内存 4GB。

分析过程：
- 第一阶段：使用哈希表统计每个词的频度（输出为 word,frequency 键值对）
  - 100GB 日志，假设平均词长 20 字节，约有 50 亿条记录
  - 不同词的数量假设在 1-10 亿级别
  - 4GB 内存可容纳约 4000 万条 HashMap 条目
  - 需要使用文件散列分治，拆分为 N 个小文件分别统计
- 第二阶段：外排序，按频度降序排列所有 (word, freq) 对
- 如果频度数据文件超出内存，使用 ExternalSorter 进行外排序

核心代码（第二阶段）：

```text
// 频度排序伪代码 — 利用 ExternalSorter 对 (词, 频度) 按频度排序
function sortByFrequency(freqFile):
    // 每行格式: "word,frequency"
    sorter = new ExternalSorter(
        comparator = by frequency descending, then by word ascending,
        chunkSize = 5_000_000,   // 每个 chunk 500 万条
        mergeDegree = 32,         // 32 路归并
        tempDir = "./sort-temp")

    batch = []
    for each line in readLines(freqFile):
        (word, freq) = parseLine(line)
        batch.add(FreqPair(word, freq))
        if batch.size() >= 5_000_000:
            sorter.addAll(batch)
            batch.clear()
    if batch not empty:
        sorter.addAll(batch)

    sortedFile = sorter.sort()
    print("Sorted result: " + sortedFile)
```

结果验证：通过 ExternalSorter 的分段排序 + 败者树多路归并，100GB 频度数据可在有限内存下完成按频度排序。败者树将每次比较的复杂度从 O(K) 降低到 O(log K)，在 K = 32 时每次取最小元素仅需约 5 次比较。

***

## 8. 单向二分查找（按位拆分）

> **一句话概括**：在极大整型数据集中查找中位数时，通过二进制位拆分逐步缩小候选范围，避免将全部数据加载到内存。

### 8.1 原理

核心思想是利用整数在计算机中的二进制表示，从最高位（MSB）到最低位（LSB）逐位确定中位数的每一位。

**算法过程**：
1. 假设 N 个整数的范围是 [0, 2^B-1]（B 为位数），要找第 K 小的数（中位数即第 N/2 小）
2. 从最高位（bit B-1）开始：
   a. 遍历全部数据，统计该位为 0 的元素个数 count0
   b. 如果 K < count0，说明中位数在 bit=0 的分区中，写入 file_0
   c. 否则，中位数在 bit=1 的分区中，K = K - count0，写入 file_1
   d. 丢弃不包含中位数的分区文件
3. 对保留的文件继续处理下一个 bit
4. 处理完所有 B 位后，即得到中位数

**为什么有效**：
- 中位数在有序序列中的位置是固定的（第 N/2 小）
- 按 bit 分组后，bit=0 的所有数全部小于 bit=1 的所有数（在该 bit 及以上更高位相同的前提下）
- 通过计数可以精确知道中位数落在哪个分区
- 每次迭代数据量约减半（期望情况下），B 次迭代后完成

**ASCII 原理图**：

```
初始: N 个整数在 [0, 2^32-1] 范围，找第 N/2 小的数

Bit 31（最高位）:
  +-------------------+-------------------+
  |  bit=0 的数       |  bit=1 的数       |
  |  范围 [0, 2^31-1]  |  范围 [2^31, 2^32-1] |
  |  共 count0 个      |  共 count1 个     |
  +-------------------+-------------------+
  K = N/2
  if K < count0 → 中位数在左边，保留 file_0
  else → K = K - count0, 中位数在右边，保留 file_1

Bit 30:
  ... 继续在保留的文件上拆分 ...
  
最终: 32 次迭代后得到精确中位数值
```

### 8.2 适用场景

| 场景 | 是否适用 | 原因 |
|------|----------|------|
| 超大整型数据集找中位数 | 是 | 内存只需几个变量，全量数据在磁盘上 |
| 极低内存环境（MB 级） | 是 | 缓冲区可配置为 KB 级 |
| 流式数据中位数 | 否 | 需多次遍历，不适合一次到达的数据 |
| 浮点数中位数 | 部分适用 | 需特殊处理 IEEE 754 位表示 |
| 在线实时查询 | 否 | 需多次 I/O 遍历，延迟高 |
| 字符串中位数 | 否 | 字符串位编码不等长，需前缀树配合 |

### 8.3 复杂度分析

| 指标 | 值 | 说明 |
|------|-----|------|
| 时间复杂度（I/O） | O(B * N) | B 为位数（int=32, long=64），N 为元素数 |
| 时间复杂度（CPU） | O(B * N) | 主要为按位判断，低开销 |
| 空间复杂度 | O(B) | 只需几个计数变量 |
| 磁盘临时空间 | 2N * sizeof(T) | 每次迭代需要读写 N 个元素 |
| 迭代次数 | B（int=32, long=64） | 每位一次 |
| 期望每次保留数据量 | N/2 | 均匀分布下每次减半 |

**优化技巧**：
- 前几轮数据量大时，使用大缓冲区（如 1MB）减少系统调用
- 后几轮数据量变小后，可切到内存中直接处理
- 对于已知数据范围的情况，可以从更低的 bit 开始（如已知所有数 < 2^30，则从 bit 29 开始）

### 8.4 伪代码

```text
// 按位中位数查找伪代码 — 逐位拆分文件，内存占用恒定
// 核心思想：从最高位到最低位，逐位统计 0/1 计数，确定中位数所在分区

// 从 int 文件中查找中位数
function findMedianInt(inputPath):
    total = countLines(inputPath)
    k = (total - 1) / 2            // 第 k 小的元素（0-indexed）
    return findKthInt(inputPath, k, 31)

// 递归查找第 k 小的 int 元素
function findKthInt(inputPath, k, bitPos):
    if bitPos < 0:
        return readFirstInt(inputPath)  // 处理完所有 bit，只剩一个值

    // 遍历文件，按当前 bit 拆分为两个临时文件
    file0 = "bit0-" + bitPos + ".dat"   // 该位为 0 的数据
    file1 = "bit1-" + bitPos + ".dat"   // 该位为 1 的数据
    count0 = 0

    for each value in inputPath:
        if (value & (1 << bitPos)) == 0:
            write(file0, value)
            count0++
        else:
            write(file1, value)

    // 判断中位数所在分区
    if k < count0:
        nextInput = file0    // 中位数在 0 分区
        delete(file1)
    else:
        nextInput = file1    // 中位数在 1 分区
        k = k - count0       // 更新目标位置
        delete(file0)

    return findKthInt(nextInput, k, bitPos - 1)

// 从 long 文件中查找中位数（支持 64 位整数）
function findMedianLong(inputPath):
    total = countLines(inputPath)
    k = (total - 1) / 2
    return findKthLong(inputPath, k, 63)
```

### 8.5 实战例子

**问题：100 亿整数找中位数（10MB 内存限制）**

问题描述：有一个包含 100 亿个整数的文件（约 40GB），可用内存仅 10MB，找出这些整数的中位数。

分析过程：
- 10MB 内存无法容纳使用 Quick Select（需要将数组加载到内存）
- 也无法使用两个堆的方案（同样需要内存）
- BitwiseMedianFinder 方案非常适合：
  - 每轮只需读入一行、判断 bit、写出一行
  - 内存只需文件 I/O 缓冲区（如 1MB）和几个计数器
  - 32 轮迭代后即可得到精确中位数
  - 每轮 I/O 总量约 40GB × 2（读写各一次），32 轮 = 2.56TB I/O
  - 期望数据每轮减半，实际 I/O 总量约 80GB × (1 + 1/2 + 1/4 + ...) ≈ 160GB

> **优化**：前 P 轮可一次性处理多个 bit（如前 8 位拆至 256 个文件），将 32 趟 I/O 减少到 25 趟。

结果验证：使用按位拆分方法，在 10MB 内存限制下可以精确找出 100 亿个整数的中位数。该方案的关键优势在于内存占用与数据量无关，仅取决于缓冲区大小配置。

***

## 9. 多线程并发算法

> **一句话概括**：通过线程间协调机制实现高效并发，涵盖无锁化、锁、工具类三种实现范式。

### 9.1 生产者-消费者模式

生产者-消费者模式是多线程编程的经典问题：生产者线程生成数据，消费者线程处理数据，两者通过共享缓冲区解耦。

**三种实现方案对比**：

#### 方案一：无锁化实现（volatile + CAS + 自旋 + 环形缓冲区）

```text
// 无锁环形缓冲区伪代码 — 基于 volatile 实现单生产者-单消费者
class LockFreeRingBuffer:
    function init(capacity):
        buffer = new Object[capacity]  // 2 的幂
        writeIndex = 0   // volatile，生产者写入位置
        readIndex = 0    // volatile，消费者读取位置

    // 生产者尝试写入（非阻塞）
    function tryProduce(item):
        if writeIndex - readIndex >= capacity:
            return false  // 缓冲区满了
        slot = writeIndex % capacity  // 位运算优化
        buffer[slot] = item
        writeIndex = writeIndex + 1   // volatile 写，保证可见性
        return true

    // 消费者尝试读取（非阻塞）
    function tryConsume():
        if readIndex >= writeIndex:
            return null   // 缓冲区空了
        slot = readIndex % capacity
        item = buffer[slot]
        buffer[slot] = null  // 帮助 GC
        readIndex = readIndex + 1
        return item
```

锁实现（synchronized + wait/notify）和 BlockingQueue 工具类实现同样可用于该模式，但无锁化方案在单生产者-单消费者场景下吞吐量最高。

### 9.2 交替打印：两个线程交替打印 A1B2C3...

volatile 自旋实现同样可用于交替打印，但在等待时间不确定的场景下 CPU 消耗较高。

**方案二：Lock + Condition**

```text
// 交替打印 A1B2C3... — Lock + Condition 实现
// 思想：两个 Condition 分别控制字母和数字的等待/唤醒
function alternatePrint():
    lock = new ReentrantLock()
    letterTurn = lock.newCondition()
    numTurn = lock.newCondition()
    isLetterTurn = true

    // 字母线程
    thread(function():
        lock.acquire()
        for c in 'A'..'Z':
            while not isLetterTurn:
                letterTurn.await()      // 等待轮到字母
            print(c)
            isLetterTurn = false
            numTurn.signal()            // 唤醒数字线程
        lock.release()
    )

    // 数字线程
    thread(function():
        lock.acquire()
        for i in 1..26:
            while isLetterTurn:
                numTurn.await()         // 等待轮到数字
            print(i)
            isLetterTurn = true
            letterTurn.signal()         // 唤醒字母线程
        lock.release()
    )
```

Semaphore 实现同理，通过两个信号量交替授权即可。

### 9.3 顺序打印：三个线程按 A→B→C 顺序各打印一次

volatile 自旋实现同样可用于顺序打印，通过共享的 turn 标志控制线程轮流执行。

Lock + Condition 实现同理，通过多个 Condition 分别唤醒各线程。

**方案三：Semaphore**

```text
// 顺序打印 A→B→C — Semaphore 实现
// 思想：每个线程持有一个许可，打印后释放下一个线程的许可
function sequentialPrint():
    sems = [new Semaphore(1), new Semaphore(0), new Semaphore(0)]
    letters = ['A', 'B', 'C']

    for id in 0..2:
        thread(function():
            for i in 0..9:
                sems[id].acquire()                  // 等待自己的许可
                print(letters[id])
                sems[(id + 1) % 3].release()        // 释放下一个线程的许可
        )
```

### 9.4 环形缓冲区（Disruptor 模式）

Disruptor 是 LMAX 公司开源的高性能无锁环形缓冲区，核心设计思想：
- 预分配固定容量的环形数组，避免 GC
- 使用 sequence（序列号）代替锁来控制并发访问
- 单个生产者下完全无锁，多个生产者下使用 CAS
- 通过缓存行填充避免伪共享（False Sharing）

**无锁环形缓冲区原理**：

```
          readSequence          writeSequence
              |                      |
              v                      v
  +----+----+----+----+----+----+----+----+
  |    |    |    |    |    |    |    |    |    环形数组（2 的幂）
  +----+----+----+----+----+----+----+----+
   ^                      ^
   |                      |
  已消费                  已生产

关键规则（单生产者-单消费者）：
- 生产者只能写入 writeSequence 位置
- 消费者只能读取 readSequence 位置
- writeSequence - readSequence <= capacity（不溢出）
- readSequence <= writeSequence（不读取未生产数据）
```

**伪代码**：

```text
// 无锁环形缓冲区伪代码 — Disruptor 模式，单生产者-单消费者，预分配避免 GC
class RingBuffer:
    function init(capacity):
        capacity = nextPowerOf2(capacity)  // 容量向上取整为 2 的幂
        mask = capacity - 1                // 位运算优化取模
        entries = new Object[capacity]     // 预分配数组
        producerSequence = -1              // volatile，生产者序列
        consumerSequence = -1              // volatile，消费者序列

    // 生产一个元素（自旋等待直到有空间）
    function produce(item, timeoutNanos):
        nextSeq = producerSequence + 1
        // 自旋等待消费者释放空间（写满时 producer - consumer >= capacity）
        while nextSeq - capacity >= consumerSequence:
            if timeout exceeded: throw TimeoutException
            spinWait()

        slot = nextSeq & mask
        entries[slot] = item
        producerSequence = nextSeq  // volatile 写，保证 item 可见

    // 消费一个元素（自旋等待直到有数据）
    function consume(timeoutNanos):
        nextSeq = consumerSequence + 1
        // 自旋等待生产者生产
        while nextSeq > producerSequence:
            if timeout exceeded: return null
            spinWait()

        slot = nextSeq & mask
        item = entries[slot]
        entries[slot] = null    // 帮助 GC
        consumerSequence = nextSeq  // volatile 写
        return item
```

### 9.5 方案对比表

| 方案 | 吞吐量 | 延迟 | CPU 消耗 | 适用场景 | 复杂度 |
|------|--------|------|----------|----------|--------|
| volatile 自旋 | 极高 | 极低（ns 级） | 高（忙等） | 低争用、短等待的临界区 | 低 |
| Lock + Condition | 中 | 中（μs 级） | 低（挂起） | 一般并发场景，等待时间不确定 | 中 |
| Semaphore | 中 | 中（μs 级） | 低（挂起） | 资源计数、许可控制 | 中 |
| BlockingQueue | 中 | 中（μs 级） | 低 | 通用生产者-消费者 | 低（API 封装） |
| 无锁 RingBuffer (CAS) | 极高 | 极低（ns 级） | 中（CAS 重试） | 高性能交易系统、日志框架 | 高 |
| synchronized + wait/notify | 中低 | 中（μs~ms 级） | 低 | 简单阻塞场景 | 低 |

**选型建议**：
- 延迟敏感且等待时间极短（< 1μs）→ volatile 自旋
- 通用生产消费 → BlockingQueue（简单可靠）
- 超高吞吐量、GC 敏感 → 无锁 RingBuffer（Disruptor 模式）
- 资源计数控制 → Semaphore
- 复杂条件等待 → Lock + Condition

***

## 10. 系统设计模式

> **一句话概括**：高频系统设计问题的架构方案和实现要点，涵盖短域名、评论入库、在线用户数、红包算法。

### 10.1 短域名系统

**架构图**：

```
                    +-----------+
                    |   客户端   |
                    +-----+-----+
                          | GET tinyurl.com/abc123
                          v
                   +-----+-----+
                   |  Web 服务器 |
                   +-----+-----+
                         |
               +---------+---------+
               |                    |
               v                    v
          +----+----+          +---+---+
          |  发号器   |          | Redis  |
          | (ID gen)  |          | K/V   |
          +----+-----+          +---+---+
               |                    |
               v                    v
          +----+----+          +---+---+
          |  MySQL   |          |  302   |
          | (ID 源)  |          | Redirect|
          +---------+          +--------+
```

**放号器设计**：
- 使用 MySQL auto_increment 或 Redis INCR 获取唯一 ID
- ID 通过 62 进制编码（0-9a-zA-Z）转为短码
- 例如 ID=123456789 → 62 进制 → "8M0k5"
- 62 进制编码的 6 位可容纳 62^6 ≈ 568 亿个 URL

**62 进制转换**：

```text
// 62 进制编码器伪代码 — 数字 ID 与短字符串互转
function encode(id):
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    if id == 0: return "0"
    result = ""
    while id > 0:
        result = chars[id % 62] + result
        id = id / 62
    return result

function decode(str):
    result = 0
    for each char c in str:
        idx = indexOf(c in chars)
        result = result * 62 + idx
    return result
```

**核心流程**：

```text
// 短域名核心服务伪代码
class UrlShortenerService:
    function init(redis, idGen):
        this.redis = redis    // Redis KV 存储
        this.idGen = idGen    // 发号器

    // 创建短链接
    function shorten(longUrl):
        id = idGen.nextId()           // 获取唯一 ID
        code = encode(id)             // 62 进制编码为短码
        redis.set("url:" + code, longUrl)   // Redis 为主存储
        redis.expire("url:" + code, 365天)  // 设置过期时间
        return "https://tinyurl.com/" + code

    // 获取原始 URL（302 重定向时调用）
    function resolve(code):
        url = redis.get("url:" + code)
        if url != null: return url
        // Redis 未命中时回查 MySQL（缓存穿透防护）
        return url
```

**容量规划**：日新增 1000 万短链，5 年 ≈ 182 亿，62 进制 6 位容量 568 亿满足需求。每条 Redis 约 200B，全量需 Redis Cluster 分片。

### 10.2 海量评论入库

**架构图**：

```
                +-----------+
                |   用户端   |
                +-----+-----+
                      | POST /comment
                      v
                +-----+-----+
                |  API 网关   |
                +-----+-----+
                      |
                +-----+-----+
                |  MQ 生产者  |
                +-----+-----+
                      |
                +-----+-----+
                |  消息队列    |
                | (Kafka/RMQ) |
                +-----+-----+
                      |
            +---------+---------+
            |                   |
            v                   v
      +-----+-----+      +-----+-----+
      |  消费者 1   |      |  消费者 N   |
      +-----+-----+      +-----+-----+
            |                   |
            v                   v
      +-----+-----+      +-----+-----+
      |  主库写入    |      |  主库写入    |
      +-----+-----+      +-----+-----+
            |
            v
      +-----+---------+
      |  从库（读）     |
      |  + 缓存（Redis）|
      +----+----------+
           |
           v
     其他查询请求 →
     优先读缓存，
     缓存未命中读从库
```

**核心流程**：
1. 用户提交评论 → API 网关接收请求
2. 写入消息队列（Kafka/RocketMQ），立即返回"提交成功"
3. 消费者从 MQ 拉取消息，批量写入数据库
4. 写入成功后，更新 Redis 缓存（热点文章评论）
5. 查询时优先读 Redis 缓存，缓存未命中读从库

**热点评论缓存**：

```text
// 热点评论缓存伪代码 — 本地 + Redis 两级缓存
class HotCommentCache:
    function init():
        localCache = new ConcurrentHashMap()  // 本地缓存
        // 每分钟清理超过 5 分钟未访问的热点

    function get(articleId):
        return localCache.get(articleId)

    function put(articleId, comments):
        localCache.put(articleId, comments)

    // 定时清理过期热点
    function evictStale():
        for each (articleId, comments) in localCache:
            if lastAccessTime > 5 minutes:
                localCache.remove(articleId)

class Comment:
    id: long
    articleId: long
    content: String
    createTime: long
```

**容量规划**：日评论 1 亿（峰值 5 万 QPS），Kafka 16 分区绰绰有余；数据库按 article_id 分 256 片；Redis 缓存最近 1000 篇热点评论约 40MB。

### 10.3 在线/并发用户数

**方案：Redis ZSET 滑动窗口**

核心原理：使用 Redis 的有序集合（ZSET），以用户 ID 为 member，以最后活跃时间戳为 score。在线用户数 = 统计 score 在 [now - window, now] 范围内的 member 数量。

**ASCII 原理图**：

```
Redis ZSET: online_users
+------------+------------------+
|   Member   |      Score       |
| (user_id)  | (timestamp_ms)   |
+------------+------------------+
| uid_1001   | 1718000000123    |
| uid_1002   | 1718000000456    |
| uid_1003   | 1717999999789    |
| ...        | ...              |
+------------+------------------+

窗口: [now - 5min, now]
查询: ZCOUNT online_users (now-5min) now

过期清理: ZREMRANGEBYSCORE online_users 0 (now-5min)
```

**Java 实现**：

```text
// 在线用户统计伪代码 — Redis ZSET 滑动窗口
// 核心：以用户 ID 为 member，最后活跃时间戳为 score
class OnlineUserCounter:
    KEY = "online_users"
    WINDOW = 5 minutes     // 5 分钟滑动窗口

    function init(redis):
        this.redis = redis

    // 用户活跃心跳（每次请求时上报）
    function heartbeat(userId):
        now = currentTimeMillis()
        redis.zadd(KEY, now, userId)

    // 查询当前在线用户数（过去 5 分钟活跃的独立用户数）
    function count():
        now = currentTimeMillis()
        windowStart = now - WINDOW
        return redis.zcount(KEY, windowStart, now)

    // 清理过期数据（可定时执行）
    function cleanup():
        now = currentTimeMillis()
        cutoff = now - WINDOW
        redis.zremrangeByScore(KEY, 0, cutoff)
```

**容量规划**：每个 ZSET 条目约 68B，DAU 1 亿时窗口内约 1000 万条 ≈ 680MB，过期清理后保持稳定。ZADD/ZCOUNT 时间复杂度 O(log N)，单机可支撑 10 万 QPS。

### 10.4 红包算法

**两种常用算法对比**：

#### 线性切割法

线性切割法在总金额区间随机选取 K-1 个切分点，排序后区间差值即为每个红包金额，但需预分配且可能产生极端值。

#### 二倍均值法

思想：每个红包的金额 = 随机范围 [1, 剩余金额 / 剩余个数 × 2]，保证每个人至少抢到 1 分且期望值相等。

```text
// 红包算法伪代码 — 二倍均值法
// 思想：每个红包金额 = 随机范围 [1, 剩余平均金额 × 2]，保证期望值相等
function splitRedPacket(totalAmount, count):
    // totalAmount: 总金额（单位：分）, count: 红包个数
    amounts = []
    remaining = totalAmount
    remainingCount = count

    for i in 0..count-2:
        max = (remaining / remainingCount) * 2  // 最大金额 = 剩余平均金额 × 2
        amount = random(1, max)                  // 随机 [1, max]
        amounts.add(amount)
        remaining = remaining - amount
        remainingCount = remainingCount - 1

    amounts.add(remaining)  // 最后一个红包拿全部剩余
    return amounts
```

#### 算法对比

| 指标 | 线性切割法 | 二倍均值法 |
|------|-----------|-----------|
| 随机性 | 完全均匀随机 | 期望值相等，后半段波动小 |
| 实现复杂度 | 需排序，O(K log K) | 无需排序，O(K) |
| 金额分布 | 可能产生极端值 | 分布更温和 |
| 适用场景 | 抢红包、"拼手气" | 金额均匀的场景 |
| 是否支持动态领取 | 否（需预分配） | 是（可逐次计算） |

**微信红包实际方案**：采用二倍均值法的变体，每次抢红包时实时计算，支持高并发下的动态领取。

***

## 11. 附录

### A. 算法选型速查表

| 场景 | 推荐算法 | 关键数据结构 | 章节 |
|------|----------|-------------|------|
| 大文件拆分 | 文件散列分治 | 哈希函数 + 文件缓冲区 | 第 1 章 |
| 海量数据去重 | 布隆过滤器 | 位数组 + K 个哈希函数 | 第 2 章 |
| 存在性判断 | 布隆过滤器 / 位图 | 位数组 | 第 2 章 |
| 大整数范围统计 | 位图法 | BitSet / RoaringBitmap | 第 2 章 |
| 词频统计（内存充足） | 哈希表 | HashMap | 第 6 章 |
| 词频统计（内存受限） | 文件散列 + 哈希表 | 分治 + HashMap | 第 1, 6 章 |
| TopK 频度词 | 小顶堆 | PriorityQueue | 第 5 章 |
| 海量数据排序 | 外排序 | 败者树 + 文件分区 | 第 7 章 |
| 超大整数中位数 | 按位拆分 | 文件位拆分 + 计数器 | 第 8 章 |
| 数据流中位数 | 双堆 | 大顶堆 + 小顶堆 | 第 5 章 |
| 高频众数 | Boyer-Moore 投票 | 计数器 | 第 2 章提及 |
| 缓存 KV 存储 | 哈希表 / Redis | HashMap / 跳表 | 第 6, 10 章 |
| 生产者-消费者 | 阻塞队列 / 无锁环形缓冲区 | BlockingQueue / RingBuffer | 第 9 章 |
| 顺序控制 | Condition / Semaphore | Lock + Condition | 第 9 章 |
| 系统高可用设计 | 读写分离 / MQ 异步 | 消息队列 + 缓存 | 第 10 章 |
| 在线用户统计 | ZSET 滑动窗口 | Redis ZSET | 第 10 章 |
| 红包分配 | 二倍均值法 | 随机数 + 动态计算 | 第 10 章 |

### B. 内存估算参考

**基础换算**：
- 1 KB = 2^10 B ≈ 1024 B
- 1 MB = 2^20 B ≈ 104 万 B
- 1 GB = 2^30 B ≈ 10.7 亿 B
- 1 TB = 2^40 B ≈ 1.1 万亿 B

**各数据结构每条目开销与 1GB 可容纳量**：

| 数据结构 | 每条目开销（近似） | 说明 | 1GB 可容纳量 |
|----------|------------------|------|-------------|
| int 原始数组 | 4 B | 纯数据 | ~2.68 亿 |
| long 原始数组 | 8 B | 纯数据 | ~1.34 亿 |
| Integer 对象 | 16 B | 对象头 + 4B 数据 + 对齐 | ~6700 万 |
| String（平均 10 字符） | 56 B | 对象头 + char[] + coder + hash | ~1900 万 |
| HashMap Node | 32 B | 对象头 + hash + 3 引用 + 对齐 | ~3350 万 |
| HashMap Node + String key | ~88 B | Node 32B + String 56B | ~1200 万 |
| HashMap Node + String key + Integer value | ~104 B | 增加 Integer 16B | ~1000 万 |
| HashSet 条目 | ~32 B | 内部使用 HashMap | ~3350 万 |
| ArrayList<Integer> | ~20 B / 元素 | 数组引用 + Integer 对象 | ~5200 万 |
| PriorityQueue（堆） | ~32 B / 元素 | 内部数组 + 对象头 | ~3350 万 |
| 位图（BitSet） | 1 bit / 条目 | 每 bit 代表一个值 | ~85.9 亿个值 |
| 布隆过滤器（1% 误判率） | ~9.6 bits / 条目 | K 个哈希函数 | ~8.9 亿条（1GB） |
| Redis ZSET 条目 | ~68 B | member + score + 元数据 | ~1500 万 |
| 文件缓冲区 | 配置决定 | byte[]，典型 64KB-1MB | 与数据量无关 |

**内存估算公式**：
- 总内存 ≈ 条目数 ×（单条目开销 + 额外元数据）×（1 + 负载因子裕量）
- HashMap 实际可用 ≈（配置内存 × 0.7）/ 单条目开销（留 30% 给 JVM 其他开销和负载因子）

### C. 复杂度总览表

| 算法 | 时间复杂度 | 空间复杂度 | 数据类型 | 额外 I/O |
|------|-----------|-----------|----------|---------|
| 文件散列分治 | O(N) | O(M × B) | 通用 | N（写一次原数据） |
| 位图统计 | O(N) | O(range / 8) | 整数 | 0（纯内存） |
| 位图排序 | O(N) | O(range / 8) | 无重复整数 | 0（纯内存） |
| 布隆过滤器（构建） | O(N × K) | O(m) | 通用 | 0（纯内存） |
| 布隆过滤器（查询） | O(K) | O(m) | 通用 | 0（纯内存） |
| Boyer-Moore 投票 | O(N) | O(1) | 可比较 | 0（纯内存） |
| 小顶堆 TopK | O(N log K) | O(K) | 可比较 | 0（纯内存） |
| 双堆中位数 | O(log N) / 插入 | O(N) | 可比较 | 0（纯内存） |
| 外排序 | O(N log N) | O(M + K × B) | 可比较 + Serializable | O(N log_K(N/M)) |
| 按位拆分找中位数 | O(B × N) | O(B) | 整数 | O(B × N) |
| HashMap 操作 | O(1) 平均 | O(N) | 通用 | 0（纯内存） |
| HashMap 扩容 | O(N) 均摊 O(1) | O(2N) 临时 | 通用 | 0（纯内存） |
| 无锁环形缓冲区 | O(1) | O(capacity) | 通用 | 0（纯内存） |
| 多路归并（败者树） | O(N log K) | O(K) | 可比较 | 0（纯内存归并阶段） |
| BitSet 位操作 | O(1) | O(N / 64) | 整数 | 0（纯内存） |
| 短域名（62 进制编码） | O(log_BASE N) | O(1) | 长整型 ID | 0（纯内存计算） |
| ZSET 滑动窗口 | O(log N) | O(N) | 用户 ID | 0（Redis 内存） |
| 红包二倍均值法 | O(K) | O(K) | 金额 | 0（纯内存计算） |

**参考说明**：
- N = 数据总量，M = 分片数，B = 每片缓冲区大小，K = 堆大小/归并路数/红包个数
- range = 整数取值范围（如 int 为 2^32）
- m = 位数组大小（布隆过滤器）
- range / 8 = 位图字节数
- O(B × N) 表示 B 次全量遍历

***

Happy learning!
