# YGC问题排查，又让我涨姿势了

<br/>

在高并发下，Java程序的GC问题属于很典型的一类问题，带来的影响往往会被进一步放大。不管是「GC频率过快」还是「GC耗时太长」，由于GC期间都存在Stop The World问题，因此很容易导致服务超时，引发性能问题。

我们团队负责的广告系统承接了比较大的C端流量，平峰期间的请求量基本达到了上千QPS，过去也遇到了很多次GC相关的线上问题。

5月份的这篇文章我介绍了[一个Full GC过于频繁的案例](http://mp.weixin.qq.com/s?__biz=MzU2MTM4NDAwMw==&mid=2247483975&idx=1&sn=fe708c07e0d45ac1fe75ab2258267c51&chksm=fc78dd6bcb0f547d8db8b85d61e57804bfb02573a41d501d960b7bc5a165a9a0aab4bcb0dd4b&scene=21#wechat_redirect)，并且针对JVM的堆内存结构和GC原理进行了系统性的总结。

这篇文章，我再分享一个更棘手的Young GC耗时过长的线上案例，同时会整理下YGC相关的知识点，希望让你有所收获。内容分成以下2个部分：

-   从一次YGC耗时过长的案例说起
-   YGC的相关知识点总结
    
<br/>
  

# 01 从一次YGC耗时过长的案例说起

今年4月份，我们的广告服务在新版本上线后，收到了大量的服务超时告警，通过下面的监控图可以看到：超时量突然大面积增加，1分钟内甚至达到了上千次接口超时。下面详细介绍下该问题的排查过程。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25a81ba4e0fe484b956dcbc1f2563f31~tplv-k3u1fbpfcp-zoom-1.image)

<br/>

## 1.1 检查监控

收到告警后，我们第一时间查看了监控系统，立马发现了YoungGC耗时过长的异常。 我们的 程序大概在 21 点50 左右上线，通过下图可以看出： 在上线之前，YGC基本几十毫秒内完成，而上线后YGC耗时明显变长，最长甚至达到了3秒多。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9bd49e08c52f4565b8795e4126c89667~tplv-k3u1fbpfcp-zoom-1.image)

由于 YGC期间程序会 Stop The World ，而我们上游系统设置的服务超时时间都在几百毫秒，因此推断：是因为YGC耗时过长引发了服务大面积超时。

按照GC问题的常规排查流程，我们立刻摘掉了一个节点，然后通过以下命令dump了堆内存文件用来保留现场。  

jmap  -dump:format=b,file=heap pid

最后对线上服务做了回滚处理，回滚后服务立马恢复了正常，接下来就是长达1天的问题排查和修复过程。

<br/>

## 1.2 确认JVM配置

用下面的命令，我们再次检查了JVM的参数

ps aux | grep "applicationName=adsearch"

> -Xms4g -Xmx4g -Xmn2g -Xss1024K 
> 
> -XX:ParallelGCThreads=5 
> 
> -XX:+UseConcMarkSweepGC 
> 
> -XX:+UseParNewGC 
> 
> -XX:+UseCMSCompactAtFullCollection 
> 
> -XX:CMSInitiatingOccupancyFraction=80

可以看到堆内存为4G，新生代和老年代均为2G，新生代采用ParNew收集器。

再通过命令 jmap  -heap pid  查到：新生代的Eden区为1.6G，S0和S1区均为0.2G。

本次上线并未修改JVM相关的任何参数，同时我们服务的请求量基本和往常持平。因此猜测：此问题大概率和上线的代码相关。

<br/>

## 1.3 检查代码

再回到YGC的原理来思考这个问题，一次YGC的过程主要包括以下两个步骤：

> 1、从GC Root扫描对象，对存活对象进行标注
> 
> 2、将存活对象复制到S1区或者晋升到Old区

根据下面的监控图可以看出：正常情况下，Survivor区的使用率一直维持在很低的水平（大概30M左右），但是上线后，Survivor区的使用率开始波动，最多的时候快占满0.2G了。而且，YGC耗时和Survivor区的使用率基本成正相关。因此，我们推测：应该是长生命周期的对象越来越多，导致标注和复制过程的耗时增加。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e7c2f15489148409e407d57dc43c965~tplv-k3u1fbpfcp-zoom-1.image)

再回到服务的整体表现：上游流量并没有出现明显变化，正常情况下，核心接口的响应时间也基本在200ms以内，YGC的频率大概每8秒进行1次。

