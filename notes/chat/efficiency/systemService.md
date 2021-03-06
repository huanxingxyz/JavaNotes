首先要表明一个观点：**脱离业务实际情况的架构都是耍流氓，所以不是所有系统都必须服务化，也不要为了服务化而服务化。**
在了解服务化的好处之前，让我们先看看传统的系统架构是什么样的，当了解传统架构的缺点之后，再去看看为什么要做服务化，就容易理解了。

# 单体架构
在单体服务的时代，我们是一台应用服务器，后面挂一台数据库。
当访问量增多的时候，会引入负载均衡、数据库读写分离、分库分表等技术，系统的一个整体的架构大概是这个样子的：

![为什么越来越多的系统在做服务化？](https://github.com/CodeDaShu/JavaNotes/blob/master/img/framework/monomer_load.jpg)

这种架构，会有什么样的痛点呢？
系统在不断发展的过程中，可能会遇到下面几种情况：

*   **数据到处都有：** 如果系统彼此独立，那么相同或类似的数据会分散存储，举个最简单的例子，如果一个公司对外的系统很多，每个系统都提供用户注册的功能，注册后用户信息保存到自己的系统，当公司内这样的系统越来越多，问题就会凸显；
*   **系统体积庞大：** 如果功能都集中在一个系统中，那么这个系统将拥有太多的功能，就会造成项目代码过多，维护、迭代、发布也会变得困难；
*   **代码到处拷贝：** 如果数据库统一，用户信息都存储到一个数据库中，开放给各个业务系统操作（事实上几乎没有公司会这样做），这样带来的一个问题就是，相同逻辑的代码，会分布在多个系统中；更严重的是，代码与数据库的耦合度太高，不易于扩展。
*   **代码质量无法保障，系统/模块之间相互影响：** 假如A功能写了SQL导致全表扫描，数据库的CPU飙到100%或造成锁表，那么影响会影响到其他功能。

# 服务化架构

这时候会考虑在代码这个级别，对用户数据的操作，进行服务化；服务化后的架构大概是这个样子（这里先不讨论是直接调用，还是服务注册、发现）：

![为什么越来越多的系统在做服务化？](https://github.com/CodeDaShu/JavaNotes/blob/master/img/framework/service_oriented.jpg)

这个服务化的过程其实也非常简单，在例子中，说白了就是把用户相关的功能单独做一个系统，并且把对用户信息的操作通过接口的方式暴露出来，那么服务化有什么好处，到底解决了哪些问题呢？我总结有这么几点：

*   **相同的数据集中存储：** 例如用户数据只保存在用户中心；
*   **业务逻辑集中，可复用**：一个功能，只需要一处实现，其他系统只需要调用接口；如果是RPC的方式实现，就像调用本地的一个方法一样；具体业务逻辑是如何实现的，调用方不需要关心；
*   **屏蔽了底层复杂度：** 用不用缓存，数据库是否需要分库分表，对调用方来说，都是黑盒；

*   **增加数据价值：** 当数据被集中在了一起，才能做下一步的处理、分析、预测；才能发挥出数据的价值；

# 服务化的缺点

*   **高可能的要求更高，事故发生后的影响更大：** 如果用户中心挂了，那么会影响到所有依赖于用户中心的系统；
*   **架构更为复杂：** 服务和服务之间彼此依赖，如果服务不断增多并且没有得到很好的治理，最终可能导致整个公司没有一个人能够了解所有的服务；
*   **运维更加困难：** 当出现问题时，查找问题也更加困难。

**总之，系统要不要拆，要不要服务化，服务化到什么粒度，还需要从实际出发；脱离业务实际情况的架构都是耍流氓！**
