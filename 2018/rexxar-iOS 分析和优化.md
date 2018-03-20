# rexxar-iOS 分析和优化

项目地址：[rexxar-iOS](https://github.com/bigyelow/rexxar-ios)

## rexxar-iOS 分析
### 1. 工作机制
#### （1）routes.json 文件

```
{

"deploy_time": "Fri, 22 Dec 2017 08:01:21 GMT",

"items": [
{
"remote_file": "https://img3.doubanio.com/dae/rexxar/pre-files/account/suspicious_login-ea05ec2361.html",
"deploy_time": "Fri, 22 Dec 2017 08:01:21 GMT",
"uri": "douban://douban.com/account/suspicious_login/\\d+[/]?",
"discard_version": null
}, ...,],

"partial_items": [
{
"remote_file": "https://img3.doubanio.com/dae/rexxar/pre-files/article/note-e466b77b3b.html",
"deploy_time": "Fri, 22 Dec 2017 08:01:21 GMT",
"uri": "douban://partial.douban.com/note/\\w+/_content[/]?.*",
"discard_version": null
},...,]

}
```

#### （2）启动
- 从本地读取 routes.json 文件 `-(NSData*)routesMapFile`
	- cache 目录：`Caches/com.douban.frodo.hybrid/routes.json`
	- resource 目录：`data/Containers/Bundle/Application/1A279542-3E7A-46E4-8645-F923F079997E/Frodo.app/hybrid`
	- 优先从 cache 读取
		- 升级的时候，由于 cache 目录中仍有数据，resource 中的资源仍然可能不会使用到
- 更新 routes 文件
	- `updateRouteFilesWithCompletion:`
	- 请求 routes.json
	- 下载 routes.json 中的资源文件 `_rxr_downloadFilesWithinRoutes:completion:`
		- 根据 `remoteURL` ，检查 cache 和 resource 目录中是否已经存在这个文件 `routeFileURLForRemoteURL:`
		- 如果不存在，会逐个校验、下载文件
			- 校验机制：校验文件名最后几位是否是文件内容的 md5 值
			- 校验机制放在 Frodo 中，可定制；
				- https://img3.doubanio.com/dae/rexxar/pre-files/account/suspicious_login-d07650f7fa.html
				- `validateRemoteHTMLFile:fileData:`
			- 下载到 cache：`https://img3.doubanio.com/dae/rexxar/pre-files/account/suspicious_login-ea05ec2361.html` --> `4385e7d1dc99e1497493b4377c8eeb11.html`
			- MD5：文件名的 MD5 值
	- 整个下载过程没有一个文件出现错误，才会更新**内存**（`[RXRRouteManager sharedInstance].routes`）中的 routes.json
		- 如果下载到某个文件时出错，那么 routes.json 未更新，之前的本地资源文件依然存在
		- `如果没有意外`或者`更新`，**内存中**的 routes.json 文件是**可靠的**
			- 如果在路由表中找到了对应的 uri，那么在本地一定有对应的 html 文件

#### （3）加载
- Route：`URLRouter.isRouteExistForURI:`
	- 检查内存中 routes 表对于某个 `uri` 是否有对应的 `remote url`
- 加载：`RXRViewController.reloadWebView._rxr_htmlURLWithUri:htmlFileURL`
	- 缓存启用：uri --> remoteURL --> `routeFilePathForRemoteURL:`
	- 没有缓存：直接加载 `remote url`

#### （4）出错
```
typedef enum : NSUInteger {
  RXRLogTypeNoRoutesMapURLError,  // 没有设置 RoutesMap 地址
  RXRLogTypeDownloadingRoutesError, // 下载 Routes 失败
  RXRLogTypeDownloadingHTMLFileError, // 下载 HTML file 失败
  RXRLogTypeValidatingHTMLFileError,  // 验证下载的 HTML file 失败（需要提供 `RXRDataValidator`）
  RXRLogTypeFailedToCreateCacheDirectoryError,  // 创建 cache 目录失败
  RXRLogTypeWebViewLoadingError,  // WebView 加载失败
  RXRLogTypeNoRemoteHTMLForURI, // 在内存中的 route 列表里找不到 uri 对应的项（没有对应的 html 文件名）
  RXRLogTypeNoLocalHTMLForURI,  // 在内存中的 route 列表里找不到 uri 对应的本地 html 文件
  RXRLogTypeWebViewLoad404,  // 没有本地 html 时，webview 加载远程 html 出现 404
  RXRLogTypeWebViewLoadNot200,  // WebView load 中返回了非 200 status code
  RXRLogTypeUnknown,
} RXRLogType;
```

frodo 中处理 404：发生 RXRLogTypeNoLocalHTMLForURI 且加载 `remote url` 失败。

### 2. 截获请求实现原理
 - 核心 `Interceptor` + `Decorator`
 - `Interceptor`
	 - 继承自 `RXRNSURLProtocol`
	 - 两类
		 - `RXRRequestInterceptor`：截获请求，定制请求。startLoading 的时候检查实例变量 decorators，并逐个 decorate request
			 - 对 decorator 顺序有要求
			 - FRDRXRAuthDecorator
			 - FRDRXRSignatureDecorator
			 - FRDRXRRequestMethodDecorator
			 - 区别经典的装饰器模式
		 - `RXRContainerInterceptor`：类似代理，截获请求，控制返回数据；
			 - `setContainerAPIs:`
			 	- FRDRXRGeoContainerAPI
			 	- FRDRXRLocContainerAPI
			 	- FRDRXRSetLocContainerAPI
			 	- FRDRXRAccountContainerAPI
			 	- FRDRXRInstalledAppsContainerAPI
 	- `RXRURLSessionDemux` + `RXRNSURLProtocol` 实现 request 和 response
- 注册 `interceptor`
	- `+ registerRXRProtocolClass:`
	- `+ unregisterRXRProtocolClass:`
- 其他：`RXRCacheFileInterceptor`
	- js 丢失：
		- 截获请求 js 文件的 request，成功后缓存；
	- html 出错：
		- 不会截获正确的 `http://xxx.com/yy.html`，会被 webview url loading system 吞掉，不会缓存
		- 如果 `http://xxx.com/yy.html` 出错会被截获 -- 404 处理

## 优化
### 痛点
- 加载出错
- 日志：`RXRLogger`
- 实现依赖 `NSURLProtocol`
	- Interceptor + Decorator 调试困难
	- `canInitWithRequest:` 泛滥
	
### 错误处理优化
- 加载出错：`RXRWebViewController.webview:didFailLoadWithError`
- 404：`RXRViewController.decidePolicyForNavigationResponse:`
	- 刷新 routes
	- `reloadLimitWhen404` = 2
- 优化重试交互

### 计划：使用 `WKURLSchemeHandler` 替换 `NSURLProtocol`
- 测试中...