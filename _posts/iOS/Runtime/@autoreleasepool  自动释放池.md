---
layout: default
title: @autoreleasepool  自动释放池
---

#@autoreleasepool  自动释放池

## 引言
在主程序运行时，会看到以下的代码：

    int main(int argc, char * argv[]) {
	    @autoreleasepool {
	        return UIApplicationMain(argc, argv, @"XUIApplication", NSStringFromClass([AppDelegate class]));
		}
    }
那么@autoreleasepool 究竟做了什么呢？

## 自动释放池的主要工作
在`MRC`时代，如果不知道一个对象什么时候释放，可以在初始化时加上autorelease，如：

    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
	NSString* str = [[[NSString alloc] initWithString:@"tutuge"] autorelease];
	//use str...
	[pool release];
也就是说，在创建时，可以给对象发送自动释放的消息，当`NSAutoreleasePool`结束时，标记过“`autorelease`”的对象就会被`release`。

在`ARC`时代，我们不用手动的发送`autoRelease`消息，`ARC`会自动的帮我们加上这些，而这时候，`@autoreleasepool`做的事情，就和`NSAutoreleasePool`一样，自动的释放池中的对象。


## `@autoreleasepool`原理

| AutoreleasePoolPage  |
| :-------- :|
| magic_t const magic;    | 
| id *next;    | 
| pthread_t const thread;    | 
| AutoreleasePoolPage * const parent;    | 
| AutoreleasePoolPage * child;    | 
| uint32_t const depth;    | 
| uint32_t hiwat;    | 


在ARC下，我们使用`@autoreleasepool{}`来创建自动释放池，随后编译器将其改写成：

    oid *context = objc_autoreleasePoolPush();
	// {}中的代码
	objc_autoreleasePoolPop(context);
这两个类都是对`AutoreleasePoolPage`的简单封装。所以自动释放机制的核心在这个类。
- `AutoreleasePoolPage`没有单独的结构，室友若干个`AutoreleasePoolPage`以`双向链表`的形式组合而成的。
- `AutoreleasePool`是按线程一一对应的。
- `AutoreleasePoolPage`每个对象会开辟4096字节内存（虚拟内存一页的大小）。除了上面的实力变量所占空间，剩下的空间全部用来储存`autorelease`对象的地址。
- `AutoReleasePoolPage`存有栈顶的下一个`autorelease`对象的下一个位置。
- 当一个`AutoReleasePoolPage`被沾满时，会新建一个对象，连接链表，后面的`autorelease`对象在新的page中加入。新`page`的`next`指针被初始化在栈底，然后继续向栈顶添加新对象。
所以，向一个对象发送`- autorelease`消息，就是将这个对象加入到当前的`AutoReleasePoolPage`的栈顶`next`指针指向的位置。

#### 调用释放池`push`
每当执行一次`objc_autoreleasePoolPush`调用时，`runtime`向当前的`AutoreleasePoolPage`中add进一个`哨兵对象`，值为0。
`objc_autoreleasePoolPush`的返回值正是这个哨兵对象的地址，被`objc_autoreleasePoolPop`(哨兵对象)作为入参，于是：
1. 根据传入的哨兵对象地址找到哨兵对象所处的`page`
2. 在当前`page`中，将晚于哨兵对象插入的所有`autorelease`对象都发送一次`- release`消息，并向回移动next指针到正确位置。
3. 补充2：从最新加入的对象一直向前清理，可以向前跨越若干个`page`，直到哨兵所在的`page`。


> **以下内容由于不理解，暂时先粘贴在此处**

##Autorelease返回值的快速释放机制

值得一提的是，ARC下，runtime有一套对autorelease返回值的优化策略。
比如一个工厂方法：

```
+ (instancetype)createSark {
    return [self new];
}
// caller
Sark *sark = [Sark createSark];
```

秉着谁创建谁释放的原则，返回值需要是一个autorelease对象才能配合调用方正确管理内存，于是乎编译器改写成了形如下面的代码：

```
+ (instancetype)createSark {
    id tmp = [self new];
    return objc_autoreleaseReturnValue(tmp); // 代替我们调用autorelease
}
// caller
id tmp = objc_retainAutoreleasedReturnValue([Sark createSark]) // 代替我们调用retain
Sark *sark = tmp;
objc_storeStrong(&sark, nil); // 相当于代替我们调用了release
```

一切看上去都很好，不过既然编译器知道了这么多信息，干嘛还要劳烦autorelease这个开销不小的机制呢？于是乎，runtime使用了一些黑魔法将这个问题解决了。

