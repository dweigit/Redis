# 从入门到精通（下）

## Redis 有序集合 (Sorted sets) 

有序集合类似于集合和哈希的混合体的一种数据类型。像集合一样，有序集合由唯一的，不重复的字符串元素组成，在某种意义上，有序集合也就是集合。 

集合中的每个元素是无序的，但有序集合中的每个元素都关联了一个浮点值，称为分数(score，这就是为什么该类型也类似于哈希，因为每一个元素都映射到一个值)。 

此外，有序集合中的元素是按序存储的（不是请求时才排序的，顺序是依赖于表示有序集合的数据结构）。他们按照如下规则排序： 

- 如果 A 和 B 是拥有不同分数的元素，A.score > B.score，则 A > B。
- 如果 A 和 B 是有相同的分数的元素，如果按字典顺序 A 大于 B，则 A > B。A 和 B 不能相同，因为排序集合只能有唯一元素。

让我们开始一个简单的例子，添加一些黑客的名字作为有序集合的元素，以他们的出生年份为分数。 

```
> zadd hackers 1940 "Alan Kay"  
(integer) 1  
> zadd hackers 1957 "Sophie Wilson"  
(integer 1)  
> zadd hackers 1953 "Richard Stallman"  
(integer) 1  
> zadd hackers 1949 "Anita Borg"  
(integer) 1  
> zadd hackers 1965 "Yukihiro Matsumoto"  
(integer) 1  
> zadd hackers 1914 "Hedy Lamarr"  
(integer) 1  
> zadd hackers 1916 "Claude Shannon"  
(integer) 1  
> zadd hackers 1969 "Linus Torvalds"  
(integer) 1  
> zadd hackers 1912 "Alan Turing"  
(integer) 1  
```

如你所见，ZADD 命令类似于 SADD，但是多一个参数(位于添加的元素之前)，即分数。ZADD 命令也是可变参数的，所以你可以自由的指定多个分数值对(score-value pairs)，尽管上面的例子中并没有使用。 

使用排序集合可以很容易返回按照出生年份排序的黑客列表，因为他们已经是排序好的。 

实现注意事项：有序集合是通过双端(dual-ported)数据结构实现的，包括跳跃表(skiplist，后续文章会详细介绍，译者注)和哈希表(hashtable)，所以我们每次添加元素时 Redis 执行 O(log(N)) 的操作。这还好，但是当我们请求有序元素时，Redis 根本不需要做什么工作，因为已经是全部有序了： 

```
> zrange hackers 0 -1  
1) "Alan Turing"  
2) "Hedy Lamarr"  
3) "Claude Shannon"  
4) "Alan Kay"  
5) "Anita Borg"  
6) "Richard Stallman"  
7) "Sophie Wilson"  
8) "Yukihiro Matsumoto"  
9) "Linus Torvalds"  
```

注意：0 和 - 1 表示从索引为 0 的元素到最后一个元素(-1 像 LRANGE 命令中一样工作)。 

如果我想按照相反的顺序排序，从最年轻到最年长？使用 ZREVRANGE 代替 ZRANGE： 

```
> zrevrange hackers 0 -1  
1) "Linus Torvalds"  
2) "Yukihiro Matsumoto"  
3) "Sophie Wilson"  
4) "Richard Stallman"  
5) "Anita Borg"  
6) "Alan Kay"  
7) "Claude Shannon"  
8) "Hedy Lamarr"  
9) "Alan Turing"  
```

也可以同时返回分数，使用 WITHSCORES 参数： 

```
> zrange hackers 0 -1 withscores  
1) "Alan Turing"  
2) "1912"  
3) "Hedy Lamarr"  
4) "1914"  
5) "Claude Shannon"  
6) "1916"  
7) "Alan Kay"  
8) "1940"  
9) "Anita Borg"  
10) "1949"  
11) "Richard Stallman"  
12) "1953"  
13) "Sophie Wilson"  
14) "1957"  
15) "Yukihiro Matsumoto"  
16) "1965"  
17) "Linus Torvalds"  
18) "1969"  
```

## 范围操作 (ranges) 

有序集合远比这些要强大。他们可以在范围上操作。让我们获取 1950 年前出生的所有人。我们使用 ZRANGEBYSCORE 命令来办到： 

```
> zrangebyscore hackers -inf 1950  
1) "Alan Turing"  
2) "Hedy Lamarr"  
3) "Claude Shannon"  
4) "Alan Kay"  
5) "Anita Borg"  
```

