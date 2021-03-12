![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea3769109f2145a6ab4e09b407fefa4c~tplv-k3u1fbpfcp-zoom-1.image)

这篇文章记录了给 Apache 顶级项目 - 分库分表中间件 ShardingSphere 提交 Bug 的历程。

说实话，这是一次比较曲折的 Bug 跟踪之旅。10月28日，我们在 GitHub 上提交 issue，中途因为官方开发者的主观臆断被 Close 了两次，直到 11 月 20 日才被认定成 Bug 并发出修复版本，历时 20 多天。

本文将还原该 Bug 的分析过程，将有价值的经验和技术点进行提炼。通过本文，你将收获到：

> 1、疑难问题的排查思路
> 
> 2、数据库中间件 Sharding Proxy 的原理
> 
> 3、MySQL 预编译的流程和交互协议
> 
> 4、Wireshark 抓包分析 MySQL 的奇淫技巧

# 01 问题描述 

这个 Bug 来源于我的公号读者，他替公司预研 ShardingProxy（属于 ShardingSphere 的子产品，可用作分库分表，后文会详细介绍）。他按照官方文档写了一个很简单的 demo，但是运行后无法查询出数据。

下面是他遇到问题后发给我的信息，希望我能帮忙一起定位下原因。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59a6c5dea5514865aff377cabfe98f23~tplv-k3u1fbpfcp-zoom-1.image)![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f5b3636022334d439507cff210d299d1~tplv-k3u1fbpfcp-zoom-1.image)

截图中的 doc 详细记录了 ShardingProxy 的配置、调试分析日志、以及问题的具体现象。

为了方便大家理解，我重新描述下这个 Demo 的业务逻辑以及问题表象。

## 1. Demo 的业务逻辑说明

这个 Demo 很简单，主要为了跑通 ShardingProxy  的分库分表功能。程序用 SpringBoot + MyBatis 实现了一个单表的查询逻辑，然后用这张表的一个 long 类型字段作为分区键，并通过 ShardingProxy 进行了分表。

下面是那张数据表的详细定义，共 16 个字段，大家关注前两个字段即可，其他字段和本文提到的 Bug 无关。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4cf031a514dc40748847b1c61bc1754a~tplv-k3u1fbpfcp-zoom-1.image)

前两个字段的作用如下：

-   BIZ_DT：业务字段，date类型，和Bug有关
    
-   ECIF\_CUST\_NO：bigint 类型，用做分区键
    

代码就是 Controller 调用 Service，Service 调用 Dao，Dao 通过 MyBatis 实现，这里就不粘贴了。

由于使用了 ShardingProxy 中间件，因此它跟直连数据库的配置会有所不同，在定义 dataSource 时，url 需要配置成这样：

> jdbc:mysql://127.0.0.1:3307/sharding_db?useServerPrepStmts=true&cachePrepStmts=true&serverTimezone=UTC

可以看到，jdbc 连接的是 ShardingProxy 的逻辑数据源 sharding_db，端口使用的是 3307，并非真正的底层数据库以及 MySQL Server 的真实端口 3306，具体原理下文会介绍到。其中，标蓝色的 useServerPrepStmts 和 cachePrepStmts 这两个参数，和本文说的 Bug 有关，这里先提一下，后面会具体分析。

另外，ShardingProxy 的分表策略是：用 long 类型的 ecif\_cust\_no 字段对 2 进行取模，分成了两张表。具体配置如下：

> shardingColumn: ecif\_cust\_no
> 
> algorithmExpression: pscst\_prdt\_cvr${ecif\_cust\_no % 2}

## 2. 问题描述

再说下遇到的问题。首先，往数据表中预先插入一条 ECIF\_CUST\_NO 等于 10000 的数据：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d83af34c53de4dc991162f5d4ab7d09b~tplv-k3u1fbpfcp-zoom-1.image)![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/663688817ddc4a70bc46ae4aaae05234~tplv-k3u1fbpfcp-zoom-1.image)

