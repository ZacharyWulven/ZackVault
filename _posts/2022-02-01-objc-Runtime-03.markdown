---
layout: post
title: Runtime(三)
date: 2022-02-01 13:00:30.000000000 +09:00
categories: [iOS, 底层原理, Runtime]
tags: [ObjC, 底层原理, Runtime]
---

### objc_msgSend

OC 中方法调用，其实都是转换为 objc_msgSend(receiver,@selector) 函数调用，即消息机制。objc_msgSend 是通过汇编实现。

### objc_msgSend 源码

{% highlight ruby %}
ENTRY \_objc_msgSend
UNWIND \_objc_msgSend, NoFrame
// p0 是寄存器，存消息接收者
cmp p0, #0 // nil check and tagged pointer check
#if SUPPORT_TAGGED_POINTERS
b.le LNilOrTagged //(MSB tagged pointer looks negative) // le 即<=, if p0<=0 { jump to LNilOrTagged }
#else
b.eq LReturnZero
#endif
ldr p13, [x0] // p13 = isa
GetClassFromIsa_p16 p13, 1, x0 // p16 = class
LGetIsaDone:
// calls imp or objc_msgSend_uncached
CacheLookup NORMAL, \_objc_msgSend, \_\_objc_msgSend_uncached

#if SUPPORT_TAGGED_POINTERS
LNilOrTagged:
b.eq LReturnZero // nil check
GetTaggedClass
b LGetIsaDone
// SUPPORT_TAGGED_POINTERS
#endif

LReturnZero:
// x0 is already zero
mov x1, #0
movi d0, #0
movi d1, #0
movi d2, #0
movi d3, #0
ret

END_ENTRY \_objc_msgSend
{% endhighlight %}

### objc_msgSend 分为 3 个阶段

#### 1. 消息发送

针对实例对象方法伪代码，类方法 find in meta-class，能找到最终会加入 receiver 类的 cache_t。
{% highlight ruby %}
objc_msgSend(receiver,@selector) {
if !receiver { objc_msgSend 会看消息接收者是否为 nil，如果为 nil 直接 return
return;
}
bool found = look up in receiverClass cache_t // CacheLookup 查找方法缓存
if found {
// 调用方法
return;
}
found = look up in receiverClass class_rw_t // 查找 receiverClass class_rw_t
// 在 class_rw_t 已排序的是二分查找，否则遍历
if found {
// 调用方法，并缓存方法在 receiverClass 的 cache_t
return;
}
superclass = receiverClass->superclass
while superclass {
found = look up in superclass cache_t // CacheLookup 查找 superclass 方法缓存
if found {
// 调用方法，并缓存方法在 receiverClass 的 cache_t
return;
}
found = look up in superclass's class_rw_t // 查找 superclass class_rw_t
if found {
// 调用方法，并缓存方法在 receiverClass 的 cache_t
return;
}  
 superclass = superclass->superclass
}

// superclass == nil, not found，进入动态解析阶段
}
{% endhighlight %}

#### 2. 动态方法解析

允许动态创建方法，若没动态解析过，triedResolver = NO，会进行动态解析 \_class_resolveMethod。调用

1. resolveInstanceMethod
2. resolveClassMethod

然后标记为 triedResolver=YES，再回到 1. 消息发送阶段，重新从 receiverClass 的 cache 中查找方法，走完消息发送流程，若还没找到方法则进入消息转发阶段。

{% highlight ruby %}
// No implementation found. Try method resolver once.
if ((behavior & LOOKUP_RESOLVER) && !triedResolver) {
methodListLock.unlock();
\_class_resolveMethod(cls, sel, inst);
triedResolver = YES;
goto retry;
}

void \_play(id self, SEL \_cmd) {
NSLog(@"\_play - %@ - %@", self, NSStringFromSelector(\_cmd));
}

- (BOOL)resolveInstanceMethod:(SEL)sel {
  if (sel == @selector(play)) {
  // Method playMethod = class_getInstanceMethod(self, @selector(other));
  // class_addMethod(self, sel, method_getImplementation(playMethod), method_getTypeEncoding(playMethod));

        // 这里要传入 class 对象
        class_addMethod(self, sel, (IMP)_play, "V16@0:8");
        return YES; // return yes or no 无所谓，因为 objc 源码没有用到 return value，但最好还是按照官方要求返回。
      }
      return [super resolveInstanceMethod:sel];

  }

