<p align="center">

<img src="Images/logo.jpeg" alt="Debug" title="Debug"/>

</p>

## Preface

> Debugging has a rather bad reputation. I mean, if the developer had a complete understanding of the program, there wouldn’t be any bugs and they wouldn’t be debugging in the first place, right?<br/>Don’t think like that.<br/>There are always going to be bugs in your software — or any software, for that matter. No amount of test coverage imposed by your product manager is going to fix that. In fact, viewing debugging as just a process of fixing something that’s broken is actually a poisonous way of thinking that will mentally hinder your analytical abilities.<br/>Instead, you should view debugging **as simply a process to better understand a program**. It’s a subtle difference, but if you truly believe it, any previous drudgery of debugging simply disappears.

Since Grace Hopper, the founder of the **Cobol** language, discovered the world's first Bug in a relay computer, the generation of Bug in software development has never stopped. As the preface to the book of《Advanced Apple Debugging & Reverse Engineering》tells us: Developers don't want to think that if there is a good understanding of how software works, there will be no Bug. Therefore, debugging is almost a inevitable phase in the software development life cycle.

## Debugging Overview

If you ask an inexperienced programmer
 about how to define debugging, he might say "Debugging is something you do to find a solution for your software problem". He is right, but that's just a tiny part of a real debugging.

Here are the steps of a real debugging:
1. Find out why it's behaving unexpectly
2. Resolve it
3. Try to make sure no new issue is involved
4. Improve the quality of your code, include readability, architecture, test coverage and performance etc.
5. Make sure that the same problem does not occur anywhere else

Among above steps, the most important step is the first step: find out the problem. Apparently, it's a prerequisite of other steps.

Research shows the time experienced programmers spend on debugging to locate the same set of defects is about one twentieth of inexperienced programmers. That means debugging experience makes an enormous difference on programming efficiency. We have lots of books on software design, unfortunately, rare of them have introduction about debugging, even the courses in school.

As the debugger improving over years, the programmers' coding style are changed thoroughly. But still，a good debugger cannot replace good debugging thought. In contrast, excellent debugging thought is not enough without good debugger. The perfect thing is we have both of them.

The following graph is the nine debugging rules described in book <Debugging: The 9 Indispensable Rules for Finding Even the Most Elusive Software and Hardware Problems>.

<p align="center">

<img src="Images/debug_rules_en.jpeg" width="500" />

</p>

## Case

This article illustrates the procedure of debugging through a real case. Some of the details are changed to protect personal privacy.

### Issue
The issue we are going to talk about was happening when I was developing a login SDK. One user claimed the app crashed when he pressed the "QQ" button in login page. As we debugged this issue, we found the crash happend if the QQ app was not installed at the same time. When user presses QQ button to require a login, the QQ login SDK tries to launch an authorization web page in our app. In this case, an unrecognized selector error `[TCWebViewController setRequestURLStr:]` occurs.

> P.S: To focus on the issue, the unneccessary bussion debug information are not listed below. Meanwhile **AADebug** is used as our app name. 

Here is the stack trace of this crash:
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