然后启动 demo 程序，使用 curl 发起 post 请求，查询 ecifCustNo 等于 10000 的那条记录，居然查询不出数据：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f936dc896b648a89998b1d42a897a35~tplv-k3u1fbpfcp-zoom-1.image)![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f22c75c375d647209e7ea596dafe465c~tplv-k3u1fbpfcp-zoom-1.image)

至此，背景基本交代清楚了，为什么数据库中明明有数据，但是程序却查询不出来呢？问题到底出现在 ShardingProxy，还是应用程序本身？

# 02 ShardingProxy 原理简介 

在开启这个问题的分析过程之前，我先快速普及下 ShardingProxy 的基本原理，以便大家能更好的理解我的分析思路。

开源的数据库中间件大家一定接触过，最流行的是 MyCat 和 ShardingSphere。其中 MyCat 是阿里开源的；ShardingSphere 是由当当网开源，并在京东逐渐发展壮大，于 2020 年成为了 Apache 顶级项目。

ShardingSphere 的目标是一个生态圈，它由非常著名的 ShardingJDBC、ShardingProxy、ShardingSidecar 3 款独立的产品组成。本文重点普及下 ShardingProxy，另外两个就不展开了。

## 1\. 什么是 ShardingProxy ？

ShardingProxy 属于和 MyCat 对标的产品，定位为透明化的数据库代理端，可以理解成：一个实现了 MySQL 协议的 Server（独立进程），可用于读写分离、分库分表、柔性事务等场景。

对于应用程序或者 DBA 来说，可以把 ShardingProxy 当做数据库代理，能用 MySQL 客户端工具（Navicat）或者命令行和它直接交互，而 ShardingProxy 内部则通过 MySQL 原生协议与真实的 MySQL 服务器通信。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b09d4305c55d4eb580b79bc317852bb8~tplv-k3u1fbpfcp-zoom-1.image)![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9565322f88394de6b22107ba02986c94~tplv-k3u1fbpfcp-zoom-1.image)

从架构图来看，ShardingProxy 就相当于 MySQL，它本身不存储数据，但是对外屏蔽了 Database 的存储细节，你可以用连接 MySQL 的方式去连接 ShardingProxy（除了端口不同），用你熟悉的 ORMapping 框架使用它。

## 2\. ShardingProxy 的内部架构

再来看下 ShardingProxy 的内部架构，后续源码分析时会涉及到此部分。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4aae448027b946a38c3e20ab57c6db22~tplv-k3u1fbpfcp-zoom-1.image)

整个架构分为前端、核心组件和后端：

前端（Frontend）负责与客户端进行网络通信，采用的是 NIO 框架，在通信的过程中完成对MySQL协议的编解码。

核心组件（Core-module）得到解码的 MySQL 命令后，开始调用 Sharding-Core 对 SQL 进行解析、改写、路由、归并等核心功能。

后端（Backend）与真实数据库交互，采用 Hikari 连接池，同样涉及到 MySQL 协议的编解码。

## 3\. ShardingProxy 的预编译 SQL 功能

本文的 Bug 跟 ShardingProxy 的预编译 SQL 有关，这里单独介绍下此功能以及与之相关的 MySQL 协议，这个是本文的关键，请耐心看完。

熟悉数据库开发的同学一定了解：预编译 SQL（PreparedStatement），在数据库收到一条 SQL 到执行完毕，一般分为以下 3 步：

> 1、词法和语义解析
> 
> 2、优化 SQL，制定执行计划
> 
> 3、执行并返回结果

但是很多情况下，一条 SQL 语句可能会反复执行，只是执行时的参数值不同。而预编译功能将这些值用占位符代替，最终达到一次编译、多次运行的效果，省去了解析优化等过程，能大大提高 SQL 的执行效率。

假设我们要执行下面这条 SQL 两次：

```
SELECT * FROM t_user WHERE user_id = 10;
```

那 JDBC 和 MySQL 之间的协议消息如下：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/15c280767ec74cbebf117be188853d5b~tplv-k3u1fbpfcp-zoom-1.image)

