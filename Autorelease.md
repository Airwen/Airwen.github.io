# Objective-c AutorealsePool

1. 自动释放池是什么？
2. 什么样的对象是怎么添加到自动释放池的？
3. 自动释放池什么时候释放掉池里的对象。


在Objective-C版本的iOS项目的main.m文件中的代码很简单：

```
int main(int argc, char * argv[]) {
    @autoreleasepool {
        /*
	     * 这行代码将所有的事件、消息全部交给了 UIApplication 来处理
	     * return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
	     */
    }
}
```

`clang -rewrite-objc main.m` 后的代码是这样的(去掉了很多不相关的代码之后)：

```
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};

//...
int main(int argc, char * argv[])
{
   /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool;

	   /*
	    * `__AtAutoreleasePool __autoreleasepool;`创建了一个`__AtAutoreleasePool`类型的局部变量__autoreleasepool，调用了其构造器方法。
	    * 所以这句相当于：
	    * void * atautoreleasepoolobj = objc_autoreleasePoolPush();
	    *
	    * 超出了pool block的作用域 __autoreleasepool本地变量被“销毁”，掉了其“析构方法”，所以这句相当于：:
	    * objc_autoreleasePoolPop(atautoreleasepoolobj);
	    */
    
   }
}
```

所以，`@autoreleasepool {....}`翻译过来相应的代码是一段类似下面代码：

> 了解一下Struct的构造器方法，使用方法。就能体会这里使用结构体的初始化和析构来简化代码的实现是一种很巧妙的办法)：

```

int main(int argc, const char * argv[]) {
    void * atautoreleasepoolobj = objc_autoreleasePoolPush();
	
	/*
	中间的所有的autorelease对象都会被添加到当前最近的autorealeasepool
	*/

	objc_autoreleasePoolPop(atautoreleasepoolobj);
}
// 就像[自动释放池的前世今生 ---- 深入解析 autoreleasepool](https://draveness.me/autoreleasepool)所说的：
```

那么问题来了：
atautoreleasepoolobj是什么？

