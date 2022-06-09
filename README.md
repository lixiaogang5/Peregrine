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
```c
void MemoryContextInit(void)
{
	//封装的assert()断言函数的功能, 即AssertState括号中的判别式需要为true
	AssertState(TopMemoryContext == NULL);

	/* 首先, 初始化TopMemoryContext,它是所有其他内存上下文的父节点. */
	TopMemoryContext = AllocSetContextCreate((MemoryContext) NULL,
						 "TopMemoryContext",
						 ALLOCSET_DEFAULT_SIZES);

	/* 没有任何其他位置指向 CurrentMemoryContext,请将其指向TopMemoryContext. 调用者应尽快更改此选项！ */
	CurrentMemoryContext = TopMemoryContext;

	/*
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
}
```

## 1.2 初始化TopMemoryContext
首先，需要初始化TopMemoryContext内存上下文，因为它是所有其他内存上下文的父节点。其初始化功能由函数AllocSetContextCreate()完成，该函数支持五个参数，第一个是用于指定待初始化的内存上下文变量，第二个参数是该内存上下文的名称（主要用于调试使用）；第三个参数是用于指定最小内存上下文大小；第四个参数用于指定初始化块大小；第五个参数用于指定最大块（block）大小。

由于是初始化TopMemoryContext内存上下文，所以第二个参数就以该内存上下文变量为名称（`TopMemoryContext`）。在这里，ALLOCSET_DEFAULT_SIZES是一个宏定义，其声明于文件src/include/utils/memutils.h中。如下：
```c
/*
 * Recommended default alloc parameters, suitable for "ordinary" contexts
 * that might hold quite a lot of data.
 */
#define ALLOCSET_DEFAULT_MINSIZE   0					//0KB
#define ALLOCSET_DEFAULT_INITSIZE  (8 * 1024)			//8KB
#define ALLOCSET_DEFAULT_MAXSIZE   (8 * 1024 * 1024)	//8MB
#define ALLOCSET_DEFAULT_SIZES \
		ALLOCSET_DEFAULT_MINSIZE, ALLOCSET_DEFAULT_INITSIZE, ALLOCSET_DEFAULT_MAXSIZE
```