很显然，对于局部变量来说，在每次YGC后就能够马上被回收了。那为什么还会有如此多的对象在YGC后存活下来呢？

我们进一步将怀疑对象锁定在：程序的全局变量或者类静态变量上。但是diff了本次上线的代码，我们并未发现代码中有引入此类变量。

<br/>

## 1.4 对dump的堆内存文件进行分析

代码排查没有进展后，我们开始从堆内存文件中寻找线索，使用MAT工具导入了第1步dump出来的堆文件后，然后通过Dominator Tree视图查看到了当前堆中的所有大对象。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f2051748e7749469bb7d6b918b730ce~tplv-k3u1fbpfcp-zoom-1.image)

立马发现NewOldMappingService这个类所占的空间很大，通过代码定位到：这个类位于第三方的client包中，由我们公司的商品团队提供，用于实现新旧类目转换（最近商品团队在对类目体系进行改造，为了兼容旧业务，需要进行新旧类目映射）。

进一步查看代码，发现这个类中存在大量的静态HashMap，用于缓存新旧类目转换时需要用到的各种数据，以减少RPC调用，提高转换性能。  

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0d76add33568483cad8cd60cd78a95f5~tplv-k3u1fbpfcp-zoom-1.image)

原本以为，非常接近问题的真相了，但是深入排查发现：这个类的所有静态变量全部在类加载时就初始化完数据了，虽然会占到100多M的内存，但是之后基本不会再新增数据。并且，这个类早在3月份就上线使用了，client包的版本也一直没变过。  

经过上面种种分析，这个类的静态HashMap会一直存活，经过多轮YGC后，最终晋升到老年代中，它不应该是YGC持续耗时过长的原因。因此，我们暂时排除了这个可疑点。

<br/>

## 1.5 分析YGC处理Reference的耗时

团队对于YGC问题的排查经验很少，不知道再往下该如何分析了。基本扫光了网上可查到的所有案例，发现原因集中在这两类上：

1、对存活对象标注时间过长：比如重载了Object类的Finalize方法，导致标注Final Reference耗时过长；或者String.intern方法使用不当，导致YGC扫描StringTable时间过长。

2、长周期对象积累过多：比如本地缓存使用不当，积累了太多存活对象；或者锁竞争严重导致线程阻塞，局部变量的生命周期变长。  

针对第1类问题，可以通过以下参数显示GC处理Reference的耗时-XX:+PrintReferenceGC。 添加此参数后，可以看到不同类型的  reference 处理耗时都很短，因此又排除了此项因素。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c228b4e4feb4a88b52ccf36653c2651~tplv-k3u1fbpfcp-zoom-1.image)

<br/>

## 1.6 再回到长周期对象进行分析

再往后，我们添加了各种GC参数试图寻找线索都没有结果，似乎要黔驴技穷，没有思路了。综合监控和种种分析来看：应该只有长周期对象才会引发我们这个问题。

折腾了好几个小时，最终峰回路转，一个小伙伴重新从MAT堆内存中找到了第二个怀疑点。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ff5ec8b637b40388195899d25f024ce~tplv-k3u1fbpfcp-zoom-1.image)

从上面的截图可以看到：大对象中排在第3位的ConfigService类进入了我们的视野，该类的一个ArrayList变量中竟然包含了270W个对象，而且大部分都是相同的元素。  

ConfigService这个类在第三方Apollo的包中，不过源代码被公司架构部进行了二次改造，通过代码可以看出：**问题出在了第11行，每次调用getConfig方法时都会往List中添加元素，并且未做去重处理。**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/964dc23cdd084be3bab42c3e88be3562~tplv-k3u1fbpfcp-zoom-1.image)

我们的广告服务在apollo中存储了大量的广告策略配置，而且大部分请求都会调用ConfigService的getConfig方法来获取配置，因此会不断地往静态变量namespaces中添加新对象，从而引发此问题。

至此，整个问题终于水落石出了。这个BUG是因为架构部在对apollo client包进行定制化开发时不小心引入的，很显然没有经过仔细测试，并且刚好在我们上线前一天发布到了中央仓库中，而公司基础组件库的版本是通过super-pom方式统一维护的，业务无感知。

<br/>

## 1.7 解决方案

