## Nacos 服务注册配置中心

Naming和Configuration的前两个字母，最后s为Service

替代Eureka服务注册中心，Config服务配置中心，Bus总线



### AP和CP切换



### 配置管理

 Namespace区分部署环境

Group和DataID逻辑上区分两个目标对象 



## Sentinel 熔断与限流



## Seata 分布式事务

Simple Extensible Autonomous Transaction Architecture 简单可扩展自治事务框架

一次业务操作需要跨多个数据源或者需要跨多个系统进行远程调用，就会产生分布式事务。解决全局数据一致性	

### **AT 模式（Automatic Transaction）**



### Seata 中有三大基本组件：

1. Transaction Coordinator(TC)：维护全局和分支事务的状态，驱动全局事务提交与回滚。
2. Transaction Manager(TM)：定义全局事务的范围：开始、提交或回滚全局事务。
3. Resource Manager(RM)：管理分支事务处理的资源，与 TC通信以注册分支事务并报告分支事务的状态，并驱动分支事务提交或回滚。

### Seata 管理分布式事务的典型生命周期

1. TM要求TC开始新的全局事务、TC生成全局事务的XID
2. XID 通过微服务的调用链传播
3. RM 在 TC 中将本地事务注册为 XID 的相应全局事务的分支。
4. TM 要求 TC 提交或回滚 XID 的相应全局事务
5. TC 驱动 XID 的相应全局事务下的所有分支事务，完成分支提交或回滚



### Seata优点

1. 应用层基于SQL解析实现了自动补偿，从而最大程度的降低业务侵入性
2. 将分布式事务中TC（事务协调者）独立部署，负责事务的注册、回滚
3. 通过全局锁实现了写隔离与读隔离

