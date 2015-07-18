title: ARC dealloc cxx_destruct
categories: IOS
tags:
  - arc
  - dealloc
date: 2015-07-17 16:24:53
author:
description:
photos:
---
[转](http://blog.sunnyxx.com/2014/04/02/objc_dig_arc_dealloc/)

# ARC下dealloc过程及.cxx_destruct的探究

## 我是前言

这次探索源自于自己一直以来对ARC的一个疑问，在MRC时代，经常写下面的代码：

```
- (void)dealloc
{
    self.array = nil;
    self.string = nil;
    // ... //
    // 非Objc对象内存的释放，如CFRelease(...)
    // ... //
    [super dealloc];
}
```

对象析构时将内部其他对象release掉，申请的非Objc对象的内存当然也一并处理掉，最后调用super，继续将父类对象做析构。而现如今到了ARC时代，只剩下了下面的代码：

```
- (void)dealloc
{
    // ... //
    // 非Objc对象内存的释放，如CFRelease(...)
    // ... //
}
```

问题来了：

1. 这个对象实例变量（Ivars）的释放去哪儿了？
2. 没有显示的调用[super dealloc]，上层的析构去哪儿了？

## ARC文档中对dealloc过程的解释

[llvm官方的ARC文档](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#dealloc) 中对ARC下的dealloc过程做了简单说明，从中还是能找出些有用的信息：

> A class may provide a method definition for an instance method named dealloc. This method will be called after the final release of the object but before it is deallocated or any of its instance variables are destroyed. The superclass’s implementation of dealloc will be called automatically when the method returns.

大概意思是：dealloc方法在最后一次release后被调用，但此时实例变量（Ivars）并未释放，父类的dealloc的方法将在子类dealloc方法返回后自动调用

> The instance variables for an ARC-compiled class will be destroyed at some point after control enters the dealloc method for the root class of the class. The ordering of the destruction of instance variables is unspecified, both within a single class and between subclasses and superclasses.

理解：ARC下对象的实例变量在根类[NSObject dealloc]中释放（通常root class都是NSObject），变量释放顺序各种不确定（一个类内的不确定，子类和父类间也不确定，也就是说不用care释放顺序）

所以，不用主调[super dealloc]是因为自动调了，后面再说如何实现的；ARC下实例变量在根类NSObject析构时析构，下面就探究下。

---

## NSObject的析构过程

通过apple的runtime源码，不难发现NSObject执行`dealloc`时调用`_objc_rootDealloc`继而调用`object_dispose`随后调用`objc_destructInstance`方法，前几步都是条件判断和简单的跳转，最后的这个函数如下：

```
void *objc_destructInstance(id obj)
{
    if (obj) {
        Class isa_gen = _object_getClass(obj);
        class_t *isa = newcls(isa_gen);

        // Read all of the flags at once for performance.
        bool cxx = hasCxxStructors(isa);
        bool assoc = !UseGC && _class_instancesHaveAssociatedObjects(isa_gen);

        // This order is important.
        if (cxx) object_cxxDestruct(obj);
        if (assoc) _object_remove_assocations(obj);

        if (!UseGC) objc_clear_deallocating(obj);
    }

    return obj;
}
```

简单明确的干了三件事：

1. 执行一个叫`object_cxxDestruct`的东西干了点什么事
2. 执行`_object_remove_assocations`去除和这个对象`assocate`的对象
3. 执行`objc_clear_deallocating`，清空引用计数表并清除弱引用表，将所有`weak`引用指nil（这也就是weak变量能安全置空的所在）

所以，所探寻的ARC自动释放实例变量的地方就在cxxDestruct这个东西里面没跑了。

---

## 探寻隐藏的.cxx_destruct

上面找到的名为`object_cxxDestruct`的方法最终成为下面的调用：

```
static void object_cxxDestructFromClass(id obj, Class cls)
{
    void (*dtor)(id);

    // Call cls's dtor first, then superclasses's dtors.

    for ( ; cls != NULL; cls = _class_getSuperclass(cls)) {
        if (!_class_hasCxxStructors(cls)) return; 
        dtor = (void(*)(id))
            lookupMethodInClassAndLoadCache(cls, SEL_cxx_destruct);
        if (dtor != (void(*)(id))_objc_msgForward_internal) {
            if (PrintCxxCtors) {
                _objc_inform("CXX: calling C++ destructors for class %s", 
                             _class_getName(cls));
            }
            (*dtor)(obj);
        }
    }
}
```

代码也不难理解，沿着继承链逐层向上搜寻SEL_cxx_destruct这个selector，找到函数实现(void (*)(id)(函数指针)并执行。
搜索这个selector的声明，发现是名为.cxx_destruct的方法，以点开头的名字，我想和unix的文件一样，是有隐藏属性的

从[这篇文章](http://my.safaribooksonline.com/book/programming/objective-c/9780132908641/3dot-memory-management/ch03)中：

> ARC actually creates a -.cxx_destruct method to handle freeing instance variables. This method was originally created for calling C++ destructors automatically when an object was destroyed.

和《Effective Objective-C 2.0》中提到的：

> When the compiler saw that an object contained C++ objects, it would generate a method called .cxx_destruct. ARC piggybacks on this method and emits the required cleanup code within it.

可以了解到，`.cxx_destruct`方法原本是为了C++对象析构的，ARC借用了这个方法插入代码实现了自动内存释放的工作

## 通过实验找出.cxx_destruct

最好的办法还是写个测试代码把这个隐藏的方法找出来，其实在runtime中运行已经没什么隐藏可言了，简单的类结构如下：

```
@interface Father : NSObject
@property (nonatomic, copy) NSString *name;
@end

@interface Son : Father
@property (nonatomic, copy) NSArray *toys;
@end
```

只有两个简单的属性，找个地方写简单的测试代码：

```
// start
{
    // before new
    Son *son = [Son new];
    son.name = @"sark";
    son.toys = @[@"sunny", @"xx"];
    // after new
}
// gone
```

主要目的是为了让这个对象走`dealloc`方法，新建的son对象过了大括号作用域就会释放了，所以在`after new`这行son对象初始化完成，在`gone`这行son对象被dealloc

个人一直喜欢使用[NSObject+DLIntrospection](https://github.com/garnett/DLIntrospection)这个扩展作为调试工具，可以轻松打出一个类的方法，变量等等。

将这个扩展引入工程内，在`after new`处设置一个断点，run，trigger后使用lldb命令用这个扩展输出Son类所有的方法名：

![](http://ww3.sinaimg.cn/large/51530583gw1ef27srhw7lj208b05ujrq.jpg)

发现了这个`.cxx_destruct`方法，经过几次试验，发现：
1. 只有在ARC下这个方法才会出现（试验代码的情况下）
2. 只有当前类拥有实例变量时（不论是不是用property）这个方法才会出现，且父类的实例变量不会导致子类拥有这个方法
3. 出现这个方法和变量是否被赋值，赋值成什么没有关系

## 使用watchpoint定位内存释放时刻

依然在after new断点处，输入lldb命令：
```
watchpoint set variable son->_name

```

将name的变量加入`watchpoint`，当这个变量被修改时会触发trigger：

![](http://ww3.sinaimg.cn/large/51530583gw1ef28rn41lcj20fs03aq3b.jpg)

从中可以看出，在这个时刻，`_name`从0x00006b98变成了0x0，也就是nil，赶紧看下调用栈：

![](http://ww1.sinaimg.cn/large/51530583gw1ef2911o40zj20a605yweu.jpg)

发现果然跟到了`.cxx_destruct`方法，而且是在objc_storeStrong的过程中释放

#  刨根问底.cxx_destruct
知道了ARC下对象实例变量的释放过程在`.cxx_destruct`内完成，但这个函数内部发生了什么，是如何调用`objc_storeStrong`释放变量的呢？
从上面的探究中知道，`.cxx_destruct`是编译器生成的代码，那它很可能在clang前端编译时完成，这让我联想到clang的Code Generation，因为之前曾经使用`clang -rewrite-objc xxx.m`时查看过官方文档留下了些印象，于是google：
```
.cxx_destruct site:clang.llvm.org
```

结果发现clang的doxygen文档中`CodeGenModule`模块正是这部分的实现代码，cxx相关的代码生成部分源码在
http://clang.llvm.org/doxygen/CodeGenModule_8cpp-source.html
位于1827行，删减掉离题部分如下：

```
/// EmitObjCIvarInitializations - Emit information for ivar initialization
/// for an implementation.
void CodeGenModule::EmitObjCIvarInitializations(ObjCImplementationDecl *D) 
{
    DeclContext* DC = const_cast<DeclContext*>(dyn_cast<DeclContext>(D));
    assert(DC && "EmitObjCIvarInitializations - null DeclContext");
    IdentifierInfo *II = &getContext().Idents.get(".cxx_destruct");
    Selector cxxSelector = getContext().Selectors.getSelector(0, &II);
    ObjCMethodDecl *DTORMethod = ObjCMethodDecl::Create(getContext(), 
                                                        D->getLocation(),
                                                        D->getLocation(), cxxSelector,
                                                        getContext().VoidTy, 0, 
                                                        DC, true, false, true,
                                                        ObjCMethodDecl::Required);
   D->addInstanceMethod(DTORMethod);
   CodeGenFunction(*this).GenerateObjCCtorDtorMethod(D, DTORMethod, false);
}
```

这个函数大概作用是：获取.`cxx_destruct`的selector，创建Method，并加入到这个Class的方法列表中，最后一行的调用才是真的创建这个方法的实现。这个方法位于
http://clang.llvm.org/doxygen/CGObjC_8cpp_source.html

1354行，包含了构造和析构的cxx方法，继续跟随`.cxx_destruct`，最终调用`emitCXXDestructMethod`函数，代码如下：

```
static void emitCXXDestructMethod(CodeGenFunction &CGF, ObjCImplementationDecl *impl) 
{
   CodeGenFunction::RunCleanupsScope scope(CGF);
 
   llvm::Value *self = CGF.LoadObjCSelf();
 
   const ObjCInterfaceDecl *iface = impl->getClassInterface();
   for (const ObjCIvarDecl *ivar = iface->all_declared_ivar_begin(); ivar; ivar = ivar->getNextIvar()) 
   {
     QualType type = ivar->getType();

     // Check whether the ivar is a destructible type.
     QualType::DestructionKind dtorKind = type.isDestructedType();
     if (!dtorKind) continue;
 
     CodeGenFunction::Destroyer *destroyer = 0;
 
     // Use a call to objc_storeStrong to destroy strong ivars, for the
     // general benefit of the tools.
     if (dtorKind == QualType::DK_objc_strong_lifetime) {
       destroyer = destroyARCStrongWithStore;
 
     // Otherwise use the default for the destruction kind.
     } else {
       destroyer = CGF.getDestroyer(dtorKind);
     }
 
     CleanupKind cleanupKind = CGF.getCleanupKind(dtorKind);
     CGF.EHStack.pushCleanup<DestroyIvar>(cleanupKind, self, ivar, destroyer,
                                          cleanupKind & EHCleanup);
   }
 
   assert(scope.requiresCleanups() && "nothing to do in .cxx_destruct?");
}
```

分析这段代码以及其中调用后发现：它遍历当前对象所有的实例变量（Ivars)，调用`objc_storeStrong`，从clang的ARC文档上可以找到`objc_storeStrong`的示意代码实现如下：

```
id objc_storeStrong(id *object, id value) {
  value = [value retain];
  id oldValue = *object;
  *object = value;
  [oldValue release];
  return value;
}
```

在`.cxx_destruct`进行形如`objc_storeStrong(&ivar, null)`的调用后，这个实例变量就被release和设置成nil了
注：真实的实现可以参考 http://clang.llvm.org/doxygen/CGObjC_8cpp_source.html 2078行

## 自动调用[super dealloc]的实现

按照上面的思路，自动调用`[super dealloc]`也一定是CodeGen干的工作了
位于 http://clang.llvm.org/doxygen/CGObjC_8cpp_source.html 492行
`StartObjCMethod`方法中：

```
if (ident->isStr("dealloc"))
   EHStack.pushCleanup<FinishARCDealloc>(getARCCleanupKind());
```

上面代码可以得知在调用`dealloc`方法时被插入了代码，由FinishARCDealloc结构定义：

```
struct FinishARCDealloc : EHScopeStack::Cleanup {
   void Emit(CodeGenFunction &CGF, Flags flags) override {
     const ObjCMethodDecl *method = cast<ObjCMethodDecl>(CGF.CurCodeDecl);
 
     const ObjCImplDecl *impl = cast<ObjCImplDecl>(method->getDeclContext());
     const ObjCInterfaceDecl *iface = impl->getClassInterface();
     if (!iface->getSuperClass()) return;
 
     bool isCategory = isa<ObjCCategoryImplDecl>(impl);
 
     // Call [super dealloc] if we have a superclass.
     llvm::Value *self = CGF.LoadObjCSelf();
 
     CallArgList args;
     CGF.CGM.getObjCRuntime().GenerateMessageSendSuper(CGF, ReturnValueSlot(),
                                                       CGF.getContext().VoidTy,
                                                       method->getSelector(),
                                                       iface,
                                                       isCategory,
                                                       self,
                                                       /*is class msg*/ false,
                                                       args,
                                                       method);
   }
};
```

上面代码基本上就是向父类转发dealloc的调用，实现了自动调用[super dealloc]方法。

## 总结
1. ARC下对象的成员变量于编译器插入的.cxx_desctruct方法自动释放
2. ARC下[super dealloc]方法也由编译器自动插入
3. 所谓编译器插入代码过程需要进一步了解，还不清楚其运作方式
4. clang的CodeGen也值得深入研究一下

## References：
http://clang.llvm.org/docs/AutomaticReferenceCounting.html
http://my.safaribooksonline.com/book/programming/objective-c/9780132908641/3dot-memory-management/ch03
http://clang.llvm.org/doxygen/CGObjC_8cpp_source.html