通过上述流程可以看到：第 1 条消息是PreparedStatement，查询语句中的参数值用问号代替了，它告诉 MySQL 对这个SQL 进行预编译；第 2 条消息 MySQL 告诉 JDBC 准备成功了；第 3 条消息 JDBC 将参数设置为 1 ；第 4 条消息 MySQL 返回查询结果；第 5 条和第 6 条消息表示第二次执行同样的 SQL，已经无需再次预编译了。

再回到 ShardingProxy，如果需要支持预编译功能，交互流程肯定是需要变的，因为 Proxy 在收到 JDBC 的PreparedStatement 命令时，SQL 里的分片键是问号，它根本不知道该路由到哪个真实数据库。

因此，流程变成了下面这样：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9332a800a05549399ff73b3ea5c0defc~tplv-k3u1fbpfcp-zoom-1.image)

可以看到，Proxy在收到 PreparedStatement 命令后，并不会把这条消息转发给MySQL，只是缓存了这个 SQL，在收到 ExecuteStatement 命令后，才根据分片键和传过来的参数值确定真实的数据库，并与 MySQL 交互。

# 03 问题分析 

上一章节基本把这个 Bug 相关的原理知识介绍清楚了，下面正式进入问题的分析过程。

最开始拿到这个问题，我也是比较头秃的，尤其看到读者下面这段信息。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f1d1800287d48fdbcbc7aeebc9d8f75~tplv-k3u1fbpfcp-zoom-1.image)

当然，我的功力是达不到盲猜水平的，说下我的完整思路。

## 第 1 步  复现问题

我让读者给我打包发了 Demo 的源代码、数据库脚本以及 ShardingProxy 配置，然后本地安装了 ShardingProxy 4.1.1 版本，再通过 Navicat 连接到 ShardingProxy 执行数据库脚本，环境基本就准备完毕了。

启动 Demo 程序后，通过 Postman 发送请求，问题稳定复现了，确实查不出数据。

## 第 2 步 确认应用程序是否有BUG

因为整个代码很简单，代码层面唯一有可能存在问题的是 Mybatis 这一层。为了确认这一点，我修改了 SpringBoot 的配置，将 MyBatis 的 debug 日志也打印了出来。再次发起请求后，能从控制台中看到以下详细日志：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9dcba179acef45819617e2dc98884e22~tplv-k3u1fbpfcp-zoom-1.image)

日志中没发现异常，而且 PreparedStatement 以及 ExecuteStatement 的参数设置都是正确的，查询结果确实是空。

为了缩小排查范围，我把 dataSource 的 配置改回了直连真实数据库，这样能将 ShardingProxy 这个干扰因素排除在外。改完后的 url 如下：

> jdbc:mysql://127.0.0.1:3306/db1?useServerPrepStmts=true&cachePrepStmts=true&serverTimezone=UTC

其中，db1 是真实数据库，3306 也是MySQL 服务器的端口了。然后再次用 Postman 发送请求，可以看到：有正确数据返回了。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/44fac4d3cb58490a84b06ba3b3acce35~tplv-k3u1fbpfcp-zoom-1.image)

通过这一步，我将怀疑对象再次转移到 ShardingProxy 上了，并将 dataSource 配置改回成原样，继续排查。

## 第 3 步 排查 ShardingProxy

首先，查看 ShardingProxy 的运行日志，没发现任何异常；其次，能看到日志中的 Actual SQL 是正确的，它已经根据分区键正确路由到了  pcsct\_prdt\_cvr0  这张表：

```
[INFO ] 17:25:48.804 [ShardingSphere-Command-15] 
ShardingSphere-SQL - Actual SQL: ds_0 ::: SELECT
BIZ_DT,ECIF_CUST_NO,DEP_FLG ...
FROM pscst_prdt_cvr0
WHERE ECIF_CUST_NO = ? ::: [10000]
```

因此可以推断：ShardingProxy 的分库分表配置应该是没有问题的。

