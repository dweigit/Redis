# 集中插入

有时候 Redis 实例需要在短时间内加载大量的已存在数据，或者用户产生的数据，这样，上百万的键将在很短的时间内被创建。 

这被称为集中插入(mass insertion)，这篇文档的目的，就是提供如何最快地向 Redis 中插入数据的一些相关信息。 

## 使用协议，伙计 

使用标准的 Redis 客户端来完成集中插入并不是一个好主意，理由是：一条一条的发送命令很慢，因为你需要为每个命令付出往返时间的花费。可以使用管道(pipelining)，但对于许多记录的集中插入而言，你在读取响应的同时还需要写新命令，以确保插入尽可能快。 

只有少部分的客户端支持非阻塞 I/O，也并不是所有的客户端都能高效地解析响应以最大化吞吐量。基于上述这些原因，首选的集中导入数据到 Redis 中的方式，是生成按照 Redis 协议的原始(raw)格式的文本文件，以调用需要的命令来插入需要的数据。 

例如，如果我需要生成一个巨大的数据集，拥有数十亿形式为”keyN->ValueN” 的键，我将创建一个按照 Redis 协议格式，包含如下命令的文件： 

```
SET Key0 Value0  
SET Key1 Value1  
...  
SET KeyN ValueN  
```

当这个文件被创建后，剩下的工作就是将其尽可能快的导入到 Redis 中。过去的办法是使用 netcat 来完成，命令如下： 

```
(cat data.txt; sleep 10) | nc localhost 6379 > /dev/null  
```

然而，这种集中导入的方式并不是十分可靠，因为 netcat 并不知道所有的数据什么时候被传输完，并且不能检查错误。在 github 上一个不稳定的 Redis 分支上，redis-cli 工具支持一种称为管道模式(pipe mode)的模式，设计用来执行集中插入。 

使用管道模式运行命令如下： 

```
cat data.txt | redis-cli --pipe  
```

输出类似如下的内容： 

```
All data transferred. Waiting for the last reply...  
Last reply received from server.  
errors: 0, replies: 1000000  
```

redis-cli 工具也能够确保仅仅将来自 Redis 实例的错误重定向到标准输出。 

## 生成 Redis 协议(Generating Redis Protocol) 

Redis 协议非常容易生成和解析，可以参考其文档(请关注后续翻译文档，译者注)。但是，为了集中插入的目标而生成协议，你不必了解协议的每一个细节，仅仅需要知道每个命令通过如下方式来表示： 

```
*<args><cr><lf>  
$<len><cr><lf>  
<arg0><cr><lf>  
<arg1><cr><lf>  
...  
<argN><cr><lf>  
```

<cr\>表示 "\r"(或 ASCII 字符 13)，<lf\> 表示 "\n"(或者 ASCII 字符 10)。 

例如，命令 SET key value 通过以下协议来表示： 

```
*3<cr><lf>  
$3<cr><lf>  
SET<cr><lf>  
$3<cr><lf>  
key<cr><lf>  
$5<cr><lf>  
value<cr><lf>  
```

或者表示为一个字符串： 

```
"*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n"  
```

为集中插入而生成的文件，就是由一条一条按照上面的方式表示的命令组成的。 

下面的 Ruby 函数生成合法的协议。 

```
def gen_redis_proto(*cmd)  
proto = ""  
        proto << "*"+cmd.length.to_s+"\r\n"  
        cmd.each{|arg|  
            proto << "$"+arg.to_s.bytesize.to_s+"\r\n"  
            proto << arg.to_s+"\r\n"  
        }  
        proto  
end  
  
puts gen_redis_proto("SET","mykey","Hello World!").inspect  
```

使用上面的函数，可以很容易地生成上面例子中的键值对。程序如下： 

```
(0...1000).each{|n|  
STDOUT.write(gen_redis_proto("SET","Key#{n}","Value#{n}"))  
}  
```

我们现在可以直接以 redis-cli 的管道模式来运行这个程序，来执行我们的第一次集中导入会话。 

```
$ ruby proto.rb | redis-cli --pipe  
All data transferred. Waiting for the last reply...  
Last reply received from server.  
errors: 0, replies: 1000  
```

## 管道模式如何工作(How works) 

redis-cli 管道模式的魔力，就是和 netcat 一样的快，并且能理解服务器同时返回的最后一条响应。 

按照以下方式获得： 

- redis-cli –pipe 尝试尽可能快的发送数据到服务器。
- 与此同时读取可用数据，并尝试解析。
- 当标准输入没有数据可读时，发送一个带有 20 字节随机字符的特殊 ECHO 命令：我们确保这是最后发送的命令，我们也确保可以匹配响应的检查，如果我们收到了相同的 20 字节的批量回复(bulk reply)。
- 一旦这个特殊的最后命令被发送，收到响应的代码开始使用这 20 个字节来匹配响应。当匹配响应到达后成功退出。

使用这个技巧，我们不需要为了知道发送了多少命令而解析发送给服务端的协议，仅仅只需要知道响应就可以。 

但是，在解析响应的时候，我们对所有已解析响应进行了计数，于是最后我们可以告诉用户，通过集中插入会话传输给服务器的命令的数。 