Lines of irrelevant information are removed here

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
	22  AADebug                             0x0000000108596d3f main + 111
	23  libdyld.dylib                       0x0000000113e7d92d start + 1
)
libc++abi.dylib: terminating with uncaught exception of type NSException
```
### Message Forwarding

Before talking about the debugging, let's get familiar with the message forwarding in Objective-C. As we know Objective-C uses a messaging structure rather than function calling. The key difference is that in the messaging structure, the runtime decides which function will be executed not compiling time. That means if an unrecognized message is sent to one object, nothing will happen during compiling time. And during runtime, when it receives a method that it doesn't understand, an object goes through message forwarding, a process designed to allow you as the developer to tell the message how to handle the unknown message.

Below four methods are usually involved during message forwarding:

1. `+ (BOOL)resolveInstanceMethod:(SEL)sel`: this method is called when an unknown message is passed to an object. This method takes the selector that was not found and return a Boolean value to indicate whether an instance method was added to the class that can now handle that selector. If the class can handle this selector, return Yes, then the message forward process is completed. This method is often used to access @dynamic properties of NSManagedObjects in CoreData in a dynamically way. `+ (BOOL)resolveClassMethod:(SEL)sel` method is similar with above method, the only difference is this one class method, the other is instance method.

2. `- (id)forwardingTargetForSelector:(SEL)aSelector`：This method provides a second receiver for handling unknow message, and it's faster than `forwardInvocation:`. This method can be used to imitate some features of multiple inheritance. Note that there is no way to manipulate the mssage using this part of the fowarding path. If the message needs to be altered before sending to the replacement receiver, the full forwarding mechanism must be used.
3. `- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`：If the forwarding algorithm has come this far, the full forwarding mechanism is started. `NSMethodSignature` is returned by this method which includes method description in aSelector paramter. Note that this method needs to be overrided if you want to create a `NSInvocation` object which contains selector, target, and arguments during the message forwarding.
4. `- (void)forwardInvocation:(NSInvocation *)anInvocation`：The implemention of this method must contains below parts: Find out the object which can handle anInvocation message; Sending message to that object, the anInvocation saves the return value, runtime then sends the return value to the original message sender. In fact, this method can have the same behavior with `forwardingTargetForSelector:` method by simply changing the invocation target and invoking it afterwards, but we barely do that.

Usually, the first two methods used for message forwarding are called as **Fast Forwarding**, becuase it provides a much faster way to do the message forwarding. To distinguish from the fast forwarding, method 3 and 4 are called as **Normal Forwarding** or **Regular Forwarding**. It's much slower because it has to create **NSInvocation** object to complete the message forwarding.

> Note: If `methodSignatureForSelector` method is not overwrided or the returned `NSMethodSignature` is nil, `forwardInvocation` will not be called, and the message forwarding is terminated with `doesNotRecognizeSelector` error raised. We can see it from the `__forwarding__` function's source code below.

The process of message forwarding can be described by a flow diagram, see below.

<p align="center">

<img src="Images/message_forward_en.png" />

</p>

Like described in the flow diagram, at each step, the receiver is given a chance to handle the message. Each step is more expensive than the one before it. The best practice is to handle the message forwarding process as early as possible. If the message is not handled through the whole process, `doesNotRecognizeSeletor` error is raised to state the selector cannot be recognized by the object.

### Debugging Process

It's time to finish the theory part and move back to the issue.

According to the `TCWebViewController` information from the trace stack, we naturally associate it with the Tencent SDK **TencentOpenAPI.framework**, but we didn't update the Tencent SDK recently which means the crash was not caused by **TencentOpenAPI.framework**.

First, we decompiled the code and got the struct of the `TCWebViewController` class

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
From the static analysis result, there was no Setter and Getter method for `requestURLStr` in `TCWebViewController`. Because there was no such crash in previous app version, we came out an idea: would the property in `TCWebViewController` be implemented in a dynamic way which uses `@dynamic` to telll the compiler not generate getter and setter for the property during compiling time but dynamically created in runtime like **Core Data** framework? Then we decided to going deeply of the idea to see if our guess was correct. During our tracking, we found there was a category `NSObject(MethodSwizzlingCategory)` for `NSObject` in **TencentOpenAPI.framework** which was very suspicious. In this category, there was a method `switchMethodForCodeZipper` whose implemention replaced the `methodSignatureForSelector` and `forwardInvocation` methods to `QQmethodSignatureForSelector` and `QQforwardInvocation` methods.


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

Then we kept tracking into `QQmethodSignatureForSelector` method, and there was a method named `_AddDynamicPropertysSetterAndGetter` in it. From the name, we can easily get that this method is to add Setter and Getter method for properties dynamically. This found can substantially verify our original guess is correct. 

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
But why the setter cannot recognized in `TCWebViewController` class? Is it because the `QQMethodSignatureForSelector` method was covered during our development of this version? However we couldn't find a clue even we went through everywhere in the code.
That was very disappointing. So far the static analysis is done. Next step is using LLDB to dynamically debug the Tencent SDK to find out which path broke the creation of Getter and Setter in message forwarding process.

> If we try to set breakpoint on `setRequestURLStr` through LLDB command, we will find that we cannot make it. The reason is because the setter is not available during compiling time. This can also verify our original guess.

According to the crash stack trace, we can conclude `setRequestURLStr` is called in ` -[TCWebViewKit open]` method, which means the crash happens during Tencent SDK checking if the QQ app is installed and opening the authentication web page progress.

Then we use below LLDB command to set breakpoint on this method:

```
br s -n "-[TCWebViewKit open]"
```

> `br s` is the abbreviation for `breakpoint set`, `-n` represents set the breakpoint according to the method name afte it, which has the same behavior with symbolic breakpoint, `br s -F` can also set the breakpoint.  `b -[TCWebViewKit open]` also works here, but `b` here is the abbreviation of `_regexp-break`, which uses  regular expression to set the breakpoint. In the end, we can also set breakpoint on memory address like `br s -a 0x000000010940b24e`, which can help to debug block if the address of the block is available.

By now the breakpoint is set successfully.


```
Breakpoint 34: where = AADebug`-[TCWebViewKit open], address = 0x0000000103157f7d
```

When app is going to lanuch the web authentication page, the project is stopped on this breakpoint. Refer to below:

<p align="center">

<img src="Images/lldb_webviewkit_open.png" />

</p>

> This screenshot is captured when app runing on simulator, so the assembly code is based on X64. If you are using the iPhone device, the assembly code should be ARM. But the analysis method is the same for them, please notice it.

Set a breakpoint on Line 96, this assembly code is the `setRequestURLStr` method invocation, then print the content of `rbx` register, then we can observer that the `TCWebViewController` instance is saved in this register.

<p align="center">

<img src="Images/lldb_webviewkit_open_1.png" />