我开始怀疑：是否跟 ShardingProxy 所使用的数据库驱动有关？因为这个 Jar 包是应用方选择版本，手动放到 ShardingProxy 安装目录中的。因此，我将驱动版本从 5.1.47 版本改成了 8.0.13 （和 Demo 使用了相同的版本），但是问题仍然存在。

另外，还能想到的是：是否是 ShardingProxy 的这个最新版本引入了 Bug？然后，我又另外安装了它的上一个版本 4.1.0，重新测试了一遍，还是有问题。

这个时候，真感觉没有其他可疑点了，所有能想到的点都排查了一遍。我再次回到了 Demo 程序本身，它和 ShardingProxy 唯一的结合点就在 DataSource 的 url 上。

> jdbc:mysql://127.0.0.1:3307/sharding_db?useServerPrepStmts=true&cachePrepStmts=true&serverTimezone=UTC

库名和端口号配置无误，唯一可疑的是另外三个参数: useServerPrepStmts、cachePrepStmts 、serverTimezone。其中，前两个参数和预编译 SQL 有关，是一个组合。

因此，我将这两个参数从 url 中去掉，测试了一下。这个时候奇迹出现了，居然返回了正确数据。至此，基本定位到了问题，但根本原因是什么呢？究竟是不是 ShardingProxy 的 Bug ？

## 第 4 步 Wireshark 抓包分析 MySQL 协议

找到这个问题的解决方案后，我同步给了读者。与此同时，他也在 ShardingProxy  的 GitHub 上提交了 issue，反馈了这个最新进展。

由于工作原因，这个问题我就暂时放一边了，准备抽空再接着排查。

大概过了一周我想起了这个问题，然后打开 issue 想了解下调查进度，让我非常惊讶的是：官方开发者居然在复现此问题后，主观臆断地认为是应用程序的问题，然后莫名奇妙的把这个 issue 关闭了，他们的答复是这样的：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ebef01bc3ab4afa85140be4beade975~tplv-k3u1fbpfcp-zoom-1.image)

意思就是：我们针对预编译 SQL 功能做了大量的测试，这个是不可能存在问题的，建议你们更换下应用程序的数据库连接池，抓包继续分析下。

第二天，我开始用 Wireshark 抓包分析 MySQL 的协议，想弄清楚根本原因到底是什么？同时联系上了官方，让他们 reopen 了这个问题。

Wireshark 如何抓取 MySQL 协议的数据包，这里就不展开了，大家可以网上查下资料。注意将 Wireshark 的过滤条件设置成：

> mysql || tcp.port==3307

其中：mysql 表示 ShardingProxy 和 MySQL Server 之间的数据包，tcp.port==3307 表示 Demo 程序和 ShardingProxy 之间的数据包。

启动 Wireshark 抓包后，再次用 Postman 发起请求，触发整个过程，然后就能顺利抓到下面截图的数据包了。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b82025c488834efb9274df148a0d5bef~tplv-k3u1fbpfcp-zoom-1.image)![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/788d0495c3a1442f9cf20ed8ea989ec7~tplv-k3u1fbpfcp-zoom-1.image)

大家关注底色为 深蓝色 的 8 个数据包即可。在本文第 2 章节的原理部分，我已经详细介绍过 ShardingProxy 的预编译功能以及该流程的 MySQL 协议消息，这里的 8 个数据包和原理介绍是完成吻合的。

那接下来如何进一步分析呢？结合 ShardingProxy 的架构图来思考下：Proxy 仅仅作为一个中间代理，介于应用程序和 MySQL Server 之间，它完全实现了 MySQL 协议，以便对 MySQL 命令进行解码和编码，然后加上自己的分库分表逻辑。

如果 ShardingProxy 内部存在 Bug，那一定是某个数据包出现了问题。顺着这个思路，很快就能发现：执行完 ExecuteStatement 后，MySQL Server 返回正确数据包给 Proxy 了，但是 Proxy 没有返回正确的数据包给应用程序。

