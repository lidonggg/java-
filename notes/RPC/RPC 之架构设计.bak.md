﻿[TOC]

# RPC 之架构设计

RPC 本质上就是一个远程调用，

## 服务发现

## 健康检测

有了集群之后，在每次发请求之前，RPC 框架都会根据负载均衡算法选择一个具体的服务提供者的实例以供调用。为了保证能够请求成功，我们需要确保每次选择的服务提供者（一个 IP 地址）是健康的。因为，我们需要一个完善的健康检测机制，能够及时发现服务集群中不健康的节点，并且能够及时告知服务的调用方。

### 基于心跳的健康检测

健康检测最常见的解决方案是基于心跳的健康检测。

## 路由策略

## 负载均衡

一般情况下 RPC 的负载均衡完全由 RPC 框架自身来实现，RPC 的服务调用者会与“注册中心”下发的所有服务节点建立长连接，在每次发起 RPC 调用的时候，服务调用者都会通过配置的负载均衡算法自主选择一个节点，发起 RPC 调用请求。

主流的负载均衡算法有随机权重法、轮询法、最小连接法以及（一致性）哈希算法等。

### （一致性）哈希算法

无论是随机权重还是轮询算法，对于一个服务调用方的多次请求，每次落到的提供方很大概率上是不同的。如果请求是无状态的，那么影响不大，但如果请求是有状态的，比如带有缓存服务等，如果请求不到一台服务器上，那么对于缓存的同步将是很大的一个挑战（当然通过分布式缓存，例如 Redis 等，可以很大程度上解决这一问题）。针对这个问题，哈希算法就可以派上用场了。

哈希算法的核心思想是设计一个哈希函数，然后请求过来之后，首先通过哈希函数计算的结果选择对应的调用节点，需要注意的是函数的参数对于特定的请求应该是一致的，比如唯一 ID 等，这样能够确保计算出来的结果是完全一致的。但是哈希算法适合处理节点数量相对比较固定的场景，因为如果一旦出现节点的增加或者删除，那么整个哈希函数可能都需要重写，因为要保证结果不会落到已被删除的节点上或者要能够落到新增的节点上，这个时候可能会产生大量的数据迁移。一致性哈希算法在这种场景之下孕育而生。

一致性哈希算法是指将服务器节点映射到一个首尾相连的哈希环上面，在经过哈希函数计算之后，会从哈希环上面进行顺时针查找，直到找到第一个对应的节点。这样当某个节点宕机或新增节点的时候，受影响的仅仅是此节点和此节点前一个节点的数据，这样数据迁移的成本会大大降低，并且节点数越多，迁移成本就会越低。所以一致性算法具有较好的容错性和可扩展性。

但是在一致性哈希中，经常会出现一种情况：调用方访问请求集中在少数的几个节点上，会出现有些服务提供方负载较高，而另外一些提供方负载较低的情况，因为我们很难设计出一个每个节点之间的间隔在哈希环上是基本一致的，就算设计出来了，如果新增或删除节点，这种基本一致也会被破坏。为了解决这个问题，一致性哈希算法中又引入了虚拟节点的概念。它的大致思想就是针对每个服务器节点计算多个哈希值，在每个计算结果的位置上，都放置一个虚拟节点，并且将虚拟节点映射到实际节点上，这样能够很大程度上使节点在哈希环上的分布相对一致。

## 异常重试

重试机制是提高接口调用成功率的一个重要手段。在请求发起之后，很有可能会因为网络抖动等原因导致请求失败，如果此时我们又正好希望请求能够成功，那么异常重试机制在这个时候就能发挥作用了。当然我们也可以自己手动写 try-cache，一旦捕捉到请求异常，则重新发起一次 RPC 调用，但是这么做显然不够优雅。如果第二次请求也失败了怎么办，难道要在 try-cache 中再嵌套一层 try-cache ？这么做显然会使代码变得异常臃肿，每个请求都得添加多层 try-cache，而且重试次数在编码的时候就已经确定了，很难根据业务需求去调整。因此我们希望能够通过 RPC 框架自身完成重试，并且对业务是透明的，这样对于调用方来说，自己只发起了一次 RPC 请求，与一般的编码方式没有任何区别。这就是 RPC 的异常重试机制。

