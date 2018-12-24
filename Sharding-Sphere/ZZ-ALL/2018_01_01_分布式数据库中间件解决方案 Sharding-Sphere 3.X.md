title: 分布式数据库中间件解决方案 Sharding-Sphere 3.X
date: 2018-01-01
tags:
categories: Sharding Sphere
permalink: Sharding-Sphere/Distributed-database-middleware-solution-Sharding-Sphere-3.x
author: ShardingSphere
from_url: https://blog.csdn.net/ok449a6x1i6qq0g660fV/article/details/80491123

---

摘要: 原创出处 https://blog.csdn.net/ok449a6x1i6qq0g660fV/article/details/80491123 「ShardingSphere」欢迎转载，保留摘要，谢谢！

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

码农少闲月，五月人倍忙！Sharding-Sphere在经历公投改名、新官网上线、版权转移等一系列重大变革后，终于迎来了它的3.X新时代！从Sharding-JDBC到Sharding-Sphere，老铁粉陪它一同走过，新朋友也在陆续加入。Sharding-Sphere是什么？做什么？做的如何？三大经典提问帮助新老朋友一同温故知新。

**Sharding-Sphere是什么?**

Sharding-Sphere是一套开源的分布式数据库中间件解决方案组成的生态圈，它由Sharding-JDBC、Sharding-Proxy和Sharding-Sidecar这3款相互独立的产品组成。他们均提供标准化的数据分片、读写分离、柔性事务和数据治理功能，可适用于如Java同构、异构语言、容器、云原生等各种多样化的应用场景。

Sharding-Sphere定位为关系型数据库中间件，旨在充分合理地在分布式的场景下利用关系型数据库的计算和存储能力，而并非实现一个全新的关系型数据库。它与NoSQL和NewSQL是并存而非互斥的关系。NoSQL和NewSQL作为新技术探索的前沿，是非常值得推荐的。而Sharding-Sphere关注未来不变的东西，进而抓住事物本质。关系型数据库当今依然占有巨大市场，是各个公司核心业务的基石，我们目前阶段更加关注在原有基础上的增量，而非颠覆。其架构如下图所示：

