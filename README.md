## 背景

<p align="center">

<img src="Images/logo.jpeg" alt="Debug" title="Debug"/>

</p>


> Debugging has a rather bad reputation. I mean, if the developer had a complete understanding of the program, there wouldn’t be any bugs and they wouldn’t be debugging in the first place, right?<br/>Don’t think like that.<br/>There are always going to be bugs in your software — or any software, for that matter. No amount of test coverage imposed by your product manager is going to fix that. In fact, viewing debugging as just a process of fixing something that’s broken is actually a poisonous way of thinking that will mentally hinder your analytical abilities.<br/>Instead, you should view debugging **as simply a process to better understand a program**. It’s a subtle difference, but if you truly believe it, any previous drudgery of debugging simply disappears.

正如《Advanced Apple Debugging & Reverse Engineering》一书前言所述：开发者不要妄图认为如果能充分了解软件的工作方式，就不会存在 Bug，事实上，任何软件中都存在 Bug。

在做登录 SDK 开发的过程中，产线接到用户反馈，在点击登录页面的 QQ 图标的时候出现应用闪退的情况，试图重现的过程中发现是在用户手机未安装 QQ 的情况下，使用 QQ 登录的时候回去拉起 QQ Web 授权页，但此时会出现 `[TCWebViewController setRequestURLStr:]` 找不到 selector 的情况。

> 注意：为了更好的讲解，下面所有涉及到具体业务，与本主题无关的地方没有列出来，同时应用名称用 **AADebug** 代替。

应用崩溃的堆栈信息如下：

```
Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[TCWebViewController setRequestURLStr:]: unrecognized selector sent to instance 0x7fe25bd84f90'
*** First throw call stack:
(
	0   CoreFoundation                      0x0000000112ce4f65 __exceptionPreprocess + 165
	1   libobjc.A.dylib                     0x00000001125f7deb objc_exception_throw + 48
	2   CoreFoundation                      0x0000000112ced58d -[NSObject(NSObject) doesNotRecognizeSelector:] + 205
	3   AADebug                             0x0000000108cffefc __ASPECTS_ARE_BEING_CALLED__ + 6172
	4   CoreFoundation                      0x0000000112c3ad97 ___forwarding___ + 487
	5   CoreFoundation                      0x0000000112c3ab28 _CF_forwarding_prep_0 + 120
	6   AADebug                             0x000000010a663100 -[TCWebViewKit open] + 387
	7   AADebug                             0x000000010a6608d0 -[TCLoginViewKit loadReqURL:webTitle:delegate:] + 175
	8   AADebug                             0x000000010a660810 -[TCLoginViewKit openWithExtraParams:] + 729
	9   AADebug                             0x000000010a66c45e -[TencentOAuth authorizeWithTencentAppAuthInSafari:permissions:andExtraParams:delegate:] + 701
	10  AADebug                             0x000000010a66d433 -[TencentOAuth authorizeWithPermissions:andExtraParams:delegate:inSafari:] + 564
………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………

省略若干无关行	

………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………………
236
	14  libdispatch.dylib                   0x0000000113e28ef9 _dispatch_call_block_and_release + 12
	15  libdispatch.dylib                   0x0000000113e4949b _dispatch_client_callout + 8
	16  libdispatch.dylib                   0x0000000113e3134b _dispatch_main_queue_callback_4CF + 1738
	17  CoreFoundation                      0x0000000112c453e9 __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__ + 9
	18  CoreFoundation                      0x0000000112c06939 __CFRunLoopRun + 2073
	19  CoreFoundation                      0x0000000112c05e98 CFRunLoopRunSpecific + 488
	20  GraphicsServices                    0x0000000114a13ad2 GSEventRunModal + 161
	21  UIKit                               0x0000000110d3f676 UIApplicationMain + 171
	22  CCTalk                              0x0000000108596d3f main + 111
	23  libdyld.dylib                       0x0000000113e7d92d start + 1
)
libc++abi.dylib: terminating with uncaught exception of type NSException
```

## 排查过程

根据上面出错信息中的 `TCWebViewController` 很自然想到与腾讯的 SDK **TencentOpenAPI.framework** 有关，但是产线的应用出现问题的时间段内没有更新腾讯的 SDK，所以应该不是直接由 **TencentOpenAPI.framework** 导致应用崩溃的。

首先通过反编译工具拿到 `TCWebViewController` 类的结构