下面截图的是倒数第 2 个 Response 数据包，由 MySQL Server 返回给 Proxy 的，Payload 中能看到那条记录的数据：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d45ec790bee407a8e88719d6c568c85~tplv-k3u1fbpfcp-zoom-1.image)

下面截图的是最后 1 个 Response 数据包，由 Proxy 返回给应用程序的，Payload 中只能看到表字段的定义，那条记录已经不翼而飞了。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c68cd60229cc4f888c30f1a494bec6b0~tplv-k3u1fbpfcp-zoom-1.image)![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71682caa56b84d9c8af5a57374c58859~tplv-k3u1fbpfcp-zoom-1.image)

通过这一步分析，就已经坐实了：ShardingProxy 是有 Bug 的。然后，我将这些依据发给了官方开发者，对方开始重视，并正式进入源码分析阶段。

# 04 根本原因定位 

当天晚上，官方开发者就定位到了根本原因，发出了 Pull Request。我看了下代码改动，仅仅修改了一行代码。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/118a2da8797f4eefadf74c944c35b0d6~tplv-k3u1fbpfcp-zoom-1.image)

改动的这行代码，就是在 ShardingProxy 再次组装数据包返回给应用程序时抛出来的。

由于我们的数据表中存在一个 date 类型的字段，改动的这行代码却强制将 date 类型转换成了 Timestamp 类型，因此抛出了异常。还有几个疑点，我结合对源代码的理解逐一解答下。

**1、为什么代码抛异常了，但是 ShardingProxy 的控制台没打印呢？**
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb8e7c40d5bc481b9746678f6ddebbb5~tplv-k3u1fbpfcp-zoom-1.image)

上面截图的是：抛出 ClassCastException 那个方法的整个调用链。由于 ShardingProxy 并没有捕获这个 RuntimeException 以及打印日志，最终这个异常被 netty 吞掉了。

**2、为什么 ShardingProxy 需要做 date 到 Timestamp 的类型转换呢？**

可以从 ShardingProxy 的架构来理解，因为 Proxy 只有对 MySQL 协议进行编解码后，才能在中间插入它的分库分表逻辑。

针对 date 类型的字段，ShardingProxy 通过 JDBC 的 API 从查询结果中拿到的仍然是 Date 类型，之所以要转换成 Timestamp，这个又跟 MySQL 的协议有关了，下面是 MySQL 官方文档的说明：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4283dc5ce36247339c1c99df55d06d09~tplv-k3u1fbpfcp-zoom-1.image)![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b0e5a115dc4642e1afe8fa94aff208d5~tplv-k3u1fbpfcp-zoom-1.image)

简单理解就是：ShardingProxy 在代码实现时，用了一个范围最大的 timestamp 存了三种可能的值 date, datetime 和 timestamp，然后再按照上面这个协议规范进行二进制的写入。

**3、这个 Bug 是只有在使用 SQL 预编译功能时才会被触发吗？**

是的，只有在处理 ExecuteStatement 命令时，这个方法才会被调用到。那普通的 SQL 查询场景为什么用不到呢？

这个又跟 MySQL 协议有关了，普通的 SQL 查询场景，payload 不是二进制协议的，而是普通的文本协议。这种情况下，无需调用这个类进行转换。

至此，整个分析过程就结束了。

# 05 写在最后 

本文详细复盘了这个 Bug 的分析过程，并对其中的原理知识和排查经验进行了总结。

对于 ShardingSphere 这种顶级开源项目来说，我个人觉得同样值得做一次深度复盘。我不认同他们对于 issue 的处理方式，另外在核心功能的自动化测试上，也一定是存在 case 不完善的，不然不可能连续多个版本都没发现这个严重 Bug。

如果你有任何疑问，欢迎评论区留言讨论。

作者简介：985硕士，前亚马逊工程师，现58转转技术总监

**欢迎扫描下方的二维码，关注我的个人公众号：IT人的职场进阶**

![](https://img-blog.csdnimg.cn/20201107215432925.jpg)