为了快速验证YGC耗时过长是因为此问题导致的，我们在一台服务器上直接用旧版本的apollo client 包进行了替换，然后重启了服务，观察了将近20分钟，YGC恢复正常。

最后，我们 通知架构部修复BUG，重新发布了super-pom ，彻底解决了这个问题。

<br/>

# 02 YGC的相关知识点总结

通过上面这个案例，可以看到YGC问题其实比较难排查。相比FGC或者OOM，YGC的日志很简单，只知道新生代内存的变化和耗时，同时dump出来的堆内存必须要仔细排查才行。

另外，如果不清楚YGC的流程，排查起来会更加困难。这里，我对YGC相关的知识点再做下梳理，方便大家更全面的理解YGC。

<br/>

## 2.1 5个问题重新认识新生代

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a875789432640ac9f3ec4a6357a6656~tplv-k3u1fbpfcp-zoom-1.image)

YGC 在新生代中进行，首先要清楚新生代的堆结构划分。新生代分为Eden区和两个Survivor区，其中Eden:from:to = 8:1:1 (比例可以通过参数 –XX:SurvivorRatio 来设定 )，这是最基本的认识。  

**为什么会有新生代？**

如果不分代，所有对象全部在一个区域，每次GC都需要对全堆进行扫描，存在效率问题。分代后，可分别控制回收频率，并采用不同的回收算法，确保GC性能全局最优。

**为什么新生代会采用复制算法？**

新生代的对象朝生夕死，大约90%的新建对象可以被很快回收，复制算法成本低，同时还能保证空间没有碎片。虽然标记整理算法也可以保证没有碎片，但是由于新生代要清理的对象数量很大，将存活的对象整理到待清理对象之前，需要大量的移动操作，时间复杂度比复制算法高。

**为什么新生代需要两个Survivor区？**

为了节省空间考虑，如果采用传统的复制算法，只有一个Survivor区，则Survivor区大小需要等于Eden区大小，此时空间消耗是8 * 2，而两块Survivor可以保持新对象始终在Eden区创建，存活对象在Survivor之间转移即可，空间消耗是8+1+1，明显后者的空间利用率更高。

**新生代的实际可用空间是多少？**

YGC后，总有一块Survivor区是空闲的，因此新生代的可用内存空间是90%。在YGC的log中或者通过 jmap -heap pid 命令查看新生代的空间时，如果发现capacity只有90%，不要觉得奇怪。

**Eden区是如何加速内存分配的？**

HotSpot虚拟机使用了两种技术来加快内存分配。分别是bump-the-pointer和TLAB（Thread Local Allocation Buffers）。

由于Eden区是连续的，因此bump-the-pointer在对象创建时，只需要检查最后一个对象后面是否有足够的内存即可，从而加快内存分配速度。

TLAB技术是对于多线程而言的，在Eden中为每个线程分配一块区域，减少内存分配时的锁冲突，加快内存分配速度，提升吞吐量。  

<br/>

## 2.2 新生代的4种回收器

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d7992adddd3448a78a116d404a4916d0~tplv-k3u1fbpfcp-zoom-1.image)

SerialGC（串行回收器），最古老的一种，单线程执行，适合单CPU场景。

ParNew（并行回收器），将串行回收器多线程化，适合多CPU场景，需要搭配老年代CMS回收器一起使用。  

ParallelGC（并行回收器），和ParNew不同点在于它关注吞吐量，可设置期望的停顿时间，它在工作时会自动调整堆大小和其他参数。  

G1（Garage-First回收器），JDK 9及以后版本的默认回收器，兼顾新生代和老年代，将堆拆成一系列Region，不要求内存块连续，新生代仍然是并行收集。

上述回收器均采用复制算法，都是独占式的，执行期间都会Stop The World.

<br/>

## 2.3 YGC的触发时机

当Eden区空间不足时，就会触发YGC。结合新生代对象的内存分配看下详细过程：

1、新对象会先尝试在栈上分配，如果不行则尝试在TLAB分配，否则再看是否满足大对象条件要在老年代分配，最后才考虑在Eden区申请空间。

2、如果Eden区没有合适的空间，则触发YGC。  

3、YGC时，对Eden区和From Survivor区的存活对象进行处理，如果满足动态年龄判断的条件或者To Survivor区空间不够则直接进入老年代，如果老年代空间也不够了，则会发生promotion failed，触发老年代的回收。否则将存活对象复制到To Survivor区。