上文在讲解负载均衡的时候我们说到，当调用方发起请求之后，会先经过负载均衡算法选取一个节点，然后给这个节点发送请求信息。当信息发送失败或者收到异常信息之后，我们就可以通过异常重试机制重新通过负载均衡算法选取一个节点发送请求。当重试次数超过用户配置的异常重试次数之后，就返回给调用端一个请求异常，否则就继续下去。

### 请求幂等

上面我们说过，如果网络抖动，一次请求没有收到成功的返回值的时候就会发起重试，这里就会存在一个问题，如果请求已经到达了服务提供方，然后在处理完成之后发生了网络抖动，这个时候调用方同样会认为请求异常了，从而会重新发起一次请求调用，这样的话就会导致一次请求被执行了多次。如果是往数据库插入数据，那么如果不做限制，就会插入两条数据，这显然是不能被接受的。那我们该怎么办呢？

这个时候就需要操作具有幂等性了。

幂等的意思是，一个请求无论被调用多少次，最终的结果都是一直的。比如对于上述的插入操作来说，正确的结果应该是两次请求执行完成之后只插入了一条数据。在数学中，可以表示为 f(x) = f(f(x))。

对于 crud 操作来说，select（查）操作是天然幂等的，对主键的删除或者条件删除也一般是幂等性的，会存在幂等问题的就是插入（增）数据和修改（改）数据了。接下来对于这两种操作，我们来看一下一些可行的解决方案。

1. 幂等地插入数据

- 添加唯一索引
- 分布式锁

2. 幂等地修改数据

- MVCC (多版本并发控制)，带条件更新，通过 version 或其他字段来做乐观锁
- 分布式锁

### 解决重试超时问题

考虑这样一个情景：我把调用端的请求超时时间设置为 5s，结果连续重试 3 次，每次都耗时 2s，那最终这个请求的耗时是 6s，那这样的话，即使最终请求成功了，但是由于超时时间设置的比较短，调用方依然是不会收到请求成功的返回值。那么这个时候该怎么办呢？

解决这个问题最直接的方式就是，在每次重试后都重置一下请求的超时时间。当调用端发起 RPC 请求时，如果发送请求发生异常并触发了异常重试，我们可以先判定下这个请求是否已经超时，如果已经超时了就直接返回超时异常，否则就先重置下这个请求的超时时间，之后再发起重试。

### 解决重复选取同一节点的问题

重试过程中，通过负载均衡算法，很有可能会选取到之前已经尝试过的节点，如果本身只是因为瞬间的网络原因而非服务器原因的请求异常，那么这么做是没问题。那么如果此服务器本身就存在问题，比如负载压力过大导致请求处理超时，再次选取它的话也会存在同样的问题，那这个时候该怎么办？

其实解决的方法很简单，我们只需要在下一次负载均衡的时候去掉已经选取过的节点，保证本次不会选到已选过的节点即可。

## 优雅关闭

在服务重启或者关机的过程中，服务的调用方可能会存在以下几种情况：

- 调用方发请求之前，目标服务已经下线了。对于调用方来说，跟目标节点的连接会断开，这个时候调用方可以立马感知到，并且在其健康列表里面把这个节点移除，从而不会被负载均衡算法选中。这是我们希望看到的一点。
- 调用方发请求的时候，目标服务正在关闭，但调用方并不知道它正在关闭，而且两者之间的连接也没有断开，这个节点还会在健康列表里面，因此该节点有一定的概率会被选中，并且有可能在请求处理还没有结束的时候服务就已经下线了，如果这个时候在执行事务操作等，那就很有可能会导致事务的不一致，这对于线上业务的影响是很大的。

服务下线的过程中，会有两次 RPC 调用，一是服务提供方通知注册中心下线操作，另外一个是注册中心通知调用方服务节点的下线，但是由于注册中心通知服务方是异步的，不能保持实时性，只能保证最终一致性，所以注册中心在收到服务提供方下线的时候，并不能成功保证把这次要下线的节点推送到所有的调用方。

因此为了避免上述第二种情况的出现，我们需要一个能够优雅地关闭服务的机制。

想要优雅地关闭服务，我们需要从两方面考虑，一方面是对于即将到来的请求要怎么办，另一方面是对于正在处理中的请求要怎么办。接下来我们就从这两方面来具体展开。

### 处理即将到来并且还没有处理的请求

