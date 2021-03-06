# 1 定义
将领域中所发生的活动建模成一系列的离散事件。每个事件都用领域对象来表 示……领域事件是领域模型的组成部分，表示领域中所发生的事情。
一个领域事件将导致进一步的业务操作，在实现业务解耦的同时，还有助于形成完整的业务闭环。

领域事件可以是业务流程的一个步骤，比如一个事件发生后触发的后续动作，比如密码连续输错三次，触发锁定账户的动作。

# 2 识别领域事件的话术
- “如果发生……，则……”
- “当做完……的时候，请通知……”（这里的通知本身并不能构成一个事 件，而只是表明我们需要向外界发出通知。）
- “发生……时，则……”等
- “如果发生 这样的事情,它并不重要；如果发生那样的事情，它就很重要了”

在这些场景中，如果发生某种事件后，会触发进一步的操作，那么这个事件很可能就是领域事件。由于领域事件需要发布到外部系统，比如发布到另一个限界上下文。由于这样的事件由订阅方处理，它将对本地和远程上下文产生深远的影响。

那领域事件为什么要用最终一致性，而不是传统SOA的直接调用？

聚合的一个原则：一个事务中最多只能更改一个聚合实例。所以

-  本地限界上下文中的其他聚合实例便可以通过领域事件的方式同步
- 用于使远程依赖系统与本地系统保持一致。解耦本地系统和远程系统还有助于提高双方协作服务的可伸缩性。