我们要求 Redis 返回分数在负无穷到 1950 之间的所有元素(包括两个极端)。 

也可以删除某个范围的元素。让我们从有序集合中删除出生于 1940 年到 1960 年之间的黑客： 

```
> zremrangebyscore hackers 1940 1960  
(integer) 4  
```

ZREMRANGEBYSCORE 也许不是最合适的命令名，但是非常有用，返回删除的元素数目。 

另一个非常有用的操作是用来获取有序集合中元素排行的操作。也就是可以询问集合中元素的排序位置。 

```
> zrank hackers "Anita Borg"  
(integer) 4  
```

ZREVRANK 命令用来按照降序排序返回元素的排行。 

## 字典分数 (Lexicographical scores) 

最近的 Redis2.8 版本引入了一个新的特性，假定集合中的元素都具有相同的分数，允许按字典顺序获取范围(元素按照 C 语言中的 memcmp 函数进行比较，因此可以保证没有整理，每个 Redis 实例会有相同的输出)。 

操作字典顺序范围的主要命令是 ZRANGEBYLEX，ZREVRANGEBYLEX，ZREMRANGEBYLEX 和 ZLEXCOUNT。例如，我们再次添加我们的著名黑客清单。但是这次为每个元素使用 0 分数： 

```
> zadd hackers 0 "Alan Kay" 0 "Sophie Wilson" 0 "Richard Stallman" 0  
  "Anita Borg" 0 "Yukihiro Matsumoto" 0 "Hedy Lamarr" 0 "Claude Shannon"  
  0 "Linus Torvalds" 0 "Alan Turing"  
```

根据有序集合的排序规则，他们已经按照字典顺序排好了： 

```
> zrange hackers 0 -1  
1) "Alan Kay"  
2) "Alan Turing"  
3) "Anita Borg"  
4) "Claude Shannon"  
5) "Hedy Lamarr"  
6) "Linus Torvalds"  
7) "Richard Stallman"  
8) "Sophie Wilson"  
9) "Yukihiro Matsumoto"  
```

使用 ZRANGEBYLEX 我们可以查询字典顺序范围： 

```
> zrangebylex hackers [B [P  
1) "Claude Shannon"  
2) "Hedy Lamarr"  
3) "Linus Torvalds"  
```

范围可以是包容性的或者排除性的(取决于第一个字符，即开闭区间，译者注)，+ 和 - 分别表示正无穷和负无穷。查看该命令的文档获取更详细信息(该文档后续即奉献，译者注)。 

这个特性非常重要，因为这允许有序集合作为通用索引。例如，如果你想用一个 128 位无符号整数来索引元素，你需要做的就是使用相同的分数(例如 0)添加元素到有序集合中，元素加上由 128 位大端(big endian)数字组成的 8 字节前缀。由于数字是大端编码，字典顺序排序(原始 raw 字节顺序)其实就是数字顺序，你可以在 128 位空间查询范围，获取元素后抛弃前缀。如果你想在一个更正式的例子中了解这个特性，可以看看 Redis 自动完成范例(后续献上，译者注)。 

## 更新分数：排行榜 (leader boards) 

这一部分是开始新的主题前最后一个关于有序集合的内容。有序集合的分数可以随时更新。对一个存在于有序集合中的元素再次调用 ZADD，将会在 O(log(N))时间复杂度更新他的分数 (和位置)，所以有序集合适合于经常更新的场合。 

由于这个特性，通常的一个使用场景就是排行榜。最典型的应用就是 facebook 游戏，你可以组合使用按分数高低存储用户，以及获取排名的操作，来展示前 N 名的用户以及用户在排行榜上的排行(你是第 4932 名最佳分数)。 

## 位图 (Bitmaps) 

位图不是一个真实的数据类型，而是定义在字符串类型上的面向位的操作的集合。由于字符串类型是二进制安全的二进制大对象(blobs)，并且最大长度是 512MB，适合于设置 232 个不同的位。 

位操作分为两组：常量时间单个位的操作，像设置一个位为 1 或者 0，或者获取该位的值。对一组位的操作，例如计算指定范围位的置位数量。 

位图的最大优势是有时是一种非常显著的节省空间来存储信息的方式。例如，在一个系统中，不同用户由递增的用户 ID 来表示，可以使用 512MB 的内存来表示 400 万用户的单个位信息(例如他们是否需要接收信件)。 

设置和检索位使用 SETBIT 和 GETBIT 命令： 

