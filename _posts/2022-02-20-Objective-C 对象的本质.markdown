---
layout: post
title: ### -2022-02-20-Objective-C 对象本质.markdown
date: 2022-02-20 14:50:30.000000000 +09:00
tag: OC 底层原理 
---

### OC 对象本质
OC 底层都是 C、C++，由 C、C++结构体实现，汇编代码受硬件影响




{% highlight ruby %}
结构体的地址就是它第一个属性的地址，所以 NSObject 第一个是 isa，NSObject 对象指针本质就是它 isa 的地址。
{% endhighlight %}

### 对象大小
class_getInstanceSize 返回成员变量大小，所以在 64bit 架构是 8 字节因为只有 isa 指针。而 NSObject 真正分配 16 字节，但真正利用的只有 8 个字节。
{% highlight ruby %}
    NSLog(@"%zu", malloc_size((__bridge const void *)(obj)));


    size_t instanceSize(size_t extraBytes) const {
        if (fastpath(cache.hasFastInstanceSize(extraBytes))) {
            return cache.fastInstanceSize(extraBytes);
        }
 
        size_t size = alignedInstanceSize() + extraBytes;
        
        if (size < 16) size = 16;   // CF requires all objects be at least 16 bytes.
        return size;
    }
{% endhighlight %}
instanceSize 函数分配内存大小，如果 size 小于 16，会默认设置为 16 字节，不论 32bit 还是 64bit 系统。

### 内存对齐
内存对齐：即使没有 oc 的 16 字节限制，结构体的大小必须是最大成员大小的倍数。分配内存对齐 iPhone OS 都是 16 的倍数。instance size 是最大成员大小的倍数，malloc size 是 16 倍数（苹果规定操作系统对内存有个桶 Buckets sized {16,32,48,64 ...} 最大 256）。最小分配内存是 16 是苹果自己的，内存对齐是通用的。
{% highlight ruby %}
class_getInstanceSize 至少需要多少内存
malloc_size 实际内存了分配
{% endhighlight %}

### 对象


#### Instance 对象
只有 isa 和成员变量
     

#### Class 对象
Class 类型放在 .data 段，想看存在哪个段可以打印内存看和那些变量类似，Class 的地址跟全局变量地址接近所以猜测放在 .data 段。
{% highlight ruby %}
每个类的类对象、元类对象在内存中有且只有一份。类，元类结构一样都是 Class 对象。
{% endhighlight %}

class 方法总是返回类对象
{% highlight ruby %}
+ (Class)class {
    return self;
}

- (Class)class {
    return object_getClass(self);
}
{% endhighlight %}

object_getClass(id obj)，直接返回 isa
* 传实例对象返回类对象
* 传类对象返回元类
* 传入元类返回基类（NSObject）的 meta-class

{% highlight ruby %}
Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}
{% endhighlight %}

Class 里有
* isa指针
* superclass 指针
* 类属性，类的对象方法
* 类协议方法
* 类的成员变量信息 ivar

meta-class 里有
* isa 指针，最终指向自己
* superclass 指针，最终指向 NSObject
* 类方法信息、协议类方法


 
NSObject 体系，非 root 元类 isa 都指向 Root 的元类，即 NSObject 的元类。NSObject 体系 Root 的元类的 super 指向 Root 也就是 NSObject。
    
{% highlight ruby %} 
class_isMetaClass(objcMetaClass) 判断是否为元类
objc_getClass(char *name) 通过类名字符串找到对应的类，返回类对象
{% endhighlight %}

### isa & superclass
* instance isa 指向 class
* class 的 isa 指向 meta-class
* class 的 superclass 指向父类的 class，如果没有父类指向 nil
* meta-class 的 superclass 指向父类的 meta-class
* 基类的 meta-classs 的 superclass 指向基类的 class
* meta-class 的 isa 指向基类的 meta-class


如果一个类方法 meta-class 都没有的话，会最终找到基类的类，若基类里有同名字实例方法会调用同名字的实例方法。

{% highlight ruby %} 
class 地址 = instance 的 isa 地址 & ISA_MASK
meta-class 地址 = class isa 地址 & ISA_MASK
{% endhighlight %}