因为服务方已经开始进入关闭流程了，那么很多对象可能已经被销毁了，关闭过程中再收到的请求肯定是没法按照正常的业务逻辑来进行处理的，因此我们需要在关闭的时候设置一个“挡板”，它的作用就是告诉调用方“我已经开始进入关闭流程了，你的请求我不能再处理了”。

基于这个思路，我们可以这么处理：当服务提供方正在关闭，如果这个时候还收到了新的请求，提供方直接返回一个特定的异常给调用方，这个异常告诉调用方“我收到了请求，但是我正在关闭，并没有处理它”，此时调用方收到这个异常响应之后，它会把请求重试到其他节点，并且 RPC 框架会直接把这个节点从健康列表中移除掉。

但是以上的方法只是一种被动等待的方式，也就是说，只有请求过来了才会通知相应的调用方。这会导致整个关闭的过程有些漫长，因为在当前时间点有的调用方可能会没有业务请求。因此除此之外，我们还可以加入一个主动通知的流程。

那么如何能够捕获到关闭事件呢？答案就是通过捕获操作系统的进程信号来获取。在 Java 语言中，对应的就是 Runtime.addShutdownHook 方法，可以注册关闭的钩子。在 RPC 启动的时候，我们提前注册关闭钩子，并在里面添加了两个处理程序，一个负责开启关闭标识，一个负责安全关闭服务对象，服务对象在关闭的时候会通知调用方下线节点。同时需要在我们调用链里面加上挡板处理器，当新的请求来的时候，会判断关闭标识，如果正在关闭，则抛出特定异常。

### 处理正在进行中的请求

对于正在处理中的请求，我们首先需要能够识别出这些请求，并且能够知道请求数量。对此我们可以在服务对象上面添加一个计数器，如果有请求过来了，计数器加一，请求处理完成之后，计数器减一，通过该计数器我们就可以知道是否还有正在处理中的请求了。

### 总结

在 RPC 里面，服务重启（关闭）看似是一个不起眼的小功能，但是如果处理的不好的话，极有可能会导致业务受损，通过优雅关闭流程，我们可以不用再担心因为重启而导致的问题，减少了许多运维成本。

## 优雅启动--启动预热

在 Java 里面，运行过程中，JVM 会把高频的代码编译成机器码，被加载过的类也会被缓存到 JVM 缓存中，再次使用的时候不会触发临时加载，这就导致在 Java 进程刚刚启动的时候，执行速度会比已经运行了一段时间的时候要慢。因此在服务刚启动的时候就承担着停机前一样的流量，会使它在启动之初就处于高负载的状态，从而导致调用方过来的请求可能出现大面的超时，进而使线上业务受损。

因此，对于服务的启动，我们也应该有一个机制，让它不会在启动之初承担大量流量，而是一开始只接受少许请求，然后逐渐提升到最佳状态，这就是 RPC 中的“启动预热”。

### 启动预热

在整个 RPC 框架中，服务调用是由服务调用方通过一定的负载均衡算法发起的，因此在进行负载均衡的时候，调用方需要能够区分刚刚启动的服务，从而降低选取到它的概率。那么要怎么区分一个服务是否为刚启动不久呢？这里主要有两种方案，都很简单。第一种方案，是服务提供方在启动的时候，把自己的启动时间告诉注册中心；另外一种方案就是注册中心在调用方注册的时候记录一下它的注册请求时间。通过这两种方案，最终的结果都是调用方通过服务发现的时候，不但能够找到所有的可用服务列表，还包括它们的启动时间，然后通过这个启动时间计算一个权值交给负载均衡算法去计算即可。

### 延迟暴露

在服务启动的过程中，都通过执行 main() 方法，顺序地把各种相关的依赖加载进来。这个时候，如果加载到了 RPC 服务相关的依赖，那么就会同时把它注册到注册中心。此时，就有可能会存在调用方已经能够从注册中心拿到该服务，但是此服务却还没有完全启动完成的情况，从而有可能会导致请求调用失败，业务受损。“延迟暴露”的概念就是为了解决该问题而出现的。