#### 黑魔法之Thread Local Storage

Thread Local Storage（TLS）线程局部存储，目的很简单，将一块内存作为某个线程专有的存储，以key-value的形式进行读写，比如在非arm架构下，使用pthread提供的方法实现：

```
void* pthread_getspecific(pthread_key_t);
int pthread_setspecific(pthread_key_t , const void *);
```

说它是黑魔法可能被懂pthread的笑话- -

在返回值身上调用`objc_autoreleaseReturnValue`方法时，`runtime`将这个返回值object储存在TLS中，然后直接返回这个`object`（不调用`autorelease`）；同时，在外部接收这个返回值的objc_retainAutoreleasedReturnValue里，发现TLS中正好存了这个对象，那么直接返回这个object（不调用retain）。
于是乎，调用方和被调方利用TLS做中转，很有默契的免去了对返回值的内存管理。

于是问题又来了，假如被调方和主调方只有一边是ARC环境编译的该咋办？（比如我们在ARC环境下用了非ARC编译的第三方库，或者反之）
只能动用更高级的黑魔法。

#### 黑魔法之__builtin_return_address

这个内建函数原型是

```
char *__builtin_return_address(int level)
```

作用是得到函数的返回地址，参数表示层数，如__builtin_return_address(0)表示当前函数体返回地址，传1是调用这个函数的外层函数的返回值地址，以此类推。

```
- (int)foo {
    NSLog(@"%p", __builtin_return_address(0)); // 根据这个地址能找到下面ret的地址
    return 1;
}
// caller
int ret = [sark foo];
```

看上去也没啥厉害的，不过要知道，函数的返回值地址，也就对应着调用者结束这次调用的地址（或者相差某个固定的偏移量，根据编译器决定）
也就是说，被调用的函数也有翻身做地主的机会了，可以反过来对主调方干点坏事。
回到上面的问题，如果一个函数返回前知道调用方是ARC还是非ARC，就有机会对于不同情况做不同的处理

#### 黑魔法之反查汇编指令

通过上面的`__builtin_return_address`加某些偏移量，被调方可以定位到主调方在返回值后面的汇编指令：

```
// caller
int ret = [sark foo];
// 内存中接下来的汇编指令（x86，我不懂汇编，瞎写的）
movq ??? ???
callq ???
```

而这些汇编指令在内存中的值是固定的，比如`movq`对应着`0x48`。
于是乎，就有了下面的这个函数，入参是调用方`__builtin_return_address`传入值

```
static bool callerAcceptsFastAutorelease(const void * const ra0) {
    const uint8_t *ra1 = (const uint8_t *)ra0;
    const uint16_t *ra2;
    const uint32_t *ra4 = (const uint32_t *)ra1;
    const void **sym;
    // 48 89 c7    movq  %rax,%rdi
    // e8          callq symbol
    if (*ra4 != 0xe8c78948) {
        return false;
    }
    ra1 += (long)*(const int32_t *)(ra1 + 4) + 8l;
    ra2 = (const uint16_t *)ra1;
    // ff 25       jmpq *symbol@DYLDMAGIC(%rip)
    if (*ra2 != 0x25ff) {
        return false;
    }
    ra1 += 6l + (long)*(const int32_t *)(ra1 + 2);
    sym = (const void **)ra1;
    if (*sym != objc_retainAutoreleasedReturnValue)
    {
        return false;
    }
    return true;
}
```

它检验了主调方在返回值之后是否紧接着调用了`objc_retainAutoreleasedReturnValue`，如果是，就知道了外部是`ARC`环境，反之就走没被优化的老逻辑。

# 其他`Autorelease`相关知识点

使用容器的block版本的枚举器时，内部会自动添加一个AutoreleasePool：

```
[array enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
    // 这里被一个局部@autoreleasepool包围着
}];
```

当然，在普通`for`循环和`for in`循环中没有，所以，还是新版的`block`版本枚举器更加方便。for循环中遍历产生大量`autorelease`变量时，就需要手加局部`AutoreleasePool`咯。


> **看不懂的部分完**


## 添加`@autoreleasepool`的场景

1. 在基于命令行的程序，即没有UI框架，比如`AppKit`等`Cocoa`框架。
2. 在写循环时，循环中包含了大量的临时创建的对象。
3. 创建了新的线程
4. 长时间在后台运行的任务



















> 参考
> [`@autoreleasepool`的Apple文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html)


