# Bugfix for WKWebView Blank Issue

> 该问题是在 豆瓣App iOS 版本（<= 5.22.0）上发现的


## 问题
- 用户反馈【书影音 tab】页面**有时**出现部分区域空白且无法点击现象
- 用户反馈【书影音 tab】页面**偶现**全部空白

## 难点
- 从出现频率上看，貌似是“非必现”的且复现路径不好找
- 无法判断上述两个问题是否由同一个问题引起
- 影响因子较多：难以确定是 “native” 还是“前端页面”问题；无法判断是否与 rexxar 实现机制有关

结论：没有明显的思路

## 解决过程
### 1. 找到复现路径
遇到这个问题并不是刻意的，而是刚好在做[【书影音 tab】优化](https://github.intra.douban.com/frodo/Frodo-iOS/issues/8916)这个 issue 时检查内存使用情况恰好遇到的，且需要恰好是一个 1G 内存的设备。

> 注：1G 内存的设备才会更频繁的出现 webContentProcess 被杀掉的情况

复现方法：

- 在该 tab 创建12个 segmengts，依次切换触发页面加载；
- 在 `RXRWebViewController` 中设置断点`webViewWebContentProcessDidTerminate:`
- 第一次进入断点后，即之前加载的某一个页面的 WKWebView 的 webContentProcess 已经被杀掉后，继续切换到下一个 segment，等出现加载动画后，**快速往下滑动**，即可出现下面部分页面空白且无法点击；
- 继续切换到新的 segment，后面的页面**不做任何操作**也会出现上述 bug

> 注：最开始以为只有页面重新加载才会出现这个 bug，后面会讲到如何找到最终复现路径的

### 2. 定位问题
分析：由于在 `webViewWebContentProcessDidTerminate:` 和 `viewWillAppear:` 中我们均有 reload 页面，几经尝试后，直接修改 RXRViewController 和 RXRWebViewController 并不能解决问题，且从现象上来说无法排除和前端渲染逻辑相关，于是不得不用“排除法”。

排除法目标：

- 确认是 native 问题还是前端页面问题
- 如果是 native 问题，那么具体是哪个类的问题

共同条件：

- 去掉 SubjectModulesSegmentController 影响，使用最基本的 JYSlideSegmentController

#### 2.1 初步确定范围
对比测试

- 1 表示有问题
- 0 表示没有问题

----  | FRDRXRSubjectModuleViewController | FRDRXRViewController  | RXRViewController | WebViewController
--------------------- | ------------------------------- | ---------------      | ------ | ----
m.douban.com          |          1     |  1 | 1 | 0
file://          |               1   |

- 不是“书影音”页面的问题
- 怀疑是 rexxar 实现的问题
- 直接使用 RXRViewController 测试

耗时：3小时以上...

> 注：m.douban.com 和 file:// 消耗的内存并不一样，m.douban.com 需要同时加载 20 个以后才开始出现内存回收现象

#### 2.2 RXRViewController Debug
（1）猜测原因：

- webview load 出错
- RXRNSURLProtocol 数据返回出错
- viewDidAppear 再 reload
- viewWillAppear 和 didTerminate 都调用了 reload，是否是重复调用引起的？—— 看起来很有可能
	- 只在 didTerminate 时 reload 	
		- RXRViewController + m.douban.com + 只在 didTerminate 中 reload：没有出现问题（当时以为找到问题了）
		- RXRViewController + file::// + 只在 didTerminate 中 reload：依然有问题（希望再次幻灭）
		- 另外发现
			- 直接在 didTerminater 调用 webView.reload 会报 NSPOSIXErrorDomain, Code = 1 错误，原因：需要调用 loadFileURL 方法
			- 直接在 didTerminater loadFileURL 报错：_WKRecoveryAttempterErrorKey=\<WKReloadFrameErrorRecoveryAttempter: 0x11c6a42a0\>，原因：见后文分析

结论：Not the Answer，不是重复调用引起的，且去掉 viewWillAppear 中的 reload 不解决问题。

（2）再次回顾现象：为什么第一次加载的时候不容易出问题？第一次加载和第二次加载有什么区别？

recall：内存回收

```
/*! @abstract Invoked when the web view's web content process is terminated.
 @param webView The web view whose underlying web content process was terminated.
 */
- (void)webViewWebContentProcessDidTerminate:(WKWebView *)webView API_AVAILABLE(macosx(10.11), ios(9.0));
```

关键字：`web content process`

```
/*! @abstract The process pool from which to obtain the view's web content
 process.
 @discussion When a web view is initialized, a new web content process
 will be created for it from the specified pool, or an existing process in
 that pool will be used.
*/
@property (nonatomic, strong) WKProcessPool *processPool;

/*! A WKProcessPool object represents a pool of web content processes.
 The process pool associated with a web view is specified by its web view
 configuration. Each web view is given its own web content process until an
 implementation-defined process limit is reached; after that, web views
 with the same process pool end up sharing web content processes.
 */
WK_EXTERN API_AVAILABLE(macosx(10.10), ios(8.0))
@interface WKProcessPool : NSObject <NSCoding>
@end
```

关键语句：

Each web view is given its own web content process until an
 implementation-defined process limit is reached; after that, web views with the same process pool end up **sharing web content processes**.

猜测：只要发生过一次内存回收，之后新加载的 webview 不做任何操作都会出现问题，而不需要重新加载之前的 webview 来复现问题。

**经验证，该猜测成立。**

（3）尝试解决

- 每次内存回收后使用新的 pool —— NO
- 检查 `WKProcessPool` 是否有释放内存等相关方法 —— NO
- 每次 viewWillAppear 的时候 setNeedsLayout —— NO

到这里几乎是山穷水尽了，看起来是 WKWebView 的内存管理机制问题，这个问题可能无解了...

Any Idea ?

### 3. 解决问题——无奈之举

recall:

```
/*! @abstract The process pool from which to obtain the view's web content
 process.
 @discussion When a web view is initialized, a new web content process
 will be created for it from the specified pool, or an existing process in
 that pool will be used.
*/
@property (nonatomic, strong) WKProcessPool *processPool;
```

关键词：initialized

暴力解法：每次创建新的 webView

尝试路径：

1. init-create + willAppear-new-webView：NO
2. viewDidLoad-create + willAppear-new-webView: YES !!

继续尝试：

3. viewDidLoad-create: NO
4. viewDidLoad-creat + willAppear-setNeedsLayout: NO
5. viewDidLoad-creat + willAppear-new-process-pool: NO

结论：viewDidLoad-create + willAppear-new-webView

回溯原因：为什么 viewDidLoad-create 可以而 init-create 不可以？
猜测：与 view 加载和 WKProcessPool 重用 webContentProcess 机制有关。

### 4. 注意事项
1. JYSlide 中的 rxrVC 在 terminate 的时候不能直接调用 webView.reload，因为有 localFileURL，需要调用 `loadFileURL:allowingReadAccessToURL:`，否则会报权限问题
2. JYSlide 中的 rxrVC 在 terminate 的时候不能立即调用 reloadWebView，有可能会报错。

	```
	Error Domain=NSURLErrorDomain
	Code=-999 "(null)"
	UserInfo=
	{
	NSErrorFailingURLStringKey=http://,
	_WKRecoveryAttempterErrorKey=<WKReloadFrameErrorRecoveryAttempter: 0x1c0822d20>
	}

	```

	原因是 webView 之前的请求还没有加载完成就发起了下一个请求，此时 webView 会取消之前的请求。

	所以需要将 terminate 回调告诉 container，由 container 控制 reload 时机。

3. 对于普通的 RXRVC 还是需要立即 reload，因为并不是所有时候都会调用到 viewWillAppear: ，比如切换 app 且 WKWebView 内存释放后，回来会白屏
4. FRDWebViewController 和其他的 JYSlide + RXRVC 会有类似的问题
5. 【书影音 tab】一般到10个左右的 segments 就会开始出现 terminate 回调；而 https://m.douban.com 差不多要到20个左右