</p>

## 序言

> Debugging has a rather bad reputation. I mean, if the developer had a complete understanding of the program, there wouldn’t be any bugs and they wouldn’t be debugging in the first place, right?<br/>Don’t think like that.<br/>There are always going to be bugs in your software — or any software, for that matter. No amount of test coverage imposed by your product manager is going to fix that. In fact, viewing debugging as just a process of fixing something that’s broken is actually a poisonous way of thinking that will mentally hinder your analytical abilities.<br/>Instead, you should view debugging **as simply a process to better understand a program**. It’s a subtle difference, but if you truly believe it, any previous drudgery of debugging simply disappears.

从 **Cobol** 语言的创始人 Grace Hopper 在继电器式计算机中发现世界上第一个 Bug 开始，软件开发中 Bug 的产生就从未停止。正如《Advanced Apple Debugging & Reverse Engineering》一书前言所述：开发者不要妄图认为如果能充分了解软件的工作方式，就不会存在 Bug，事实上，任何软件中都存在 Bug。所以在软件开发周期中，Debugging 几乎是一个无法避免的环节。

## 调试概述

如果你问一个经验不丰富的程序员该如何定义调试，他也许会回答你调试就是找出解决问题的方案。事实上，这只是调试中目标的一小部分，甚至都不算是最重要的一部分。
有效的调试需要如下步骤：

1. 找出为什么软件没有按照期望的行为运行
2. 解决问题
3. 避免引发其他问题
4. 提升代码的整体质量，包括可读性、架构、测试覆盖率、性能等方面
5. 确保类似问题不会在其他地方再次出现

在上面步骤中，最重要是第一步——找出导致问题的根源，这是后面其他步骤的先决条件。

研究表明经验丰富的程序员调试找出 Bug 的所用的时间大约是缺乏经验的程序员的 1/20。经验丰富的程序员与缺乏经验的程序员之间存在巨大的调试效率的差异。不幸的是，有很多关于软件设计的书籍，但深入讲解调试的却比较少，学校的课程中几乎也看不到关于调试的内容。

这些年来，调试器在不断在发展，也彻底改变了程序员们的编程方式。当然调试器无法代替良好的思维，思维也无法替代优秀的调试器，最完美的组合就是优秀的调试器加上良好的思维。

下图是《调试九法:软硬件错误的排查之道》一书提及的九大调试规则。
<p align="center">

<img src="Images/debug_rules.jpeg" width="500" />

</p>

## 案例

文章通过一个真实的“案例故事”来描述调试过程，有些细节被我改动了，以便保护个人隐私。

### 问题

