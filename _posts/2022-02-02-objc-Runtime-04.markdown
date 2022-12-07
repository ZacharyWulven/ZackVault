---
layout: post
title: Runtime(四)
date: 2022-02-02 11:00:30.000000000 +09:00
categories: [iOS, 底层原理, Runtime]
tags: [ObjC, 底层原理, Runtime]
---

### [super message]

{% highlight ruby %}
@interface Student: Person

- (instanceType)init {
  NSLog(@"%@", [self class]); // Student
  NSLog(@"%@", [self superclass]); // Person
  NSLog(@"%@", [super class]); // Student
  NSLog(@"%@", [super superclass]); // Person
  }   
  @end

生成 cpp 文件
NSLog((NSString _)&**NSConstantStringImpl**var_folders_70_5wpd8v2s40g1g0jxm0x_13hc0000gn_T_Student_361c0e_mi_0, ((Class (_)(id, SEL))(void \*)objc_msgSend)((id)self, sel_registerName("class")));

NSLog((NSString _)&**NSConstantStringImpl**var_folders_70_5wpd8v2s40g1g0jxm0x_13hc0000gn_T_Student_361c0e_mi_1, ((Class (_)(id, SEL))(void \*)objc_msgSend)((id)self, sel_registerName("superclass")));

NSLog((NSString _)&**NSConstantStringImpl**var_folders_70_5wpd8v2s40g1g0jxm0x_13hc0000gn_T_Student_361c0e_mi_2, ((Class (_)(**rw*objc_super *, SEL))(void \_)objc_msgSendSuper)((**rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("Student"))}, sel_registerName("class")));

NSLog((NSString _)&**NSConstantStringImpl**var_folders_70_5wpd8v2s40g1g0jxm0x_13hc0000gn_T_Student_361c0e_mi_3, ((Class (_)(**rw*objc_super *, SEL))(void \_)objc_msgSendSuper)((**rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("Student"))}, sel_registerName("superclass")));

// 以上 cpp 代码可以看出，调用的消息接收者 self = Student

runtime super 定义
struct objc_super {
/// Specifies an instance of a class.
  **unsafe_unretained \_Nonnull id receiver; // 消息接收者
  **unsafe_unretained \_Nonnull Class super_class; // 消息接收者父类，表示从父类开始查找方法实现
};

// super 调用的 receiver 仍然是 Student，从 Student 的父类找 class 方法
[super class]
receiver = self(Student)
struct objc_super arg = {self, class_getSuperclass}
objc_msgSendSuper(self, @selector(class))

// class 源码

- (Class)class {
  return self;
  }

* (Class)class {
  return object_getClass(self);
  }

// superclass 源码

- (Class)superclass {
  return self->getSuperclass();
  }

* (Class)superclass {
  return [self class]->getSuperclass();
  }
  {% endhighlight %}

因为 super 调用的消息接收者 receiver = Student，只不过是从父类开始查找方法。而 class 实现是 object_getClass 取决于 self 也就是 receiver 是谁。所以 [super class] 返回 Student。

结论：

1. 消息接收者依然是 self，上例也就是 Student。

2. objc_msgSendSuper 时因为传入了 super 表示从父类开始查找方法实现。

### isMemberOfClass & isKindOfClass

{% highlight ruby %}
NSLog(@"%d", [[NSObject class] isKindOfClass:[NSObject class]]); // 1
NSLog(@"%d", [[NSObject class] isMemberOfClass:[NSObject class]]); // 0
NSLog(@"%d", [[Person class] isKindOfClass:[Person class]]); // 0
NSLog(@"%d", [[Person class] isMemberOfClass:[Person class]]); // 0
NSLog(@"%d", [Person isKindOfClass:[NSObject class]]); // 1 元类的父类是 NSObject

- (BOOL)isMemberOfClass:(Class)cls {
  return self->ISA() == cls;
  }

* (BOOL)isMemberOfClass:(Class)cls {
  return [self class] == cls;
  }

- (BOOL)isKindOfClass:(Class)cls {
  for (Class tcls = self->ISA(); tcls; tcls = tcls->getSuperclass()) {
  if (tcls == cls) return YES;
  }
  return NO;
  }

* (BOOL)isKindOfClass:(Class)cls {
  for (Class tcls = [self class]; tcls; tcls = tcls->getSuperclass()) {
  if (tcls == cls) return YES;
  }
  return NO;
  }
  {% endhighlight %}

可以理解为 self->ISA() = [self class]，这样 - + 方法就一致了

### 面试题:下面 viewDidLoad 代码能 build 过吗 ?能运行吗 ？如能运行结果是什么？

{% highlight ruby %}
@interface Person : NSObject

- (void)print;

@end

@implementation Person

- (void)print {
  NSLog(@"my name is %@", self.name);

}

@end

- (void)viewDidLoad {
  [super viewDidLoad];

  id cls = [Person class];

  void \*obj = &cls;
  // 在内存中这个结构跟普通对象结构是一样的，所以能调用成功，就是找前 8 个字节就是 isa
  [(__bridge id)obj print]; // obj->cls(isa)->[Person class]

  // Person \*person = [[Person alloc] init];
  // person->isa->[Person class]

}
{% endhighlight %}

先看下函数调用时栈空间分配

{% highlight ruby %}

