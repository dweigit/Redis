# 高可用客户端指引

本文档是一篇草案，其包含的指引将来可能会随着Sentinel项目的进展而改变。 

## 支持Redis Sentinel的Redis客户端指引 

Redis Sentinel是Redis实例的监控解决方案，处理Redis主服务器的自动故障转移和服务发现(谁是一组实例中的当前主服务器)。由于Sentinel具有在故障转移期间重新配置实例，以及提供配置给连接Redis主服务器或者从服务器的客户端的双重责任，客户端需要有对Redis Sentinel的显式支持。 

这篇文档针对Redis客户端开发人员，他们想在其客户端实现中支持Sentinel，以达到如下目标： 

- 通过Sentinel实现客户端的自动配置。
- 改进Sentinel自动故障转移的安全性。# 高可用客户端指引

本文档是一篇草案，其包含的指引将来可能会随着 Sentinel 项目的进展而改变。 

## 支持 Redis Sentinel 的 Redis 客户端指引 

Redis Sentinel 是 Redis 实例的监控解决方案，处理 Redis 主服务器的自动故障转移和服务发现(谁是一组实例中的当前主服务器)。由于 Sentinel 具有在故障转移期间重新配置实例，以及提供配置给连接 Redis 主服务器或者从服务器的客户端的双重责任，客户端需要有对 Redis Sentinel 的显式支持。 

这篇文档针对 Redis 客户端开发人员，他们想在其客户端实现中支持 Sentinel，以达到如下目标： 

- 通过 Sentinel 实现客户端的自动配置。
- 改进 Sentinel 自动故障转移的安全性。

要想获得 Redis Sentinel 如何工作的细节，请查看相关文档(请查看本系列相关文章，译者注)，本文只包含 Redis 客户端开发人员需要的信息，期待读者已经比较熟悉 Redis Sentinel 的工作方式。 

## 通过 Sentinel 实现 Redis 服务发现(Redis service discovery) 

Redis Sentinel 通过像”stats”或”cache”这样的名字来识别每个主服务器。每个名字实际上标识了一组实例，由一个主服务器和若干个从服务器组成。 

网络中用于特定目的的 Redis 主服务器的地址，在一些像自动故障转移，手工触发故障转移(例如，为了提升一个 Redis 实例)，或者其他原因引起的这样的事件后可能会改变。 

通常，Redis 客户端中有一些硬编码的配置来指定 IP 地址和端口作为网络中 Redis 主服务器的地址。但是，如果主服务器的地址改变了，就需要手工介入到每个客户端了。 

支持 Sentinel 的 Redis 客户端可以从使用 Sentinel 的主服务器的名称自动发现 Redis 的地址。所以支持 Sentinel 的客户端应该可以从输入中获得，而不是硬编码的 IP 地址和端口： 

- 指向已知的 Sentinel 实例的 ip:port 对列表。
- 服务的名称，像”timelines”或者”cache”。

下面是客户端为了从 Sentinel 列表和服务名称获得主服务器地址而需要遵循的步骤。 

### 第 1 步：连接第一个 Sentinel(connecting to the first Sentinel) 

客户端应该迭代 Sentinel 地址列表。应该尝试使用较短的超时(大约几百毫秒)来连接到每一个地址的 Sentinel。遇到错误或者超时就尝试下一个 Sentinel 地址。 

如果所有的 Sentinel 地址都没有尝试成功，就返回一个错误给客户端。 

第一个回应客户端请求的 Sentinel 被置于列表的开头，这样在下次重连时，我们会首先尝试在上一次连接尝试是可达的 Sentinel，以最小化延迟。 

### 第 2 步：请求主服务器地址(ask for master address) 

一旦与 Sentinel 的连接建立起来，客户端应该重新尝试在 Sentinel 上执行下面的命令： 

```
SENTINEL get-master-addr-by-name master-name
```

这里的 master-name 应该被替换为用户指定的真实服务名称。 

调用的结果可能是下面两种回复之一： 

- ip:port 对。
- 一个 null 回复。这表示 Sentinel 不知道这个主服务器。

如果收到了 ip:port 对，这个地址应该用来连接到 Redis 主服务器。否则，如果收到了一个 null 回复，客户端应该尝试列表中的下一个 Sentinel。 

### 第 3 步：在目标实例中调用 ROLE 命令(call the ROLE command in the target instance) 

一旦客户端发现了主服务器实例的地址，就应该尝试与主服务器的连接，然后调用 ROLE 命令来验证实例的角色真的是一个主服务器。 