故事发生我在做登录 SDK 开发的过程中，产线接到用户反馈，在点击登录页面的 QQ 图标的时候出现应用闪退的情况，试图重现的过程中发现是在用户手机未安装 QQ 的情况下，使用 QQ 登录的时候会去拉起 QQ Web 授权页，但此时会出现 `[TCWebViewController setRequestURLStr:]` 找不到 selector 的情况。

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
	22  AADebug                             0x0000000108596d3f main + 111
	23  libdyld.dylib                       0x0000000113e7d92d start + 1
)
libc++abi.dylib: terminating with uncaught exception of type NSException
```


### 消息转发

在开始调试之前，准备用一些篇幅讲解 Objective-C 中的消息转发（message forwarding）机制。熟悉 Objective-C 的读者应该清楚，该语言使用的是“消息结构”，而非像 C 语言的“函数调用”，如果在编译期间向对象发送了其无法解读的消息并没什么大碍，因为当运行时期间对象接收到无法解读的对象后，它可以通过开启消息转发机制来做一些补救措施，具体来说就是由程序员来告诉对象应该如何处理这条未知消息。

消息转发中通常会涉及到下面四个方法：

1. `+ (BOOL)resolveInstanceMethod:(SEL)sel`：对象收到未知消息后，首先会调用该方法，参数就是未知消息的 selector，返回值则表示能否新增一个实例方法处理 selector 参数。如果这一步成功处理了 selector 后，返回 `YES`，后续的转发机制不再进行。事实上，这个被经常使用在要访问 **CoreData** 框架中的 NSManagedObjects 对象的 @dynamic 属性中，以动态的插入存取方法。<br/>`+ (BOOL)resolveClassMethod:(SEL)sel`：和上面方法类似，区别就是上面是实例方法，这个是类方法。
2. `- (id)forwardingTargetForSelector:(SEL)aSelector`：这个方法提供处理未知消息的备援接收者，这个比 `forwardInvocation:` 标准转发机制更快。通常可以用这个方案来模拟多继承的某些特性。这一步我们无法操作转发的消息，如果想修改消息的内容，则应该开启完整的消息转发流程来实现。
3. `- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`：如果消息转发的算法执行到这一步，代表已经开启了完整的消息转发机制，这个方法返回 `NSMethodSignature` 对象，其中包含了指定 selector 参数中的有关方法的描述，在消息转发流程中，如果需要创建 `NSInvocation` 对象也需要重写这个方法，`NSInvocation` 对象包含了 SEL、Target 和参数。
4. `- (void)forwardInvocation:(NSInvocation *)anInvocation`：方法的实现通常需要完成以下任务：找出能够处理 `anInvocation` 对象封装的消息的对象；使用 `anInvocation` 给前面找出的对象发送消息，`anInvocation` 会保存返回值，运行时会将返回值发送给原来的 sender。其实通过简单的改变调用目标，然后在改变后的目标上调用，该方法就能实现与 `forwardingTargetForSelector:` 一样的行为，然而基本不这样做。

通常将1和2的消息转发称为 **Fast Forwarding**，它提供了一种更为简便的方式进行消息转发，而为了与 **Fast Forwarding** 区分，3和4的消息转发被称之为 Normal Forwarding 或者 Regular Forwarding。 Normal Forwarding 因为要创建 **NSInvocation** 对象，所以更慢一些。

> 注意：如果 `methodSignatureForSelector` 方法返回的 `NSMethodSignature` 是 nil 或者根本没有重写 `methodSignatureForSelector`，则 `forwardInvocation` 不会被执行，消息转发流程终止，抛出无法处理的异常，这个在下文 `___forwarding___`函数的源码中可以看出。

下面这张流程图清晰地阐述了消息转发的流程。

<p align="center">

<img src="Images/message_forward.png" />

</p>

正如消息转发的流程图描述的，对象在上述流程的每一步都有机会处理消息。然而上文已经提到，消息转发流程越往后，处理消息所付出的代价也就越大。所以若非必要，应当尽早结束消息转发流程。如果消息转发的流程中都没有处理未知消息，最终会调用 `doesNotRecognizeSelector:` 抛出异常，表示对象无法正确识别此 SEL。


### 汇编语言

汇编语言是面向机器的程序设计语言，可以将其看成是各种 CPU 的机器指令的助记符集合。程序员可以使用汇编代码直接控制硬件系统工作，而且用汇编语言编写的程序具备执行速度快和占用内存少等优点。

在 Apple 平台上主流的汇编语言有 x86 汇编 和 ARM 汇编，在移动设备上使用的是 ARM 汇编，这主要是因为 ARM 采用的是 RISC 架构，具备功耗低的优势，而桌面平台使用的则是 x86 汇编。iOS 模拟器的程序实际就是以 iOS 模拟器作为容器，运行在该容器中的 Mac OS 程序，所以它使用的汇编也是 x86 汇编。由于我们的案例是在 iOS 模拟器中进行调试的，所以主要的研究目标是 x86 汇编。而 x86 汇编语言演变出两个语法分支: Intel（最初用于x86平台的文档中）和 AT&T，其中 Intel 语法在 MS-DOS 和 Windows 家族中占主导地位，而 AT&T 语法则常见于 UNIX 家族中。Intel 和 AT&T 汇编在语法上存在巨大的差异，这主要体现在变量、常量、寄存器访问、间接寻址和偏移量等方面。虽然两者语法上存在巨大差异，但所基于的硬件体系是相同的，因此可以将两者其一移植到另一种的汇编格式。

Intel 和 AT&T 汇编语法的差异主要有以下几个方面：

1. 操作数前缀：AT&T 汇编语法中的寄存器名称都以 `%` 作为前缀，立即操作数则以 `$` 作为前缀，而 Intel 汇编语法的寄存器和立即数均无前缀修饰。另外一个区别是 AT&T 汇编语法中的十六进制会加上 `0x` 前缀。下表给出了两者操作数前缀区别的示例：

	| AT&T | Intel |
	|:-------:|:-------:|
	| movq %rax, %rbx | mov rbx, rax |
	| addq $0x10, %rsp | add rsp, 010h |

2. 操作数方向：在 AT&T 汇编语法中，第一个操作数为源操作数，第二个操作数为目的操作数，Intel 汇编语法的操作数顺序正好相反。在这个方面 AT&T 汇编语法更接近人们日常的阅读习惯。
3. 寻址方式：与 Intel 汇编语法相比，AT&T 的间接寻址方式会显得更难读懂一些，但是二者地址计算的公式都是：`address = disp + base + index * scale`，其中 `base` 为基址，`disp` 为偏移地址，`index * scale` 决定了第几个元素，`scale` 为元素长度，只能为2的幂，`disp/base/index/scale` 全部都是可选的, `index` 默认为0，`size` 默认为1。最终 AT&T 汇编指令的格式是 `%segreg: disp(base,index,scale)`，Intel 汇编指令的格式是 `segreg: [base+index*scale+disp]`。上面两种格式中给出的其实是段式寻址，其中 `segreg` 是段寄存器，使用在实模式下，当 CPU 可以寻址空间的位数超过寄存器的位数时，例如 CPU 可以寻址20位地址空间时，但寄存器只有16位，为了达到20位的寻址，就需要使用 `segreg:offset` 的方式来寻址，计算出来的偏移地址是 `segreg * 16 + offset`，这种方式比平坦内存模式的寻址要复杂。在保护模式下，是在线性地址下进行寻址，不必考虑段基址。
4. 操作码的后缀：AT&T 汇编语法会在操作码后面带上一个后缀，其含义就是明确操作码的大小。后缀一般有 `b`、`w`、`l` 和 `q` 这四种，其中 `b` 是8位的字节（byte），`w` 是16位的字（word），`l` 是32位的双字（dword），在许多机器上32位数都被称为长字（long word），这其实是16位字作为标准的那个时代遗留下来的历史称呼，`q` 是64位的四字（qword）。 

### 调试过程

根据上面出错信息中的 `TCWebViewController` 很自然联想到与腾讯的 SDK **TencentOpenAPI.framework** 有关，但是产线的应用出现问题的时间段内并没有更新腾讯的 SDK，所以应该不是直接由 **TencentOpenAPI.framework** 导致应用崩溃的。

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

静态分析的结果发现 `TCWebViewController` 类中确实没有 `requestURLStr` 的 Setter 和 Getter 方法。因为之前版本没有出现崩溃，

此时产生一个想法：`TCWebViewController` 类中的 Property 会不会像 **Core Data** 框架一样，使用 `@dynamic` 告诉编译器不做处理，然后 Getter 和 Setter 方法是在运行时动态创建。于是带着这个猜想继续追踪下去，发现在 **TencentOpenAPI.framework** 中有个 `NSObject` 的 Category：`NSObject(MethodSwizzlingCategory)` 非常可疑，其中 `switchMethodForCodeZipper:` 方法分别将消息转发过程中的 `methodSignatureForSelector` 和 `forwardInvocation`方法替换为 `QQmethodSignatureForSelector` 和 `QQforwardInvocation`。

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

于是将视线转移到 `QQmethodSignatureForSelector` 中，发现在其中有个方法：`_AddDynamicPropertysSetterAndGetter`，从方法名称很容易知道这个方法就是动态地给属性添加 Setter 和 Getter 方法。基本验证了 `TCWebViewController` 类中的 Property 的 Setter 和 Getter 方法是在 Runtime 动态添加这个猜想。

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

那究竟为什么 `TCWebViewController` 找不到 Setter 的 Selector 呢？是否在开发新版本的过程覆盖了 `QQMethodSignatureForSelector` 导致的呢？然而在搜遍项目的所有角落，并没有发现项目中替换有 `NSObject` 的 `methodSignatureForSelector`，问题有点棘手，分析到这一步，静态分析暂时告一段落，下一步将使用 LLDB 来动态调试腾讯的三方库，从而找出是哪一个环节破坏了消息转发过程中动态生成 Getter 和 Setter。

> 这里其实如果通过 LLDB 命令给 `setRequestURLStr` 方法打断点，会发现不能成功打上这个断点，原因也是因为 Setter 方法其实在编译时还没有，也能作为上面猜想的佐证。

根据崩溃堆栈中可以推测出 `setRequestURLStr` 是在 `-[TCWebViewKit open]` 方法中调用的，也就是发生在腾讯的 SDK 检查设备没有安装 QQ 从而去打开 Web 授权页这一过程中。

我们使用 LLDB 在这个方法下断点，命令如下：

```
br s -n "-[TCWebViewKit open]"
```

> `br s -n` 的前半部分 `br s` 是 `breakpoint set` 的意思，`-n` 表示根据函数名称来下断点，作用和符号断点一样，使用 `br s -F` 也是可以的。直接使用 `b -[TCWebViewKit open]` 能达到同样的效果，只不过 `b` 命令是 `_regexp-break` 的缩写，是用正则匹配的方式下断点。最后也可以给在指定的内存地址设置断点，下断点的命令为 `br s -a 0x000000010940b24e`，这种方式可以用在知道 block 内存地址的时候，给 block 设置断点。

断点成功打上。

```
Breakpoint 34: where = AADebug`-[TCWebViewKit open], address = 0x0000000103157f7d
```

