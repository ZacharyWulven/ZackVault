---
layout: post
title: Runtime(一)
date: 2022-01-30 11:00:30.000000000 +09:00
categories: [iOS, 底层原理, Runtime]
tags: [ObjC, 底层原理, Runtime]
---

### 简介

OC 的动态性是由 Runtime 支撑的，Runtime 提供的 API 基本都是 C 语言的，源码由 C/C++/汇编编写。

### 共用体（union）与位运算的应用

{% highlight ruby %}

& 可以用来取出特定的位，想取哪一位就让哪一位是 1，其他位是 0
| 可以让某一位为 1

#define PS5Mask (1<<0)
#define XBOXMask (1<<1)
#define MACMask (1<<2)

@interface Person () {
union {
char bits; // 0b 0000 0111
struct { // struct 为了可读性，表示 bits 的结构，不影响存储
char PS5 : 1; // 只占一位
char XBOX : 1;
char MAC : 1;

        };
    } _device;

}
@end

@implementation Person

- (void)setPS5:(BOOL)PS5 {
  if (PS5) {
  \_device.bits |= PS5Mask;
  }
  else {
  \_device.bits &= ~PS5Mask;
  }
  }

- (void)setXBOX:(BOOL)XBOX {
  if (XBOX) {
  \_device.bits |= XBOXMask;
  }
  else {
  \_device.bits &= ~XBOXMask;
  }
  }

- (void)setMAC:(BOOL)MAC {
  if (MAC) {
  \_device.bits |= MACMask;
  }
  else {
  \_device.bits &= ~MACMask;
  }
  }

- (BOOL)isPS5 {
  return !!(\_device.bits & PS5Mask);
  }

- (BOOL)isXBOX {
  return !!(\_device.bits & XBOXMask);
  }

- (BOOL)isMAC {
  return !!(\_device.bits & MACMask);
  }

@end

测试
Person \*person = [[Person alloc] init];
[person setPS5:NO];
[person setMAC:YES];

(lldb) p/x &(person->\_device)
((anonymous union) \*) $0 = 0x0000600002ab8008
(lldb) x 0x0000600002ab8008
0x600002ab8008: 04 00 00 00 00 00 00 00 20 25 40 20 25 40 20 25 ........ %@ %@ %
04 即 0100 设置的 MAC 值
{% endhighlight %}

讨论
{% highlight ruby %}
如果 union struct 占 4 位则 MASK 也要变
#define PS5Mask (0b1111<<0) //0b0000 1111 (or 15<<0)
#define XBOXMask (0b1111<<4) //0b1111 0000 (or 15<<4)
#define MACMask (0b1111<<8)
{% endhighlight %}

### isa 详解

在 arm64 之前，isa 地址就是类或元类对象的地址， arm64 架构对 isa 进行了优化，变成了共用体（union）结构，使用位域来存储更多信息，将一个 64 位(8 字节)内存数据存了很多东西，其中用 33 位存储 class 的信息，需要 & MASK 才能得到类对象地址。isa 是一个 isa_t(union)。

{% highlight ruby %}
实例对象->isa & ISA_MASK = 类对象地址
类对象->isa & ISA_MASK = 元类对象地址
{% endhighlight %}

源码：本篇基于 objc4-818.2.tar.gz
{% highlight ruby %}
struct objc_object {
private:
isa_t isa;
}

// isa union 定义
union isa_t {
isa_t() { }
isa_t(uintptr_t value) : bits(value) { }

    uintptr_t bits;

private:
// Accessing the class requires custom ptrauth operations, so
// force clients to go through setClass/getClass by making this
// private.
Class cls;

public:
#if defined(ISA_BITFIELD)
struct {
ISA_BITFIELD; // defined in isa.h
};

    bool isDeallocating() {
        return extra_rc == 0 && has_sidetable_rc == 0;
    }
    void setDeallocating() {
        extra_rc = 0;
        has_sidetable_rc = 0;
    }

#endif
void setClass(Class cls, objc_object \*obj);
Class getClass(bool authenticated);
Class getDecodedClass(bool authenticated);
};
{% endhighlight %}

ISA_BITFIELD 在 arm64 定义

{% highlight ruby %}
#define ISA_MASK 0x0000000ffffffff8ULL 用来取中间 33 位类对象地址
#define ISA_MAGIC_MASK 0x000003f000000001ULL
#define ISA_MAGIC_VALUE 0x000001a000000001ULL
#define ISA_HAS_CXX_DTOR_BIT 1
#define ISA_BITFIELD  
uintptr_t nonpointer : 1; 为 0 代表是普通指针，存储 class、meta-class 地址；为 1 代表优化过 union 结构
uintptr_t has_assoc : 1; 是否曾经被设置过关联对象，如果没有，释放时候更快，即使被删除也还是 1  
uintptr_t has_cxx_dtor : 1; 是否有 C++ 的析构函数，如果没有，释放时候更快  
uintptr_t shiftcls : 33; 类对象、meta-class 地址值 /MACH_VM_MAX_ADDRESS 0x1000000000/
uintptr_t magic : 6; 用于在调试时分辨对象是否未完成初始化，值为 ISA_MAGIC_VALUE 表示已完成初始化  
uintptr_t weakly_referenced : 1; 是否曾经被弱引用指向过，如果没有，释放时更快  
uintptr_t unused : 1; 对象是否正在释放
uintptr_t has_sidetable_rc : 1; 若引用计数多大，导致 extra_rc 19 位不够存储，则使用 sidetable_rc 进行存储
uintptr_t extra_rc : 19 引用计数器，存放引用计数 - 1，刚 alloc 出来则 19 位都是 0
#define RC_ONE (1ULL<<45)
#define RC_HALF (1ULL<<18)
#endif
{% endhighlight %}

对象销毁源码
{% highlight ruby %}
void \*objc_destructInstance(id obj)
{
if (obj) {
Class isa = obj->getIsa();
// 处理 C++ 析构函数，对应 has_cxx_dtor
if (isa->hasCxxDtor()) {
object_cxxDestruct(obj);
}
// 处理关联对象，对应 has_assoc
if (isa->instancesHaveAssociatedObjects()) {
\_object_remove_assocations(obj);
}

        objc_clear_deallocating(obj);
    }
    return obj;

}
{% endhighlight %}

{% highlight ruby %}
类对象地址值最后三位永远是 0，类对象地址最后一个十六进制位要是 0 要么是 8
{% endhighlight %}

### objc 源码地址

{% highlight ruby %}
https://opensource.apple.com/tarballs/objc4/
{% endhighlight %}
