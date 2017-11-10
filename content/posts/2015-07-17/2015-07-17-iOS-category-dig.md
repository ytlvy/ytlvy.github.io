---
author: null
categories:
- IOS
date: '2015-07-17'
description: null
photos: null
tags:
- dig
- category
title: iOS category dig
---

[转自](http://blog.sunnyxx.com/2014/03/05/objc_category_secret/)

## objc category的秘密

### category的真面目

objc所有类和对象都是c结构体，category当然也一样，下面是runtime中category的结构：

```
struct _category_t {
    const char *name; // 1
    struct _class_t *cls; // 2
    const struct _method_list_t *instance_methods; // 3
    const struct _method_list_t *class_methods; // 4
    const struct _protocol_list_t *protocols; // 5
    const struct _prop_list_t *properties; // 6
};
```

1. `name` 注意，并不是`category`小括号里写的名字，而是类的名字
2. `cls` 要扩展的类对象，编译期间这个值是不会有的，在app被runtime加载时才会根据name对应到类对象
3. `instance_methods`这个category所有的-方法
4. `class_methods`这个category所有的+方法
5. `protocols`这个category实现的protocol，比较不常用在category里面实现协议，但是确实支持的
6. `properties`这个category所有的property，这也是category里面可以定义属性的原因，不过这个property不会@synthesize实例变量，一般有需求添加实例变量属性时会采用`objc_setAssociatedObject`和`objc_getAssociatedObject`方法绑定方法绑定，不过这种方法生成的与一个普通的实例变量完全是两码事。

### 编译器，你对category干了什么？

举个栗子看，定义下面一个类和它的category，实现忽略，保存为sark.h和sark.m

```
@interface Sark : NSObject
- (void)speak;
@end

@interface Sark (GayExtention)
- (void)burst;
@end
```

请出clang的重写命令：

```
$ clang -rewrite-objc sark.m
```

同级目录下会生成sark.cpp，这就是objc代码重写成c++(基本就是c)的实现。
打开生成的文件，发现茫茫多，排除include进来的header，自己的代码都在文件尾部了，看看上面的category被编译器搞成什么样子了：

```
static struct _category_t _OBJC_$_CATEGORY_Sark_$_GayExtention __attribute__ ((used, section ("__DATA,__objc_const"))) =
{
    "Sark",
    0, // &OBJC_CLASS_$_Sark,
    (const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_Sark_$_GayExtention,
    0,
    0,
    0,
};
```

先注意这个category的名字`_OBJC_$_CATEGORY_Sark_$_GayExtention`，这是一个按规则生成的符号了，中间的`Sark`是类名，后面的`GayExtention`是类别的名字，这也就是为什么同一个类的`category`名不能冲突了
对应看上面`_category_t`的定义，因为category里面只添加了一个`- burst`方法，所以只有实例方法那一项被填充了值`_OBJC_$_CATEGORY_INSTANCE_METHODS_Sark_$_GayExtention`

其中`_I_Sark_GayExtention_burst`符号就代表了category里面的`- burst`方法，同样遵循了一定的命名规范，里面的`I`表示实例方法

最后，这个类的category们生成了一个数组，存在了`__DATA`段下的`__objc_catlistsection`里
```
static struct _category_t *L_OBJC_LABEL_CATEGORY_$ [1] __attribute__((used, section ("__DATA, __objc_catlist, regular, no_dead_strip")))= {
    &_OBJC_$_CATEGORY_Sark_$_GayExtention,
};
```

至此编译器的任务完成了。

### runtime，我的category哪儿去了？

我们知道，category动态扩展了原来类的方法，在调用者看来好像原来类本来就有这些方法似的，有两个事实：

1. 不论有没有import category 的.h，都可以成功调用category的方法，都影响不到category的加载流程，import只是帮助了编译检查和链接过程
2. runtime加载完成后，category的原始信息在类结构里将不会存在

这需要探究下runtime对category的加载过程，这里就简单说一下

1. objc runtime的加载入口是一个叫_objc_init的方法，在library加载前由libSystem dyld调用，进行初始化操作
2. 调用map_images方法将文件中的imagemap到内存
3. 调用_read_images方法初始化map后的image，这里面干了很多的事情，像load所有的类、协议和category，著名的+ load方法就是这一步调用的
4. 仔细看category的初始化，循环调用了_getObjc2CategoryList方法，这个方法拿出来看看：
5. …

```
#define GETSECT(name, type, sectname)                                   \
    type *name(const header_info *hi, size_t *outCount)  \
    {                                                                   \
        unsigned long byteCount = 0;                                    \
        type *data = (type *)                                           \
            getsectiondata(hi->mhdr, SEG_DATA, sectname, &byteCount);   \
        *outCount = byteCount / sizeof(type);                           \
        return data;                                                    \
    }

// ... //

GETSECT(_getObjc2CategoryList, category_t *, "__objc_catlist");
```

眼熟的`__objc_catlist`，就是上面category存放的数据段了，可以串连起来了

在调用完`_getObjc2CategoryList`后，runtime终于开始了category的处理，简化的代码如下

```
// Process this category.
// First, register the category with its target class.
// Then, rebuild the class's method lists (etc) if
// the class is realized.
BOOL classExists = NO;
if (cat->instanceMethods ||  cat->protocols  ||  cat->instanceProperties)
{
    addUnattachedCategoryForClass(cat, cls, hi);
    if (isRealized(cls)) {
        remethodizeClass(cls);
        classExists = YES;
    }
}

if (cat->classMethods  ||  cat->protocols )
{
    addUnattachedCategoryForClass(cat, cls->isa, hi);
    if (isRealized(cls->isa)) {
        remethodizeClass(cls->isa);
    }
}
```

首先分成两拨，一拨是实例对象相关的调用a`ddUnattachedCategoryForClass`，一拨是类对象相关的调用`addUnattachedCategoryForClass`，然后会调到`attachCategoryMethods`方法，这个方法把一个类所有的`category_list`的所有方法取出来组成一个`method_list_t **`，注意，这里是倒序添加的，也就是说，新生成的category的方法会先于旧的category的方法插入

```
static void
attachCategoryMethods(class_t *cls, category_list *cats,
                      BOOL *inoutVtablesAffected)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    BOOL isMeta = isMetaClass(cls);
    method_list_t **mlists = (method_list_t **)
        _malloc_internal(cats->count * sizeof(*mlists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int i = cats->count;
    BOOL fromBundle = NO;
    while (i--) {
        method_list_t *mlist = cat_method_list(cats->list[i].cat, isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= cats->list[i].fromBundle;
        }
    }

    attachMethodLists(cls, mlists, mcount, NO, fromBundle, inoutVtablesAffected);

    _free_internal(mlists);
}
```

生成了所有method的list之后，调用attachMethodLists将所有方法前序添加进类的方法的数组中，也就是说，如果原来类的方法是a,b,c，类别的方法是1,2,3，那么插入之后的方法将会是1,2,3,a,b,c，也就是说，原来类的方法被category的方法覆盖了，但被覆盖的方法确实还在那里。








