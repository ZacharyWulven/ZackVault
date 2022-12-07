---
layout: post
title: Runtime(二)
date: 2022-01-31 11:00:30.000000000 +09:00
categories: [iOS, 底层原理, Runtime]
tags: [ObjC, 底层原理, Runtime]
---

### class_rw_t

class_rw_t 里的 methods、properties、protocols 是二维数组，可读写，包含类的初始内容和分类内容

{% highlight ruby %}
struct objc_class : objc_object {
objc_class(const objc_class&) = delete;
objc_class(objc_class&&) = delete;
void operator=(const objc_class&) = delete;
void operator=(objc_class&&) = delete;
// Class ISA;
Class superclass;
cache_t cache; // formerly cache pointer and vtable
class_data_bits_t bits; // class_rw_t \* plus custom rr/alloc flags
}

class_rw_t = bits & FAST_DATA_MASK
class_rw_t* data() const {
return (class_rw_t *)(bits & FAST_DATA_MASK);
}
{% endhighlight %}

{% highlight ruby %}
struct class_rw_t {
class_ro_t
method_array_t methods // 二维数组 method_array_t = [method_list_t, method_list_t]; method_list_t = [method_t, method_t]
property_array_t properties // 二维数组 property_array_t = [property_list_t, property_list_t]; property_list_t = [property_t]
protocol_array_t protocols // 二维数组 protocol_array_t = [protocol_list_t, protocol_list_t]; protocol_list_t = [protocol_ref_t]
}
{% endhighlight %}

### class_ro_t

class_ro_t 里边 baseMethodList、baseProtocols、ivars、baseProperties 是一维数组，只读的，存储类的初始方法、属性、成员等。一开始 class_data_bits_t bits 指向 class_ro_t，之后 runtime 把 class_ro_t 赋值给 class_rw_t->ro, 最后 class_data_bits_t bits 的 data 指向 class_rw_t。

{% highlight ruby %}
struct class*ro_t {
void \_baseMethodList;
protocol_list_t * baseProtocols;
const ivar*list_t * ivars;
const uint8*t * weakIvarLayout;
property_list_t \*baseProperties;
}

auto ro = (const class_ro_t \*)cls->data(); // 最初 cls data 就是 ro
auto isMeta = ro->flags & RO_META;
if (ro->flags & RO_FUTURE) {
// This was a future class. rw data is already allocated.
rw = cls->data();
ro = cls->data()->ro();
ASSERT(!isMeta);
cls->changeInfo(RW_REALIZED|RW_REALIZING, RW_FUTURE);
} else {
// Normal class. Allocate writeable class data.
rw = objc::zalloc<class_rw_t>();  
 rw->set_ro(ro); // ro 赋值给 rw
rw->flags = RW_REALIZED|RW_REALIZING|isMeta;
cls->setData(rw); // cls data 指向 rw
}
{% endhighlight %}

## method_t

method_t 是对方法/函数的封装

{% highlight ruby %}
struct method_t {
SEL name(); // 函数名，选择器
const char \*types(); // 编码（返回值类型，参数类型）
IMP imp(bool needsLock); // 指向函数的指针（函数地址），函数的具体实现
}

typedef id \_Nullable (_IMP)(id \_Nonnull, SEL \_Nonnull, ...);
SEL 类似 char _，可以通过 @selector() 或 sel_registerName() 获取
Note: 不同类中的相同名字方法的 SEL 是一样的
{% endhighlight %}

Type Encoding 类型编码
iOS 提供一个叫 @encode 的指令，可以将具体的类型表示成字符串编码

Types 结构

返回值|参数 1|参数 2|...|参数 n

{% highlight ruby %}

例如 v16@0:8
v void, 返回值
@ 第一个参数，id 类型
0 从第 0 个字节开始
: 第二个参数，是选择器 SEL 类型，从第 8 个字节开始
16 表示所有参数的字节数，不包括返回值

NSLog(@"%s", @encode(id)); // 打印 @

{% endhighlight %}

### 方法缓存

class 内有个 cache_t(方法缓存)，用散列表存储曾经调用过的方法，可提高方法查找速度。

{% highlight ruby %}

struct objc_class : objc_object {
union {
struct {
explicit_atomic<mask_t> \_maybeMask; // 散列表长度 - 1，\_maybeMask is the buckets mask
uint16_t \_occupied;
};
};
struct bucket_t \*buckets() const; // 散列表
mask_t occupied() const; // 已缓存的方法数量
}
散列表长度 >= 已缓存的方法数量

struct bucket_t {
private:
// IMP-first is better for arm64e ptrauth and no worse for arm64.
// SEL-first is better for armv7\* and i386 and x86_64.
#if **arm64**
explicit_atomic<uintptr_t> \_imp; // 函数地址
explicit_atomic<SEL> \_sel; // SEL 作为 key，以前类型是 cache_key_t
#else
explicit_atomic<SEL> \_sel;
explicit_atomic<uintptr_t> \_imp;
#endif
}
{% endhighlight %}

原理是哈希表，bucket_t \* 里元素 index = @selector(test) & \_maybeMask。& 出来的值一定小于等于 \_maybeMask。

{% highlight ruby %}
static inline mask_t cache_hash(SEL sel, mask_t mask)
{
uintptr_t value = (uintptr_t)sel;
#if CONFIG_USE_PREOPT_CACHES
value ^= value >> 7;
#endif
return (mask_t)(value & mask); // SEL & MASK
}
{% endhighlight %}

若产生了碰撞 💥，会调用 cache_next 函数重新生成 index

{% highlight ruby %}
do {
if (fastpath(b[i].sel() == 0)) {
incrementOccupied();
b[i].set<Atomic, Encoded>(b, sel, imp, cls());
return;
}
if (b[i].sel() == sel) {
// The entry was added to the cache by some other thread
// before we grabbed the cacheUpdateLock.
return;
}
} while (fastpath((i = cache_next(i, m)) != begin));

#elif **arm64**
static inline mask_t cache_next(mask_t i, mask_t mask) {
return i ? i-1 : mask;
}
{% endhighlight %}

若散列表容量不够会扩容，然后重新缓存，因为 mask 变了，也就是长度变了
{% highlight ruby %}
capacity = capacity ? capacity \* 2 : INIT_CACHE_SIZE;
if (capacity > MAX_CACHE_SIZE) {
capacity = MAX_CACHE_SIZE;
}
reallocate(oldCapacity, capacity, true);
{% endhighlight %}

方法查找时先找类对象从类对象的方法缓存(cache_t)里找，没有在去 class_rw_t 里找，如果方法在父类找到，这个方法也会缓存到自己类对象的 cache_t 中。

### Tips

{% highlight ruby %}
哈希表核心原理，即 f(key) = index。也可以用 % 代替 &，但 & 效率更高。
{% endhighlight %}

### 参考链接

{% highlight ruby %}
https://developer.apple.com/documentation/foundation/nsmethodsignature
https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100
{% endhighlight %}
