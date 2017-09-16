--- 
layout:     post
title:      Objective-C学以致用-Method Swizzling
subtitle:   Method Swizzling解析
date:       2015-12-02
author:     奇风
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - iOS
    - Object-C
    - runtime
    - Method Swizzling
---
Method Swizzling还没有一个广泛接受的译名，我个人认为比较容易理解的一个是方法变换。简单的说，它就是在运行期修改类中方法所对应的实现的技术。
在本文中，我们就将方法变换的来龙去脉捋一遍。

在捋这个来龙去脉的时候，我们需要把握住三个原则：格物致知，深入浅出，学以致用。
其中，格物致知是方法，深入浅出是成果，学以致用是目的。

#1.格物：明白其原理
方法变换的技术基础在于Objective-C是一个具有运行时动态特性的面向对象语言，要想彻底的明白方法变换的实现细节，那么就需要对Objective-C有一个整体的认识，尤其是其运行期的原理。
我们这里格的物就是runtime。

##1.1.世界构造
Objective-C是一个面向对象语言，那么面向对象就是核心的思想，下面这张图对此表述的足够清晰。
![Objective-C的世界观](http://blog.devtang.com/images/class-diagram.jpg)

《道德经》中有云：道生一，一生二，二生三，三生万物。
而Objective-C的世界观与道教的世界观很像。

###Objective-C的“道”
万物皆对象。这是整个世界的最高准则。

###Objective-C的“一”
既然万物皆对象，并且考虑到面向对象思想中的抽象与继承概念，那么这个世界中就必然存在第一个对象，也是所有对象的根。
而这个根就是“一”，NSObject。

###Objective-C的“二”
我们知道NSObject是一切类的基类，所以它是一个类。但“道”又说了“万物皆对象”，那这个NSObject呢？没错，它也是对象。扩展开来，所有的类其实都是对象。
不过我们知道，在Objective-C中，对象都是有类型的，它们都是由类定义出来的。那么，NSObject的类型是什么呢？或者说类的类是什么呢？
NSObject的类型是NSObject的MetaClass（元类）。NSObject是NSObject元类的一个实例。所有的类都存在一个对应的元类。
而这NSObject的MetaClass就是“二”！

说到这里，大家或许会有一个问题，既然NSObject（一）是由NSObject的元类（二）所定义的，那么为什么是“道生一，一生二”？而不是“二生一”呢？

但是，不要忘了，整个Objective-C的世界中只有一个根类NSObject，所有其他的类都必须继承自NSObject，所以不可避免的，NSObject元类也是继承自NSObject的一个子类。
因此才说：一生二。

###Objective-C的“三”
上面我们说到，NSObject是NSObject元类的实例。那么如果脑洞足够大的话你应该会想到一个问题：那这个NSObject的元类是个什么东东？
按照最高准则——万物都是对象，那它也是对象。那么它的类型是什么？是NSObject元类的元类？那NSObject元类的元类是谁的实例呢？这不成了一个无限死循环了么？

其实，你的这个担忧是不存在的，因为NSObject元类的类型是它自己。实际上，所有类的元类的类型都是NSObject元类。因此，不存在无限死循环的问题。

这样一来，类和元类形成了一个闭环：所有的东西都是对象，且所有的对象都是由类型（类或元类）定义的。
NSObject类与NSObject元类你依赖我，我继承你，两位一体。在这对阴阳双子的交互作用下，再加上面向对象中基本的继承体系，Objective-C世界总就衍生出无数的类。
这，便是“三生万物”!

##1.2.运转规则
在1.1中我们只是讲了整个Objective-C世界的构造是怎么样的，但并没有说明它是怎么运行的，万物对象之间是怎么发生联系的。

###消息机制
Objective-C的运转依靠的是消息机制，其实相对于直接调用，这种方法更接近于现实世界。
比如说，你中午来到一个小饭馆吃饭，吃完后喊道：“服务员，买单！”
那么这个时候，“你”就向“服务员”发出了一个名为“买单”的消息，下面是卧槽(OC)语言的代码：
```
服务员类 *服务员 = [饭馆 可用的服务员];
[服务员 买单];
```
但是这里有一个问题，那就是你并不知道“服务员”对象是否能够处理“买单”这个消息。或者在你的经验中其他饭馆的“服务员”可以“买单”，但是实际上这个饭馆的不可以。
这中差异其实与Objective-C的运转规则是一样的，在Objective-C中你也可以向指定对象发送一个不知道是否可以响应的消息。

不过扩展一下，在现实世界中这样的问题必然会出现，因为现在世界中没有IDE环境帮你做强类型校验，你也不可能每次向对象发送消息前都问一下对方是否能够响应该消息。
但是Objective-C的软件开发中，尤其是业务逻辑开发，一般不会遇到，因为Xcode环境还是做了基本的类型校验。但这是IDE环境的功能，并不是Objective-C世界运转规则的一部分。
这便是Objective-C中的动态特性。

那么一旦出现这种情况，要怎么处理呢？实际上是会有几种不同的结果：

 1. 服务员根本不理你。因为服务员处理不了买单消息，所以他没有给出任何响应，这就有如APP的闪退。当然现实世界是不会闪退重启的……
 2. 服务员只是告诉你他不能买单。这是一个简单的容错处理，当服务员遇到不能处理的[XX]消息时，他就直接告诉你“我不能XX”。这样你就可以选择其他的处理方法。
 3. 服务员直接将你领到柜台处，转由收银员或者老板处理。这就是一个相当好的，接近无缝连接的处理方法了。当服务员接收到自己无法处理的“买单”消息时，她应该判断消息的类型是“买单”，并且转给能处理“买单”消息的对收银员或者老板进行处理。

###代码展现
说了这么多，其实上面只是讲了Objective-C世界运转的基本规则——具有动态特性的消息机制。
那么这个消息机制在整个世界构造中是怎么运转的呢？

在此之前，我们要了解Objective-C世界的代码展现方式。
我们先看一下“一”：NSObject
```
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}

+ (void)load;
+ (void)initialize;
.
.
.
@end
```
我们可以看到其中有一个`Class isa  OBJC_ISA_AVAILABILITY;`变量，那么这个Class类型是什么呢？
```
typedef struct objc_class *Class;
```
可见Class其实是一个结构体指针，该结构体定义如下：
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
```
其中*methodLists中存储的便是当前类的实例方法列表，表示当前类的实例所能够响应的所有消息列表，以及响应该消息时执行的函数指针地址。
其中的数据结构类似下表：
| Selector | IMP |
|:-------------:|:-------------:|
| SelectorA | IMPA |
| SelectorB | IMPB |
| SelectorC | IMPC |
| Selector... | IMP... |
其中Selector是方法的编号变量，IMP是对应的实现函数指针。

现在我们通过买单的例子讲一下整体的流程：
当向服务员发送买单消息为时：
 1. 先根据@Selector(买单)从服务员类的cache方法列表（cache methodLists）里去找；
 2. 找到了，则取得对应的IMP值，然后调用执行该函数；
 3. 没找到，就从服务员类的方法列表（methodLists）里找；
 4. 找到了则执行对应的IMP；还找不到，就到super class的方法列表里找，直到找到基类(NSObject)为止；
 5. 最后再找不到，就会进入动态方法解析和消息转发的机制。(这部分知识，以后再细谈)
![这里写图片描述](http://img.ptcms.csdn.net/article/201507/06/5599f2fb67847.jpg)
比如，服务员类：
| Selector | IMP |
|:-------------:|:-------------:|
| 点菜 | IMP点菜 |
| 拿餐具 | IMP拿餐具 |
| 拿餐巾纸 | IMP拿餐巾纸 |
| ... | IMP... |
如果没有找到，则将消息转发给父类；
比如，服务员类的父类是人类：
| Selector | IMP |
|:-------------:|:-------------:|
| 吃饭 | IMP吃饭 |
| 睡觉 | IMP睡觉 |
| 打豆豆 | IMP打豆豆 |
| 穿衣服 | IMP穿衣服 |
| 脱衣服 | IMP脱衣服 |
| ... | IMP... |

这里就涉及了一个问题，如果是类方法，那么它是存储在什么地方的呢？
没错，类方法是存储在类的元类方法列表中。
比如，服务员类有这样一个类方法：
```
服务员类 *服务员 = [服务员类 穿着女仆装的服务员];
```
那么，服务员类的元类，方法列表就是
| Selector | IMP |
|:-------------:|:-------------:|
| 穿着女仆装的服务员 | IMP穿着女仆装的服务员 |
| ... | IMP... |

#2.致知：通晓其实质
通过上面的格物，我们已经知道了Objective-C的世界构成与运转规则，那么结合以上的内容，我们本文的主角Method Swizzling的实质是什么呢？
没错，就是变换MethodList中Selector所对应的IMP！

在上一节消息机制的讲述中，找到Selector之后总是需要取对应IMP再调用执行该IMP。那么只要我们修改了MethodList中一个Selector对应的IMP，那么程序运行时，就会调用我们修改后的IMP，运行出我们想要的效果。

##2.1.系统函数
基于这样的逻辑，OC的objc/runtime.h单元中提供了一些函数来修改类的MethodList。

###交换方法的实现：
```
OBJC_EXPORT void method_exchangeImplementations(Method m1, Method m2) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);
```
该函数的功能是：交换两个方法的实现函数指针。

举例如下：
```
Method Method吃饭 = class_getInstanceMethod([服务员类 class], @selector*(吃饭));
Method Method脱衣服 = class_getInstanceMethod([服务员类 class], @selector*(脱衣服));
method_exchangeImplementations(Method吃饭, Method脱衣服);
```
执行之后，服务员类父类的MethodList如下：
| Selector | IMP |
|:-------------:|:-------------:|
| 吃饭 | IMP脱衣服 |
| 睡觉 | IMP睡觉 |
| 打豆豆 | IMP打豆豆 |
| 穿衣服 | IMP穿衣服 |
| 脱衣服 | IMP吃饭 |
| ... | IMP... |
那么当执行以下代码时……
```
服务员类 *服务员 = [服务员类 穿着女仆装的服务员];
[服务员 吃饭];
```

###新增方法：
```
/** 
OBJC_EXPORT BOOL class_addMethod(Class cls, SEL name, IMP imp, 
                                 const char *types) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);
```
该函数的功能是：给cls添加一个新的方法,若cls的MethodList（不包括父类的MethodList）中存在这个方法则添加失败返回NO。如果cls的父类中存在SEL方法，那么执行此函数后就会在cls的MethodList中添加一个同名的方法覆盖父类的实现。

###设置方法的函数实现：
```
OBJC_EXPORT IMP method_setImplementation(Method m, IMP imp) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);
```
该函数的功能是：设置方法的新的实现函数指针，返回值是老的实现函数指针。

###替换方法：
```
OBJC_EXPORT IMP class_replaceMethod(Class cls, SEL name, IMP imp, 
                                    const char *types) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);
```
该函数的功能是：若cls的MethodList（不包括父类的MethodList）中存在这个方法则调用method_setImplementation；否则调用class_addMethod；

#3.致用：理论结合实践
讲到这里，我们对于MethodSwizzling技术以及其实现原理就有了一个基本完整的了解，算是完成了格物致知这个学习过程。
不过无论什么技术，学习最终都是为了应用，即学以致用，而Method Swizzling也不例外。所以我们接下来要思考一下为什么要用Method Swizzling，和最终要怎么去使用这项技术。

##why
如果要用一句话解释Method Swizzling，那么就是：在运行期修改一个类中的方法实现。
于是问题来了，如果要修改方法实现的话，继承、类别等技术也可以实现，为什么要用Method Swizzling呢？
 - 避免多次继承；
例如，在项目中，框架会提供一个VC基类，然后项目引用了框架之后会在此基础上再声名一个VC基类，一个类型的业务模块可能又会在此基类的基础上再抽象出一个基类。
而实际上，这些继承链中的很多代码逻辑都是与业务无关的。如果使用MethodSwizzling技术，则可以替换多重的继承体系。
 - 修复系统BUG；
当系统提供的类存在BUG时，你无法直接修改系统类的代码；如果定义一个子类又会导致大量的代码修改；如果使用类别，又无法调用该方法的原函数实现。
这时候使用方法变换技术可以在不修改业务代码的基础上替换系统类的方法实现，并且保证新的函数实现会调用原方法实现。
 - 避免修改大量代码；
比如使用友盟统计页面的进入和退出次数，正常情况下需要在每个VC的viewWillAppear:和viewWillDisappear:方法中添加统计代码。这样即便你可以直接修改类的代码，修改的工作量也比较大。
如果使用方法变换技术，那么可以替换UIViewController的方法实现，通过Class名称来进行页面统计。
 - 增加调试日志；
这个场景更加暴力，如果想要给方法增加开始和结束日志，挨个方法的去改肯定非常的麻烦。
 - 其他AOP场景（权限、缓存、性能统计……）；
其实像友盟统计、调试日志这些场景都属于AOP需要解决的问题范畴，这些问题具有通用性，且与业务逻辑无关。所以，所有AOP编程所要解决的问题，在Objective-C基本上都需要通过方法变换来完成。

##what
现在我们需要思考的一个问题是：如果我们需要使用方法变换技术，那么站在使用者的角度上，这个技术是什么样的？
 - 根本目的：替换一个类中的方法实现；
 - 兼容性：替换后是否需要调用原来的实现；
 - 顺序：新增的代码和原方法实现的调用顺序；在开始，中间或者最后调用原方法实现；
 - 多样性：需要考虑子类实例方法替换、子类实例方法新增、继承自父类的实例方法替换；子类类方法替换、子类类方法新增、继承自父类的类方法替换；
 - 影响范围：子类替换继承自父类的方法时，是否会影响其他继承自此父类的类功能；同一继承链条上的多个类，同时替换方法实现时，如何保证最终的替换效果；

##how
在我们思考怎么根据方法变换技术满足使用者的要求之前，我们必须考虑一下这个技术的缺点。所谓“未虑胜，先虑败”，这样才能立于不败之地。
具体的问题可见参考资料1。
不过原文中有一个问题是没有提出的，即：
**对同一个类的同一方法，多次进行方法变换的顺序问题。**
这个问题其实是很重要的，如原文所讲，我们在声明类别来存放新的方法实现，而在这个类别的+load;方法中进行方法变换。
那么考虑到框架的情况，那么就有可能存在多个方法变换的类别（框架里声明了一个类别，项目里面又声明了一个类别）；那这个时候，这两个类别的执行顺序是按编译顺序执行的，我们无法控制确保其执行顺序。

若是AOP编程，每个Aspect之间互相独立，是没有调用顺序要求的。所以该问题影响并不大。
但是对于框架而言，这个问题就比较大了。比如框架的类别中将view的背景色设置为白色，项目的类别中将view的背景色设置为灰色。那么不同的加载顺序会导致不同的执行顺序，最终的效果也不一样。
要解决该问题，那么就不能在类别的`+load;`方法中进行隐式的方法变换，而是在APP启动时进行显式的调用。

基于使用者的要求，可以有两种不同的实现（封装以达到直接可用的目的）思路：
 - 类hook方式；
hook，钩子，即将自定义代码钩在指定方法身上，和指定方法同时执行；
优点：使用者使用方便；
缺点：不够灵活；实现该方案需要大量的编码；

 - 类继承方法；
继承，即默认覆盖原实现方法，但可以通过例如`[super viewDidLoad];`的显式代码调用来调用方法的原实现；
优点：灵活，可在自己的逻辑中控制是否执行原实现；编码工作量少；
缺点：使用者需要写更多的代码；

如果只是这样比较的话大家可能还无法明白其间的差别，我们以界面统计和权限控制场景来体会一下其中的差别。
在界面统计的场景中，我们的要求其实很简单：在每次调用viewWillAppear:方法前根据Class调用一下对应的统计代码。这个场景必然需要调用原方法实现。所以使用hook方案的用户体验会更好。
在权限控制的场景中，我们要求用户在点击功能按钮时需要校验当前用户是否已登录，若未登录则弹出登录界面，已登录则调用原方法实现。那么hook方案其实是无法满足场景要求的（参考资料1中虽然说的是hook方案，其实现方式实际上还是类似于继承的方案）。

基于时间的考虑，我本次采用的是类继承的方案。
采用此方案，我们需要满足以下几点：

 1. 支持新增方法，替换方法实现；
 2. 替换方法实现时，需要保存方法的原实现指针，新的方法实现中可以显式的调用此实现指针；
 3. 需要支持实例方法和类方法；

以上三点是我们作为技术方案的提供者可以控制的，至于以下四点则需要使用者注意：

 1. 若小规模的使用方法变换功能，则需要在类别的`+load;`方法中进行方法变换；若大规模使用，则要在APP启动时，显式的调用方法变换类方法；
 2. 进行方法变换时，需要使用dispatch_once进行控制只执行一次；
 3. 多个有继承关系的同时进行方法变换时，先从父类开始；
 4. 在方法变换时，新方法的声明注意使用前缀，避免命名冲突；

具体实现请看DEMO，最终NSObject+Swizzle.h提供的接口如下：
```
/**
 *  通过新的方法名调配当前类的实例方法，支持新增方法
 *
 *  @param originalSelector 需要调配的方法名
 *  @param replacement      新的方法名
 *  @param store            存储原实现IMP的指针
 *
 *  @return 是否成功
 */
+ (BOOL)swizzleInstanceMethod:(SEL)aOriginalSelector withReplacement:(SEL)aReplacementSelector andStore:(IMPPointer)aStorePointer;

/**
 *  通过新的方法名调配当前类的类方法，支持新增方法
 *
 *  @param originalSelector 需要调配的方法名
 *  @param replacement      新的方法名
 *  @param store            存储原实现IMP的指针
 *
 *  @return 是否成功
 */
+ (BOOL)swizzleClassMethod:(SEL)aOriginalSelector withReplacement:(SEL)aReplacementSelector andStore:(IMPPointer)aStorePointer;
```

#参考资料
1.[Objective-C的hook方案（一）: Method Swizzling](http://blog.csdn.net/yiyaaixuexi/article/details/9374411)
2.[Objective-C method及相关方法分析](http://blog.csdn.net/uxyheaven/article/details/40350911)
3.[深入理解Objective-C：Category](http://tech.meituan.com/DiveIntoCategory.html)