![img](http://static.iocoder.cn/5abc5b9da4893739693e8cdca93b6e81)

**Sharding-Sphere家族都有谁?**

Sharding-JDBC, Sharding-Proxy以及Sharding-Sidecar 共同组成了Sharding-Sphere。他们分别定位、适用于不同的应用场景。您也可以将他们组合使用以得到增益的性能表现。

**1. Sharding-JDBC**

Sharding-JDBC是Sharding-Sphere的第一个产品，也是Sharding-Sphere的前身。 它定位为轻量级Java框架，在Java的JDBC层提供分库分表、读写分离、数据库治理、柔性事务等服务。它使用客户端直连数据库，以jar包形式提供服务，无需额外部署和依赖，可理解为增强版的JDBC驱动，完全兼容JDBC和各种ORM框架。

**2. Sharding-Proxy**

Sharding-Proxy是Sharding-Sphere的第二个产品。 它定位为透明化的数据库代理端，提供封装了数据库二进制协议的服务端版本，用于完成对异构语言的支持。 Sharding-Proxy屏蔽了底层的分库分表，您可以像使用一个简单的数据库一样来操作分库分表的数据。目前提供MySQL版本，它可以使用任何兼容MySQL协议的访问客户端(如：MySQL Command Client, MySQL Workbench等)来访问Sharding-Proxy，进而进行DDL/DML等操作来变更数据，对DBA更加友好。

**3. Mixed scheme of Sharding-JDBC & Sharding-Proxy**

为了得到更好的性能以及友好的交互体验，您可以同时使用Sharding-JDBC和Sharding-Proxy。

线上应用使用Sharding-JDBC直连数据库以获取最优性能，使用MySQL命令行或UI客户端连接Sharding-Proxy方便的查询数据和执行各种DDL语句。它们使用同一个注册中心集群，通过管理端配置注册中心中的数据，即可由注册中心自动将配置变更推送至JDBC和Proxy应用。若数据库拆分的过多而导致连接数会暴涨，则可以考虑直接在线上使用Sharding-Proxy，以达到有效控制连接数的目的。其架构如下如所示：

![img](http://static.iocoder.cn/6402bfbd3b9de1603b74dbdb964c90bd)

**4. Sharding-Sidecar**

Sharding-Sidecar是Sharding-Sphere的第三个产品，目前仍处在孵化中。 定位为Kubernetes或Mesos的云原生数据库代理。其核心思想是Database Mesh，即通过无中心、零侵入的方案提供与数据库交互的啮合层。关注重点在于如何将分布式的数据访问应用与数据库有机串联起来。

 **Sharding-Sphere的功能特性**

**1. 分库分表**

为解决关系型数据库面对海量数据由于数据量过大而导致的性能问题，将数据进行分片是行之有效的解决方案，而将集中于单一节点的数据拆分并分别存储到多个数据库或表，称为分库分表。作为分布式数据库中间件，我们的目标是透明化分库分表所带来的影响，让使用方尽量像使用一个数据库一样使用水平拆分之后的数据库。

**2. 读写分离**

面对日益增加的系统访问量，数据库的吞吐量面临着巨大瓶颈。 对于同一时间有大量并发读操作和较少写操作类型的应用系统来说，将单一的数据库拆分为主库和从库，主库负责处理事务性的增删改操作，从库负责处理查询操作，能够有效的避免由数据更新导致的行锁，使得整个系统的查询性能得到极大的改善。透明化读写分离所带来的影响，让使用方尽量像使用一个数据库一样使用主从数据库，是读写分离中间件的主要功能。

**3. 柔性事务**

对于分布式的数据库来说，强一致性分布式事务在性能方面存在明显不足。追求最终一致性的柔性事务，在性能和一致性上则显得更加平衡。 Sharding-Sphere目前支持最大努力送达型柔性事务，未来也将支持TCC柔性事务。若不使用柔性事务，Sharding-Sphere也会自动包含弱XA事务支持。

**4. 数据治理**

Sharding-Sphere提供注册中心、配置动态化、数据库熔断禁用、调用链路等治理能力。

**Sharding-Sphere 3.X新功能**

1. Sharding-Proxy MySQL版本上线，支持DML/DDL/DAL/DQL等基本 SQL。屏蔽底层所有分库分表，可像使用单一MySQL数据库一样处理分库分表数据。
2. 新增对OR SQL语句的支持，例如：select * from t_order where (id>10 and id<20) or status=‘init’;
3. 新增对INSERT批量插入的支持，例如 insert into t_order(order_id, user_id, status) values (1, 2, ‘init’), (2, 3, ‘init’), (3, 4, ‘init’);
4. 优化对INSERT SQL语句的支持，主要包括不指定具体列进行INSERT操作，例如：insert into t_order values(1, 2,‘init’);
5. 增加解析引擎对SQL的缓存，进一步提升解析性能。
6. Sharding-JDBC新增对InlineExpression占位符$->{}的支持。

新的产品、新的特性、新的优化是不是让你眼前一亮？那就赶快把Sharding-Sphere 3.X用起来吧！

**暖心Tips：**

① 使用Sharding-JDBC，可加入以下Maven依赖：

![img](http://static.iocoder.cn/5c97c09c0262336364827aa4b9ae573a)

② 使用Sharding-Proxy，可在这里下载：

https://github.com/sharding-sphere/sharding-sphere-doc/raw/master/dist/sharding-proxy-3.0.0.M1.tar.gz

此外，我们的DOCKER下载命令如下所示：

docker pull shardingsphere/sharding-proxy

**星路历程**

Sharding-Sphere自2016开源以来，不断精进、不断发展，被越来越多的企业和个人认可：在Github上收获4000+的star，1700+forks，60+的各大公司企业使用它，为Sharding-Sphere提供了重要的成功案例。此外，越来越多的企业伙伴和个人也加入到Sharding-Sphere的开源项目中，为它的成长和发展贡献了巨大力量。

未来，我们将不断优化当前的特性，精益求精；同时，大家关注的柔性事务、数据治理等更多新特性也会陆续登场。Sharding-Sidecar也将成为云原生的数据库中间件！

愿所有有识之士能加入我们，一同描绘Sharding-Sidecar的新未来！

愿正在阅读的你也能助我们一臂之力，转载分享文章、加入关注我们！

**项目地址**

https://github.com/sharding-sphere/sharding-sphere/

https://gitee.com/sharding-sphere/sharding-sphere/

更多信息请浏览官网：

http://shardingsphere.io/