# 高可用（上）

Redis Sentinel 是 Redis 的官方高可用解决方案，是设计用来帮助管理 Redis 实例的系统。用于完成下面 4 个任务： 

- 监控(Monitoring)。Sentinel 不断检查你的主从实例是否运转正常。
- 通知(Notification)。Sentinel 可以通过 API 来通知系统管理员，或者其他计算机程序，被监控的 Redis 实例出了问题。
- 自动故障转移(Automatic failover)。如果一台主服务器运行不正常，Sentinel 会开始一个故障转移过程，将从服务器提升为主服务器，配置其他的从服务器使用新的主服务器，使用 Redis 服务器的应用程序在连接时会收到新的服务器地址通知。
- 配置提供者(Configuration provider)。Sentinel 充当客户端服务发现的权威来源：客户端连接到 Sentinel 来询问某个服务的当前 Redis 主服务器的地址。当故障转移发生时，Sentinel 会报告新地址。

## 分布式特性(Distributed nature) 

Redis Sentinel 是一个分布式系统，这意味着，你通常想要在你的基础设施中运行多个 Sentinel 进程，这些进程使用 gossip 协议来判断一台主服务器是否下线(down)，使用 agreement 协议来获得授权以执行故障转移，并更新相关配置。 

分布式系统具有特定的安全(safety)和活性(liveness)的问题，为了更好地使用 Redis Sentinel，你应该去理解 Sentinel 是如何作为一个分布式系统运转的，至少在较高的层面上。这会让 Sentinel 变得更复杂，但是比单进程系统更好，例如： 

- Sentinel 集群能对主服务器故障转移，即使部分 Sentinel 失败。
- 单个 Sentinel 工作不正常，或者连接不正常，在没有别的 Sentinel 授权的情况下不能故障转移主服务器。
- 客户端可以随机连接到任何一个 Sentinel 来获取主服务器的配置信息。

## 获取 Sentinel(Obtaining Sentinel) 

当前版本的 Sentinel 被称为 Sentinel 2。使用了更强大和简单的算法来重写最初的 Sentinel 实现 (本文后面会解释)。 

稳定版本的 Redis Sentinel 被打包在 Redis 2.8 中，这是最新的 Redis 版本。 

新的开发在不稳定的分支中进行，新的特性一旦稳定了就会合并回 2.8 分支。 

重要：即使你使用的是 Redis 2.6，你也应该使用 Redis 2.8 自带的 Sentinel。Redis 2.6 自带的 Sentinel，也就是 Sentinel 1，已经不赞成使用，并且有很多的 bug。总之，你应该尽快把你的 Redis 和 Sentinel 实例都迁移到 Redis 2.8，以获得更好的全面体验。 

## 运行 Sentinel(Running Sentinel) 

如果你使用 redis-sentinel 可执行文件 (或者如果你有一个叫这个名字的到 redis-server 的符号链接)，你可以使用下面的命令行来运行 Sentinel： 

```
redis-sentinel  /path/to/sentinel.conf  
```

另外，你可以直接使用 redis-server 可执行文件并作为 Sentinel 模式启动： 

```
redis-server  /path/to/sentinel.conf  --sentinel  
```

两种方式是一样的。 

但是，运行 Sentinel 强制使用配置文件，这个文件被系统用来保存当前状态，在重启时能重新加载。如果没有指定配置文件，或者配置文件的路径不可写，Sentinel 将拒绝启动。 

Sentinel 运行时默认监听 TCP 端口 26379，所以为了让 Sentinel 正常运行，你的服务器必须开放 26379 端口，以接受从其他 Sentinel 实例 IP 地址的连接。否则，Sentinel 间就没法通信，没法协调，也不会执行故障转移。 

## 配置 Sentinel(Configuring Sentinel) 

Redis 的源码发行版中包含一个叫做 sentinel.conf 的自说明示例配置文件，可以用来配置 Sentinel，一个典型的最小配置文件看起来就像下面这样： 

```
sentinel  monitor  mymaster  127.0.0.1  6379  2  
sentinel  down-after-milliseconds  mymaster  60000  
sentinel  failover-timeout  mymaster  180000  
sentinel  parallel-syncs  mymaster  1  
  
sentinel  monitor  resque  192.168.1.3  6380  4  
sentinel  down-after-milliseconds  resque  10000  
sentinel  failover-timeout  resque  180000  
sentinel  parallel-syncs  resque  5  
```

你只需要指定你要监控的主服务器，并给每一个主服务器(可以拥有任意多个从服务器)一个不同的名字。没有必要指定从服务器，因为它们会被自动发现。Sentinel 会根据从服务器的额外信息来自动更新配置(为了在重启时还能保留配置)。每次故障转移时将一台从服务器提升为主服务器时都会重写配置文件。 

上面的示例配置监控了两个 Redis 实例集合，每个由一个主服务器和未知数量的从服务器组成。其中一个实例集合叫做 mymaster，另一个叫做 resque。 

为了说得再清楚一点，我们一行一行地来看看这些配置选项是什么意思： 