当应用准备跳到 Web 授权页的时候，断点会被断住，会看到下图

<p align="center">

<img src="Images/lldb_webviewkit_open.png" />

</p>

> 案例给出的是在模拟器中运行的截图，所以汇编代码是 X64 的，真机上看到的汇编代码是 ARM 汇编，但是分析的方法都是一样的，这点读者需要注意。

在下图 96 行的汇编代码打一个断点，这条汇编代码就是调用 `setRequestURLStr` 方法，然后打印出 `rbx` 寄存器的内容，可以观察到 `rbx` 寄存器保存的就是 `TCWebViewController` 实例。

<p align="center">

<img src="Images/lldb_webviewkit_open_1.png" />

</p>

#### methodSignatureForSelector

接下来用 LLDB 给 `QQmethodSignatureForSelector` 方法下断点

```
br s -n "-[NSObject QQmethodSignatureForSelector:]"
```

LLDB 中输入 `c` 命令让端点继续执行，这个时候断点断在了 `QQmethodSignatureForSelector` 方法内部，所以推翻了 `QQmethodSignatureForSelector` 方法被我们项目替换的猜想。
<p align="center">

<img src="Images/lldb_method_signature.png" />

</p>

在 `QQmethodSignatureForSelector` 方法汇编代码的最后，也就是31行的 `retq` 指令处下一个断点，然后将寄存器 `rax` 存放的内存地址打印出来，如下图

