---
layout: post
title: Runtime(五)
date: 2022-02-03 15:00:30.000000000 +09:00
categories: [iOS, 底层原理, Runtime]
tags: [ObjC, 底层原理, Runtime]
---

### 窥探底层代码几种方式

{% highlight ruby %}
1 转为 cpp 代码
2 Debug Disassembly
3 生成汇编代码 Xcode Produc -> Perform Action -> Assemble "xxx.m"，:行号 进行代码查找
{% endhighlight %}

OC 在变为机器代码之前，会被 LLVM 编译器转换为中间代码（Intermediate Representation）。中间代码是跨平台的。中间代码语法参考网站 https://llvm.org/docs/LangRef.html

可以使用以下命令生成中间代码
{% highlight ruby %}
clang -emit-llvm -S main.m // 生成 .ll 文件
{% endhighlight %}

### Runtime API

{% highlight ruby %}
// 获取 isa 指向的 Class
objc_getClass()

// 设置 isa 指向的 Class
object_setClass(person, [Student class]);

// 判断一个 O C 对象是否是 Class
object_isClass(person)

// 动态创建一个新的类 (参数:父类， 类名， 额外的内存空间)
// 创建一个继承自 NSObject 的类
Class newClass = objc_allocateClassPair([NSObject class], "Dog", 0);

// 添加成员变量，参数 类、成员名、占字节数、对齐、encode
// 已经注册完的类不能添加成员变量
class_addIvar(newClass, "\_age", 4, 1, @encode(int));
class_addIvar(newClass, "\_weight", 4, 1, @encode(int));

// 添加方法
class_addMethod(newClass, @selector(run), (IMP)run, "v@:");

// 注册一个类(要再注册之前添加成员变量),
// 类一旦注册完 类对象就创建好了 class_addIvar 不能放在 objc_registerClassPair 后
objc_registerClassPair(newClass);

id dog = [[newClass alloc] init];

[dog setValue:@10 forKey:@"_age"];
[dog setValue:@20 forKey:@"_weight"];

NSLog(@"dog age=%@, weight=%@", [dog valueForKey:@"_age"], [dog valueForKey:@"_weight"]);

[dog run];

object_setClass(person, newClass);
[person run];

// 获取成员变量信息
Ivar weightIvar = class_getInstanceVariable([Person class], "\_weight");
NSLog(@"weightIvar name=%s %s", ivar_getName(weightIvar), ivar_getTypeEncoding(weightIvar));

Ivar nameIvar = class_getInstanceVariable([Person class], "\_name");
object_setIvar(person, nameIvar, @"jerry");
NSLog(@"person name = %@", person.name);

object_setIvar(person, ageIvar, (\_\_bridge id)(void \*)10);
NSLog(@"person age = %d", person->\_age);

unsigned int count;
// 获取成员变量列表
Ivar \*ivars = class_copyIvarList([Person class], &count);
for (int i = 0; i < count; i++) {
Ivar ivar = ivars[i];
NSLog(@"ivars name=%s TypeEncoding=%s", ivar_getName(ivar), ivar_getTypeEncoding(ivar));
}
//copy 出来的要释放
free(ivars);

// 应用修改私有属性的值
self.textField.placeholder = @"请输入用户名";
UILabel \*label = [self.textField valueForKeyPath:@"_placeholderLabel"];
label.textColor = [UIColor redColor];

//替换方法
class_replaceMethod([Person class], @selector(run), (IMP)run, "v@:");

class_replaceMethod([Person class], @selector(run), imp_implementationWithBlock(^{
NSLog(@"hook run to block");
}), "v");
[person run];

//exchange 方法
Method walkMethod = class_getInstanceMethod([Person class], @selector(walk));
Method seatMethod = class_getInstanceMethod([Person class], @selector(seat));

method_exchangeImplementations(walkMethod, seatMethod);
[person walk];

// 销毁动态创建对象
// objc_disposeClassPair(newClass);

{% endhighlight %}

### 方法交换，交换的是 Method 里的 IMP

button 可以 hook UIControl 的 sendAction:to:forEvent: 方法。  
{% highlight ruby %}
// 第一个参数要注意类型是否是对的，类簇如 NSArray 的真实类型是其他 \_\_NSArrayM
Method method1 = class_getInstanceMethod(self, @selector(sendAction:to:forEvent:));
Method method2 = class_getInstanceMethod(self, @selector(xxx_sendAction:to:forEvent:));

method_exchangeImplementations(method1, method2);

objc 源码
void method_exchangeImplementations(Method m1, Method m2)
{
if (!m1 || !m2) return;

    mutex_locker_t lock(runtimeLock);

    IMP imp1 = m1->imp(false);
    IMP imp2 = m2->imp(false);
    SEL sel1 = m1->name();
    SEL sel2 = m2->name();

    m1->setImp(imp2);
    m2->setImp(imp1);


    // RR/AWZ updates are slow because class is unknown
    // Cache updates are slow because class is unknown
    // fixme build list of classes whose Methods are known externally?

    flushCaches(nil, __func__, [sel1, sel2, imp1, imp2](Class c){
        return c->cache.shouldFlush(sel1, imp1) || c->cache.shouldFlush(sel2, imp2);
    });

    adjustCustomFlagsForMethodChange(nil, m1);
    adjustCustomFlagsForMethodChange(nil, m2);

}
{% endhighlight %}

### 总结 Runtime

OC 是一门动态性比较强的编程语言，允许很多操作推迟到程序运行时再运行。OC 的动态性是由 runtime 来支持实现的，runtime 是一套 C 语言的 API。

应用
{% highlight ruby %}
1 关联对象
2 遍历类的所有成员变量
2.1 字典转模型
2.2 应用修改私有属性的值
2.3 归档 解档
3 交换方法实现
4 2 遍历类的所有方法
。。。
{% endhighlight %}
