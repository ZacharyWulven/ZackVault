---
layout: post
title: ### 2022-02-11-AutoreleasePool.markdown
date: 2022-02-11 15:50:30.000000000 +09:00
tag: OC 底层原理 
---

### Autorelease
自动释放池主要底层数据结构：
* __AtAutoreleasePool
* AutoreleasePoolPage

__AtAutoreleasePool 调用 push、pop。push、pop 内通过 AutoreleasePoolPage 管理，调用了 autorelease 对象最终都通过 AutoreleasePoolPage 来管理。

{% highlight ruby %}
结构体定义，可以认为 C++ 结构体就是一个类
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {    // 构造函数，创建结构体变量时候调用
      atautoreleasepoolobj = objc_autoreleasePoolPush();
  }
  
  ~__AtAutoreleasePool() {   // 析构函数
      objc_autoreleasePoolPop(atautoreleasepoolobj);
      
  }
  void * atautoreleasepoolobj;
};

@autoreleasepool 代码

/* @autoreleasepool */ 
{ 
    __AtAutoreleasePool __autoreleasepool; 

    appDelegateClassName = NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class")));

    Person *person = ((Person *(*)(id, SEL))(void *)objc_msgSend)((id)((Person *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("Person"), sel_registerName("alloc")), sel_registerName("init"));
}

@autoreleasepool 本质
{
  atautoreleasepoolobj = objc_autoreleasePoolPush()
 
  Person *person = ((Person *(*)(id, SEL))(void *)objc_msgSend)((id)((Person *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("Person"), sel_registerName("alloc")), sel_registerName("init"));
  
  objc_autoreleasePoolPop(atautoreleasepoolobj)
}

{% endhighlight %}

{% highlight ruby %}
class AutoreleasePoolPage : private AutoreleasePoolPageData {}

struct AutoreleasePoolPageData       
{                                       // 主要成员
  magic_t const magic;
  __unsafe_unretained id *next;         // 指向下一个存放调用了 autorelease 对象地址的地址，刚开始就是 begin()
  pthread_t const thread;
  AutoreleasePoolPage * const parent;   // 指向上一个 AutoreleasePoolPage 的地址
  AutoreleasePoolPage *child;           // 指向下一个 AutoreleasePoolPage 的地址
  uint32_t const depth;
  uint32_t hiwat;
};

{% endhighlight %}

### 流程
* 每个 AutoreleasePoolPage 对象占 4096 字节，除存放它内部的成员变量，剩下空间用来存放 autorelease 对象地址
* 所以 AutoreleasePoolPage 对象是通过双向链表形式连接在一起的
* AutoreleasePoolPage 通过 begin 函数获取存 autorelease 对象的起始地址（this+成员 size）
* AutoreleasePoolPage 通过 end 函数获取存 autorelease 对象的结束地址 （this+4096）


假设 AutoreleasePoolPage 起始地址是 0x1000，结束时 0x2000（end() 函数），所有成员 size 结束地址是 0x1038（begin() 函数），begin() 到 end() 之间的空间用来存储调用了 autorelease 方法的对象的地址。


objc_autoreleasePoolPush()
{% highlight ruby %}
static inline void *push() 
{
    id *dest;
    if (slowpath(DebugPoolAllocation)) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_BOUNDARY);         // POOL_BOUNDARY = nil
    } else {
        dest = autoreleaseFast(POOL_BOUNDARY);
    }
    ASSERT(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}
{% endhighlight %}
objc_autoreleasePoolPush 将 POOL_BOUNDARY 放入 0x1038 地址，即 begin() 函数地址，如果有调用 autorelease 方法的对象，就被依次放到 POOL_BOUNDARY 后边。

当 @autorelease 大括号结束时调用 objc_autoreleasePoolPop()，会从最后一个调用 autorelease 方法的对象开始调用 release，然后往回找，一直找到 POOL_BOUNDARY 的地址停止，上例也就是 0x1038。当然如果有嵌套逻辑会有多个 POOL_BOUNDARY。

#### 嵌套
{% highlight ruby %}
int main(int argc, char * argv[]) {
    @autoreleasepool {       // r1 = push()
      
      Person *person1 = [[Person alloc] init];
      
      @autoreleasepool {     // r2 = push()
        
        Person *person2 = [[Person alloc] init];

        Person *person3 = [[Person alloc] init];
        
        @autoreleasepool {   // r3 = push()
          
          Person *person4 = [[Person alloc] init];
          
        }                    // pop(r3)
        
      }                      // pop(r2)
      
    }                        // pop(r1)
    return 0;
}
{% endhighlight %}

AutoreleasePoolPage 内存展示

{% highlight ruby %}

magic_t const magic;
__unsafe_unretained id *next;  
pthread_t const thread;
AutoreleasePoolPage * const parent;   
AutoreleasePoolPage *child;           
uint32_t const depth;
uint32_t hiwat;
POOL_BOUNDARY          // 0x1038，r1 = push()
person1
POOL_BOUNDARY
person2
person3
POOL_BOUNDARY         // r3 = push()
person4               // pop(r3) 从这里开始直到遇到 POOL_BOUNDARY 停止

{% endhighlight %}

可通过调用私有 API 查看 autoreleasePool 情况，需要声明后调用。
{% highlight ruby %}
extern void _objc_autoreleasePoolPrint(void); 
{% endhighlight %}

PAGE  (hot) 是正在使用的 page，(cold) 不是当前在使用的。
{% highlight ruby %}
objc[59905]: ##############
objc[59905]: AUTORELEASE POOLS for thread 0x105bdce00
objc[59905]: 525 releases pending.
objc[59905]: [0x7ff03c809000]  ................  PAGE (full)  (cold)
objc[59905]: [0x7ff03c809038]  ################  POOL 0x7ff03c809038   // POOL_BOUNDARY
objc[59905]: [0x7ff03c809040]    0x6000017c8000  Person
objc[59905]: [0x7ff03c809048]  ################  POOL 0x7ff03c809048   // POOL_BOUNDARY
objc[59905]: [0x7ff03c809050]    0x6000017c8010  Person
objc[59905]: [0x7ff03c809058]    0x6000017c8020  Person
.
.
.
objc[59905]: [0x7ff03c809fe8]    0x6000017c9f40  Person
objc[59905]: [0x7ff03c809ff0]    0x6000017c9f50  Person
objc[59905]: [0x7ff03c809ff8]    0x6000017c9f60  Person
objc[59905]: [0x7ff03d00e000]  ................  PAGE  (hot)        ## //当前正在使用的 page
objc[59905]: [0x7ff03d00e038]    0x6000017c9f70  Person
objc[59905]: [0x7ff03d00e040]    0x6000017c9f80  Person
objc[59905]: [0x7ff03d00e048]    0x6000017c9f90  Person
objc[59905]: [0x7ff03d00e090]    0x6000017ca020  Person
objc[59905]: [0x7ff03d00e098]    0x6000017ca030  Person
objc[59905]: [0x7ff03d00e0a0]    0x6000017ca040  Person
objc[59905]: [0x7ff03d00e0a8]    0x6000017ca050  Person
objc[59905]: [0x7ff03d00e0b0]    0x6000017ca060  Person
objc[59905]: [0x7ff03d00e0b8]    0x6000017ca070  Person
objc[59905]: [0x7ff03d00e0c0]    0x6000017ca080  Person
objc[59905]: [0x7ff03d00e0c8]  ################  POOL 0x7ff03d00e0c8
{% endhighlight %}

### RunLoop 和 Autorelease
iOS 在主线程的 RunLoop 中注册了 2 个 Observer，回调都是 _wrapRunLoopWithAutoreleasePoolHandler。
* 一个 Observer 监听进入 Runloop 时候（kCFRunLoopEntry），调用 autoreleasePoolPush
* 另一个 Observer 监听休眠前和退出时候（），休眠前会调用一次 autoreleasePollPop 再 autoreleasePoolPush，退出时候调用 autoreleasePollPop


所以如果被 @autorelease{} ，在 } 结束后也就是调用 pop 后。其他由 RunLoop 控制释放的，所处的那一次 RunLoop 休眠前释放。autorelease 对象不是被 main 函数的 autorelease 管理的。
