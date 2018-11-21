# Objective-c Category

1. Category中的method
2. Category中的property
3. Extension


##Category中的method

我们都知道Category可以为原类扩展方法，无论是“原类”的实例对象还是“其”子类的都可以调用Category中的方法。

美团的[深入理解Objective-C：Category](https://tech.meituan.com/DiveIntoCategory.html)中的：

* **4、追本溯源-category如何加载**小节，如果Category和“原类”有相同的方法时调用机制？
  
  1. category的方法没有“完全替换掉”原来类已经有的方法，也就是说如果category和原来类都有methodA，那么category附加完成之后，类的方法列表里会有两个methodA
  2. category的方法被放到了新方法列表的前面，而原来类的方法被放到了新方法列表的后面，这也就是我们平常所说的category的方法会“覆盖”掉原来类的同名方法，这是因为运行时在查找方法的时候是顺着方法列表的顺序查找的，它只要一找到对应名字的方法，就会罢休^_^，殊不知后面可能还有一样名字的方法。（也就是调用Category中的实现）。
* **5、旁枝末叶-category和+load方法**小节，做了对多个Category重复声明`+ (void)load`声明的方法调用机制？
  
  所有的load方法都调用了，而且

  1. 可以调用，因为附加category到类的工作会先于+load方法的执行
  2. +load的执行顺序是先类，后category，而category的+load执行顺序是根据编译顺序决定的。——也就是编译顺序是文件在TARGET->Build Phases->Complie Sources中的先后顺序。


我这里又做了下面两种测试：

1. 如果不同的Category在不同的文件声明并实现了相同的方法？
   根据是文件在TARGET->Build Phases->Complie Sources中的先后顺序，只调用在最后（最下面）的Category中的方法实现。

2. 如果不同的Category在同一文件声明并实现了相同的方法？
   只调用@implementation（最下面）的Category中的方法实现。


所以，Category的方法调用应该和`@implementation`被编译的先后顺意有关，那个Category的`@implementation`最后编译，就调用其的方法实现 —— 这套规则值针对于

##Category中的property

在Category不能声明实例变量。
可以在Category声明property，但是需要自己实现getter/setter方法。其实，普通的property实际上是： setter/getter + ivar。所以，要想通过Category为原类增加“实例变量”，是可以通过runtime的Associated Objects来实现，Category的的property实际上是：setter/getter + AssociatedObjects

但是有个问题：

1. 关联对象被存储在什么地方，是不是存放在被关联对象本身的内存中？
2. 关联对象的生命周期是怎样的，什么时候被释放，什么时候被移除？
3. 关联对象的五种关联策略有什么区别，有什么坑？

从调用栈开始

```
objc_setAssociatedObject
   |_ _object_set_associative_reference
```

从 AssociationsManager 代码上可以了解到：

```
// class AssociationsManager manages a lock / hash table singleton pair.
// Allocating an instance acquires the lock, and calling its assocations()
// method lazily allocates the hash table.

spinlock_t AssociationsManagerLock;

class AssociationsManager {
    // associative references: object pointer -> PtrPtrHashMap.
    static AssociationsHashMap *_map;
public:
    AssociationsManager()   { AssociationsManagerLock.lock(); }
    ~AssociationsManager()  { AssociationsManagerLock.unlock(); }
    
    AssociationsHashMap &associations() {
        if (_map == NULL)
            _map = new AssociationsHashMap();
        return *_map;
    }
};

AssociationsHashMap *AssociationsManager::_map = NULL;
```

> 从AssociationsManager的注释粗浅地翻译过来：
> AssociationsManager类管理一个lock(一个spinlock_t类型的全局锁对象AssociationsManagerLock，实际用到的是`os_unfair_lock`)和一个单对哈希表(hash table singleton pair —— unordered_map的子类AssociationsHashMap。)。

AssociationsHashMap的结构：

```
class AssociationsHashMap : public unordered_map<disguised_ptr_t, ObjectAssociationMap *, DisguisedPointerHash, DisguisedPointerEqual, AssociationsHashMapAllocator> {
    public:
        void *operator new(size_t n) { return ::malloc(n); }
        void operator delete(void *ptr) { ::free(ptr); }
    };
```

AssociationsHashMap 与 unordered_map 模版类的声明对比

```
disguised_ptr_t              - class _Key                   					- key_type
ObjectAssociationMap *       - class _Tp                    					- mapped_type
DisguisedPointerHash         - class _Hash = hash<_Key>     					- hasher
DisguisedPointerEqual        - class _Pred = equal_to<_Key>                     - key_equal
AssociationsHashMapAllocator - class _Alloc = allocator<pair<const _Key, _Tp> >	- allocator_type
```

也就是以`disguised_ptr_t`作为Key，`ObjectAssociationMap *`作为对应的value。

DisguisedPointerHash是实现了根据_Key生成相应Hash值的相应算法。

ObjectAssociationMap 是`std::map`的子类，它是以“你”为关联对象制定的Key作为map中的key, ObjectAssociationMap 与 unordered_map 模版类的声明对比

```
void *             				- class _Key                 		- key_type
ObjcAssociation    				- class _Tp                  		- mapped_type
ObjectPointerLess  				- class _Compare = less<_Key>		- key_compare
ObjectAssociationMapAllocator   - allocator<pair<const _Key, _Tp> > - allocator_type
```

总的全局关联对象_map结构大概是：

```

AssociationsHashMap
        |_ key - DisguisedPointerHash(DISGUISE(id object))
        |_ value - ObjectAssociationMap *
                          |_ key(client指定的) - void *key(client)
                          |_ value - ObjcAssociation(id value, uintptr_t policy)

//可以看出AssociationsHashMap 是一个无序的哈希表，维护了从对象地址到 ObjectAssociationMap 的映射；
// ObjectAssociationMap 是一个 C++ 中的 map ，维护了从 key 到 ObjcAssociation 的映射，即关联记录；
// ObjcAssociation 是一个 C++ 的类，表示一个具体的关联结构，主要包括两个实例变量，_policy 表示关联策略，_value 表示关联对象。
```

现在，可以回答问题：1. 关联对象被存储在什么地方，是不是存放在被关联对象本身的内存中？ 

答： 关联对象对象放在了全局的AssociationsHashMap中，并不是放在被关联对象本身的内存中（所以关联对象并不是一个类的实例变量-ivar）。
    所以类的内存的大小是在编译时期就决定了，而类的实例变量是通过对象的指针和相应的offset查找的。
    原类或者Extension中的property是getter+setter+ivar；
    Category可以声明property，但是需要实现getter和setter方法，具体怎么实现要程序员要扩展什么？可以是getter/setter一个关联对象；读写磁盘上的某个数据；某个计算表达式的结果。
    
由property联想的题外话，具体看[这里](https://github.com/ibireme/YYModel/issues/30)的讨论：

property，自上至下主要有 3 种方法：KVC、getter/setter(Accessor Methods)、ivar。

1. KVC 兼容性最好，它会尝试用 getter/setter，失败时则直接访问 ivar。缺点是性能差，非对象类型需要包装成值对象。
2. getter/setter 性能最高，和直接手动访问属性性能相同。缺点就是运行时代码写起来太麻烦。
3. ivar 访问时，性能并不比 getter/setter 高，而且对没有 ivar 的 property 无效，另外相对上面两种方法，它不支持 CoreData 或其他 dynamic 属性的对象。

为什么，ivar 性能并不比 getter/setter 高？
通常，在 OC 中直接访问 ivar，是通过 object_getIvar() object_setIvar() 来进行的，我测试过这两个方法性能并不高。getter/setter 是直接通过编译时对象 struct 内存地址偏移来访问的，性能自然很高。

通过 OC 运行时实际上也能获得属性在对象 struct 里的地址偏移，是一个 ptrdiff_t 类型的数值，但我没见过有人能用这个值来直接访问内存地址。一方面 OC 对象的结构并不是公开的，而且在过去也经过好几次的变更；另一方面，如果通过 ivar 访问，那 assign、weak、strong 等逻辑都需要自己实现一遍。

代码里直接访问 self 的 ivar，是编译后通过地址偏移访问的。self.property 比直接访问 _propertyName 多了个函数调用，但实际性能都很高不用担心。


第二个问题的探索，就是针对关联对象的内存管理：

当我们设置的时候关联对象时，在`_object_set_associative_reference`函数的一开始通过`acquireValue`函数置换出新的对象`new_value`，具体怎么置换出了新的的new_value呢？在`acquireValue`函数中根据设置的关联策略(policy)对value做相应的内存策略：

1. 对RETAIN策略做retain操作：返回`objc_retain`函数的结果；
2. 对COPY策略的操作： value 发送一个copy消息，并返回其操作结果；
3. 其他不做任何操作直接返回其结果

当被关联的对象是方时的调用栈可以看出：

```
-[NSObject dealloc]
      |__objc_rootDealloc(id obj) //全局函数
            |_ objc_object::rootDealloc() 
                  |_object_dispose //全局函数
                      |_ objc_destructInstance //全局函数
                             |_ _object_remove_assocations //全局函数
```

这个对象dealloc的时候，在`_object_remove_assocations`函数中将对象从AssociationsHashMap中移除，然后遍历该对象所有的关联值，关联值做“release”操作：

```
static void releaseValue(id value, uintptr_t policy) {
    if (policy & OBJC_ASSOCIATION_SETTER_RETAIN) {
        return objc_release(value);
    }
}
```

也就是对copy和retain策略的关联对象做“release”操作。

现在，可以回答问题：2. 关联对象的生命周期是怎样的，什么时候被释放，什么时候被移除？
答： 关联对象的生命周期和其关联策略有关，对象的property的copy和retian(strong)的内存管理策略的一样的：所以使用ASSIGN策略关联一个对象类型的value时要小心，注意它的生命周期是不受“你”的控制的。


3. 关联对象的五种关联策略有什么区别，有什么坑？

其实在分析问题二的时候，已经对ASSIGN，RETAIN，COPY着三种策略做了分析。但是，对RETAIN，COPY策略分别有not made atomically和atomically不同版本：

```
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```

但是从runtime的源代码上没有看出来其间区别...


参照文章：

[Objective-C Associated Objects 的实现原理](http://blog.leichunfeng.com/blog/2015/06/26/objective-c-associated-objects-implementation-principle/)

[Objective-C Category 的实现原理](http://blog.leichunfeng.com/blog/2015/05/18/objective-c-category-implementation-principle/)

[深入理解Objective-C：Category](https://tech.meituan.com/DiveIntoCategory.html)