如果 ROLE 命令不可用(Redis 2.8.12 引进的)，客户端可以使用 INFO 复制命令来解析角色：输出中的某一个字段。 

如果实例不是期待中的主服务器，客户端应该等待一小段时间(几百毫秒)然后再尝试从第 1 步开始。 

## 处理重连(Handling reconnections) 

一旦服务名称被解析为主服务器地址，并且与 Redis 主服务器实例的连接已经建立，每次需要重新连接时，客户端应该重新从第 1 步开始使用 Sentinel 来解析地址。例如，下面的情况下需要重新联系 Sentinel： 

- 如果客户端在超时或者 socket 错误后重连。
- 如果客户端因为被显式关闭或者被用户重连而重连。

在上面的情况下或者任何客户端丢失了与 Redis 服务器连接的情况下，客户端应该再次解析主服务器地址。 

## Sentinel 故障转移断开(Sentinel failover disconnection) 

从 Redis 2.8.12 开始，当 Redis Sentinel 改变了实例的配置，例如，提升从服务器为主服务器，故障转移后降级主服务器来复制新的主服务器，或者只是改变一个旧的(stale)从服务器的主服务器地址，会发送一个 CLIENT KILL 类型的命令给实例，来确保所有的客户端都与重新配置过的实例断开。这会强制客户端再次解析主服务器地址。 

如果客户端要联系一个还未更新信息的 Sentinel，通过 ROLE 命令验证 Redis 实例角色会失败，允许客户端发现联系上的 Sentinel 提供了旧的(stale)信息，然后会重试。 

注意：一个旧的主服务器返回在线的同时，客户端联系一个旧的 Sentinel 实例是有可能的，所以客户端可能连接了一个旧的主服务器，然而 ROLE 的输出也是匹配的。但是，当主服务器恢复回来以后，Sentinel 将会尝试将其降级为从服务器，触发一次新的断开。这个逻辑也适用于连接到一个旧的从服务器，其会被重新配置来复制一个不同的主服务器。 

## 连接从服务器(Connecting to slaves) 

有时候客户端有兴趣连接到从服务器，例如，为了分离(scale)读请求。简单修改一下第 2 步就可以支持连接从服务器。不是调用下面的命令： 

```
SENTINEL get-master-addr-by-name master-name
```

客户端应该调用： 

```
SENTINEL slaves master-name
```

用于检索从服务器实例的清单。 

相应地，客户端应该使用 ROLE 命令来验证实例真的是一个从服务器，以防止分离读请求到主服务器。 

## 连接池(Connection pools) 

对于实现了连接池的客户端，当单个连接重连时，应该要再次联系 Sentinel，如果是主服务器的地址改变了，所有已经存在的连接都要关闭并且重新连接到新的地址。 

## 错误报告(Error reporting) 

客户端应该在遇到错误时正确的返回信息给用户，尤其是： 

- 如果没有 Sentinel 能够联系上(这样客户端不可能从 SENTINEL get-master-addr-by-name 获得回复)，应该返回明确表明 Redis Sentinel 不可达的错误。
- 如果所有池中的 Sentinel 返回 null 回复，用户必须被通知 Sentinel 不认识这个主服务器名称的错误。

## Sentinel 列表自动刷新(Sentinels list automatic refresh) 

一旦收到 get-master-addr-by-name 的成功回复，客户端会按照下面的步骤来更新其内部的 Sentinel 节点的列表： 

- 使用 SENTINEL sentinels <master-name>命令获取这台主服务器的其他 Sentinel 列表。
- 添加每个不在列表中的 ip:port 对到列表的后面。

客户端不需要更新自己的配置文件来持久化列表。更新内存中表示的 Sentinel 列表的能力对改进可靠性已经很有用了。 

## 订阅 Sentinel 事件来改进响应能力(Subscribe to Sentinel events to improve responsiveness) 

介绍 Sentinel 的文档中展示了客户端可以使用发布订阅来连接 Sentinel 以订阅 Redis 实例的配置变更。 

这种机制可以用来加快客户端的重配置，也就是，客户端可以监听发布订阅，以知道配置变更什么时候发生，从而运行上文解释的三步协议来解析新的 Redis 主服务器(或者从服务器)地址。 

但是，通过发布订阅收到的变更消息不能代替上面的步骤，因为不能保证客户端可以收到所有的变更消息。 

## 额外信息(Additional information) 

要获得额外信息或者讨论这个指引的特定方面，请发消息到 Redis Google Group。 