void test() {
long long a = 4; // 0x7ffee4a53a78
long long b = 5; // 0x7ffee4a53a70
long long c = 6; // 0x7ffee4a53a68
long long d = 7; // 0x7ffee4a53a60

NSLog(@"%p, %p, %p, %p", &a, &b, &c, &d);
}

局部变量分配在栈空间，栈空间分配，从高地址到低地址。
{% endhighlight %}

如果加一行 NSString *test = @"134"; 代码
{% highlight ruby %}
NSString *test = @"134";  
id cls = [Person class];
void \*obj = &cls;
[(__bridge id)obj print]; // obj->cls(isa)->[Person class]

test、cls、obj 都在栈空间，test 地址最大，此时打印 my name is 134。
{% endhighlight %}

内存指向关系
{% highlight ruby %}
obj -> cls // 栈低地址
cls -> [Person class]
test -> @"134" // 栈高地址
{% endhighlight %}

其实 self 就是 obj，此时获取 name 内容相当于找到 obj->isa (obj->cls) 之后的 8 个字节的内存
{% highlight ruby %}

- (void)print {
  NSLog(@"my name is %@"，self->name); // self 就是 obj
  }
  // Person 实现
  struct Person_IMPL {
  Class isa;
  NSString \*\_name; // 相当于找到 isa 之后的 8 个字节的内存
  }
  {% endhighlight %}

结论
{% highlight ruby %}
栈空间分配内存，从高地址到低地址。所以 self->name 就是找比 cls 地址高的 8 个字节。
{% endhighlight %}

现在回到原题
{% highlight ruby %}

- (void)viewDidLoad {
  // struct objc_super arg = {self, UIViewController.class} arg 就是隐含的局部变量
  // [super viewDidLoad] 等于 objc_msgSendSuper({arg, sel_registerName("viewDidLoad")})
  [super viewDidLoad];

      id cls = [Person class];

      void *obj = &cls;
      [(__bridge id)obj print];

  }

obj -> cls // 栈低地址
cls -> [Person class]
{self, UIViewController.class} -> arg // 栈高地址

所以最后打印 my name is <ViewController: 0x6000035e9580>
{% endhighlight %}

涉及到知识点

1. super 的本质

2. 函数栈空间分配

3. 消息机制，找 isa

4. 访问成员变量本质，找到对应的成员那块内存

5. 结构体如何访问成员

### objc_msgSendSuper 本质

Debug 汇编：Xcode Debug->Debug Workflow->Always Show Disassembly

通过汇编可以知道 objc_msgSendSuper 最终会调用 objc_msgSendSuper2。
{% highlight ruby %}
0x102d6d03f <+31>: movq %rax, -0x18(%rbp)
0x102d6d043 <+35>: movq 0x5bb6(%rip), %rsi ; "viewDidLoad"
0x102d6d04a <+42>: leaq -0x20(%rbp), %rdi
0x102d6d04e <+46>: callq 0x102d6d2d6 ; symbol stub for: objc_msgSendSuper2
0x102d6d053 <+51>: movq 0x5bce(%rip), %rdi ; (void \*)0x0000000102d72cb8: Person
{% endhighlight %}

{% highlight ruby %}
struct objc_super arg = {self, ViewController.class}  
// objc_msgSendSuper2 内部会 ViewController->superclass 所以和 objc_msgSendSuper 效果一样
objc_msgSendSuper2({arg, sel_registerName("viewDidLoad")})

汇编代码
ENTRY \_objc_msgSendSuper2
UNWIND \_objc_msgSendSuper2, NoFrame

#if \_\_has_feature(ptrauth_calls)
ldp x0, x17, [x0] // x0 = real receiver, x17 = class
add x17, x17, #SUPERCLASS // x17 = &class->superclass 这里获取 superclass
ldr x16, [x17] // x16 = class->superclass
AuthISASuper x16, x17, ISA_SIGNING_DISCRIMINATOR_CLASS_SUPERCLASS
LMsgSendSuperResume:
#else
ldp p0, p16, [x0] // p0 = real receiver, p16 = class
ldr p16, [x16, #SUPERCLASS] // p16 = class->superclass
#endif

LLDB 查看内存

(lldb) p/x obj
(Person \*) $0 = 0x00007ffeef2a8aa8
(lldb) x 0x00007ffeef2a8aa8
0x7ffeef2a8aa8: b8 ec 95 00 01 00 00 00 00 0a ae 01 00 60 00 00 .............`..
0x7ffeef2a8ab8: 40 ec 95 00 01 00 00 00 a0 e7 c2 7b ff 7f 00 00 @..........{....
(lldb) x/4g 0x00007ffeef2a8aa8
0x7ffeef2a8aa8: 0x000000010095ecb8 0x0000600001ae0a00
0x7ffeef2a8ab8: 0x000000010095ec40 0x00007fff7bc2e7a0
(lldb) p (Class)0x000000010095ecb8 // cls
(Class) $1 = Person
(lldb) p (Class)0x0000600001ae0a00 // viewController 对象
(Class) $2 = 0x0000600001ae0a00
(lldb) p (Class)0x000000010095ec40 // viewController class
(Class) $3 = ViewController
(lldb)
{% endhighlight %}

小结：

1. 生成的 cpp 代码不一定是最终的运行代码，但大概率是对的。

2. 在 arm64 架构，函数参数一般是放寄存器，因为效率更高。
