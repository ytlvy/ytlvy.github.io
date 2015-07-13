title: iOS Runtime Message And Category
categories: IOS
tags:
  - IOS
  - Category
description:
date: 2015-07-13 21:20:02
author:
photos:
---
### 习题内容

下面的代码会？Compile Error / Runtime Crash / NSLog…?
```
@interface NSObject (Sark)
+ (void)foo;
@end

@implementation NSObject (Sark)

- (void)foo
{
    NSLog(@"IMP: -[NSObject(Sark) foo]");
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        [NSObject foo];
        [[NSObject new] foo];
    }
    return 0;
}
```

答案：代码正常输出，输出结果如下：
```
2014-11-06 13:11:46.694 Test[14872:1110786] IMP: -[NSObject(Sark) foo]
2014-11-06 13:11:46.695 Test[14872:1110786] IMP: -[NSObject(Sark) foo]
```
<!-- more -->
使用`clang -rewrite-objc main.m`重写，我们可以发现 main 函数中两个方法调用被转换成如下代码：
```
((void (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("foo"));
 ((void (*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new")), sel_registerName("foo"));
```

我们发现上述两个方法最终转换成使用 `objc_msgSend` 函数传递消息

### 这里先看几个概念
`objc_msgSend`函数定义如下:

```
id objc_msgSend(id self, SEL op, ...)
```

关于 id 的解释请看objc runtime系列第二篇博文： objc runtime中Object & Class & Meta Class的细节

### 什么是 SEL
打开objc.h文件，看下SEL的定义如下:
```
typedef struct objc_selector *SEL;
```

SEL是一个指向`objc_selector`结构体的指针。而 `objc_selector` 的定义并没有在runtime.h中给出定义。我们可以尝试运行如下代码:
```
SEL sel = @selector(foo);
NSLog(@"%s", (char *)sel);
NSLog(@"%p", sel);

const char *selName = [@"foo" UTF8String];
SEL sel2 = sel_registerName(selName);
NSLog(@"%s", (char *)sel2);
NSLog(@"%p", sel2);
```

输出如下:
```
2014-11-06 13:46:08.058 Test[15053:1132268] foo
2014-11-06 13:46:08.058 Test[15053:1132268] 0x7fff8fde5114
2014-11-06 13:46:08.058 Test[15053:1132268] foo
2014-11-06 13:46:08.058 Test[15053:1132268] 0x7fff8fde5114
```

Objective-C在编译时，会根据方法的名字生成一个用来区分这个方法的唯一的一个ID。只要方法名称相同，***那么它们的ID就是相同的***。

两个类之间，不管它们是父类与子类的关系，还是之间没有这种关系，只要方法名相同，那么它的SEL就是一样的。每一个方法都对应着一个SEL。编译器会根据每个方法的方法名为那个方法生成唯一的SEL。这些SEL组成了一个Set集合，当我们在这个集合中查找某个方法时，只需要去找这个方法对应的SEL即可。而SEL本质是一个字符串，所以直接比较它们的地址即可。

当然，不同的类可以拥有相同的selector。不同类的实例对象执行相同的selector时，会在各自的方法列表中去根据selector去寻找自己对应的IMP。

### 那么什么是IMP呢
继续看定义:
```
typedef id (*IMP)(id, SEL, ...); 
```

IMP本质就是一个函数指针，这个被指向的函数包含一个接收消息的对象id，调用方法的SEL，以及一些方法参数，并返回一个id。

因此我们可以***通过SEL获得它所对应的IMP，在取得了函数指针之后，也就意味着我们取得了需要执行方法的代码入口，这样我们就可以像普通的C语言函数调用一样使用这个函数指针。***

### 那么 objc_msgSend 到底是怎么工作的呢
在Objective-C中，消息直到运行时才会绑定到方法的实现上。编译器会把代码中[target doSth]转换成 objc_msgSend消息函数，这个函数完成了动态绑定的所有事情。它的运行流程如下:

1. 检查selector是否需要忽略。(ps: Mac开发中开启GC就会忽略retain,release方法。)
2. 检查target是否为nil。如果为nil，直接cleanup，然后return。(这就是我们可以向nil发送消息的原因。)
3. 然后在target的Class中根据Selector去找IMP

寻找IMP的过程:

1. 先从当前class的cache方法列表（cache methodLists）里去找
2. 找到了，跳到对应函数实现
3. 没找到，就从class的方法列表（methodLists）里找
4. 还找不到，就到super class的方法列表里找，直到找到基类(NSObject)为止
5. 最后再找不到，就会进入动态方法解析和消息转发的机制。(这部分知识，下次再细谈)

###  那么什么是方法列表呢
objc_class结构体定义，如下
```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
    #if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
    #endif
} OBJC2_UNAVAILABLE;

struct objc_method_list {
    struct objc_method_list *obsolete                        OBJC2_UNAVAILABLE;

    int method_count                                         OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
    /* variable length structure */
    struct objc_method method_list[1]                        OBJC2_UNAVAILABLE;
}
```