第一行告诉 Redis 监控一个叫做 mymaster 的主服务器，地址为 127.0.0.1，端口为 6379，判断这台主服务器失效需要 2 个 Sentinel 同意(如果同意数没有达到，自动故障转移则不会开始)。 

但是要注意，无论你指定多少个同意来检测实例是否正常工作，Sentinel 需要系统中已知的大多数 Sentinel 的投票才能开始故障转移，并且在故障转移之后获取一个新的配置纪元(configuration Epoch) 赋予新的配置。 

在例子中，仲裁人数 (quorum) 被设置为 2，所以使用 2 个 Sentinel 同意某台主服务器不可到达或者在一个错误的情况中，来触发故障转移(但是，你后面还会看到，触发一个故障转移还不足以开始一次成功的故障转移，还需要授权)。 

其他的选项基本上都是这样的形式： 

```
sentinel  <option_name>  <master_name>  <option_value>  
```

它们的作用如下： 

down-after-milliseconds 表示要使 Sentinel 开始认为实例已下线(down)，实例不可到达(没有响应我们的 PING，或者响应一个错误) 的毫秒数。这个时间过后，Sentinel 将标记实例为主观下线(subjectively down，也称 SDOWN)，这还不足以开启自动故障转移。但是，如果足够的实例认为具备主观下线条件，实例就会被标记为客观下线(objectively down)。需要同意的 Sentinel 数量依赖于为这台主服务器的配置。 

parallel-syncs 设置在一次故障转移之后，被配置为同时使用新主服务器的从服务器数量。这个数字越小，完成故障转移过程需要的时间就越多，如果从服务器配置为服务旧数据，你可能不太希望所有的从服务器同时从新的主服务器重同步，尽管复制过程通常不会阻塞从服务器，但是在重同步过程中仍然会有一段停下来的时间来加载来自于主服务器的大量数据。设置这个选项的值为 1 可以确保每次只有一个从服务器不可用。 

其他的选项将在本文的剩余篇幅里介绍，Redis 发行版本中自带的示例 sentinel.conf 文件中也有详细的文档。 

所有的配置参数可以在运行时用 SENTINEL SET 命令修改。请看下文中运行时重新配置 Sentinel 这一部分获取更多的信息。 

## 仲裁人数(Quorum) 

本文前面的部分展示了每一个被 Sentinel 监控的主服务器都关联了一个仲裁人数的配置。它指定了同意主服务器不可达或者错误条件需要的 Sentinel 进程数，以触发一次故障转移。 

但是，故障转移被触发后，为了让故障转移真正执行，必须至少大多数的 Sentinel 授权某个 Sentinel 才能错误转移。 

让我们解释的再清楚一些： 

- 仲裁人数：检测错误条件以标记主服务器为 ODOWN 所需要的 Sentinel 进程数。
- 故障转移由 ODOWN 状态触发。
- 一旦故障转移被触发，故障转移的 Sentinel 需要向大多数 Sentinel 请求授权(或者大于大多数，如果仲裁人数设置为大于大多数的话)。

差别看起来很微妙，但是实际上理解和使用起来都相当简单。例如，如果你有 5 个 Sentinel 实例，然后设置仲裁人数为 2，只要有 2 个 Sentinel 认为主服务器不可达就会触发一次故障转移，这两个 Sentinel 仅当得到至少 3 个 Sentinel 的授权时才能故障转移。 

如果设置仲裁人数为 5，所有的 Sentinel 都必须同意主服务器的错误条件，故障转移需要所有 Sentinel 的授权。 

## 配置纪元 (Configuration epochs) 

Sentinel 需要大多数的授权来开启故障转移是有几个重要原因的： 

当一个 Sentinel 得到授权了，就会为故障转移的主服务器获得一个唯一的配置纪元。这是在故障转移完成后用于标记新的配置的一个版本数字。因为大多数同意将一个指定的版本赋予一个指定的 Sentinel，所以其它的 Sentinel 不能使用它。这意味着，每一次故障转移的配置都使用一个唯一的版本来标记。我们会看到为什么这个是如此的重要。 

另外，Sentinel 有一个规则：如果一个 Sentinel 为了指定的主服务器故障转移而投票给另一个 Sentinel，将会等待一段时间后试图再次故障转移这台主服务器。这个延时(delay)是 failover-timeout，你可以在 sentinel.conf 中配置。这意味着，Sentinel 不会同时故障转移同一台主服务器，第一个请求被授权的将会尝试，如果失败了，过一会后另一个将会尝试，等等。 

Redis Sentinel 保证活性(liveness)属性，如果大多数 Sentinel 能够对话，如果主服务器下线，最后只会有一个被授权来故障转移。 

Redis Sentinel 也保证安全(safety)属性，每个 Sentinel 将会使用不同的配置纪元来故障转移同一台主服务器。 

## 配置传播(Configuration propagation) 

一旦一个 Sentinel 能够成功故障转移一台主服务器，会开始广播新的配置，从而使其他 Sentinel 更新关于这台主服务器的信息。 