```
  ; @class TCWebViewController : UIViewController<UIWebViewDelegate, NSURLConnectionDelegate, NSURLConnectionDataDelegate> {
                                       ;     @property webview
                                       ;     @property webTitle
                                       ;     @property requestURLStr
                                       ;     @property error
                                       ;     @property delegate
                                       ;     @property activityIndicatorView
                                       ;     @property finished
                                       ;     @property theData
                                       ;     @property retryCount
                                       ;     @property hash
                                       ;     @property superclass
                                       ;     @property description
                                       ;     @property debugDescription
                                       ;     ivar _nloadCount
                                       ;     ivar _webview
                                       ;     ivar _webTitle
                                       ;     ivar _requestURLStr
                                       ;     ivar _error
                                       ;     ivar _delegate
                                       ;     ivar _xo
                                       ;     ivar _activityIndicatorView
                                       ;     ivar _finished
                                       ;     ivar _theData
                                       ;     ivar _retryCount
                                       ;     -setError:
                                       ;     -initWithNibName:bundle:
                                       ;     -dealloc
                                       ;     -stopLoad
                                       ;     -doClose
                                       ;     -viewDidLoad
                                       ;     -loadReqURL
                                       ;     -viewDidDisappear:
                                       ;     -shouldAutorotateToInterfaceOrientation:
                                       ;     -supportedInterfaceOrientations
                                       ;     -shouldAutorotate
                                       ;     -webViewDidStartLoad:
                                       ;     -webViewDidFinishLoad:
                                       ;     -webView:didFailLoadWithError:
                                       ;     -webView:shouldStartLoadWithRequest:navigationType:
                                       ; }
                     _OBJC_CLASS_$_TCWebViewController:
```

静态分析的结果发现 `TCWebViewController` 类中确实没有 `requestURLStr` 的 Setter 和 Getter，因为之前版本没有出现崩溃。

此时产生一个想法：`TCWebViewController` 类中的 Property 会不会像 **Core Data** 框架一样，使用 `@dynamic` 告诉编译器不做处理，然后 Getter 和 Setter 方法是在运行时动态创建。于是带着这个猜想继续查下去，发现在 **TencentOpenAPI.framework** 中有个 `NSObject` 的 Category：`NSObject(MethodSwizzlingCategory)` 非常可疑，其中 `switchMethodForCodeZipper:` 方法分别将消息转发过程中的 `methodSignatureForSelector` 和 `forwardInvocation`方法替换为 `QQmethodSignatureForSelector` 和 `QQforwardInvocation`。

```objective-c
void +[NSObject switchMethodForCodeZipper](void * self, void * _cmd) {
    rbx = self;
    objc_sync_enter(self);
    if (*(int8_t *)_g_instance == 0x0) {
            [NSObject swizzleMethod:@selector(methodSignatureForSelector:) withMethod:@selector(QQmethodSignatureForSelector:)];
            [NSObject swizzleMethod:@selector(forwardInvocation:) withMethod:@selector(QQforwardInvocation:)];
            *(int8_t *)_g_instance = 0x1;
    }
    rdi = rbx;
    objc_sync_exit(rdi);
    return;
}
```

于是将视线转移到 `QQmethodSignatureForSelector` 中，发现在其中有个方法：`_AddDynamicPropertysSetterAndGetter`，从方法名称很容易知道这个方法是动态地给属性添加 Setter 和 Getter 方法。基本验证了 `TCWebViewController` 类中的 Property 的 Setter 和 Getter 方法是在 Runtime 动态添加。

```objective-c
void * -[NSObject QQmethodSignatureForSelector:](void * self, void * _cmd, void * arg2) {
    r14 = arg2;
    rbx = self;
    rax = [self QQmethodSignatureForSelector:rdx];
    if (rax == 0x0) {
            rax = sel_getName(r14);
            _AddDynamicPropertysSetterAndGetter();
            rax = 0x0;
            if (0x0 != 0x0) {
                    rax = [rbx methodSignatureForSelector:r14];
            }
    }
    return rax;
}
```

那究竟为什么 `TCWebViewController` 找不到 Setter 的 Selector 呢？是否在开发新版本的过程覆盖了 `QQmethodSignatureForSelector` 导致的呢？然而在搜遍项目的所有角落，并没有发现项目中替换有 `NSObject` 的 `methodSignatureForSelector`，问题有点棘手，分析到这一步，静态分析暂时告一段落，下一步将使用 LLDB 来动态调试腾讯的三方库，从而找出是哪一个环节破坏了消息转发过程中动态生成 Getter 和 Setter。

> 这里其实如果通过 LLDB 命令给 `setRequestURLStr` 方法打断点，会发现不能成功打上这个断点，原因也是因为 Setter 方法其实在编译时还没有，也能作为上面猜想的佐证。

根据崩溃堆栈中可以推测出 `setRequestURLStr` 是在 `-[TCWebViewKit open]` 方法中调用的，也就是腾讯的 SDK 发现没有安装 QQ 时，会打开 QQ 的 Web 授权页。

我们使用 LLDB 在这个方法下断点，命令如下：

```
br s -n "-[TCWebViewKit open]"
```
断点成功打上。

```
Breakpoint 34: where = AADebug`-[TCWebViewKit open], address = 0x0000000103157f7d
```

当应用准备跳到 Web 授权页的时候，断点会被断住，会看到下图

<p align="center">

<img src="Images/lldb_webviewkit_open.png" />

</p>

在下图 96 行的汇编代码打一个断点，这条汇编代码就是调用 `setRequestURLStr` 方法，然后打印出 `rbx` 寄存器的内容，可以观察到 `rbx` 保存的就是 `TCWebViewController` 实例。

<p align="center">

<img src="Images/lldb_webviewkit_open_1.png" />

</p>











