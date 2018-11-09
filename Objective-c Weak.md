Objective-c Weak

通过clang将代码编译`clang -S -fobjc-arc -emit-llvm main.m -o main.ll`，分析llvm的中间语言，通过以下命令将代码编译成中间语言(通过[ARC原理探究](http://luoxianming.cn/2017/05/06/arc/)了解到的)：

```
 #import <Foundation/Foundation.h>
int main(int argc, char * argv[])
{
    NSObject *obj = [[NSObject alloc] init];
    id __weak obj1 = obj;
}
```

中间代码为：

```
define i32 @main(i32, i8**) #0 {
  //抽调冗余代码
  %18 = call i8* @objc_initWeak(i8** %6, i8* %17) #2
  call void @objc_destroyWeak(i8** %6) #2
  %19 = bitcast %0** %5 to i8**
  call void @objc_storeStrong(i8** %19, i8* null) #2
  ret i32 0
}
```

另外，也可以通过断点在`id __weak obj1 = obj;`语句，调试objc的源码([runtime Debug项目](https://github.com/Airwen/objc4-723-debug))，在通过`Debug bar`上的`step into`按钮，一步一步看weak的调用栈：

```
id objc_initWeak(id *location, id newObj)
      |_ static id storeWeak(id *location, objc_object *newObj) 
                      |_ id weak_register_no_lock(weak_table_t *weak_table, id referent_id, id *referrer_id, bool crashIfDeallocating)
```

weak referrer 是“放在”`SideTable`的`weak_table`(`weak_table_t`结构体)中。

```
struct SideTable
          |_ **weak_table_t** weak_table
                   |_ **size_t** num_entries
                   |_ **uintptr_t** mask
                   |_ **weak_entry_t * **weak_entries(struct ``，初始大小为64，)
                             |_ weak_referrer_t * inline_referrers
                             |_ weak_referrer_t ** referrer
					               |_ referent(DisguisedPtr<objc_object>)

struct `weak_table_t`: The global weak references table. <Key(object ids): Value(weak_entry_t structs)>

DisguisedPtr<objc_object> referent;
typedef DisguisedPtr<objc_object *> weak_referrer_t;

DisguisedPtr 是一个C++类，它的注释是这样写到：
DisguisedPtr<T> acts like pointer type T*, except the stored value is disguised to hide it from tools like `leaks`. nil is disguised as itself so zero-filled memory works as expected, which means 0x80..00 is also disguised as itself but we don't care. Note that weak_entry_t knows about this encoding.

DisguisedPtr.value = disguise(PointerType)
PointerType = undisguise(DisguisedPtr.value)
```

> `weak_entry_t`结构中有两个关键的部分：
> refernt 和 referrers
> 根据`DisguisedPtr<T>`定义，其实指针类型 —— `T*`，那么：
> refernt 是 `objc_object *`
> referrers 是 `objc_object ** *` —— 就是一个“引用数组”，所谓的“引用” —— 其实就是存着“指针值”的一小块内存(8byte or 4byte)的地址: `inline_referrers`是固定长度`4`，`referrers`动态长度。
>    “指针值” —— 就是一个(要被弱引用的)“实例对象”的内存地址，也就是`objc_object *`

debug一下面的的代码看一下，会比较清晰：

```
Person *objc = [[Person alloc] init];
id __weak wobjc = objc;
```

# Debug信息

```
(lldb) p new_entry
(weak_entry_t *) $1 = 0x00007ffeefbff548
(lldb) p *$1
(weak_entry_t) $2 = {
  referent = (value = 18446744069401947312)
   = {
     = {
      referrers = 0xffff800110400920
      out_of_line_ness = 0
      num_refs = 0
      mask = 0
      max_hash_displacement = 0
    }
     = {
      inline_referrers = {
        [0] = (value = 18446603340788795680)
        [1] = (value = 0)
        [2] = (value = 0)
        [3] = (value = 0)
      }
    }
  }
}
(lldb) p $1->referent
(DisguisedPtr<objc_object>) $3 = (value = 18446744069401947312)
(lldb) p $3.value
(uintptr_t) $4 = 18446744069401947312
(lldb) p undisguise($4)
(Person *) $5 = 0x0000000100c0d350
//回到main方法的调用栈中，打印一下objc信息是
Printing description of objc:
(Person *) objc = 0x0000000100c0d350
(lldb) p $2.inline_referrers[0]
(weak_referrer_t) $10 = (value = 18446603340788795680)
(lldb) p $10.value
(uintptr_t) $11 = 18446603340788795680
(lldb) p undisguise($11)
(id) $12 = 0x00007ffeefbff6e0
//回到`objc_initWeak`和`storeWeak`打印location
Printing description of location:
(id *) location = 0x00007ffeefbff6e0
(lldb) p *location
(Person *) $14 = 0x0000000100c0d350
(lldb) p wobjc
(Person *) $15 = 0x0000000100c0d350
(lldb) p &wobjc
(id *) $16 = 0x00007ffeefbff6e0
```

注意看：`&wobjc`，`location`，`undisguise(weak_entry_t.inline_referrers[0].value)`三个值是一样的，是内存中“引用的地址”。而`weak_entry_t.referent`，`wobjc`就是“对象”的指针地址。“引用的地址”指向的那块内存村的是“对象”的指针地址。最好了解一下“指针”与“引用”。


NSObject释放是的调用栈：

```
-[NSObject dealloc]
    |_ _objc_rootDealloc(id obj)
          |_ objc_object::rootDealloc()
               |_ object_dispose(id obj)  `objc-runtime-new.mm`
                    |_ void *objc_destructInstance(id obj)
                          |_ objc_object::clearDeallocating()
                               |_ objc_object::clearDeallocating_slow() // Slow path for non-pointer isa with weak refs and/or side table data.
                               |_ weak_clear_no_lock
```

在最后的objc_clear_deallocating函数中，从weak表中找到弱引用指针的地址，然后置为nil,并从weak表删除记录。

关于“weak表“的操作(增加，删除，释放)都是在`objc_weak.h`和`objc_weak.mm`中进行操作的。

遗留问题：
1. 在函数`weak_resize`中，创建一个`new_entries`，会调用`calloc(new_size, sizeof(weak_entry_t))`(void *calloc(size_t n, size_t size)；功能： 在内存的动态存储区中分配n个长度为size的连续空间): 
创建是默认是64个长度为40（byte?）的连续空间，每次 num_entries >= (mask+1)*3/4，就会进行“扩容” —— (mask+1)*2。

我的问题是通过对referent一定的位运算后，转换成了某个数字(index)，根据这个结果在`weak_table_t->weak_entries`中相应的`weak_entry_t`，这是怎样的操作？weak_entries到底是个什么数据结构，`weak_entries(weak_entry_t *)`这样的数据结构怎么看都不像是是散列表:

```
static weak_entry_t *
weak_entry_for_referent(weak_table_t *weak_table, objc_object *referent)
{
    assert(referent);

    weak_entry_t *weak_entries = weak_table->weak_entries;

    if (!weak_entries) return nil;

    size_t begin = hash_pointer(referent) & weak_table->mask;
    size_t index = begin;
    size_t hash_displacement = 0;
    while (weak_table->weak_entries[index].referent != referent) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_table->weak_entries);
        hash_displacement++;
        if (hash_displacement > weak_table->max_hash_displacement) {
            return nil;
        }
    }
    
    return &weak_table->weak_entries[index];
}
```

下面是[细看objc-weak源码](https://wangwangok.github.io/2018/05/18/source_code_objc_weak_t/)这篇文章的的截取：

为`weak_entries`计算散列值的两个函数：

```
static inline uintptr_t hash_pointer(objc_object *key);
static inline uintptr_t w_hash_pointer(objc_object **key);
```

它们会根据对象的指针（不管是指针还是指向指针的指针）调用一个fast-hash函数来生成一个key，其原理是基于fast_hash，而这个key的作用目前我们无从得知。

我打印了一下`hash_pointer`生成的index好想都不应该都不会超过`weak_table_t->mask`，不然就会“越界”了？

很好看的这些文章：

写的很详细[细看objc-weak源码](https://wangwangok.github.io/2018/05/18/source_code_objc_weak_t/)

[iOS管理对象内存的数据结构以及操作算法--SideTables、RefcountMap、weak_table_t-一](https://www.jianshu.com/p/ef6d9bf8fe59)

[weak 弱引用的实现方式](https://github.com/Desgard/iOS-Source-Probe/blob/master/Objective-C/Runtime/weak%20%E5%BC%B1%E5%BC%95%E7%94%A8%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F.md)

[Objective-C runtime机制(7)——SideTables, SideTable, weak_table, weak_entry_t](https://blog.csdn.net/u013378438/article/details/82790332)