在应用启动加载、解析 Bean 的时候，如果遇到了 RPC 服务的 Bean，只先把这个 Bean 注册到 Spring-BeanFactory 里面去，而并不把这个 Bean 对应的接口注册到注册中心，只有等应用启动完成后，才把接口注册到注册中心用于服务发现，从而实现让服务调用方延迟获取到服务提供方地址。同时我们可以在服务提供方应用启动后，接口注册到注册中心前，预留一个 Hook 过程，让用户可以实现可扩展的 Hook 逻辑。用户可以在 Hook 里面模拟调用逻辑，从而使 JVM 指令能够预热起来，并且用户也可以在 Hook 里面事先预加载一些资源，只有等所有的资源都加载完成后，最后才把接口注册到注册中心，这样的话对于调用方来说，服务列表中的每一个服务都是一致的状态，从而可以不考虑服务的启动时间，简化负载均衡逻辑。

### 总结

启动预热与延迟暴露并不是 RPC 的专属功能，我们在开发其它系统时，也可以利用这两点来减少冷启动对业务的影响。

## 熔断限流

在高并发的场景下，我们的服务很有可能由于访问量过大而产生一系列问题，例如业务处理耗时过长、CPU 飘高、频繁的 full GC 以及更严重的比如服务进程直接宕机等。因此，如果想要确保服务的高可用，我们的整个 RPC 框架就需要有一定的自我保护能力。

RPC 框架包含服务提供方和服务调用方，因此我们可用从提供方和调用方两方面来实现自我保护。自我保护的手段主要有熔断和限流两种方式，其中调用方一般会采取熔断机制，而提供方一般会采用限流机制。

### 服务提供方的自我保护--限流

服务端的自我保护主要体现在如何解决负载压力过高的问题，因为提供方往往会进行频繁的 IO 访问以及计算等，性能在负载压力过高的时候往往会成为瓶颈。想要解决负载压力过高的问题，无非就是让它不要接收或者处理过多的请求就好了，这就是所谓的“限流”。

限流的方法很多，一方面是集成到 RPC 框架中，让开发者自己去配置限流阈值，由框架来决定是否应该执行限流逻辑。另一方面我们可用在服务提供方去手动添加限流逻辑，让服务端在进行请求处理之前，先执行限流逻辑，如果发现访问量过大，已经超过阈值了，那么就直接抛出一个限流异常。

服务端的限流方式有很多，这里我们介绍两种最常见的限流方式，一个是漏桶策略，典型的实现有阿里开源的流量控制框架 Sentinel 中的匀速排队限流策略；另外一个是令牌桶策略，典型的实现有 Google 开源工具包 Guava 提供的限流工具类 RateLimiter。

#### 漏桶策略

漏桶策略的思想就是无论用户请求有多少，无论请求速率有多大，“漏桶”都会接收下来，但从漏桶里出来的请求是固定速率的，保证服务器可以处理得游刃有余。当“漏桶”因为容量限制放不下更多的请求时，就会选择丢弃部分请求。这种思路其实就是一种“宽进严出”的策略。

这种策略的好处是，做到了流量整形，即无论流量多大，即便是突发的大流量，输出依旧是一个稳定的流量。但其缺点是，对于突发流量的情况，因为服务器处理速度与正常流量的处理速度一致，会丢弃比较多的请求。但是，当突发大流量到来时，服务器最好能够更快地处理用户请求，这也是分布式系统大多数情况下想要达到的效果。

所以说，漏桶策略适用于间隔性突发流量且流量不用即时处理的场景，即可以在流量较小时的“空闲期”，处理大流量时流入漏桶的流量；不适合流量需要即时处理的场景，即突发流量时可以放入桶中，但缺乏效率，始终以固定速率进行处理。

#### 令牌桶策略

令牌桶策略的思想是有一个固定容量的存放令牌的桶，我们以固定速率向桶里放入令牌，桶满时会丢弃多出的令牌。每当请求到来时，必须先到桶里取一个令牌才可被服务器处理，也就是说只有拿到了令牌的请求才会被服务器处理。所以，你可以将令牌理解为门卡，只有拿到了门卡才能顺利进入房间。这种方法有点类似于添加一个请求计数器，每次请求来了之后，计数器的值就加一，当到达预设的阈值之后，就执行限流逻辑。

这种策略的好处：当有突发大流量时，只要令牌桶里有足够多的令牌，请求就会被迅速执行。通常情况下，令牌桶容量的设置，可以接近服务器处理的极限，这样就可以有效利用服务器的资源。因此，这种策略适用于有突发特性的流量，且流量需要即时处理的场景。

#### 两种策略的对比图

