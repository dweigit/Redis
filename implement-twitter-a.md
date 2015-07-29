# 使用 Redis 实现 Twitter（上）


本文讲述使用 PHP 以及 Redis 来设计和实现一个简单的微博。编程社区传统上认为，在开发 web 应用程序时，作为特殊目的的键值存储数据库不能用于替换关系型数据库。本文将向你展示 Redis 在键值层之上的数据结构是实现各种应用程序的有效数据模型。 

在继续之前，你可以花点时间体验一下在线演示(http://retwis.redis.io，译者注)，看看我们究竟要做什么。长话短说：这是个练手，但是已经足够复杂到让你学习如何创建一个更复杂的程序的基础。 

注意：这篇文章的原始版本写于 2009 年 Redis 发布时。当时还不清楚 Redis 的数据模型适合整个程序。5 年以后的今天，已经有许多应用程序使用 Redis 作为他们的主要存储，所以今天这篇文章的目的就是作为新学者的教程。你讲学习如何使用 Redis 设计一个简单的数据层，如何应用不同的数据结构。 

我们的微博系统，叫做 Retwis，结构简单，具有很高的性能，只需少许努力能够分布于任意数量的 web 和 Redis 服务器。你可以在这里找到源代。 

我使用 PHP 来做这个例子，是因为每个人都能看懂。使用 Ruby，Python，Erlang 等等语言也能得到同样(或更好)的结果。也有一些其他的实现(但是不是所有的实现都使用和当前版本教程同样的数据层，所以请使用 PHP 官方实现会更好)。 

- Retwis-RB 是由 Daniel Lucraft 使用 Ruby 和 Sinatra 实现的版本！当然包含了全部的源代码，以及文章底部一个指向 Git 仓库的链接。本文剩下的部分定位为 PHP，但是 Ruby 程序员也可以查看 Redis-RB 的代码，因为他们从概念上非常相似。
- Retwis-J 是由 Costin Leau 使用 Spring Data Framework 和 Java 实现的版本。代码可以在 GitHub 上找到，springsource.org 上有更全面的文档介绍。

此处省略一万字。。。 

(原文此处是对 Redis 数据类型的介绍，可以参考本系列文章的第 2 篇和第 3 篇，译者注) 

## 前提条件(Prerequisites) 

如果你还没有下载 Retwis 源码请先下载。包含一些 PHP 文件和 Predis 的一份拷贝(例子中我们使用的客户端库)。 

另外你想要做的一件事是一个运行的 Redis 服务器。下载源码，使用 make 构建，使用./redis-server 运行，你就可以开始了。只是玩玩或者运行我们的 Retwis 的话，不需要配置。 

## 数据设计(Layout) 

当使用关系型数据库时，必须先设计数据库模式，这样我们先需要知道表，索引等数据库确定的东西。Redis 没有表，那我们需要设计什么呢？我们需要确定需要什么键来表示我们的对象，以及这些键需要存储什么值。 

让我们从用户开始。我们需要用户名、用户 id，密码，用户粉丝(following)，关注列表等等来表示用户。第一个问题是，我们如果标识一个用户？像在关系型数据库，一个好的解决方案是用不同的号码来标识不同的用户，所以我们可以关联一个唯一 ID 给每个用户。对这个用户的引用通过其 ID。产生唯一 ID 非常简单，使用我们的原子 INCR 操作。当我们创建一个新用户我们就可以(假设用户名为 antirez)： 

```
INCR next_user_id => 1000  
HMSET user:1000 username antirez password p1pp0  
```

注意：在真实程序中你应该使用哈希的密码，为了简化我们直接存储密码明文。 

我们使用 next_user_id 键为每一位新用户提供唯一 ID。然后我们使用唯一 ID 来命名存储用户数据的哈希结构的键。记住，这是使用键值存储的通用设计模式！除了字段已经被定义了以外，我们还需要更多东西来完整定义一个用户。例如，有时通过用户名获得用户 ID，于是我们每次添加一个用户，我们也需要操作用户的键，使用用户名作为字段，用 ID 作为值的哈希。 

```
HSET users antirez 1000  
```

这一开始看起来有点奇怪，但是记住，我们只能采取直接访问数据的方式，而没有第二层索引。没法告诉 Redis 根据一个指定值返回其键。这也是我们的优势。强制我们使用按照主键来访问一切的新的范式来组织数据，此处主键是关系型数据库中的术语。 

## 粉丝(followers)，关注(following)，和帖子(updates) 

我们的系统还有一个核心需求。一个用户可能有很多关注他的用户，我们称他们为其粉丝。一个用户也可能会关注其他用户，我们称他们为其关注者。我们有一个为此量身打造的数据结构，就是集合。独一无二的集合元素，常量时间测试存在性，是两个非常有趣的特性。然而，记录一个用户开始关注另一个用户的时间怎么办？在我们加强版的微博系统里面。我们使用有序集合而不是一个简单的集合，用粉丝或者粉儿的用户 ID 作为元素，用用户关系创建时的 unix 时间作为分数。 

让我们来定义我们的键： 

```
followers:1000 => Sorted Set of uids of all the followers users  
following:1000 => Sorted Set of uids of all the following users  
```

我们添加一个粉丝： 

```
ZADD followers:1000 1401267618 1234 => Add user 1234 with time 1401267618  
```

另外一件重要的事情我们需要一个用户首页的位置来展示用户的更新。我们需要按照时间顺序来访问这些数据，从最近的到最老的，为此最好的数据结构就是列表。基本上每一个更新都会被 LPUSH 到用户更新键，多亏了 LRANGE，我们能实现分页等等。注意，我们可以互换地使用更新(updates)和帖子(posts)这两个词，因为某种意义上说，更新其实就是小型帖子。 

```
posts:1000 => a List of post ids - every new post is LPUSHed here.  
```

这个列表基本上就是用户的时间轴。我们会加入他自己帖子 ID，以及其关注者创建的帖子。基本上我们实现了一个写分列。 

## 身份验证(Authentication) 

好了，我们或多或少已经有了关于用户的一切，除了身份验证。我们会用一种简单而又健壮的方式处理身份验证：我们不想使用 PHP 的会话机制，我们的系统要为轻松地分布式部署于很多 web 服务器上而准备，所以我们会保存全部状态到 Redis 数据库中。所有我们要做的就是要设置一个猜不出来的字符串作为认证用户的 cookie，以及一个持有该字符串的客户端的用户 ID 的一个键。 

我们需要两件事情来使得这个可以工作得健壮。第一，当前认证秘钥(不可猜测的字符串)是用户对象的一部分，所以当创建用户时，我们需要在哈希中设置一个认证字段： 

```
HSET user:1000 auth fea5e81ac8ca77622bed1c2132a021f9  
```

另外，我们需要映射认证秘钥到用户 ID，所以我们也需要一个认证键，使用哈希来映射秘钥和用户 ID。 

```
HSET auths fea5e81ac8ca77622bed1c2132a021f9 1000  
```

为了认证一个用户，我们只需要简单几步(请查看 Retwis 项目中的 login.php 源代码)： 

- 从登陆表单获取用户名和密码。
- 检查用户名是否存在于 users 哈希中。
- 如果存在，我们获取其 ID(例如 1000)。
- 检查 user:1000 的密码是否匹配，否则返回错误消息。
- 认证完毕，设置 "fea5e81ac8ca77622bed1c2132a021f9"(user:1000 的 auth 字段)作为认证 cookie。

这是真实的代码： 

```
include("retwis.php");  
  
# Form sanity checks  
if (!gt("username") || !gt("password"))  
    goback("You need to enter both username and password to login.");  
  
# The form is ok, check if the username is available  
$username = gt("username");  
$password = gt("password");  
$r = redisLink();  
$userid = $r->hget("users",$username);  
if (!$userid)  
    goback("Wrong username or password");  
$realpassword = $r->hget("user:$userid","password");  
if ($realpassword != $password)  
    goback("Wrong useranme or password");  
  
# Username / password OK, set the cookie and redirect to index.php  
$authsecret = $r->hget("user:$userid","auth");  
setcookie("auth",$authsecret,time()+3600*24*365);  
header("Location: index.php");  
```

这些发生在每次用户登录时，但是我们还需要一个 isLoggedIn 函数来检查用户是否已经通过身份认证。以下是 isLoggedIn 函数的逻辑步骤： 

- 从用户获取 auth cookie。如果没有 cookie 则用户没有登录。我们称这个 cookie 值为 <authcookie\>。
- 检查 <authcookie\> 是否存在于 auths 哈希字段中，以及其值(即用户 ID，本例中是 1000)。
- 为了系统更加健壮，验证 user:1000 的 auth 字段是否匹配。
- 用户验证完成，我们从$User 全局变量中加载一些信息。

代码也许比上面的描述更简单： 

```
function isLoggedIn() {  
    global $User, $_COOKIE;  
  
    if (isset($User)) return true;  
  
    if (isset($_COOKIE['auth'])) {  
        $r = redisLink();  
        $authcookie = $_COOKIE['auth'];  
        if ($userid = $r->hget("auths",$authcookie)) {  
            if ($r->hget("user:$userid","auth") != $authcookie) return false;  
            loadUserInfo($userid);  
            return true;  
        }  
    }  
    return false;  
}  
  
function loadUserInfo($userid) {  
    global $User;  
  
    $r = redisLink();  
    $User['id'] = $userid;  
    $User['username'] = $r->hget("user:$userid","username");  
    return true;  
}  
```
