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

函数MemoryContextInit的实现如下：
`void
MemoryContextInit(void)
{
	AssertState(TopMemoryContext == NULL);

	/*
	 * First, initialize TopMemoryContext, which is the parent of all others.
	 * 首先,初始化TopMemoryContext,它是所有其他内容的父节点.
	 */
	TopMemoryContext = AllocSetContextCreate((MemoryContext) NULL,
											 "TopMemoryContext",
											 ALLOCSET_DEFAULT_SIZES);

	/*
	 * Not having any other place to point CurrentMemoryContext, make it point
	 * to TopMemoryContext.  Caller should change this soon!
	 没有任何其他位置指向 CurrentMemoryContext,请将其指向TopMemoryContext.
 	 调用者应尽快更改此选项！
	 */
	CurrentMemoryContext = TopMemoryContext;

	/*
	 * Initialize ErrorContext as an AllocSetContext with slow growth rate ---
	 * we don't really expect much to be allocated in it. More to the point,
	 * require it to contain at least 8K at all times. This is the only case
	 * where retained memory in a context is *essential* --- we want to be
	 * sure ErrorContext still has some memory even if we've run out
	 * elsewhere! Also, allow allocations in ErrorContext within a critical
	 * section. Otherwise a PANIC will cause an assertion failure in the error
	 * reporting code, before printing out the real cause of the failure.
	将 ErrorContext 初始化为一个增长速度较慢的 AllocSetContext ——我们并不真正期望在其中分配太多内存.
 	更重要的是,要求它在任何时候至少8K. 只有在这种情况下,上下文中的保留内存才是必不可少的——
 	我们要确保ErrorContext仍然有一些内存,即使我们在其他地方已经用完了! 此外,
 	允许在关键部分的ErrorContext中进行分配.否则,在打印出失败的真正原因之前,
 	死机将导致错误报告代码中的断言失败.
	 *
	 * This should be the last step in this function, as elog.c assumes memory
	 * management works once ErrorContext is non-null.
	 这应该是此函数的最后一步,因为elog.c假设一旦ErrorContext为非空,内存管理器就可以工作.
	 */
	ErrorContext = AllocSetContextCreate(TopMemoryContext,
										 "ErrorContext",
										 8 * 1024,
										 8 * 1024,
										 8 * 1024);
	MemoryContextAllowInCriticalSection(ErrorContext, true);
}`