<p align="center">

<img src="Images/lldb_method_signature_1.png" />

</p>

图中在方法返回的时候，将 `rax` 寄存器存放的内存地址 `0x00007fdb36d38df0` 打印出来的结果是一个 `NSMethodSignature` 对象，熟悉 X86 汇编语言调用约定的读者应该知道在 X86 汇编中，函数的返回值存放在 `rax` 寄存器中。结果表明腾讯的 `QQmethodSignatureForSelector` 方法正确被调用了，并且有返回值。所以排除了这一步出现问题。

#### forwardInvocation

使用 LLDB 给 `QQforwardInvocation` 方法下断点

```
br s -n "-[NSObject QQforwardInvocation:]"
```

断点成功添加后，继续运行后，此时应用会崩溃，没有执行 `QQforwardInvocation` 方法，所以基本能够断定是我们项目中覆盖了腾讯 Hook 的 `QQforwardInvocation` 方法。

<p align="center">

<img src="Images/lldb_method_signature_2.png" />

</p>

`___forwarding___` 函数包含了消息转发的完整实现，反编译的代码摘自[Objective-C 消息发送与转发机制原理](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/)一文，文中反编译代码中调用 `forwardingTargetForSelector` 时判断 forwarding 和 receiver 是否相等处应该写错了，应当是判断 forwardingTarget 和 receiver 是否相等，代码如下：

```
int __forwarding__(void *frameStackPointer, int isStret) {
  id receiver = *(id *)frameStackPointer;
  SEL sel = *(SEL *)(frameStackPointer + 8);
  const char *selName = sel_getName(sel);
  Class receiverClass = object_getClass(receiver);

  // 调用 forwardingTargetForSelector:
  if (class_respondsToSelector(receiverClass, @selector(forwardingTargetForSelector:))) {
    id forwardingTarget = [receiver forwardingTargetForSelector:sel];
    if (forwardingTarget && forwardingTarget != receiver) {
    	if (isStret == 1) {
    		int ret;
    		objc_msgSend_stret(&ret,forwardingTarget, sel, ...);
    		return ret;
    	}
      return objc_msgSend(forwardingTarget, sel, ...);
    }
  }

  // 僵尸对象
  const char *className = class_getName(receiverClass);
  const char *zombiePrefix = "_NSZombie_";
  size_t prefixLen = strlen(zombiePrefix); // 0xa
  if (strncmp(className, zombiePrefix, prefixLen) == 0) {
    CFLog(kCFLogLevelError,
          @"*** -[%s %s]: message sent to deallocated instance %p",
          className + prefixLen,
          selName,
          receiver);
    <breakpoint-interrupt>
  }

  // 调用 methodSignatureForSelector 获取方法签名后再调用 forwardInvocation
  if (class_respondsToSelector(receiverClass, @selector(methodSignatureForSelector:))) {
    NSMethodSignature *methodSignature = [receiver methodSignatureForSelector:sel];
    if (methodSignature) {
      BOOL signatureIsStret = [methodSignature _frameDescriptor]->returnArgInfo.flags.isStruct;
      if (signatureIsStret != isStret) {
        CFLog(kCFLogLevelWarning ,
              @"*** NSForwarding: warning: method signature and compiler disagree on struct-return-edness of '%s'.  Signature thinks it does%s return a struct, and compiler thinks it does%s.",
              selName,
              signatureIsStret ? "" : not,
              isStret ? "" : not);
      }
      if (class_respondsToSelector(receiverClass, @selector(forwardInvocation:))) {
        NSInvocation *invocation = [NSInvocation _invocationWithMethodSignature:methodSignature frame:frameStackPointer];

        [receiver forwardInvocation:invocation];

        void *returnValue = NULL;
        [invocation getReturnValue:&value];
        return returnValue;
      } else {
        CFLog(kCFLogLevelWarning ,
              @"*** NSForwarding: warning: object %p of class '%s' does not implement forwardInvocation: -- dropping message",
              receiver,
              className);
        return 0;
      }
    }
  }

  SEL *registeredSel = sel_getUid(selName);

  // selector 是否已经在 Runtime 注册过
  if (sel != registeredSel) {
    CFLog(kCFLogLevelWarning ,
          @"*** NSForwarding: warning: selector (%p) for message '%s' does not match selector known to Objective C runtime (%p)-- abort",
          sel,
          selName,
          registeredSel);
  } // doesNotRecognizeSelector
  else if (class_respondsToSelector(receiverClass,@selector(doesNotRecognizeSelector:))) {
    [receiver doesNotRecognizeSelector:sel];
  } 
  else {
    CFLog(kCFLogLevelWarning ,
          @"*** NSForwarding: warning: object %p of class '%s' does not implement doesNotRecognizeSelector: -- abort",
          receiver,
          className);
  }

  // The point of no return.
  kill(getpid(), 9);
}
```