- (BOOL)resolveClassMethod:(SEL)sel {
  if (sel == @selector(personClassMethod)) {
  // 这里要传入 meta-class
  class_addMethod(object_getClass(self), sel, (IMP)c_other, "v16@0:8");
  return YES; // return yes or no 无所谓，因为 objc 源码没有用到 return value，但最好还是按照官方要求返回。
  }
  return [super resolveClassMethod:sel];
  }
  {% endhighlight %}

#### 消息转发

消息转发代码不开源，来到消息转发会调用 \_**\_ forwarding \_\_** 函数，步骤如下:

Step1. 调用 forwardingTargetForSelector 方法

1.1 若 forwardingTargetForSelector 有返回值，调用 objc_msgSend(返回值(ForwardingTarget), aSelector)，给 forwardingTarget 发消息。

1.2 若 forwardingTargetForSelector 返回值为 nil，进入 Step 2
{% highlight ruby %}
类方法对应转发

- (id)forwardingTargetForSelector:(SEL)aSelector {
  return [ForwardingTarget class]; // objc_msgSend([ForwardingTarget class], aSelector)
  return [[ForwardingTarget alloc] init]; // objc_msgSend([[ForwardingTarget alloc] init], aSelector)
  }

实例方法对应转发

- (id)forwardingTargetForSelector:(SEL)aSelector;
  {% endhighlight %}

Step2. 调用 (NSMethodSignature \*)methodSignatureForSelector:(SEL)aSelector，返回 NSMethodSignature 方法签名：返回值类型、参数类型。如果有方法签名会调用 forwardInvocation 方法，进入 Step 3。否则会调用 doesNotRecognizeSelector。最终如果消息转发都失败则会触发 unrecognized selector sent to instance。

{% highlight ruby %}
类方法签名

- (NSMethodSignature \*)methodSignatureForSelector:(SEL)aSelector {
  if (aSelector == @selector(hit:)) {
  return [NSMethodSignature signatureWithObjCTypes:"v@:i"];
  }
  return [super methodSignatureForSelector:aSelector];
  }
  实例方法签名

* (NSMethodSignature \*)methodSignatureForSelector:(SEL)aSelector;

方法签名决定了 Step3 里 NSInvocation 的返回值和参数
{% endhighlight %}

Step3. 调用 forwardInvocation NSInvocation 封装了一个方法调用。包括调用者，方法名、方法参数。
{% highlight ruby %}

类方法签名

- (void)forwardInvocation:(NSInvocation *)anInvocation {
  ForwardingTarget *target = [[ForwardingTarget alloc] init];
  int count;
  [anInvocation getArgument:&count atIndex:2];
  count = 20;
  [anInvocation setArgument:&count atIndex:2];
  [anInvocation invokeWithTarget:target];
  }

实例方法

- (void)forwardInvocation:(NSInvocation \*)anInvocation;

anInvocation.target 调用者
anInvocation。selector 方法名
// Note: index=0 is self, index=1 is SEL，so first param index is 2
[anInvocation getArgument:&count atIndex:2];

{% endhighlight %}

### 讨论

@dynamic 告诉编译器不要自动生成 setter getter，这时可以利用 resolveInstanceMethod 添加方法。

### Tips

{% highlight ruby %}
C 函数 test()
汇编会在前边加个下划线, \_test()
{% endhighlight %}

### 小结

1. 第一次调用一个方法会走消息发送两次，因为动态解析后会再走回消息发送。
2. forwardingTargetForSelector 返回值后会调用 objc_msgSend(ForwardingTarget, aSelector)，ForwardingTarget 可以是实例或类对象。
3. forwardingTargetForSelector、methodSignatureForSelector、forwardInvocation 都有 - 实例方法/+ 类方法。
4. 进入消息转发会直接调用 **_ forwarding _** 函数。然后调用 forwardingTargetForSelector 等方法。