那就要想看看 `objc_autoreleasePoolPush()`具体的方法实现(也就是读obj4的源码，我下载的是[objc4-723.tar,gz](https://opensource.apple.com/tarballs/objc4/))：

调用栈

```
void *objc_autoreleasePoolPush(void)
		|_ AutoreleasePoolPage::push();
			 |_ autoreleaseFast(POOL_BOUNDARY);
```

看一下代码实现：

```
static inline id *autoreleaseFast(id obj)
{
        AutoreleasePoolPage *page = hotPage();
        if (page && !page->full()) {
            return page->add(obj);
        } else if (page) {
            return autoreleaseFullPage(obj, page);
        } else {
            return autoreleaseNoPage(obj);
        }
}

    ...

    class AutoreleasePoolPage 
{

	....

	#   define POOL_BOUNDARY nil

	....

	static size_t const SIZE = PAGE_MAX_SIZE;
	
	....

	//精简了若干行代码的add
    id *add(id obj)
    {
        id *ret = next;  // faster than `return next-1` because of aliasing
        *next++ = obj;
        return ret;
    }

    ...

}
```

也就是`@autoreleasepool {....}`在一开始在 hotPage中 add 了一个`POOL_BOUNDARY`，并返回`next`初始化值：

那么，next的初始化值可以在AutoreleasePoolPage的构造器方法了解到：`begin()`函数的返回值赋值给了`next`。

```
id * begin() {
        return (id *) ((uint8_t *)this+sizeof(*this));
}
//没看懂这是啥，应该是内存访问的一种写法：指针+offset，
```

所以，我就degub了一下obje4代码，可以直接下载这个[项目](https://github.com/Airwen/objc4-723-debug)debug rumtime 源码：

```
(lldb) (lldb) expression AutoreleasePoolPage::hotPage()
((anonymous namespace)::AutoreleasePoolPage *) $0 = 0x000000010100d000
(lldb) p *$0
((anonymous namespace)::AutoreleasePoolPage) $1 = {
  magic = {
    m = ([0] = 2711724449, [1] = 1330926913, [2] = 1162626386, [3] = 558191425)
  }
  next = 0x000000010100d048
  thread = 0x000000010099d380
  parent = 0x0000000000000000
  child = 0x0000000000000000
  depth = 0
  hiwat = 0
}
(lldb) p $1.printAll()
objc[69922]: ##############
objc[69922]: AUTORELEASE POOLS for thread 0x10099d380
objc[69922]: 2 releases pending.
objc[69922]: [0x10100d000]  ................  PAGE  (hot) (cold)
objc[69922]: [0x10100d038]  ################  POOL 0x10100d038
objc[69922]: [0x10100d040]       0x100b3ef70  __NSCFString
objc[69922]: ##############
(lldb) p $1.begin()
(id *) $2 = 0x000000010100d038
(lldb) p *$2
(id) $3 = nil
(lldb) p sizeof($1)
(unsigned long) $4 = 56
(lldb) p $1.end()
(id *) $5 = 0x000000010100e000
(lldb) 
```

明白了begin()函数的意思，this是AutoreleasePoolPage内存(开始的)地址，sizeof(*this)应该是AutoreleasePoolPage的成员变量的大小，换句话说：
**从this开始“越过”变量块达到Pool开始的地方**

> 有空的话也可以算一下 p sizeof($1) = 56 = 0x38
> SIZE = PAGE_MAX_SIZE = 4096/* bytes per 80386 page */ = 0x1000
> 0x000000010100d000 + 0x38 = 0x000000010100d038 // POOL_BOUNDARY == $1.begin()
> 0x000000010100d000 + 0x1000 = 0x000000010100e000 // page.end()
> 正如[draveness在“自动释放池中的栈”](https://draveness.me/autoreleasepool#objc_autoreleasePoolPush)这一小节中所说的：AutoreleasePoolPage的内存结构中有 `56 bit` 用于存储 AutoreleasePoolPage 的成员变量，剩下的 0x10100d038 ~ 0x10100e000 都是用来存储加入到自动释放池中的对象。
> begin() 和 end() 这两个类的实例方法帮助我们快速获取 0x10100d038 ~ 0x10100e000 这一范围的边界地址。

所以，atautoreleasepoolobj是就是 POOL_BOUNDARY：每个自动释放池初始化调用 objc_autoreleasePoolPush 的时候，都会把一个 POOL_BOUNDARY push 到自动释放池的栈顶，并且返回这个 POOL_BOUNDARY “边界对象”。

`POOL_BOUNDARY`是什么从源代码和debug都看的出，它就是个`nil` —— 一个pool的开始。


2. 什么样的对象是怎么添加到自动释放池的？

MRC时，想读对象发出了`-[NSObject autorelease]`消息的对象都会被添加到“最近的”autorelease pool，但是在ARC这个方法已经被“藏起来了”，但现在可以用`__autoreleasing`关键字将对象添加到自动释放池中。

看一下NSbOject.mm的`-[NSObject autorelease]`调用栈

```
-[NSObject autorelease]
	|_ objc_object::rootAutorelease()
		 |_ objc_object::rootAutorelease2()
		 	  |_ AutoreleasePoolPage::autorelease((id)this)
		 	  	   |_ AutoreleasePoolPage::autoreleaseFast(obj); //略看眼熟，就是push里面调用的方法啊

/*
 * 在 autorelease 方法的调用栈中:
 * 最终都会调用上面提到的 autoreleaseFast 方法，将当前对象加到 AutoreleasePoolPage 中。
 */
```

那么`__autoreleasing`是怎么回事儿？我们来`clang -S -fobjc-arc -emit-llvm main.m -o main_autorelease.ll`一下main.m的实现

```
 #import <Foundation/Foundation.h>
int main(int argc, char * argv[])
{
    @autoreleasepool {
        NSString __autoreleasing *obj = [[NSObject alloc] init];
    }
}
```

得到：

```
; Function Attrs: noinline optnone ssp uwtable
define i32 @main(i32, i8**) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i8**, align 8
  %5 = alloca %0*, align 8
  store i32 %0, i32* %3, align 4
  store i8** %1, i8*** %4, align 8
  %6 = call i8* @objc_autoreleasePoolPush() #2
  %7 = load %struct._class_t*, %struct._class_t** @"OBJC_CLASSLIST_REFERENCES_$_", align 8
  %8 = load i8*, i8** @OBJC_SELECTOR_REFERENCES_, align 8, !invariant.load !8
  %9 = bitcast %struct._class_t* %7 to i8*
  %10 = call i8* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to i8* (i8*, i8*)*)(i8* %9, i8* %8)
  %11 = bitcast i8* %10 to %1*
  %12 = load i8*, i8** @OBJC_SELECTOR_REFERENCES_.2, align 8, !invariant.load !8
  %13 = bitcast %1* %11 to i8*
  %14 = call i8* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to i8* (i8*, i8*)*)(i8* %13, i8* %12)
  %15 = bitcast i8* %14 to %1*
  %16 = bitcast %1* %15 to %0*
  %17 = bitcast %0* %16 to i8*
  %18 = call i8* @objc_autorelease(i8* %17) #2
  %19 = bitcast i8* %18 to %0*
  store %0* %19, %0** %5, align 8
  call void @objc_autoreleasePoolPop(i8* %6)
  ret i32 0
}

//change一下伪代码

int main(int argc, const char * argv[]) {
    void * atautoreleasepoolobj = objc_autoreleasePoolPush();
	
	NSString *obj = [[NSObject alloc] init];

	objc_autorelease(obj)

	objc_autoreleasePoolPop(atautoreleasepoolobj);
}

```


看一下`objc_autorelease(id obj)`调用栈

```
id objc_autorelease(id obj)
	 |_ objc_object::autorelease()// 这个方法还有一个路径时发送`SEL_autorelease`消息
	      |_ objc_object::rootAutorelease()
			   |_ objc_object::rootAutorelease2()
			 	   |_ AutoreleasePoolPage::autorelease((id)this)
			 	  	    |_ AutoreleasePoolPage::autoreleaseFast(obj); //略看眼熟，就是push里面调用的方法啊
```


类似通过类方法实现创建对象实例并返回的话，同样还是`clang -S -fobjc-arc -emit-llvm main.m -o main_autorelease2.ll`：

```
#import <Foundation/Foundation.h>

@interface Person : NSObject
+ (Person *)person;
@end


@implementation Person
+ (Person *)person {
    return [[Person alloc] init];
}
@end

int main(int argc, char * argv[])
{ 
    Person *person = [Person person];
}
```

直接伪代码了，并注释了：

```
define internal %0* @"\01+[Person person]"(i8*, i8*) #0 {
  return objc_autoreleaseReturnValue([[Person alloc] init])
}

; Function Attrs: nonlazybind
declare i8* @objc_msgSend(i8*, i8*, ...) #1

declare i8* @objc_autoreleaseReturnValue(i8*)

; Function Attrs: noinline optnone ssp uwtable
define i32 @main(i32, i8**) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i8**, align 8
  %5 = alloca %0*, align 8
  store i32 %0, i32* %3, align 4
  store i8** %1, i8*** %4, align 8
  %6 = load %struct._class_t*, %struct._class_t** @"OBJC_CLASSLIST_REFERENCES_$_", align 8
  %7 = load i8*, i8** @OBJC_SELECTOR_REFERENCES_.4, align 8, !invariant.load !8
  %8 = bitcast %struct._class_t* %6 to i8*
  %9 = call %0* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to %0* (i8*, i8*)*)(i8* %8, i8* %7)
  %10 = bitcast %0* %9 to i8*
  %11 = call i8* @objc_retainAutoreleasedReturnValue(i8* %10) #2 
  %12 = bitcast i8* %11 to %0*
  store %0* %12, %0** %5, align 8
  %13 = bitcast %0** %5 to i8**
  call void @objc_storeStrong(i8** %13, i8* null) #2 //将location变量置为null，并release一下location变量指向的那个对象
  ret i32 0
}
```

主要看`objc_autoreleaseReturnValue`和`objc_retainAutoreleasedReturnValue`这两个函数是成对出现的。它用于**alloc/new/copy/mutableCopy**方法之外的类方法等返回对象的实现上。

```
// Prepare a value at +1 for return through a +0 autoreleasing convention.
id 
objc_autoreleaseReturnValue(id obj)
{
    if (prepareOptimizedReturn(ReturnAtPlus1)) return obj;

    return objc_autorelease(obj);
}
``

从`objc_autoreleaseReturnValue`函数是想如果`prepareOptimizedReturn`的话会直接返回这个对象，否放在autoreleasepool中。所以，如果方法或函数的调用方在调用了方法和函数后紧接着调用`objc_retainAutoreleasedReturnValue`函数，那么就不讲返回对象注册到autoreleasepool中，而是直接传递方法或函数的调用方。通过`objc_autoreleaseReturnValue`和`objc_retainAutoreleasedReturnValue`这两个函数协作，可以不讲对象注册到autoreleasepool中，而直接传递出去，这一过程达到了最优化。


```
id
objc_retainAutoreleasedReturnValue(id obj)
{
    if (acceptOptimizedReturn() == ReturnAtPlus1) return obj;

    return objc_retain(obj);
}
```

如果是`acceptOptimizedReturn`方式的话不做任何操作直接返回这个对象，否则返回`objc_retain`的对象。看来通过

> 1. 有时间可以研究一下`prepareOptimizedReturn`函数和`acceptOptimizedReturn`函数的实现。
> 2. 如果`main`函数中的`Person *person = [Person person];`调用改成`[Person person];`后，执行clang命令得到的中间代码，在main函数中没有调用`objc_retainAutoreleasedReturnValue`和`objc_storeStrong`函数，而是调用`objc_unsafeClaimAutoreleasedReturnValue`函数。`objc_unsafeClaimAutoreleasedReturnValue`函数也在`prepareOptimizedReturn`函数的优化范围之内。

所以，确定的发送autoreleasese消息的对象会被添加到pool中：`__autoreleasing`；
但是，通过类方法得到实例对象会被添加到autoreleasepool中么，《Objective-C高级编程 iOS与OS X多线程和内存管理》中貌似不会的，但是我debug了下面代码：

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Person  *objc = [Person person];
        NSMutableArray *array = [NSMutableArray array];
        [Person person];
        /*
        //test for the autorelease of subThread
        [NSThread detachNewThreadWithBlock:^{
            NSLog(@"current thread %@", [NSThread currentThread]);
            NSString *s = @"Air";
            [s stringByAppendingString:@"Loves Lala"];
            NSLog(@"current thread %@", [NSThread currentThread]);
        }];
         */
    }
    return 0;
}
```

debug了一下，貌似是会被添加到pool中的，lldb信息是：

```
(lldb) expression AutoreleasePoolPage::hotPage()
((anonymous namespace)::AutoreleasePoolPage *) $0 = 0x0000000103802000
(lldb) p $0
((anonymous namespace)::AutoreleasePoolPage *) $0 = 0x0000000103802000
(lldb) p *$0
((anonymous namespace)::AutoreleasePoolPage) $1 = {
  magic = {
    m = ([0] = 2711724449, [1] = 1330926913, [2] = 1162626386, [3] = 558191425)
  }
  next = 0x0000000103802058
  thread = 0x000000010099d380
  parent = 0x0000000000000000
  child = 0x0000000000000000
  depth = 0
  hiwat = 0
}
(lldb) p $1.printAll()
objc[79793]: ##############
objc[79793]: AUTORELEASE POOLS for thread 0x10099d380
objc[79793]: 4 releases pending.
objc[79793]: [0x103802000]  ................  PAGE  (hot) (cold)
objc[79793]: [0x103802038]  ################  POOL 0x103802038
objc[79793]: [0x103802040]       0x102f3e350  Person
objc[79793]: [0x103802048]       0x102f3e4d0  __NSArrayM
objc[79793]: [0x103802050]       0x102f3e3f0  Person
objc[79793]: ##############
(lldb) 
```


3. 自动释放池什么时候释放掉池里的对象。

就是在`@autoreleasepool {....}`作用于结束时，会调用`objc_autoreleasePoolPop(atautoreleasepoolobj);`，传递给函数的参数时之前的得到的POOL_BOUNDARY “边界对象”。

pop调用栈：

```
void objc_autoreleasePoolPop(void *ctxt)
	   |_ AutoreleasePoolPage::pop(ctxt);
```

`objc_autoreleasePoolPop`调用时，就会向自动释放池中的对象发送 `release` 消息，直到你传入的`POOL_BOUNDARY`。

最后回答 1. 自动释放池是什么？


在NSObject.mm文件中 627-639

/***********************************************************************
   Autorelease pool implementation

   A thread's autorelease pool is a stack of pointers. 
   Each pointer is either an object to release, or POOL_BOUNDARY which is 
     an autorelease pool boundary.
   A pool token is a pointer to the POOL_BOUNDARY for that pool. When 
     the pool is popped, every object hotter than the sentinel is released.
   The stack is divided into a doubly-linked list of pages. Pages are added 
     and deleted as necessary. 
   Thread-local storage points to the hot page, where newly autoreleased 
     objects are stored. 
**********************************************************************/

雷纯锋[Objective-C Autorelease Pool 的实现原理的AutoreleasePoolPage小节](http://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/)的也给出了很赞的翻译：

* 每一个线程的 autoreleasepool 其实就是一个指针的堆栈；
* 每一个指针代表一个需要 release 的对象或者 POOL_SENTINEL（哨兵对象，代表一个 autoreleasepool 的边界）；
* 一个 pool token 就是这个 pool 所对应的 POOL_SENTINEL 的内存地址。当这个 pool 被 pop 的时候，所有内存地址在 pool token 之后的对象都会被 release ；
* 这个堆栈被划分成了一个以 page 为结点的双向链表。pages 会在必要的时候动态地增加或删除；
* Thread-local storage（线程局部存储）指向 hot page ，即最新添加的 autoreleased 对象所在的那个 page 。

而[Draveness](https://draveness.me)的[自动释放池的前世今生 ---- 深入解析 autoreleasepool](https://draveness.me/autoreleasepool#objc_autoreleasePoolPush)画出了AutoreleasePool很赞的内存结构，以及pop操作的过程。

至此，还有一个问题？

1. Thread\ Runloop\ AutorealsePool 之间的关系，雷纯锋[Objective-C Autorelease Pool 的实现原理的NSThread、NSRunLoop 和 NSAutoreleasePool小节](http://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/)做了一些描述：

每个 run loop 开始前，系统会自动创建一个 autoreleasepool ，并在 run loop 结束时 drain 。但是，根据runloop的特性只有在你“访问”子，线程的runloop时会自动创建 NSRunLoop 对象 —— 也就是如果你不去访问也就没有runloop，这是就没有autoreleasepool了？那么这是自线程中的autorelease对象会怎么样？

嗯，将当前对象加到 AutoreleasePoolPage 中，最终都会调用上面提到的 `autoreleaseFast` 方法，在查找不到当前的page时，会调用`autoreleaseNoPage(obj)`创建一个 hotPage，然后调用 page->add(obj) 方法将对象添加至 AutoreleasePoolPage 的栈中。那么，问题来了这pool什么时候被释放呢？

可以参照一下这些文章

[does NSThread create autoreleasepool automatically now?](https://stackoverflow.com/questions/24952549/does-nsthread-create-autoreleasepool-automatically-now)

苹果官方文档[Using Autorelease Pool Blocks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html#//apple_ref/doc/uid/20000047-CJBFBEDI)这样描述：

Cocoa always expects code to be executed within an autorelease pool block, otherwise autoreleased objects do not get released and your application leaks memory. (If you send an autorelease message outside of an autorelease pool block, Cocoa logs a suitable error message.) The AppKit and UIKit frameworks process each event-loop iteration (such as a mouse down event or a tap) within an autorelease pool block. Therefore you typically do not have to create an autorelease pool block yourself, or even see the code that is used to create one. There are, however, three occasions when you might use your own autorelease pool blocks:

* If you are writing a program that is not based on a UI framework, such as a command-line tool.
* If you write a loop that creates many temporary objects.
* You may use an autorelease pool block inside the loop to dispose of those objects before the next iteration. Using an autorelease pool block in the loop helps to reduce the maximum memory footprint of the application.

**If you spawn a secondary thread.
You must create your own autorelease pool block as soon as the thread begins executing; otherwise, your application will leak objects. (See [Autorelease Pool Blocks and Threads](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html#//apple_ref/doc/uid/20000047-1041876) for details.)**


Autorelease Pool Blocks and Threads
Each thread in a Cocoa application maintains its own stack of autorelease pool blocks. If you are writing a Foundation-only program or if you detach a thread, you need to create your own autorelease pool block.

If your application or thread is long-lived and potentially generates a lot of autoreleased objects, you should use autorelease pool blocks (like AppKit and UIKit do on the main thread); otherwise, autoreleased objects accumulate and your memory footprint grows. If your detached thread does not make Cocoa calls, you do not need to use an autorelease pool block.

Note: If you create secondary threads using the POSIX thread APIs instead of NSThread, you cannot use Cocoa unless Cocoa is in multithreading mode. Cocoa enters multithreading mode only after detaching its first NSThread object. To use Cocoa on secondary POSIX threads, your application must first detach at least one NSThread object, which can immediately exit. You can test whether Cocoa is in multithreading mode with the NSThread class method isMultiThreaded.