1) objc_method_list 就是用来存储当前类的方法链表，objc_method存储了类的某个方法的信息。

Method
```
typedef struct objc_method *Method;
```

Method 是用来代表类中某个方法的类型，它实际就指向objc_method结构体，如下:
```
struct objc_method {
    SEL method_name                                          OBJC2_UNAVAILABLE;
    char *method_types                                       OBJC2_UNAVAILABLE;
    IMP method_imp                                           OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;
```

- `method_types`是个char指针，存储着方法的参数类型和返回值类型。
- SEL 和 IMP 就是我们上文提到的，所以我们可以理解为objc_class中 method list保存了一组SEL<->IMP的映射

2) objc_cache 用来缓存用过的方法，提高性能。 
Cache
```
typedef struct objc_cache *Cache                             OBJC2_UNAVAILABLE;
```

实际指向`objc_cache`结构体，如下:

```
struct objc_cache {
    unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;
    unsigned int occupied                                    OBJC2_UNAVAILABLE;
    Method buckets[1]                                        OBJC2_UNAVAILABLE;
};
```

- mask: 指定分配cache buckets的总数。在方法查找中，Runtime使用这个字段确定数组的索引位置
- occupied: 实际占用cache buckets的总数
- buckets: 指定Method数据结构指针的数组。这个数组可能包含不超过mask+1个元素。需要注意的是，指针可能是NULL，表示这个缓存bucket没有被占用，另外被占用的bucket可能是不连续的。这个数组可能会随着时间而增长。

`objc_msgSend`每调用一次方法后，就会把该方法缓存到cache列表中，下次的时候，就直接优先从cache列表中寻找，如果cache没有，才从methodLists中查找方法。

### 说完了 objc_msgSend， 那么题目中的Category又是怎么工作的呢?
我们知道`Catagory`可以动态地为已经存在的类添加新的方法。这样可以保证类的原始设计规模较小，功能增加时再逐步扩展。在runtime.h中查看定义:
```
typedef struct objc_category *Category;
```

同样也是指向一个 `objc_category` 的C 结构体，定义如下:

```
struct objc_category {
    char *category_name                                      OBJC2_UNAVAILABLE;
    char *class_name                                         OBJC2_UNAVAILABLE;
    struct objc_method_list *instance_methods                OBJC2_UNAVAILABLE;
    struct objc_method_list *class_methods                   OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;
```

通过上面的结构体，大家可以很清楚的看出存储的内容。我们继续往下看，打开objc源代码，在 objc-runtime-new.h中我们可以发现如下定义:
```
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
};
```

上面的定义需要提到的地方有三点:

- name 是指 class_name 而不是 category_name
- cls是要扩展的类对象，编译期间是不会定义的，而是在Runtime阶段通过name对应到对应的类对象
- instanceProperties表示Category里所有的properties，这就是我们可以通过objc_setAssociatedObject和objc_getAssociatedObject增加实例变量的原因，不过这个和一般的实例变量是不一样的

为了验证上述内容，我们使用clang -rewrite-objc main.m重写，题目中的Category被编译器转换成了这样:
```
// @interface NSObject (Sark)
// + (void)foo;
/* @end */


// @implementation NSObject (Sark)


static void _I_NSObject_Sark_foo(NSObject * self, SEL _cmd) {
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_gm_0jk35cwn1d3326x0061qym280000gn_T_main_dd1ee3_mi_0);
}

// @end


static struct _category_t _OBJC_$_CATEGORY_NSObject_$_Sark __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
    "NSObject",
    0, // &OBJC_CLASS_$_NSObject,
    (const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_NSObject_$_Sark,
    0,
    0,
    0,
};

static struct _category_t *L_OBJC_LABEL_CATEGORY_$ [1] __attribute__((used, section ("__DATA, __objc_catlist,regular,no_dead_strip")))= {
    &_OBJC_$_CATEGORY_NSObject_$_Sark,
};
```

- _OBJC_$_CATEGORY_NSObject_$_Sark是按规则生成的字符串，我们可以清楚的看到是NSObject类,且Sark是NSObject类的Category
- _category_t结构体第二项 classref_t 没有数据，验证了我们上面的说法
- 由于题目中只有 - (void)foo方法，所以结构体中存储的list只有第三项instanceMethods被填充。
- _I_NSObject_Sark_foo代表了Category的foo方法，I表示实例方法
- 最后这个类的Category生成了一个数组，存在了__objc_catlist里，目前数组的内容只有一个&_OBJC_$_CATEGORY_NSObject_$_Sark

### 最终这些Category里面的方法是如何被加载的呢?

1. 打开objc源代码，找到 objc-os.mm, 函数`_objc_init`为runtime的加载入口，由libSystem调用，进行初始化操作。

2. 之后调用`objc-runtime-new.mm` -> `map_images`加载`map`到内存

