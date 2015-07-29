# 使用 Redis 实现 Twitter（下）

把 loadUserInfo 作为一个单独的函数有点大题小做了，但是在复杂的程序中这是一个很好的方法。认证中唯一被遗漏的事情就是登出了。我们怎么来做登出呢？很简单，我们改变 user:1000 的 auth 字段中的随机串，从 auths 哈希中删除旧的认证秘钥，然后添加一个新的。 

重要：登出的步骤解释了为什么我们不是仅仅在 auths 哈希中查看认证秘钥以后认证用户，而是双重检查 user:1000 的 auth 字段。真正的认证字符串是后者，auths 哈希只不过是一个会挥发(volatile)的认证字段，或者，如果程序中 bug 或者脚本被中断，我们会发现 auths 键中有多个对应同一个用户 ID 的入口。登出代码如下(logout.php)： 

```
include("retwis.php");  
  
if (!isLoggedIn()) {  
    header("Location: index.php");  
    exit;  
}  
  
$r = redisLink();  
$newauthsecret = getrand();  
$userid = $User['id'];  
$oldauthsecret = $r->hget("user:$userid","auth");  
  
$r->hset("user:$userid","auth",$newauthsecret);  
$r->hset("auths",$newauthsecret,$userid);  
$r->hdel("auths",$oldauthsecret);  
  
header("Location: index.php");  
```

这就是我们所描述的，你需要去理解的。 

## 帖子(Updates) 

更新(updates)，也就是我们知道的帖子(posts)，更加简单。为了创建一个新的帖子我们这么干： 

```
INCR next_post_id => 10343  
HMSET post:10343 user_id $owner_id time $time body "I'm having fun with Retwis"  
```

如你所见，每篇帖子由有 3 个字段的哈希组成。帖子拥有者的用户 ID，帖子撞见时间，最后是帖子的正文，真正的状态消息。 

创建一个帖子后，我们获取其帖子 ID，LPUSH 其 ID 到帖子作者的每个粉丝用户的时间轴中，当然还有作者自己的帖子列表中(每个人事实上关注了他自己)。post.php 文件展示了这一切是怎么执行的： 

```
include("retwis.php");  
  
if (!isLoggedIn() || !gt("status")) {  
    header("Location:index.php");  
    exit;  
}  
  
$r = redisLink();  
$postid = $r->incr("next_post_id");  
$status = str_replace("\n"," ",gt("status"));  
$r->hmset("post:$postid","user_id",$User['id'],"time",time(),"body",$status);  
$followers = $r->zrange("followers:".$User['id'],0,-1);  
$followers[] = $User['id']; /* Add the post to our own posts too */  
  
foreach($followers as $fid) {  
    $r->lpush("posts:$fid",$postid);  
}  
# Push the post on the timeline, and trim the timeline to the  
# newest 1000 elements.  
$r->lpush("timeline",$postid);  
$r->ltrim("timeline",0,1000);  
  
header("Location: index.php");  
```

函数的核心是这个 foreach 循环。我们使用 ZRANGE 获取当前用户的所有粉丝，然后通过遍历 LPUSH 帖子到每一位粉丝的时间轴列表中。 

注意，我们也为所有的帖子维护了一个全局的时间轴，这样我们就可以在 Retwis 首页轻易的展示每个人的帖子。这只需要执行 LPUSH 到时间轴列表。回到现实，我们难道没有开始觉得在 SQL 中使用 ORDER BY 来排序按照时间顺序添加的东西有一点点奇怪吗？至少我只这么认为的。 

上面的代码有个有意思的地方值得注意：我们对全局时间轴执行完 LPUSH 操作之后使用了一个新命令 LTRIM。这是为了裁剪列表到 1000 个元素。全局时间轴事实上只会用在首页展示少量帖子，没有必要获取全部历史帖子。 

基本上 LTRIM+LPUSH 是 Redis 中创建上限 (capped) 集合的一种方式。 

## 帖子分页(Paginating) 