```
> setbit key 10 1  
(integer) 1  
> getbit key 10  
(integer) 1  
> getbit key 11  
(integer) 0  
```

SETBIT 命令把第一个参数作为位数，第二个参数作为要给位设置的值，0 或者 。如果位的位置超过了当前字符串的长度，这个命令或自动扩充这个字符串。 

GETBIT 命令只是返货指定下标处的位的值。超出范围的位(指定的位超出了该键下字符串的长度)被认为是 0。 

有 3 个操作一组位的命令： 

1. BITOP 命令对不同字符串执行逐位操。提供的操作包括与，或，异或和非。 
2. BITCOUNT 命令执行计数操作，返回被设置为 1 的位的数量。 
3. BITPOS 命令找到第一个值为指定值(0 或者 1)的位。 

BITOPS 和 BITCOUNT 命令都可以操作字符串的字节范围，而不仅仅是运行于整个字符串长度。下面是 BITCOUNT 调用的一个简单例子： 

```
> setbit key 0 1  
(integer) 0  
> setbit key 100 1  
(integer) 0  
> bitcount key  
(integer) 2  
```

位图的通用场景： 

- 各种实时分析
- 需要高性能和高效率的空间利用来存储与对象 ID 关联的布尔信息。


例如，假设你想知道你的网站的用户的最长日访问曲线。你从 0 开始计算天数，也就是你的网站可访问的那天，并且每当用户访问你的网站的时候，就用 SETBIT 命令设置一个位。你可以使用当前 unix 时间减去初始位移，然后除以 3600*24，作为位的下标。 

这种方式下每个用户都有一个记录每天访问信息的一个小字符串。使用 BITCOUNT 命令以及几次 BITPOS 调用，可以很容易获得指定用户访问网站的天数，或者只是获取并在客户端分析位图，也很容易计算出最长曲线。 

位图可以很容易的拆分为多个键，例如为了数据集分片，因为通常要避免使用很大的键。为了将位图拆分为不同的键，而不是将所有的位设置到一个键，一个简单的策略就是每个键只存储 M 位，并且使用位数除以 M 作为键名，在键中使用位数模 M 来定位第 N 位。 

## 超重对数 (HyperLogLogs) 

超重对数是用于计算唯一事物数量的概率性数据结构(学术上指的是估算集合的基数)。通常计算唯一项数量需要使用和你想计算的项成正比的大量内存，因为你需要记住你已经看到的元素，以避免被多次计算。然而，有一组用内存换精度的算法：你会得到一个估算的测量，伴随一个标准错误，在 Redis 的实现中误差低于 1%，但是这些算法的魔力在于，你不再需要使用和你要计算的量成比例的大量内存，你只需要使用常量内存！最坏情况下 12K 字节，或者当你的超重对数(后续称它们为 HLL)只发现了少量元素时更是省内存。 

Redis 中的超重对数，虽然技术上是一个不同的数据结构，但被编码为 Redis 字符串，所以你可以调用 GET 来序列化超重对数，使用 SET 反序列化回服务器。 

从概念上讲，超重对数的 API 像是使用集合来做同样的事情。你会 SADD 元素到集合，使用 SCARD 来检查集合中元素数量，这些元素都是唯一的，因为 SCARD 捕获重复添加已经添加的元素。 

你并没有真正添加项到超重对数中，因为这种数据结构只是包含了状态而没有包含真正的元素，其 API 也是一样： 

- 每次你看到一个新元素，你就使用 PFADD 命令添加。
- 每次你想检索到目前为止当前近似的已添加进去的唯一元素数，你就使用 PFCOUNT 命令。

```
redis> PFADD hll a b c d  
(integer) 1  
redis> PFCOUNT hll  
(integer) 4  
```

这种数据结构的使用场景的一个例子是，计算每天搜索的唯一请求数。 

Redis 也可以执行超重对数的并集，更多信息请继续关注相关命令(请关注此公众号，逐一揭晓)。 

## 其他值得注意的特性 (notable features) 

还有一些重要的 Redis API 没有在此文中探索，但是非常值得你关注： 

你可以增量迭代键空间或者一个很大的集合。 

你可以在服务端运行 Lua 脚本以赢得延迟和带宽。 

Redis 还是也是一个订阅发布服务器。 

## 了解更多 (Learn more) 

这个教程并不完整，只涵盖了基本的 API。阅读命令参考去发现更多。 

欢迎阅读此教程，和 Redis 一起狂舞！ 