下面贴出以上两种限流策略的对比图（来自“分布式技术原理与算法解析”中的分布式高可用之流量控制篇）：

<div align=center><img src="https://github.com/lidonggg/Learning-notes/blob/master/notes/RPC/images/%E6%BC%8F%E6%A1%B6%E5%92%8C%E4%BB%A4%E7%89%8C%E6%A1%B6%E9%99%90%E6%B5%81%E7%AD%96%E7%95%A5%E5%AF%B9%E6%AF%94.jpg"/></div>
<br>
<div align=center>漏桶策略和令牌桶策略的对比</div>
<br>

#### 动态限流

RPC 框架真正强大的地方在于它的治理功能，而治理功能大多都需要依赖一个注册中心或者配置中心，我们可以通过 RPC 治理的管理端进行配置，再通过注册中心或者配置中心将限流阈值的配置下发到服务提供方的每个节点上，实现动态配置。

我们可以提供一个专门的限流服务，让每个节点都依赖一个限流服务，当请求流量打过来时，服务节点触发限流逻辑，调用这个限流服务来判断是否到达了限流阈值。我们甚至可以将限流逻辑放在调用端，调用端在发出请求时先触发限流逻辑，调用限流服务，如果请求量已经到达了限流阈值，请求都不需要发出去，直接返回给动态代理一个限流异常即可。

这种限流方式可以让整个服务集群的限流变得更加精确，但也由于依赖了一个限流服务，它在性能和耗时上与单机的限流方式相比是有很大劣势的。

### 服务调用方的自我保护--熔断

调用方的自我保护主要体现在当某个服务不可用的时候，不会引起整个调用链路出现问题，甚至是下游所有服务的宕机。熔断就是为了解决这个问题而存在的。

熔断器的工作机制主要是关闭、打开和半打开这三个状态之间的切换。在正常情况下，熔断器是关闭的；当调用端调用下游服务出现异常时，熔断器会收集异常指标信息进行计算，当达到熔断条件时熔断器打开，这时调用端再发起请求是会直接被熔断器拦截，并快速地执行失败逻辑；当熔断器打开一段时间后，会转为半打开状态，这时熔断器允许调用端发送一个请求给服务端，如果这次请求能够正常地得到服务端的响应，则将状态置为关闭状态，否则设置为打开。

#### 熔断器整合

我们可用在动态代理中去使用熔断器，因为在 RPC 调用的流程中，动态代理是 RPC 调用的第一个关口。在发出请求时先经过熔断器，如果状态是闭合则正常发出请求，如果状态是打开则执行熔断器的失败策略。

### 总结

服务端主要是通过限流来进行自我保护，我们在实现限流时要考虑到应用和 IP 级别，方便我们在服务治理的时候，对部分访问量特别大的应用进行合理的限流；服务端的限流阈值配置都是作用于单机的，而在有些场景下，例如对整个服务设置限流阈值，服务进行扩容时，限流的配置并不方便，我们可以在注册中心或配置中心下发限流阈值配置的时候，将总服务节点数也下发给服务节点，让 RPC 框架自己去计算限流阈值；我们还可以让 RPC 框架的限流模块依赖一个专门的限流服务，对服务设置限流阈值进行精准地控制，但是这种方式依赖了限流服务，相比单机的限流方式，在性能和耗时上有劣势。

## 业务分组

业务分组的目的是隔离流量，也就是说将不同业务的不同接口或者同一个应用的不同类型的接口设计成不同的服务提供方，分别供不同的调用方来使用，从而达到流量隔离的作用。

分组的方法很简单，无非就是把不同的服务划分成不同的组，分别注册到注册中心，调用分别根据自己的需要，去拉去对应的某一（多）个提供方列表。对于整个架构而言，无非就是新增了一个分组参数。问题的关键是如何确定分组的标准。

一个比较好的分组的标准就是：非核心应用不要和核心应用放进同一个分组里面，核心应用之间做好隔离，尽量保证核心应用的高可用。一般情况下，核心应用对于高可用的要求比较高，而非核心应用则次之，如果将非核心和核心应用放在一起，则非核心的接口出现问题，则有可能会同时影响到核心业务，这在任何时候都是不应该存在的问题。

通过业务分组，我们不但可以把一个庞大的服务提供方划分成不同的小规模集群，同时在一些特殊的情形之下，也可以针对不同的接口提供不同的实现。





