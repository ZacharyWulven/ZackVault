---
layout: post
title: ### -2022-02-25-Objective-C 分类.markdown
date: 2022-02-25 14:50:30.000000000 +09:00
tag: OC 底层原理 
---

编译阶段：分类编译完后变成 category_t 结构，每一个分类都是是一个 _category_t（分类的数据结构）。
编译完并没有合并到本类中，运行中通过 runtime 动态将分类的方法合并到类对象、元类对象中。

{% highlight ruby %}
   struct _category_t {
     const char *name;      类名字符串
     struct _class_t *cls;  类名
     const struct _method_list_t *instance_methods; 实例方法列表
     const struct _method_list_t *class_methods;    类方法列表
     const struct _protocol_list_t *protocols;      协议方法列表
     const struct _prop_list_t *properties;         属性列表
   };
   
   struct _class_t {
     struct _class_t *isa;
     struct _class_t *superclass;
     void *cache;
     void *vtable;
     struct _class_ro_t *ro;
   };
{% endhighlight %}

### objc 源码解读
* objc-os.mm
* _objc_init
* map_images
* map_images_nolock
* objc-runtime-new.mm
* _read_images
* remethodizeClass
* attachCategories
* attachLists
* realloc、memmove、memcpy

#### 源码看做了哪些事   
1. 通过 runtime 加载某个类的所有分类数据，把所有分类方法、属性、协议数据合并到一个大数组中
2. 后参与编译的分类数据会在数组前边，将合并的分类数据（方法、属性、协议）插入到类原来数据前边，最后编译的分类放在前边，因为源码 while 循环 i--。
3. 将合并后的分类数据，插入到类原来的数据的前面   
4. 获取分类方法数组 后编译先放前边， 即同名方法后编译的分类先调用
{% highlight ruby %}
如何调整编译顺序，在工程的 build phase 里拖拽，之后查看分类顺序即可。
控制编译顺序就是 Build Phases 里的实现文件顺序。
{% endhighlight %}

#### 源码核心方法
* remethodizeClass 重新组织方法
* remethodizeClass(class)
* remethodizeClass(meta-class)

#### 核心方法解读   
1. 将分类信息附加到类里 category list= [category_t(分类1), category_t(分类2)]
2. attachCategories(cls, category list)
3. 类对象里边的数据 copy to rw

{% highlight ruby %}
   class_rw_t = class.bits & FAST_DATA_MASK
   
   rw = cls->data()
   
   rw->methods.attachLists(methodlist, methodlistCount)       //所以分类对象方法附加到类对象方法列表中
   
   rw->properties.attachLists(propertiesList, propertiesList) // 将所有分类属性附加到类对象方法列表中
  
   rw->protocols.attachLists(protolists, protolistsCount)     // 将所有分类协议方法附加到类对象方法列表中
{% endhighlight %}


#### 细节 attachLists方法 挪动
{% highlight ruby %}
   根据 newMethodCount + oldMethodCount扩容
   把 oldMethodList移动到最后
   例：若有两个分类 分别用 [category_t(3)]、[category_t(4)]表示
           [ro(1)]
   =>扩容   [ro(1)][][]
   =>移动   [][][本类(1)]
   =>add   [category_t(3)][category_t(4)][ro(1)]
   把 ro 的方法列表移动到最后
   所以分类方法会方法本类方法前边，这就造成了分类覆盖本类方法 ,但不是真覆盖，因为这相当于找到了方法就不往后找了。
   如果两个分类有相同方法 最终调用哪个看编译顺序
{% endhighlight %}


#### memmove 和 memcpy 区别
{% highlight ruby %}
如想 [3][4][1][2] 变成 [][3][4][2]，memcpy 会 one-by-one 的 copy，直接覆盖
[3][4][1][2] => [3][3][1][2] => [3][4][1][2] 显然不是想要的

memmove 会根据地址大小决定往左还是右挪，能保证数据的完整性
[3][4][1][2] => [3][][4][2] => [][3][4][2]
{% endhighlight %}

  
{% highlight ruby %}
Category 和 class extension 区别？
类延展在编译时候就合并到类里了，分类是运行时候合并到类里   
{% endhighlight %}


### +load 方法
runtime 加载类和分类时候，就会调用 load 方法
{% highlight ruby %}
每个类、分类的 +load 方法，在程序运行过程中只调用一次。
{% endhighlight %}

#### 类，分类调用顺序
{% highlight ruby %}
1 先调用本类的 +load 方法，按照编译先后顺序调用（先编译，先调用），
调用子类 load 前会先调用父类的 load （有个递归传入 superclass）
schedule_class_load(cls->getSuperclass()); // Ensure superclass-first ordering

2 再调用分类的 +load 方法，按照编译先后顺序调用（先编译，先调用）
{% endhighlight %}

#### 本质
{% highlight ruby %}
load 的本质是拿到函数地址直接调用。
{% endhighlight %}


### +initialize 
initialize 方法在类第一次接收到消息时调用，调用走的是 msgSend 发送消息机制。Objc_msgSend 里会判断是否第一次给这个类发消息，如果是第一次会调用 initialize，Objc_msgSend 底层是汇编实现的，消息机制就用 isa 指针找方法。initialize 最终调是通过 objc_msgSend 实现，保证父类先初始化。

#### 步骤
1. Objc_msgSend 里会判断是否第一次给这个类发消息，如果是第一次会调用 initialize。
2. 最终 lookUpImpOrForward 里会判断这个类是否已经 initialize，如果没有初始化就调用 _class_initialize 初始化，最终 _class_initialize 里调用
3. 看父类初始化没有，如果没有，调用父类 initialize
4. msg_send（initialize）

{% highlight ruby %}
   // 伪代码
   if (自己没有初始化) {
      if (父类没有初始化) {
        objc_msgSend(父类, @selector(initialize))
      }
      objc_msgSend(自己, @selector(initialize))
   }
   
class_getInstanceMethod 找到对象方法，class_getClassMethod 找到类方法
{% endhighlight %}

   
#### 调用顺序
1. 先调用父类的 initialize（如果父类没有初始化），再调用子类的 initialize，先初始化父类再初始化子类，每个类只初始化一次
2. 父类 initialize 有可能被调用多次
   

### +initialize 与 +load 方法区别
1. 调用方式不同：initialize 通过 objc_msgSend 进行调用，load 通过函数地址直接调用
* 如果分类实现了 initialize 就覆盖了本类的 initialize
* 如果子类没有实现 initialize，会调用父类的 initialize (父类的 initialize可能被调用多次)
* 如果父类没调用过 initialize，会先调用父类的 initialize

2. 调用时机不同：load 是 runtime 加载类分类调用只会调用一次，initialize 是类第一次接收到消息时候调用

3. 调用顺序
* load 调用顺序，先调用类的 load，先编译的类优先调用，调用子类 load 前会先调用父类的 load，再调用分类的 load，先编译的先调用
* initialize 调用顺序，先初始化父类，再初始化子类（可能最终调用父类的initialize）
   