3. 之后调用objc-runtime-new.mm->_`read_images`初始化内存中的map, 这个时候将会load所有的类，协议还有Category。NSOBject的`+load`方法就是这个时候调用的

```
// Discover categories. 
for (EACH_HEADER) {
    category_t **catlist = 
        _getObjc2CategoryList(hi, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = catlist[i];
        Class cls = remapClass(cat->cls);

        if (!cls) {
            // Category's target class is missing (probably weak-linked).
            // Disavow any knowledge of this category.
            catlist[i] = nil;
            if (PrintConnecting) {
                _objc_inform("CLASS: IGNORING category \?\?\?(%s) %p with "
                             "missing weak-linked target class", 
                             cat->name, cat);
            }
            continue;
        }

        // Process this category. 
        // First, register the category with its target class. 
        // Then, rebuild the class's method lists (etc) if 
        // the class is realized. 
        BOOL classExists = NO;
        if (cat->instanceMethods ||  cat->protocols  
            ||  cat->instanceProperties) 
        {
            addUnattachedCategoryForClass(cat, cls, hi);
            if (cls->isRealized()) {
                remethodizeClass(cls);
                classExists = YES;
            }
            if (PrintConnecting) {
                _objc_inform("CLASS: found category -%s(%s) %s", 
                             cls->nameForLogging(), cat->name, 
                             classExists ? "on existing class" : "");
            }
        }

        if (cat->classMethods  ||  cat->protocols  
            /* ||  cat->classProperties */) 
        {
            addUnattachedCategoryForClass(cat, cls->ISA(), hi);
            if (cls->ISA()->isRealized()) {
                remethodizeClass(cls->ISA());
            }
            if (PrintConnecting) {
                _objc_inform("CLASS: found category +%s(%s)", 
                             cls->nameForLogging(), cat->name);
            }
        }
    }
}
```

1) 循环调用了 _getObjc2CategoryList方法，这个方法的实现是:
```
GETSECT(_getObjc2CategoryList,        category_t *,    "__objc_catlist");
```
方法中最后一个参数`__objc_catlist`就是编译器刚刚生成的category数组

2) load完所有的categories之后，开始对Category进行处理。
> 从上面的代码中我们可以发现：实例方法被加入到了当前的类对象中, 类方法被加入到了当前类的Meta Class中 (cls->ISA)

Step 1. 调用`addUnattachedCategoryForClass`方法

Step 2. 调用`remethodizeClass`方法, 在`remethodizeClass`的实现里调用`attachCategoryMethods`
```
static void 
attachCategoryMethods(Class cls, category_list *cats, bool flushCaches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();
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

    attachMethodLists(cls, mlists, mcount, NO, fromBundle, flushCaches);

    _free_internal(mlists);
}
```

这里把一个类的`category_lis`t的所有方法取出来生成了`method list`。这里是倒序添加的，也就是说，新生成的category的方法会先于旧的category的方法插入。

之后调用`attachMethodLists`将所有方法前序添加进类的`method list`中，如果原来类的方法列表是a，b，Category的方法列表是c，d。那么插入之后的方法列表将会是c，d，a，b。

### 小发现

- 看上面被编译器转换的代码，我们发现Category头文件被注释掉了，结合上面category的加载过程。这就是我们即使没有import category的头文件，都能够成功调用到Category方法的原因。

- runtime加载完成后，Category的原始信息在类结构中将不会存在。

### 解惑
根据上面提到的知识，我们对题目中的代码进行分析。

1) objc runtime加载完后，NSObject的`Sark Category`被加载。而NSObject的`Sark Category`的头文件` + (void)foo` 并没有实质参与到工作中，只是给编译器进行静态检查，所有我们编译上述代码会出现警告，提示我们没有实现 `+ (void)foo` 方法。而在代码编译中，它已经被注释掉了。

2) 实际被加入到Class的method list的方法是 `- (void)foo`，它是一个实例方法，所以加入到当前类对象NSObject的方法列表中，而不是NSObject `Meta class`的方法列表中。

3) 当执行 `[NSObject foo]`时，我们看下整个objc_msgSend的过程:

```
结合上一篇Meta Class的知识：

1. objc_msgSend 第一个参数是  “(id)objc_getClass("NSObject")”，获得NSObject Class的对象
2. 类方法在Meta Class的方法列表中找，我们在load Category方法时加入的是- (void)foo实例方法，所以
并不在NSOBject Meta Class的方法列表中
3. 继续往 super class中找，在上一篇博客中我们知道，NSObject Meta Class的super class是
NSObject本身。所以，这个时候我们能够找到- (void)foo 这个方法。
4. 所以正常输出结果
```

4) 当执行`[[NSObject new] foo]`，我们看下整个`objc_msgSend`的过程:
```
1. [NSObject new]生成一个NSObject对象
2. 直接在该对象的类（NSObject）的方法列表里找
3. 能够找到，所以正常输出结果
```