通过阅读反编译代码，基本能将消息转发的流程梳理清楚，首先是调用消息转发流程中的 `forwardingTargetForSelector` 方法获取备援接收者，也就是上文说的 Fast Forwarding 阶段，从代码中可以看出如果 `forwardingTarget` 返回空，或者和 `receiver` 相同，则进入 Regular Forwarding 阶段，具体来说就是先调用 
`methodSignatureForSelector` 拿到方法签名，然后使用前面获取到的方法签名对象和 `frameStackPointer` 实例化 `invocation` 对象，调用 `receiver` 的 `forwardInvocation:` 方法，并将刚才实例化的 `invocation` 对象传入。最后如果没有实现 `methodSignatureForSelector` 方法并且 `selector` 已经在 Runtime 注册过了，则调用 `doesNotRecognizeSelector:` 以抛出异常。

观察我们项目中崩溃堆栈中的 `___forwarding___`，会发现他的执行路径是第二步，也就是调用了 `forwardInvocation` 执行 `NSInvocation` 对象。

> 也可以在断点之后逐步执行命令，观察汇编代码的执行路径，得出结论与上面应该是一致的。

<p align="center">

<img src="Images/___forwarding___.png" />

</p>

然而调用 `forwardInvocation` 方法究竟执行了哪个方法呢？从堆栈中我们可以看到 `__ASPECTS_ARE_BEING_CALLED__` 方法，这个是 `Aspects` 库 Hook `forwardInvocation` 的方法。

```objective-c
static void aspect_swizzleForwardInvocation(Class klass) {
    NSCParameterAssert(klass);
    // If there is no method, replace will act like class_addMethod.
    IMP originalImplementation = class_replaceMethod(klass, @selector(forwardInvocation:), (IMP)__ASPECTS_ARE_BEING_CALLED__, "v@:@");
    
    if (originalImplementation) {
        class_addMethod(klass, NSSelectorFromString(AspectsForwardInvocationSelectorName), originalImplementation, "v@:@");
    }
    AspectLog(@"Aspects: %@ is now aspect aware.", NSStringFromClass(klass));
}
```

```objective-c
// This is the swizzled forwardInvocation: method.
static void __ASPECTS_ARE_BEING_CALLED__(__unsafe_unretained NSObject *self, SEL selector, NSInvocation *invocation) {
    NSLog(@"selector:%@",  NSStringFromSelector(invocation.selector));
    NSCParameterAssert(self);
    NSCParameterAssert(invocation);
    SEL originalSelector = invocation.selector;
	SEL aliasSelector = aspect_aliasForSelector(invocation.selector);
    invocation.selector = aliasSelector;
    AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
    AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
    AspectInfo *info = [[AspectInfo alloc] initWithInstance:self invocation:invocation];
    NSArray *aspectsToRemove = nil;

    // Before hooks.
    aspect_invoke(classContainer.beforeAspects, info);
    aspect_invoke(objectContainer.beforeAspects, info);

    // Instead hooks.
    BOOL respondsToAlias = YES;
    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
        aspect_invoke(classContainer.insteadAspects, info);
        aspect_invoke(objectContainer.insteadAspects, info);
    }else {
        Class klass = object_getClass(invocation.target);
        do {
            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
                [invocation invoke];
                break;
            }
        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
    }

    // After hooks.
    aspect_invoke(classContainer.afterAspects, info);
    aspect_invoke(objectContainer.afterAspects, info);

    // If no hooks are installed, call original implementation (usually to throw an exception)
    if (!respondsToAlias) {
        invocation.selector = originalSelector;
        SEL originalForwardInvocationSEL = NSSelectorFromString(AspectsForwardInvocationSelectorName);
        if ([self respondsToSelector:originalForwardInvocationSEL]) {
            ((void( *)(id, SEL, NSInvocation *))objc_msgSend)(self, originalForwardInvocationSEL, invocation);
        }else {
            [self doesNotRecognizeSelector:invocation.selector];
        }
    }

    // Remove any hooks that are queued for deregistration.
    [aspectsToRemove makeObjectsPerformSelector:@selector(remove)];
}
```

因为 `TCWebViewController` 是腾讯 SDK 中的私有类，猜想项目不大可能直接对其进行 Hook，很有可能是 Hook 了一个父类导致这个子类也受到影响，于是在项目中继续排查。啊哈！答案终于浮出水面。将 Hook `UIViewController` 的部分删除或者注释，此时再去使用 QQ 登录，应用不再崩溃。于是初步断定是 Aspects 库导致的。

<p align="center">

