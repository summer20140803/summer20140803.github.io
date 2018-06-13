---
layout:     post
title:      iOS如何在Block中安全调用Super方法
date:       2016-11-04
author:     开不了口的猫
header-img: img/icon_如何在Block中安全调用Super方法_bg.jpg
catalog: true
tags:
    - iOS
    - 安全
---

## 前言
其实这不算一个比较常见的问题，因为没人会在某一个类的子类中去覆写父类的某一个方法实现然后还在这个方法中调用父类原来的方法实现，如果子类没有覆写，那一般直接就会通过self去调用。  
  
不过有一次晚上在公司跟同事闲聊正好谈到过这种场景，特别针对于某一个类申明的某一个block中使用super来调用的情景，因为我之前也不曾想象过这样的场景，所以不由心生了许多疑问，最大的疑问莫过于，在一个双持引用的block中，直接调用super是否会直接产生循环引用导致block和被捕获的对象都不能被释放呢？  

对应的实践代码我都放在了[Sample Repo](https://github.com/summer20140803/MyBlogCodeSamples)下，找到与本文标题对应的文件夹下的工程即可。

## 实践
首先我们创建了两个类，父类SZYParentObject和子类SZYChildObject。  

`SZYParentObject.h`
```objc
#import <Foundation/Foundation.h>

@interface SZYParentObject : NSObject

- (void)superMethod;

@end
```
  
`SZYParentObject.m`
```objc
#import "SZYParentObject.h"

@implementation SZYParentObject

- (void)superMethod {
    NSLog(@"%@, %@", NSStringFromClass([self class]), NSStringFromSelector(_cmd));
}

@end
```

`SZYChildObject.h`
```objc
#import "SZYParentObject.h"

@interface SZYChildObject : SZYParentObject

@property (nonatomic, copy) void (^customizedBlockHandler)(void);

@end
```

`SZYChildObject.m`
```objc
#import "SZYChildObject.h"

@interface SZYChildObject ()

@end

@implementation SZYChildObject

- (instancetype)init {
    self = [super init];
    if (self) {
        _customizedBlockHandler = ^{
            // 假如这里我就是想要调用父类的方法
            [super superMethod];
        };
    }
    return self;
}

#pragma mark - override method
- (void)superMethod {
    NSLog(@"子类自己的code实现");
}

@end
```
  
然后咱们在ViewController引入SZYChildObject类，初始化一个SZYChildObject类的实例并申明一个weak的属性来弱引用这个实例，方便我们检测这个实例在后续是否被释放了。
```objc
#import "ViewController.h"
#import "SZYChildObject.h"

@interface ViewController ()

@property (nonatomic, weak) SZYChildObject *obj;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    SZYChildObject *obj = [[SZYChildObject alloc] init];
    obj.customizedBlockHandler();
    self.obj = obj;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"%@", self.obj);
}

@end
```

运行之后，我们担心的事情发生了。
```objc
2018-06-07 12:41:23.133185+0800 iOS如何在Block中安全调用Super[21963:10096063] SZYChildObject, superMethod
2018-06-07 12:41:36.714578+0800 iOS如何在Block中安全调用Super[21963:10096063] <SZYChildObject: 0x60000000f0f0>
2018-06-07 12:41:36.855900+0800 iOS如何在Block中安全调用Super[21963:10096063] <SZYChildObject: 0x60000000f0f0>
2018-06-07 12:41:36.999397+0800 iOS如何在Block中安全调用Super[21963:10096063] <SZYChildObject: 0x60000000f0f0>
```

我们通过Xcode的[Debug Memory Graph](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/debugging_with_xcode/chapters/special_debugging_workflows.html)自动分析可以看到，在应用堆内存中，这个实例一直存在，并且原因也一目了然，和一个指定block双持强引用了。

![ctdpic](https://ws1.sinaimg.cn/large/006tKfTcgy1fs2iqzmlnrj31jm0dajvv.jpg)

那么再让我们回到导致循环引用的这个block。

```objc
_customizedBlockHandler = ^{
    // 假如这里我就是想要调用父类的方法
    [super superMethod];
};
```
现在结论已经有了，在一个被self持有的block中调用super确实会引起**循环引用**，那么应该怎样在必须调用super方法的前提下如何避免循环引用呢？首先我们要来了解一下super这个复杂且带有欺骗性的关键字。  

在iOS语言中，self是类的隐藏的参数，指向当前调用方法的类，另一个隐藏的参数是_cmd，代表当前类方法对应的selector。而看似近似的super关键字却绝非是个隐藏的参数，它是一个**"编译器指示符"**。我们都知道通过self调用方法，实际上会调用**objc_msgSend函数**，而通过下面官方文档的说明截图我们发现，如果通过super调用方法实际上则是调用**objc_msgSendSuper函数**。
![ctdpic](https://ws4.sinaimg.cn/large/006tKfTcgy1fs2luovqgbj31ay0iiadb.jpg)

那我们继续看下**objc_msgSend**和**objc_msgSendSuper**的区别。首先是objc_msgSend:

```objc
id objc_msgSend(id self, SEL op, ...);
```

第一个参数是消息接收者，第二个参数是调用的具体类方法的selector，后面跟着selector方法的可变参数。拿上面demo中的两个类举栗，如果我们在子类也就是SZYChildObject类中调用  
**[self superMethod];**  
则第一个入参就是SZYChildObject实例自己，第二个参数就是从当前self的class的方法列表开始向上找到的superMethod的selector。   

然后是objc_msgSendSuper:

```objc
id objc_msgSendSuper(struct objc_super *super, SEL op, ...);
```

第一个参数是一个objc_super的结构体，第二个参数跟objc_msgSend的第二个参数是一致的。那么我们就需要把焦点转移到结构体**objc_super**上了。一样通过runtime文档找到这个结构体的申明：

```objc
/// Specifies the superclass of an instance. 
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained _Nonnull id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained _Nonnull Class class;
#else
    __unsafe_unretained _Nonnull Class super_class;
#endif
    /* super_class is the first class to search */
};
#endif  
```

简化之后其实就是：

```objc
struct objc_super {
    __unsafe_unretained id receiver;
    __unsafe_unretained Class super_class;
};
```

我们可以看到这个结构体包含了两个成员，第一个成员变量**receiver**就是**子类实例本身**，和**self**相同。而第二个成员变量**super_class**其实就是指向的父类**SZYParentObject**了。  

当使用[self someMethod]时，会调用**objc_msgSend**函数，第一个参数receiver就是self，而第二个参数，要先找到self所在的这个class的方法列表，如果有，则返回对应的selector并执行，如果没有，则会一层层向上寻找，直到找到为止，如果最后都没能找到，ok，那我们进入**消息转发流程**。  

当使用[super someMethod]时，会调用**objc_msgSendSuper**函数，此时会先构造一个**objc_super**的结构体，然后第一个成员变量receiver仍然是self，而第二个成员变量super_class即是所在类的父类。构造完之后，把结构体传入**objc_msgSendSuper**函数中，然后会从super_class这个类对应的方法列表开始找selector，如果有，则返回对应的selector并执行，如果没有，则会一层层向上寻找，直到找到为止，如果最后都没能找到，会进入**消息转发流程**。  

如此一探究，super调用的流程以及与self去调用的区别就真相大白了。现在再回头来看示例中的block中调用super会导致循环引用的原因以及block如何安全使用super调用问题的答案便已浮出水面了。  

将super用源码展开后：

```objc
_customizedBlockHandler = ^{
    struct objc_super superInfo = {
        .receiver = self,
        .super_class = class_getSuperclass(NSClassFromString(@"SZYChildObject"))
    };
    void (*msgSendSuperFunction)(struct objc_super *, SEL) = (__typeof__(msgSendSuperFunction))objc_msgSendSuper;
    msgSendSuperFunction(&superInfo, @selector(superMethod));
};
```

可以很明显的看到问题，block强引用了self，而self也强持有了这个block。

而正确的调用姿势跟平常我们切断block的循环引用的姿势一模一样：

```objc
__weak __typeof(self) weak_self = self;
_customizedBlockHandler = ^{
    struct objc_super superInfo = {
        .receiver = weak_self,
        .super_class = class_getSuperclass(NSClassFromString(@"SZYChildObject"))
    };
    void (*msgSendSuperFunction)(struct objc_super *, SEL) = (__typeof__(msgSendSuperFunction))objc_msgSendSuper;
    msgSendSuperFunction(&superInfo, @selector(superMethod));
};
```

改完咱们再重新Run一下示例Demo，发现一切都正常了~

## 后续
其实看到这里，应该有人会对

```objc
.super_class = class_getSuperclass(NSClassFromString(@"SZYChildObject"))   
```

这个方法产生些疑惑，在这里为什么是选择用写死**SZYChildObject**类名而不是通过  

```objc
.super_class = class_getSuperclass(objc_getClass(self))
```

去调用到当前的类并赋值给成员变量**super_class**。  

诚然，在示例Demo中的场景下这样调用确实不会产生任何问题，但是如果有人在SZYChildObject的superMethod中也通过手动构造super_class结构体的方法去调用了父类SZYParentObject类的superMethod方法，并且一旦有人又创建了一个SZYChildObject的子类X，这个子类X如果没有覆写SZYChildObject的superMethod方法，在viewController中创建的是子类X的实例并直接执行子类X实例的block实现的话，你会发现，程序会出现superMethod方法的调用无限递归死循环并crash。纠其原因，还是因为superMethod和block内的super_class不能通过self去控制，因为objc_getClass(self)永远是指向当前方法调用者的类。  

对于这个问题，[这篇文章](https://www.jianshu.com/p/87fe5efe104e)中也有解释，并且这位作者质疑了clang rewrite的可靠性，认为super关键字其实是直接指明本类Son，再结合_objc_msgSendSuper2函数直接获取父类去查找方法的，而并非像clang重写的那样，指明本类，再通过runtime查找父类，super其真正的调用链路应该是：
1. 编译器指定一个**struct._objc_super**结构体， 结构体中self为接收对象，直接指明自身的类为结构体第二个class类型的值。
2. 调用**_objc_msgSendSuper2**函数，传入上述**struct._objc_super**结构体。
3. 在**_objc_msgSendSuper2**函数中直接通过偏移量直接查找父类。
4. 调用**CacheLookup**函数去父类中查找指定方法。

## 参考
* [https://developer.apple.com/documentation/objectivec/1456716-objc_msgsendsuper?language=occ](https://developer.apple.com/documentation/objectivec/1456716-objc_msgsendsuper?language=occ)  
* [https://developer.apple.com/documentation/objectivec/objc_super?language=objc](https://developer.apple.com/documentation/objectivec/objc_super?language=objc)  
* [https://www.jianshu.com/p/87fe5efe104e](https://www.jianshu.com/p/87fe5efe104e)
