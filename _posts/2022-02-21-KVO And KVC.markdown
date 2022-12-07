---
layout: post
title: ### -2022-02-21KVO&KVC.markdown
date: 2022-02-21 14:50:30.000000000 +09:00
tag: OC 底层原理 
---

### KVO
KVO 使用了 isa-Swizzling 技术，将对象的 isa 指向新的子类 NSKVONotifying_[类名]。
{% highlight ruby %}
   person.isa = NSKVONotifying_Person   
   [person->isa superclass] = Person
{% endhighlight %}

当修改 instance 对象的属性时，会调用 Foundation 的 _NSSetXXXValueAndNotify 函数
{% highlight ruby %}
   willChangeValueForKey
   父类原来调用
   didChangeValueForKey 内部触发 observeValueForKeyPath:ofObject:change:context
{% endhighlight %}

子类重写了 setAge、dealloc、class、_isKVOA 方法，setAge 的实现会调用 Foundation 里的 _NSSetIntValueAndNotify()。
{% highlight ruby %}
   - (void)setAge:(int)age
   {
       _NSSetIntValueAndNotify(); // 如果是 double 类型 就是 _NSSetDoubleValueAndNotify
   }
   
   伪代码
   void _NSSetIntValueAndNotify()
   {
       [self willChangeValueForKey:@"age"];
       [super setAge:age];
       [self didChangeValueForKey:@"age"];
   }
   
   - (void)didChangeValueForKey:(NSString *)key
   {
       [oberser observeValueForKeyPath:key ofObject:self change:nil context:nil]; // 通知监听器，某某属性值发生了改变
   }
   {% endhighlight %}
Note：didChangeValueForKey 里边判断了是否调用过 willChangeValueForKey，如果没调用过 willChangeValueForKey，直接 didChangeValueForKey 手动 KVO 不会生效。

KVO 子类重写了 class 方法，重写目的就是屏蔽内部实现
{% highlight ruby %}
   object_getClass(self.person) 才是真正的类
   object_getClass(self.person) => NSKVONotifying_Person
   [self.person class] => Person
{% endhighlight %}

### KVC
KVC 可以通过 key 访问属性，setValue:forKey: 和 setValue:forKeyPath:。KVC修改成员变量底层逻辑：
{% highlight ruby %}
   [self willChangeValueForKey:@"age"];
   obj->key = 10
   [self didChangeValueForKey:@"age"];
{% endhighlight %}

#### setValue:forKey: 原理
1. 找 setKey: 和 _setKey: 方法如果有就直接调用了
2. 如果没有的话会看 + (BOOL)accessInstanceVariablesDirectly 方法返回值， 默认返回 YES，返回 NO 不允许访问成员变量，调用 setValue:forUndefinedKey: 抛出异常
3. 如果返回 YES 严格按照 _key, _iskey, key, isKey 顺序找成员变量，如果找到直接赋值
4. _key, _iskey, key, isKey 都找不到，调用 setValue:forUndefinedKey: 抛出异常

{% highlight ruby %}
以 age 属性为例，若 setValue:ForKey 不是 @"age", 是 @"_age" @"_isAge" @"isAge"，
有对应成员变量不会 crash，会赋值成功但不会触发 KVO。
{% endhighlight %}

#### value:forKey: 原理
1. 按照 getKey、key、isKey、_key 查找方法
2. 如果没有的话会看 + (BOOL)accessInstanceVariablesDirectly 方法返回值， 默认返回 YES，返回 NO 不允许访问成员变量，调用 valueForUndefinedKey: 抛出异常
3. 返回 YES 严格按照 _key, _iskey, key, isKey 顺序找成员变量，如果能找到直接取值
4. _key, _iskey, key, isKey 都找不到，调用 valueForUndefinedKey: 抛出异常