我们如何使用 LRANGE 来获取一个范围的帖子，展现这些帖子在屏幕上，现在已经相当清楚了。代码很简单：

```
function showPost($id) {  
    $r = redisLink();  
    $post = $r->hgetall("post:$id");  
    if (emptyempty($post)) return false;  
  
    $userid = $post['user_id'];  
    $username = $r->hget("user:$userid","username");  
    $elapsed = strElapsed($post['time']);  
    $userlink = "<a class=\"username\"href=\"profile.php?u=".urlencode($username)."\">".utf8entities($username)."</a>";  
  
    echo('<div class="post">'.$userlink.' '.utf8entities($post['body'])."<br>");  
    echo('<i>posted '.$elapsed.'ago via web</i></div>');  
    return true;  
}  
  
function showUserPosts($userid,$start,$count) {  
    $r = redisLink();  
    $key = ($userid == -1) ? "timeline" : "posts:$userid";  
    $posts = $r->lrange($key,$start,$start+$count);  
    $c = 0;  
    foreach($posts as $p) {  
        if (showPost($p)) $c++;  
        if ($c == $count) break;  
    }  
    return count($posts) == $count+1;  
}  
```

showPost 只是转换和打印一篇 HTML 帖子，showUserPosts 获取一个范围的帖子然后传递给 showPost。 

注意：如果帖子列表很大的话，LRANGE 比较低效，我们想访问列表的中间元素，因为 Redis 列表的背后实现是链表。如果系统设计为为几百万的项分页，那最好求助于有序集合。 

## 关注用户(Following users) 

我们还没有讨论如何创建关注 / 粉丝关系，尽管这并不困难。如果 ID 为 1000 的用户(antirez) 想关注用户 ID 为 5000 的用户(pippo)，我们需要同时创建关注和被关注关系。我们只需要调用 ZADD： 

```
ZADD following:1000 5000  
ZADD followers:5000 1000  
```

仔细关注一下同一个模式。理论上，在关系型数据库中，关注者列表和粉丝列表会在同一张表中，使用像 following_id 和 follower_id 这样的列。你可以使用 SQL 查询来抽取每个用户的关注者和粉丝。在键值数据库中则有一些不同，因为我们需要设置 1000 关注 5000，同时 5000 被 1000 关注的双重关系。这是要付出的代价，但是另一方面，访问数据很简单并相当的快。将这些作为独立的集合可以让我们做一些有意思的事情。例如，使用 ZINTERSTORE 我们可以获得两个不同用户的粉丝的交集，于是我们可以给我们的 Twitter 系统增加一个特性，当你访问某个人的主页时，可以很快的告诉你” 你和 Alice 有 34 个共同粉丝” 这样类似的事情。 

你可以在 follow.php 中找到设置和删除关注/粉丝关系的代码。 

## 水平伸缩(horizontally scalable) 

亲爱的读者，如果你意识到了这一点你就已经是一个英雄了。谢谢你。在讨论水平伸缩之前有必要查看一下单台服务器的性能。Retwis 相当的快，没有任何的缓存。在一台很慢的过载的服务器上，apache 的 benchmark 使用 100 个并发客户端发出 10000 个请求，测量出平均 uv 为 5 毫秒。这意味着单台 Linux 服务器每天可以服务数以百万计的用户，这个像猴子屁股一样的慢，想象一下如果用更新的硬件会是什么结果。 

然而，你不可能永远使用单台服务器，如何伸缩一个键值存储？ 

Retwis 不执行任何多键操作，所以伸缩很简单：你可以使用客户端分片，或者类似于 Twemproxy 的分片代理，或者是即将横空出世的 Redis 集群。 

想更多的了解这个主题请阅读我们的分片文档。这里我们想强调的是，在键值存储系统中，如果你小心设计，数据集是可以拆分到相互独立的小的键上去。相比较使用语义上更复杂的数据库系统，分布这些键到多个节点更简单直接和可预见。 