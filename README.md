# 1. 启动内存上下文子系统
这必须在创建上下文或在上下文中分配内存之前调用。TopMemoryContext和ErrorContext在这里初始化；之后必须创建其他内存上下文。
## 1.1 初始化内存上下文
当使用pg_ctl或postmaster（postgres）启动PostgreSQL数据库服务时候，在main()函数中会先对名为TopMemoryContext、CurrentMemoryContext和ErrorContext的内存上下文全局变量进行初始化操作，由函数MemoryContextInit()负责完成。其中TopMemoryContext是所有其他内存上下文的父节点。在一XXX文中有提到过，当前PostgreSQL数据库中，有以下几个比较重要的全局变量内存上下文。分别是：
- CurrentMemoryContext
- TopMemoryContext
- ErrorContext
- PostmasterContext
- CacheMemoryContext
- MessageContext
- TopTransactionContext
- CurTransactionContext
- PortalContext

CurrentMemoryContext、ErrorContext、PostmasterContext、CacheMemoryContext、MessageContext、TopTransactionContext、CurTransactionContext和PortalContext均作为TopMemoryContext的子上下文。它们之间以树的形式进行分布与管理，这几个内存上下文之间的关系如图所示：
![内存上下文示意图](https://user-images.githubusercontent.com/63132178/172780970-ead483b7-b700-476f-9d01-e1a876a797f8.png)
