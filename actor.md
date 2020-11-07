# Orleans与Actor

* Actor是存在于分布式集群中可以远程访问的对象
* Actor可以通过Id进行远程调用
* Actor的调用都是顺序性的，不需要关心线程安全问题
* Orleans会自动在集群中动态分配Actor，不需要关注Actor在集群的位置
* Orleans会自动根据节点的压力对Actor进行动态负载调度
* Orleans提供友好的生命周期接口，可以在Actor激活和失活时进行业务和数据处理

## Vertex与Orleans的关系

* Vertex构建在Orleans之上，基于Orleans来提供分布式和线程安全
* Vertex拓展和增强了Orleans中的基础Grain的功能
* Vertex实现了一些Grain来提供内部服务，例如分布式锁、分布式ID生成等

## 常用的Vertex内置Actor

* __VertexActor：__ 最基本Actor，实现了事件提交、事件发布、事件执行、快照读取、事件读取、快照还原等功能。
  
* __FlowActor：__ 流程节点Actor，负责监听VertexActor的事件和校验事件完整性并自动补全丢失的事件，然后把事件送到EventHandle方法进行处理。
* __ShadowActor：__ 影子Actor，负责监听VertexActor的事件并自动还原状态，保证内部状态和VertexActor的状态一致，常用来执行VertexActor的异步逻辑或进行读写分离。
* __InnerTxActor：__ 继承自VertexActor，但支持使用内置事务功能一次性提交N个事件(VertexActor每次只能提交一个事件)。
* __ReentryActor：__ 继承自InnerTxActor，在Actor打开并发开关时提供线程安全的事件提交入口，防止状态异常，能大幅提高热点Actor的吞吐能力。
* __DTxActor：__ 继承自InnerTxActor，支持强一致性事务编程模型(多个关联的Actor在事务过程中产生的事件能进行统一的提交或回滚)。
* __ReentryDTxActor：__ 继承自ReentryActor，支持并发的强一致性事务编程模型。
* __DTxUnitActor：__ 事务的入口和控制器，负责单元内部的事务流程控制。