4、此时Eden区和From Survivor区的剩余对象均为垃圾对象，可直接抹掉回收。

此外，老年代如果采用的是CMS回收器，为了减少CMS Remark阶段的耗时，也有可能会触发一次YGC，这里不作展开。

<br/>

## 2.4 YGC的执行过程

YGC采用的复制算法，主要分成以下两个步骤：

> 1、查找GC Roots，将其引用的对象拷贝到S1区
> 
> 2、递归遍历第1步的对象，拷贝其引用的对象到S1区或者晋升到Old区

上述整个过程都是需要暂停业务线程的（STW），不过ParNew等新生代回收器可以多线程并行执行，提高处理效率。

YGC通过可达性分析算法，从GC Root（可达对象的起点）开始向下搜索，标记出当前存活的对象，那么剩下未被标记的对象就是需要回收的对象。  

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc6ea5bcdc334bf8a93395f03debe299~tplv-k3u1fbpfcp-zoom-1.image)

可作为YGC时GC Root的对象包括以下几种：

> 1、虚拟机栈中引用的对象
> 
> 2、方法区中静态属性、常量引用的对象
> 
> 3、本地方法栈中引用的对象
> 
> 4、被Synchronized锁持有的对象
> 
> 5、记录当前被加载类的SystemDictionary
> 
> 6、记录字符串常量引用的StringTable
> 
> 7、存在跨代引用的对象
> 
> 8、和GC Root处于同一CardTable的对象

其中1-3是大家容易想到的，而4-8很容易被忽视，却极有可能是分析YGC问题时的线索入口。

另外需要注意的是，针对下图中跨代引用的情况，老年代的对象A也必须作为GC Root的一部分，但是如果每次YGC时都去扫描老年代，肯定存在效率问题。在HotSpot JVM，引入卡表（Card Table）来对跨代引用的标记进行加速。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c18aee1116c4fee93743bf1b37f354d~tplv-k3u1fbpfcp-zoom-1.image)

Card Table，简单理解是一种空间换时间的思路，因为存在跨代引用的对象大概占比不到1%，因此可将堆空间划分成大小为512字节的卡页，如果卡页中有一个对象存在跨代引用，则可以用1个字节来标识该卡页是dirty状态，卡页状态进一步通过写屏障技术进行维护。  

遍历完GC Roots后，便能够找出第一批存活的对象，然后将其拷贝到S1区。接下来，就是一个递归查找和拷贝存活对象的过程。

S1区为了方便维护内存区域，引入了两个指针变量： \_saved\_mark\_word和\_top，其中\_saved\_mark\_word表示当前遍历对象的位置，\_top表示当前可分配内存的位置，很显然，\_saved\_mark\_word到\_top之间的对象都是已拷贝但未扫描的对象。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a986eeac13e479a9fac0f95e3125120~tplv-k3u1fbpfcp-zoom-1.image)

如上图所示，每次扫描完一个对象，\_saved\_mark\_word会往前移动，期间如果有新对象也会拷贝到S1区，\_top也会往前移动，直到\_saved\_mark\_word追上\_top，说明S1区所有对象都已经遍历完成。

有一个细节点需要注意的是：拷贝对象的目标空间不一定是S1区，也可能是老年代。如果一个对象的年龄（经历的YGC次数）满足动态年龄判定条件便直接晋升到老年代中。对象的年龄保存在Java对象头的mark word数据结构中（如果大家对Java并发锁熟悉，肯定了解这个数据结构，不熟悉的建议查阅资料了解下，这里不做展开）。  
  
<br/>

# 写在最后

这篇文章通过线上案例分析并结合原理讲解，详细介绍了YGC的相关知识。从YGC实战角度出发，再 简单总结一下：

1、首先要清楚YGC的执行原理，比如年轻代的堆内存结构、Eden区的内存分配机制、GC Roots扫描、对象拷贝过程等。

2、YGC的核心步骤是标注和复制，绝部分YGC问题都集中在这两步，因此可以结合YGC日志和堆内存变化情况逐一排查，同时d ump的堆内存文件 需要仔细分析。


<br/>


---

作者简介：985硕士，前亚马逊工程师，现58转转技术总监

**欢迎扫描下方的二维码，关注我的个人公众号：武哥漫谈IT，精彩原创不断！**

![](https://img-blog.csdnimg.cn/20201107215432925.jpg)
