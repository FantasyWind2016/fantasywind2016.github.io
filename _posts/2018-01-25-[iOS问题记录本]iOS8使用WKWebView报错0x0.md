# iOS问题记录本：iOS8/Swift/WKWebView/address=0x0错误

## 0、背景描述
前一段时间，公司有个小的新项目，因为考虑到项目本身没有任何技术要求，所以决定采用Swift加上WKWebView的方案进行开发；而且前一段时间团队内部也进行了基础开发库Swift化的工作，这个项目也可以用来检验之前的工作。

当然，吃螃蟹必然是要付出一些代价的，不过因为项目本身的规模很小，所以Swift语言引起的问题并不多，最麻烦的地方在于WKWebView，其中最麻烦的一个是WKWebView的JS交互阻塞AJAX请求的问题，不过这并不是本文叙述的要点，就不详述了。

项目测试快结束的时候，测试人员在进行兼容性测试时，发现在iOS8系统上启动就闪退，我调试发现原因是Swift基础库pod中依赖了Contacts系统framework，而这个framework是iOS9系统之后才有的，所以启动必闪退。
解决了这个问题后，我运行发现应用在iOS8系统上还是闪退，断点停留在AppDelegate上，报错信息为：
``` C
EXC_BAD_ACCESS(code=1,address=0x0)
```

## 1、问题查找
因为闪退时断点停留在AppDelegate上，且没有更详细的报错信息，所以无法使用常规的方法查找问题所在。

不过类似的问题还是遇到过的，比如`EXC_BAD_ACCESS(code=1,address=0x10)`这个问题，它是block释放后被执行造成的闪退问题，所以我怀疑现在这个问题也类似，但是在网上无法搜索到类似的信息，StackOverFlow上有人提问，但没有最终结果。

所以最后只能采用笨办法——排除法，把系统启动时执行的代码挨个注释掉查看效果。最终定位到问题所在是为WebView设置UserAgent的代码：
``` Objective-C
// 获取默认User-Agent
let wkWebView = WKWebView()
wkWebView.evaluateJavaScript("navigator.userAgent") { (result, error) in
    var oldAgent = ""
    if result is String {
        oldAgent = result as! String
        if oldAgent.count > 0 {
            oldAgent += " "
        }
    }
    let newAgent = oldAgent + "app-iOS"
    // 设置global User-Agent
    UserDefaults.standard.register(defaults: ["UserAgent":newAgent])
    UserDefaults.standard.synchronize()
}
```

## 2、原因分析
经过测试，即便是在一个空白的Swift工程里进行最简单的调用，如：
``` Objective-C
WKWebView().evaluateJavaScript("") { (result, error) in }
```

同样会导致闪退。

而该方法的回调参数不传时`WKWebView().evaluateJavaScript("")`则不会报错。

我们上一节说过有一个类似的问题，而`address=0x10`问题的原因在是block的结构体寻址问题：
``` C
//__block_imp：  这个是编译器给我们生成的结构体，每一个block都会用到这个结构体
struct __block_impl {
  void *isa;　　　　　　　　　//类型
  int Flags;　　　　　　　　  //标识字段
  int Reserved;　　　　　　　//保留字段　　　　　　　
  void *FuncPtr;　　　　　　 //函数指针，这个会指向编译器给我们生成的下面的静态函数__main_block_func_0
};
```

0x10是十六进制，也就是struct基地址后的第16个字节，其中void *类型占8个字节，int类型占4个字节，所以0x10的地址就是FuncPtr的地址，而address=0x10的问题也正是对值为nil的block强行调用导致的。

那么，举一反三来看，`address=0x0`的问题，0x0地址就是指向isa函数指针字段的地址，这个错误发生的原因就是调用isa造成的。

WKWebView执行JavaScript时为何调用isa我们暂不去考虑，已经可以确定如果block的值为nil时，调用isa字段，是会造成`address=0x0`问题的。所以，下面我们需要考虑的就是block为什么变成nil？

在本节开头贴的简单代码中，有几点是需要注意的：

- WKWebView()会生成一个临时变量；
- evaluateJavaScript方法执行时是异步的；
- 异步执行时，尾随闭包可能是直接或者间接被WKWebView的实例持有的；

那么，block的值变为nil的原因应该是WKWebView的临时变量实例被自动释放，所以block变量也随同被释放了。

经过多次测试也发现，该问题并非是100%重现的，我在iPhone6 plus测试机上运行时，有90%左右的重现几率。而这一现象如果用autoReleasePool来解释也是通的。

## 3、解决方案

这个问题最好的解决方法当然是让WKWebView在执行JavaScript脚本时，对block变为nil的情况进行容错处理；但是很抱歉，这需要苹果爸爸去修改，而且它们可能也无法修复iOS8这种低版本系统的问题。

那么头痛医头脚痛医脚，既然问题的原因可能是block被释放导致的，那就让它不被释放好了。

于是，我让在类里面直接声明并持有闭包变量，然后调用WKWebView方法的时候传递闭包变量进去，可是然并卵！！应用还是闪退了！

然后我就意识到我犯蠢了，作为参数传递的闭包变量，在WKWebView内部一般都要copy一份，所以我直接持有闭包变量并无法影响WKWebView内部闭包变量的释放。

最终还是将WKWebView声明为类的属性，让类持有它，这样它就不会因为是临时变量而被autoReleasePool所释放。

最后测试，效果上OK，闪退问题没有重现。

## 4.说明

最终的解决方案只是基于对问题原因的猜测，至于问题的原因是否确实如此还没有时间深入探寻，所以如果最终并非如此而对其他人造成误导，还请大家谅解。如果文章中有错误，还请多多指点。