<img src="Images/answer.png" />

</p>

`doesNotRecognizeSelector:` 是从 `__ASPECTS_ARE_BEING_CALLED__` 方法抛出来的，`__ASPECTS_ARE_BEING_CALLED__` 是 **Aspects** 用来替换 `forwardInvocation:` 的 IMP，方法内部包含了 before、instead、after 对应时间 Aspects 切片的 hook 的逻辑。`aliasSelector` 是 **Aspects** 处理后的 SEL，如`aspects__setRequestURLStr:`。

在 Instead hooks 部分会检查 `invocation.target` 的类是否能响应 `aliasSelector`，如果子类不能响应，再检查父类是否响应，一直往上寻找直到 root，由于不能响应 `aliasSelector`，所以 `respondsToAlias` 为 false。随后，则会去将 `originalSelector` 赋值给 `invocation` 的 `selector`, 再通过 `objc_msgSend` 调用 `invocation`，企图去调用原始的 SEL，由于 `TCWebViewController` 原本就无法响应 `originalSelector`:`setRequestURLStr:`，Setter 方法本身就是在 Runtime 生成的，所以最终会运行到 `__ASPECTS_ARE_BEING_CALLED__` 方法中的 `doesNotRecognizeSelector:` 方法，也就会出现上文所述的崩溃的情况。

其实细心的读者在崩溃堆栈的第3行看到 `__ASPECTS_ARE_BEING_CALLED__` 时就大概猜到这个崩溃与 **Aspects** 有关系，然而上述分析的过程可以解释为什么程序会运行 `__ASPECTS_ARE_BEING_CALLED__` 方法，并且通过这个案例我们也明白了如何使用静态分析和动态调试的方法去分析没有源码的第三方库，希望文章提及的一些技巧和思想能在读者以后调试的过程中有所帮助。

#### 解决方案

要修复这个问题其实有两种思路，第一种思路，使用一种侵入性更小的 hook 方案来替换 **Aspects**，比如 Method Swizzling，这样就不会出现 **TencentOpenAPI** 生成 `Setter` 方法的消息转发流程被打断；第二种思路，**Aspects** 是直接将 `forwardInvocation:` 替换成自己实现，如果 `aliasSelector` 和 `originalSelector` 都无法响应时抛出异常，可以采取一种更合理的处理方式，如果出现上面的情况，将消息转发流程调回到原始的转发流程中，代码如下：

```objective-c
     if (!respondsToAlias) {
          invocation.selector = originalSelector;
          SEL originalForwardInvocationSEL = NSSelectorFromString(AspectsForwardInvocationSelectorName);
         ((void( *)(id, SEL, NSInvocation *))objc_msgSend)(self, originalForwardInvocationSEL, invocation);
      }
```

事实上，除了我们遇到的问题，**Aspects** 与 **JSPatch** 也存在兼容问题，由于两者的实现原理也类似，也会出现本文中遇到的 `doesNotRecognizeSelector:`，具体请阅读[微信读书的文章](http://wereadteam.github.io/2016/06/30/Aspects/)。笔者也逛了下 **Aspects** 与 **JSPatch** 的 Issues，也有与两者兼容问题相关的信息。

#### Aspects 与 TencentOpenAPI 的一次完美邂逅

导致崩溃的原因是 **Aspects** 与 **TencentOpenAPI** 两个库的一次完美邂逅。项目中使用 Aspects 库 Hook `UIViewController` 类的页面生命周期方法，Aspects 的 Hook 实现会替换掉 `forwardInvocation` 方法，由于 `TCWebViewController` 的父类是 `UIViewController`，所以也会被 Hook 住，`QQforwardInvocation` 方法被覆盖，导致消息转发失败，从而无法动态生成属性的 Setter 和 Getter 方法。

上面案例给了我们一个警示，在使用一个第三方框架和技术的时候，我们不应该只停留在会用的层面上，而是要深入了解他背后的工作原理。这样定位问题时才会事半功倍。


## 总结

文章花了比较多的篇幅介绍了各种技巧，但是比起这些花哨的技巧，更希望读者能掌握良好的调试思想，因为良好的思维不是一朝一夕能养成的，而技巧通常都能现学现卖。只有具备良好的解决问题的思路，再辅以各种调试技巧，那么问题就迎刃而解了。


## 参考资料

* 《代码大全》
* 《调试九法:软硬件错误的排查之道》
* 《Advanced Apple Debugging & Reverse Engineering》
* 《Debug It!: Find, Repair, and Prevent Bugs in Your Code》
* 《Effective Objective-C 2.0：编写高质量iOS与OS X代码的52个有效方法》
* [Objective-C 消息发送与转发机制原理](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/)

## 作者

* [Alex Ao](https://github.com/aozhimin)
* [NewDu](https://github.com/NewDu)

## 致谢

特别致谢以下读者，感谢他们对文章的支持，并提出了非常宝贵的建议。

* [ZenonHuang](https://github.com/ZenonHuang)