![](https://img-blog.csdnimg.cn/20201010013815802.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNTg5NTEw,size_1,color_FFFFFF,t_70#pic_center)
聚合创建并发布事件。订阅方可以先存储事件，然后再将其转发到远程订阅方，或不经存 储，直接转发。除非MQ共享了模型的数据存储，不然即时转发需要XA（两阶段提交）。

考虑在系统非高峰时期，批处理过程通常进行一些系统维护工作，比如删除过期对象、创建新对象以支持新业务需求或通知用户所发生的重要事件。这样的批处理过程通常需复杂 查询且需庞大事务支持。若这些批处理过程存在冗余会怎么样？
系统中发生的每一件事情，我们都用事件形式捕获，然后将事件发布给订阅方处理，能简化系统吗？肯定的！它可消除先前批处理过程中的复杂查询，因为我们能够准确知道在何时发生何事，限界上下文也由此知道接下来应该做啥。在接收到领域事件时，系统可立即处理。原本批量集中处理的过程可以分散成许多粒度较小的处理单元，业务需求也由此更快满足，用户也可及时进行下一步操作。

领域事件驱动设计可以切断领域模型之间的强依赖关系，事件发布完成后，发布方不必关心后续订阅方事件处理是否成功，这样可以实现领域模型的解耦，维护领域模型的独立性和数据的一致性。在领域模型映射到微服务系统架构时，领域事件可以解耦微服务，微服务之间的数据不必要求强一致性，而是基于事件的最终一致性。

有的领域事件发生在微服务内的聚合之间，有的则发生在微服务之间，还有两者皆有的场景，一般来说跨微服务的领域事件处理居多。在微服务设计时不同领域事件的处理方式会不一样。

#  3 微服务内
当领域事件发生在微服务内的聚合间，领域事件发生后完成事件实体构建和事件数据持久化，发布方聚合将事件发布到事件总线，订阅方接收事件数据完成后续业务操作。

微服务内大部分事件的集成，都发生在同一进程，进程自身可很好控制事务，因此不一定需要引入MQ。但一个事件若同时更新多个聚合，按“一次事务只更新一个聚合”原则，可考虑引入事件总线。

微服务内应用服务，可通过跨聚合的服务编排和组合，以服务调用的方式完成跨聚合访问，这种方式通常应用于实时性和数据一致性要求高的场景。这个过程会用到分布式事务，以保证发布方和订阅方的数据同时更新成功。

在微服务内，不是少用领域事件，而是推荐少用事件总线。在DDD中是以聚合为单位进行数据管理，若一次操作会修改同一微服务内的多个聚合的数据，就需保证多个聚合的数据一致性，为了解耦不同聚合，需采用分布式事务或事件总线两种方式，用事件总线不太方便管理服务和数据的关系，可用类似saga之类的分布式事务技术。总之需要确保你的不同聚合的业务规则和数据一致性，尽量减少系统建设复杂度。

# 4 微服务间
跨微服务的领域事件会在不同限界上下文或领域模型间实现业务协作，主要都是为解耦，减轻微服务间实时服务访问压力。

领域事件发生在微服务间较多，事件处理机制也更复杂。跨微服务事件可推动业务流程或数据在不同子域或微服务间直接流转。

跨微服务的事件机制要总体考虑事件构建、发布和订阅、事件数据持久化、MQ，甚至事件数据持久化时还可能需考虑引入分布式事务。

微服务间访问也可采用应用服务直接调用，实现数据和服务的实时访问，弊端就是跨微服务的数据同时变更需要引入分布式事务。分布式事务会影响系统性能，增加微服务间耦合，尽量避免使用。 

# 5 领域事件总体架构
![](https://img-blog.csdnimg.cn/20201010004736799.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNTg5NTEw,size_1,color_FFFFFF,t_70#pic_center)

## 5.1 事件构建和发布
### 事件基本属性
至少包括：
- 事件唯一标识（全局唯一，事件能够无歧义在多个限界上下文中传递）
- 发生时间
- 事件类型
- 事件源

即主要记录事件本身以及事件发生背景的数据。

### 业务属性
记录事件发生那一刻的业务数据，这些数据会随事件传输到订阅方，以开展下一步业务操作。

事件基本属性和业务属性一起构成事件实体，事件实体依赖聚合根。领域事件发生后，事件中的业务数据不再修改，因此业务数据可以以序列化值对象的形式保存，这种存储格式在消息中间件中也比较容易解析和获取。

为保证事件结构的统一，还会创建事件基类 DomainEvent，子类可以扩充属性和方法。由于事件没有太多的业务行为，实现方法一般比较简单。


事件发布之前需要先构建事件实体并持久化。事件发布的方式有很多种
- 可通过应用服务或者领域服务发布到事件总线或MQ
- 也可从事件表中利用定时程序或数据库日志捕获技术获取增量事件数据，发布到MQ

## 5.2 事件数据持久化
### 意义
- 系统之间数据对账
- 实现发布方和订阅方事件数据的审计

当遇到MQ、订阅方系统宕机或网络中断，在问题解决后仍可继续后续业务流转，保证数据一致性。

### 实现方案
- 持久化到本地业务DB的事件表，利用本地事务保证业务和事件数据的一致性
- 持久化到共享的事件DB。业务、事件DB不在同一DB，它们的数据持久化操作会跨DB，因此需分布式事务保证业务和事件数据强一致性，对系统性能有影响

## 5.3 事件总线(EventBus)

### 意义
实现微服务内聚合间领域事件，提供事件分发和接收等服务。
是进程内模型，会在微服务内聚合之间遍历订阅者列表，采取同步或异步传递数据。

### 事件分发流程
- 若是微服务内的订阅者（其它聚合），则直接分发到指定订阅者
- 微服务外的订阅者，将事件数据保存到事件库（表）并异步发送到MQ
- 同时存在微服务内和外订阅者，则先分发到内部订阅者，将事件消息保存到事件库（表），再异步发送到MQ

## 5.4 MQ
跨微服务的领域事件大多会用到MQ，实现跨微服务的事件发布和订阅。
虽然MQ自身有持久化功能，但中间过程或在订阅到数据后，在处理之前出问题，需要进行数据对账，这样就没法找到发布时和处理后的数据版本。关键的业务数据推荐还是落库。

## 5.5 事件接收和处理
微服务订阅方在应用层采用监听机制，接收MQ中的事件数据，完成事件数据的持久化后，就可以开始进一步的业务处理。领域事件处理可在领域服务中实现。

- 有同学会问了，事件有没有被消费成功（消费端成功拿到消息或消费端业务处理成功），一般如何通知到消息生产端?
因为事件发布方有事件实体的原始的持久化数据，事件订阅方也有自己接收的持久化数据。一般可以通过定期对账的方式检查数据的一致性。

# 6 总结
今天我们主要讲了领域事件以及领域事件的处理机制。领域事件驱动是很成熟的技术，在很多分布式架构中得到了大量的使用。领域事件是DDD的一个重要概念，在设计时我们要重点关注领域事件，用领域事件来驱动业务的流转，尽量采用基于事件的最终一致，降低微服务之间直接访问的压力，实现微服务之间的解耦，维护领域模型的独立性和数据一致性。

除此之外，领域事件驱动机制可以实现一个发布方N个订阅方的模式，这在传统的直接服务调用设计中基本是不可能做到的。

参考
- 领域事件：解耦微服务的关键