为了认定故障转移是成功的，需要 Sentinel 能发送 SLAVEOF NO ONE 给选定的从服务器，并将其切换为主服务器，稍后可以在主服务器的 INFO 输出中观察到。 

这时，即使从服务器的重新配置还在进行中，故障转移被认为是成功的，所有的 Sentinel 被要求开始报告新的配置。 

新配置传播的方式，就是为什么我们需要每次 Sentinel 故障转移时被授权一个不同的版本号(配置纪元)的原因。 

每一个 Sentinel 使用 Redis 的发布订阅(Pub/Sub)消息不断地广播主服务器的配置版本，在主服务器上以及所有从服务器上。与此同时，所有的 Sentinel 等待其它 Sentinel 通知的配置消息。 

配置信息在\_\_sentinel\_\_:hello 频道中广播。 

因为每一个配置有一个不同的版本号，所以更大的版本号总是胜过更小的版本号。 

例如，一开始所有的 Sentinel 认为主服务器 mymaster 的配置为 192.168.1.50:6379。这个配置拥有版本 1。一段时间以后，一个 Sentinel 被授权以版本 2 来故障转移。如果故障转移成功，会广播一个新的配置，比如说 192.168.1.50:9000，作为版本 2。所有其他实例会看到这个配置，并相应地更新它们的配置，因为新的配置拥有一个更大的版本号。 

这意味着，Sentinel 保证第二个活性属性：一个可以相互通信的 Sentinel 集合会统一到一个拥有更高版本号的相同配置上。 

基本上，如果网络是分割的，每个分区会统一到一个更高版本的本地配置。在没有分割的特殊情况下，只有一个分区，每个 Sentinel 将会配置一致。 

## SDOWN 和 ODOWN 更多细节 

正如本文已经简要提到的，Redis Sentinel 有两个不同的下线概念，一个被称为主观下线条件(SDOWN)，一个本地 Sentinel 实例的下线条件。另一个称为客观下线条件(ODOWN)，当足够的 Sentinel(至少为主服务器 quorum 参数配置的数量) 具有 SDOWN 条件时就满足 ODOWN，并且使用 SENTINEL is-master-down-by-addr 命令从其它 Sentinel 获得反馈。 

从 Sentinel 的角度来看，如果我们没有在配置的 is-master-down-after-milliseconds 参数的指定时间内收到一个 PING 请求的合法响应，就达到了 SDOWN 的条件。 

PING 的可接受响应可以是以下其中之一： 

- 回复 + PONG。
- 回复 - LOADING 错误。
- 回复 - MASTERDOWN 错误。

其它回复 (或者没有回复) 都被认为是不合法的。 

注意，SDOWN 需要在配置的整个时间区间内没有收到可以接受的回复，例如，如果间隔配置为 30000 毫秒(30 秒)，我们每隔 29 秒收到一个可以接受的 ping 回复，实例被认为是正常工作的。 

从 SDOWN 切换到 ODOWN 没有使用强一致性算法，而仅仅是 gossip 的形式：如果一个指定的 Sentinel 在指定的时间范围内从足够多的 Sentinel 那里获得关于主服务器不工作的报告，SDOWN 就被提升为 ODOWN。如果这种报告不再收到，(ODOWN)标记就会被清除。 

正如已经解释过的，真正开始故障转移需要更严格的授权，但是，如果没有达到 ODOWN 状态，是不会触发故障转移的。 

ODOWN 条件只适用于主服务器。对于其他的实例，Sentinel 不需要任何同意，所以从服务器和其它 Sentinel 永远都不会达到 ODOWN 状态。 

## 自动发现(Auto discovery)	

Sentinel 之间保持着连接来互相检查彼此的可用性，互相交换信息，你不需要在每个你运行的 Sentinel 实例中配置其他 Sentinel 的地址，因为 Sentinel 使用 Redis 主服务器的发布订阅能力来发现监控同一台主服务器的其他 Sentinel。 

这是通过向名为\_\_sentinel\_\_:hello 频道发送问候消息 (Hello Messages) 实现的。 

同样，你不需要配置连接在主服务器上的从服务器列表，因为 Sentinel 会通过询问 Redis 自动发现这个列表。 

- 每个 Sentinel 每隔 2 秒向每个被监控的主服务器和从服务器的发布订阅频道\_\_sentinel\_\_:hello 发送一条消息，报告自己的存在状态：IP 地址，端口号和 runid。
- 每个 Sentinel 订阅了每个主服务器和从服务器的发布订阅频道\_\_sentinel\_\_:hello，寻找未知的 Sentinel。当检测到新的 Sentinel，就将其添加到这台主服务器的 Sentinel 列表中。
- 问候消息也包括主服务器当前的完整配置。如果另一个 Sentinel 拥有一个比接收到的更老的主服务器配置，会立刻更新为新的配置。
- 在添加一个新的 Sentinel 到主服务器前，Sentinel 总是检查是否已经有一个相同的 runid 或者相同地址(IP 地址和端口对)的 Sentinel。如果是的话，所有匹配的 Sentinel 将会被删除，新的被添加。
