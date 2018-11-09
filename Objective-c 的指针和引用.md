# Objective-c 的指针和引用

本篇是[objective-c指针解引用](https://www.jianshu.com/p/1dc7c31fa06f)这个文章的阅读笔记，相似度达80%
所以大家可以从引用的文章，一步一步debug理解这篇文章。

## 一切起源于C语言指针变量解引用

C语言中**指针变量**的值存储的是一个地址，这个地址指向一个其他变量(可以是基本类型，结构体，甚至其他指针变量)，当想要操作这个地址指向的变量是，需要对**指针变量**解引用，可以认为取地址`&`和解引用`*`是一组相对操作，可以恍惚转化：

C语言基本类型的指针操作：

```
int a = 1;
int *pInt = &a; //指针变量pInt指向了int变量啊的地址(使用`&`符号“取”地址)
int b = *pInt; //效果等同于 b = a; *pInt取得的是pInt指针指向的变量
```

C语言结构题的指针操作：

```
typedef struct Person {
	int age;
} Person;

Person person;
Person *p2person = &person; // `p2person`指向结构题`person`的指针
p2person->age = 18; //p2person指向的结构体变量的age
(*p2person).age = 101; //结构体（*p2person）的age
Person person2 =  (*p2person); //效果等同于 person2 = person
```

> Objective-C中的类最终都会转化位一个结构体(类对象`objc_class`)而存在，
> 类的实例对象其实是一个指向该结构题的指针(instancePointer->objc_object.isa->objc_class)
> id类型就是一个结构体指针：`typedef objc_object *id`,
> 所以对于实例对象我们可以像C语言(指针)一样操作，所以我们可以用使用`instance->`语法和`(*instance)`语法，这两种使用起来是等效的
> 所谓的Objective-C对象其实是个指针变量

## iOS开发中变量**取地址**和**指针解引用**的使用场景

* 值类型的引用传参

如果想在方法内部改变外部传递进来的基本数据类型变量我们可以像下面一样通过地址引用传参，其实传递的是一个指针参数，使用的时候通过指针解引用可以获取外面定义的变量：

```
- (void)changeInteger:(NSInteger *)reference {
	*reference = 100; // 解引用后进行赋值
	return YES;
}

- (void)testReference {
	NSInteger valueForInt;
	if ([self changeInteger:&valueForInt]) {
		NSLog(@"值变成了%zi", valueForInt);
	}

	/*
	NSInteger valueForInt;
	NSInteger *p2Reference = &valueForInt;
	if ([self changeInteger:p2Reference]) {
		NSLog(@"值变成了%zi", valueForInt);
	}
	*/
}
```

```
// 系统数组遍历方法中也包含一个BOOL类型的引用传参，因为block的调用者想知道block执行后的这个stop值，所以使用引用传参
    [@[] enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        *stop = YES;
    }];
```

* 对象类型的引用参数

OC的对象是一个结构体指针对象，所以如果从C语言的用法来说我们可以传递这个指针对象，然后在方法内部通过指针解引用，改变这个结构体的值，但是实际上是不可行的，我们需要遵循OC对象的生成方式，经过`alloc` `init`等方法才能创建一个正确的对象：

```
- (void)changeObject:(Person **)obj {
    // obj是一个指针变量(指向另一个指针变量 = Person的实例对象)
    // *obj将obj解引用后将得到这个指针指向的一个指针变量 = 外部的Person的实例对象
    *obj = ({
        Person *o = [Person new];
        o->studentNo = 22;
//        o->age = 18;
        o.name = @"张艾文";
        o;
    });
    
    // 这里的ob改变后只是将指针指向了另外一个对象并不能改变外部的Person的实例对象
    __autoreleasing Person *o1 = ({
        Person *o = [Person new];
        o->studentNo = 101;
        o.name = @"王老板";
        o;
    });;
    obj = &o1;
}

- (void)testRefernce {
    NSObject *o;
    [self changeObject:&o];// BreakePointer 1
    NSLog(@"调用方法后的变量o:%@", o); // BreakePointer 2
}
```

我在`BreakePointer 1`的Debug输出是：

```
(lldb) p o
(NSObject *) $0 = nil
(lldb) p &o
(NSObject **) $1 = 0x00007ffeec3be738
(lldb) p *$0
error: Couldn't apply expression side effects : Couldn't dematerialize a result variable: couldn't read its memory
(lldb) p &$0
(NSObject **) $3 = 0x00007ffeec3be738
```

我在`BreakePointer 1`的Debug输出是：

```
(lldb) p o
(Person *) $5 = 0x0000604000257100
(lldb) p *$5
(NSObject) $6 = {
  isa = Person
}
(lldb) p $5->studentNo
error: 'NSObject' does not have a member named 'studentNo'
(lldb) p ((Person *)$5)->studentNo
(int) $7 = 22
(lldb) p ((Person *)$5).name
(__NSCFConstantString *) $8 = 0x00000001251f4d88 @"张艾文"
(lldb) p &o
(NSObject **) $9 = 0x00007ffeec3be738
```

嗯，到这里我想说一下我自己的理解：

(在`testRefernce`)`&o`指向一块内存的地址，`o`里面存放的是一个对象的地址，通过操作`&o`可以改变`o`只想的对象：

```
&0(0x00007ffeec3be738)->[0x0000604000257100]
								|-> 指向一个实例对象(`NSObject *` 或 `Person *`)，或者可以说是实例变量在内存中开始的地方？
								    可以通过这个地址和偏移量来访问实例对变量的值。
```

所以，`o`可以看作是一个结构体(`objc_object *`)的指针，需要访问这个结构体成员变量时，只需要对这个指针解引用。
`*o`可以看作是一个结构体(`objc_object`)，可以访问他的成员，但不能对它进行赋值？

> 总结总结：
> 因为OC方法的形参变量都是对实参变量进行了拷贝（拷贝基本类型变量的值，或者拷贝指针变量存储的地址），如果方法内部想改变外部的变量那么只要传递外部变量的地址即可；

有个问题：`testRefernce`中关于形参，实参，传值方式的注释本身是没有问题的，但是对`- (void)testRefernce`方法执行的debug后

```
- (void)testRefernce {
    NSObject *o;  // o = nil; &o = 0x00007ffee82ee738
    [self changeObject:&o];
    /* 在方法中的debug

      - (void)changeObject:(Person **)obj {
       // obj = (Person **) $2 = 0x00007ffee82ee730
    	*obj = ({
        	Person *o = [Person new];

        	// o = (Person *) $3 = 0x000060000044b370
        	// &o = (Person **) $4 = 0x00007ffee82ee6f0

        	o->studentNo = 22;
        	o.name = @"张艾文";
        	o;
    	});

    	// obj = (Person **) $5 = 0x00007ffee82ee730

    	__autoreleasing Person *o1 = ({
        	Person *o = [Person new];

        	// o = (Person *) $6 = 0x000060000044a8c0
        	// &o = (Person **) $7 = 0x00007ffee82ee6d8
        	
        	o->studentNo = 101;
        	o.name = @"王老板";
        	o;
    	});

        // o1 = (Person *) $8 = 0x000060000044a8c0
        // &o = (Person **) $9 = 0x00007ffee82ee6e0
    	
    	obj = &o1;

    	// obj = (Person **) $10 = 0x00007ffee82ee6e0
	}
     */
    NSLog(@"调用方法后的变量o:%@", o); // BreakePointer 2

    // o = (Person *) $11 = 0x000060000044b370
    // &o = (NSObject **) $12 = 0x00007ffee82ee738
}
```

发现原本方法中的*obj和方法外的形参&o理论上应该是相等，然而运行deubg并不相等，这个问题的原因归结为编译器的优化。这个优化是基于下面2个事实：

1. OC的内存管理的三大原则其中一条是“谁生成的对象谁负责释放”，那么method2内部生成的对象就应该method2负责释放，但是这个对象生成本来就是给方法外部使用的，所以不能在方法作用域结束的时候直接释放该对象，而是要延迟释放，没错，将这个对象加入到自动释放池中即可；这点系统编译器已经给我们做了。编译器会将方法转换成如下：

```
// 方法内部给obj指向的对象赋值以后会添加到自动释放池中
- (void)changeObject:(Person **)obj {
```

2. `__strong`修饰的`testRefernce`方法局部变量`o`，编译器会在方法结束之前调用`release`方法释放该局部变量，这种前提下`changeObject:`内部生成的对象(引用计数1)已经由自动释放池来管理了，如果`o`在出作用域之前调用了`release`(引用计数变为0), 那么在自动释放池清理调用该对象的release方法导致过度释放出现崩溃； 所以系统编译器自动为我们做了转换，由于编译器是不会改变我们声明的对象`o`的内存管理方式，所以会再生成了一个临时对象，最后在方法调用完以后又将临时对象`temp`赋给`o`，这样就给我们造成一直使用的都是`o`的假象，真是完美的伪装。

```
- (void)testRefernce {
	__strong NSObject *o; // nil
	__autoreleasing NSObject *temp = o;  // nil，这句话会将temp指向的对象加入到自动释放池，因为此时指向对象为nil，系统不会将这个nil添加到自动释放池
	[self changeObject:&temp]; // 调用后temp指向对象引用计数为1
	o = temp; // 因为o是__strong修饰的，所以此时o指向对象引用计数变为2 ， 最后当o出作用域时会调用一次release, 然后自动释放池会调用一次release
}
```

问题：如果我们`testRefernce`测试方法中声明的对象`NSObject *o`前面加上`__autoreleasing`会怎样？

```
__autoreleasing  NSObject *o; 
```

结果就是编译器不会再为我们添加temp的临时变量了，这样我们方法调用前的&o和方法内部的obj是相等的；这种就避免了编译器的优化，但编译器优化同时会带来额外的代价(至少多了2行代码)，所以如果你想更好的话，就在变量声明的时候显示指定用__autoreleasing来修饰。


引用[objective-c指针解引用](https://www.jianshu.com/p/1dc7c31fa06f)

关于堆栈的相关文站: [栈（Stack）和堆（Heap）](https://wangwangok.github.io/2017/03/21/stack_heap_with_c/)

