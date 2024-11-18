
# 前言


最近阅读[Aravis](https://github.com)源码，其中大量运用了GObject，于是打算学习一下。


此系列笔记仅主要面向初学者，不会很深入探讨源码的细节，专注于介绍GObject的基本用法。


此系列笔记参考[GObject Tutorial for beginners](https://github.com)


本文可在[个人博客](https://github.com)中阅读，体验更加


套个盾：文中定义的名词只是为了更好地理解GObject，不具备权威性。


# 类和实例


在GObject中，每个可实例化类类型都与两个结构体相关联：一个是类结构体，一个是实例结构体。


* 类结构体会被注册到类型系统中（具体注册方式在[下一节](https://github.com):[楚门加速器p](https://tianchuang88.com)讨论），在`g_object_new`首次调用时，类型系统会检查相应的类结构体是否已经被初始化为一个类变量，没有则创建并初始化。此后所有该类的实例变量都将共享这个已初始化的类变量。每个类变量只会被创建一次。
* 每次调用`g_object_new`时都会创建实例变量。


在GObject系统中，类结构体和实例结构体都会被实例化，在内存中占有特定的空间。为了便于描述，我们将分配给类结构体的实例称为“类变量”，而分配给实例结构体的实例称为“实例变量”。


GObject实例的结构体定义如下



```


|  | //file: gobject.h |
| --- | --- |
|  | typedef struct _GObject  GObject; |
|  | struct  _GObject |
|  | { |
|  | GTypeInstance  g_type_instance; |
|  |  |
|  | /*< private >*/ |
|  | guint          ref_count;  /* (atomic) */ |
|  | GData         *qdata; |
|  | }; |


```

GObject类的结构体定义如下（我们可以先不用了解结构的细节）：



```


|  | //file: gobject.h |
| --- | --- |
|  | typedef struct _GObjectClass             GObjectClass; |
|  | struct  _GObjectClass |
|  | { |
|  | GTypeClass   g_type_class; |
|  |  |
|  | /*< private >*/ |
|  | GSList      *construct_properties; |
|  |  |
|  | /*< public >*/ |
|  | /* seldom overridden */ |
|  | GObject*   (*constructor)     (GType                  type, |
|  | guint                  n_construct_properties, |
|  | GObjectConstructParam *construct_properties); |
|  | /* overridable methods */ |
|  | void       (*set_property)		(GObject        *object, |
|  | guint           property_id, |
|  | const GValue   *value, |
|  | GParamSpec     *pspec); |
|  | void       (*get_property)		(GObject        *object, |
|  | guint           property_id, |
|  | GValue         *value, |
|  | GParamSpec     *pspec); |
|  | void       (*dispose)			(GObject        *object); |
|  | void       (*finalize)		(GObject        *object); |
|  | /* seldom overridden */ |
|  | void       (*dispatch_properties_changed) (GObject      *object, |
|  | guint	   n_pspecs, |
|  | GParamSpec  **pspecs); |
|  | /* signals */ |
|  | void	     (*notify)			(GObject	*object, |
|  | GParamSpec	*pspec); |
|  |  |
|  | /* called when done constructing */ |
|  | void	     (*constructed)		(GObject	*object); |
|  |  |
|  | /*< private >*/ |
|  | gsize		flags; |
|  |  |
|  | gsize         n_construct_properties; |
|  |  |
|  | gpointer pspecs; |
|  | gsize n_pspecs; |
|  |  |
|  | /* padding */ |
|  | gpointer	pdummy[3]; |
|  | }; |


```

下面使用一个简单示例，来演示GObject的类和实例的使用



```


|  | //file: example01.c |
| --- | --- |
|  | #include |
|  |  |
|  | int main (int argc, char **argv) |
|  | { |
|  |  |
|  | GObject* instance1,* instance2;     //指向实例的指针 |
|  | GObjectClass* class1,* class2;      //指向类的指针 |
|  |  |
|  | instance1 = g_object_new (G_TYPE_OBJECT, NULL); |
|  | instance2 = g_object_new (G_TYPE_OBJECT, NULL); |
|  | g_print ("The address of instance1 is %p\n", instance1); |
|  | g_print ("The address of instance2 is %p\n", instance2); |
|  |  |
|  | class1 = G_OBJECT_GET_CLASS (instance1); |
|  | class2 = G_OBJECT_GET_CLASS (instance2); |
|  | g_print ("The address of the class of instance1 is %p\n", class1); |
|  | g_print ("The address of the class of instance2 is %p\n", class2); |
|  |  |
|  | g_object_unref (instance1); |
|  | g_object_unref (instance2); |
|  |  |
|  | return 0; |
|  | } |


```

其中：


* `g_object_new`函数创建实例变量并返回指向它的指针。在实例变量第一次被创建之前，它对应的类变量也会被创建并初始化。
* 参数`G_TYPE_OBJECT`是GObject基类的类型标识符，这是GObject类型系统的核心，所有其他GObject类型都从这个基类型派生。
* 宏`G_OBJECT_GET_CLASS`返回指向参数所属类变量的指针
* `g_object_unref`会销毁实例变量并释放内存。


输出：



```


|  | The address of instance1 is 0x55d3ddc05600 |
| --- | --- |
|  | The address of instance2 is 0x55d3ddc05620 |
|  | The address of the class of instance1 is 0x55d3ddc05370 |
|  | The address of the class of instance2 is 0x55d3ddc05370 |


```

可以发现，两个实例变量的地址不同，但两个实例变量对应的类变量的地址相同，因为两个实例变量共享一个类变量


# 引用计数


引用计数机制的概念在此不做介绍


在GObject中，GObject实例具有引用计数机制：



```


|  | //file: example02.c |
| --- | --- |
|  | #include |
|  |  |
|  | static void show_ref_count (GObject* instance) |
|  | { |
|  | if (G_IS_OBJECT (instance)) |
|  | /* Users should not use ref_count member in their program. */ |
|  | /* This is only for demonstration. */ |
|  | g_print ("Reference count is %d.\n", instance->ref_count); |
|  | else |
|  | g_print ("Instance is not GObject.\n"); |
|  | } |
|  |  |
|  | int main (int argc, char **argv) |
|  | { |
|  | GObject* instance; |
|  |  |
|  | instance = g_object_new (G_TYPE_OBJECT, NULL); |
|  | g_print ("Call g_object_new.\n"); |
|  | show_ref_count (instance); |
|  | g_object_ref (instance); |
|  | g_print ("Call g_object_ref.\n"); |
|  | show_ref_count (instance); |
|  | g_object_unref (instance); |
|  | g_print ("Call g_object_unref.\n"); |
|  | show_ref_count (instance); |
|  | g_object_unref (instance); |
|  | g_print ("Call g_object_unref.\n"); |
|  | g_print ("Now the reference count is zero and the instance is destroyed.\n"); |
|  | g_print ("The instance memories are possibly returned to the system.\n"); |
|  | g_print ("Therefore, the access to the same address may cause a segmentation error.\n"); |
|  |  |
|  | return 0; |
|  | } |


```

其中：


* `g_object_new`创建一个实例变量，然后将变量的引用计数置为1
* `g_object_ref`将其引用计数加1
* `g_object_unref`将引用计数减1，如果此时引用计数为0，则析构变量。


输出：



```


|  | Call g_object_new. |
| --- | --- |
|  | Reference count is 1. |
|  | Call g_object_ref. |
|  | Reference count is 2. |
|  | Call g_object_unref. |
|  | Reference count is 1. |
|  | Call g_object_unref. |
|  | Now the reference count is zero and the instance is destroyed. |
|  | The instance memories are possibly returned to the system. |
|  | Therefore, the access to the same address may cause a segmentation error. |


```

# 初始化和析构过程


GObject初始化和销毁的实际过程比较复杂。以下是简单的描述，不做详细说明.


## 初始化


1\.用类型系统注册GObject类型。这是在调用main函数之前的GLib的初始化过程中完成的。(如果编译器是gcc，则`__attribute__ ((constructor))`用于限定初始化函数。)
2\.为GObjectClass和GObject结构分配内存
3\.初始化GObjectClass结构内存。这个内存将是GObject的类变量。
4\.初始化GObject结构内存。这个内存将是GObject的实例变量。


上述初始化过程在第一次调用`g_object_new`函数时执行。在第二次及后续调用`g_object_new`时，它只执行两个过程：①为GObject结构分配内存②初始化内存。


## 析构


1\.销毁GObject实例。释放实例的内存


GObject变量类型是静态类型。静态类型永远不会破坏它的类。因此，即使被销毁的实例变量是最后一个，类变量仍然存在，直到程序终止。


# 参考文章


1\.[GObject Tutorial for beginners](https://github.com)


# 推荐


下一篇：[GObject学习笔记（二）类型创建与注册](https://github.com)


