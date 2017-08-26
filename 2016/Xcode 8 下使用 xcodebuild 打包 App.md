Goto [Introduction](https://github.com/bigyelow/Hblog/blob/master/Introduction.md)

# Xcode 8 下使用 xcodebuild 打包 App 
之前我们的项目一直使用 Xcode 7 来打包项目，用 Cocoapods 来管理第三方库。最近我们升级到了 Xcode 8 并且由于**模块化**的需要，需要将部分 Swift 或者 OC+Swift 混编的代码抽出去成为单独的库，所以需要使用 Cocoapods 的 use_frameworks! 特性将包含了 Swift 源文件的库引用进来，这期间遇到了不少问题，在这里梳理下处理的办法以供他人参考。

> Note: 上述说的打包都是使用 xcodebuild 命令完成的。


## 开启 use_frameworks! 后由于第三方库 bundle id 问题导致安装失败
使用 xcodebuild 生成内测版 ipa 后，在安装的时候失败，查看 device log 有如下报错：

>Bundle identifier of pods is the same as the app (instead of org.cocoapods.PodName)

这是由于使用 use_frameworks! 之后，每个第三方库（framework）自成为一个一个 bundle，每一个 bundle 都应该有一个合法的 bundle identifier。 但是 xcodebuild 打包的时候会使所有的 framework 使用同一个 bundle id。

```
解决办法：在 project.pbxproj 中定义新的变量比如 APP_BUNDLE_IDENTIFIER 为主项目的
bundle id，然后修改 PRODUCT_BUNDLE_IDENTIFIER = $(APP_BUNDLE_IDENTIFIER)。然后在 
plist 文件中将 $(PRODUCT_BUNDLE_IDENTIFIER) 替换为 $(APP_BUNDLE_IDENTIFIER)。
```
相关链接：[https://github.com/CocoaPods/CocoaPods/issues/5003](https://github.com/CocoaPods/CocoaPods/issues/5003)


## Xcode 8 下使用 xcodebuild 打包 App
命令行打包的问题在于两个

* 充分理解 code sign 机制
* 熟悉使用 xcodebuild 

### 关于 code sign
这是一个老生常谈也是一个无数文章说过的问题了，其实如果理解了本质就不需要看那么多文章。理解这个问题需要理解证书、签名、沙盒、授权（Entitlements）、Provisioning profile 之间的关系及作用。在此推荐两篇文章，看懂了这两篇文章你就知道每当 code sign 出问题时应该怎么解决了。

* [https://www.objc.io/issues/17-security/inside-code-signing/](https://www.objc.io/issues/17-security/inside-code-signing/)
* [http://pewpewthespells.com/blog/migrating_code_signing.html](http://pewpewthespells.com/blog/migrating_code_signing.html)

### Xcode 8 auto signing
Xcode 8 引入了 auto signing，好处是真机调试时方便多了；坏处则是之前在 Xcode 7 下正常的打包命令在 Xcode 8 下会报错，具体的错误信息记不清了，意思是 xcodebuild 制定了 code_sign_identity 和 provisioning_profile，但是 Xcode 使用了 auto sign。这时候需要手动修改 **project.pbxproj** 文件：

	sed -i '' 's/DevelopmentTeam = *****/ProvisioningStyle = Manual/g' Frodo.xcodeproj/project.pbxproj

也就是需要设置 ProvisioningStyle = Manual，至于说具体替换的是哪段配置因项目而异。

### 使用 xcodebuild 
也许你最终还是可以使用之前的命令打包成功，但是最后上传 App Store 的时候会遇到如下问题：

>ERROR ITMS-90668: "Invalid Bundle Executable. The executable file 'xxx.app/Frameworks/yyy.framework/zzz' contains incomplete bitcode. To compile binaries with complete bitcode, open Xcode and choose Archive in the Product menu."

bitcode 出错。

或者还会遇到下面的问题，也就是 xxx.app 中多了一个 swift 的 dylib
>ERROR ITMS-90171: "Invalid Bundle Structure - The binary file 'XXX.app/**libswiftRemoteMirror.dylib**' is not permitted. Your app can't contain standalone executables or libraries, other than the CFBundleExecutable of supported bundles. Refer to the Bundle Programming Guide at https://developer.apple.com/go/?id=bundle-structure for information on the iOS app bundle structure."

这时候有两种解决办法：

	解决办法1：就像提示的，在 Xcode 中使用 Archive 功能。 Archive 功能打包的 App 可以成功上传。
	
但是我们需要持续集成，如果每次都手动 Archive、手动上传 dsym 那不是弱爆了？
你之前生成 ipa 的命令可能和下面的类似：

	/usr/bin/xcrun -sdk iphoneos PackageApplication -v path/To/xxx.app -o xxx.ipa
	
但实际上 PackageApplication 已经 deprecated 了。你是不是也和我一样，明明 log 里面有 deprecated 的日志，你却没有看？都 deprecated 了你说能不出问题吗。解决思路是我们需要用上同 Archive 一样的功能。

实际上已经不需要 xcrun 了，xcodebuild 新增了 **-exportArchive** 选项。

	解决办法2：
	
	1. Build app:

	xcodebuild archive \
             -workspace XXX.xcworkspace \
             -scheme "XXX" \
             -configuration Distribution \
             -derivedDataPath ./build \
             -archivePath ./build/Products/XXX.xcarchive \
             CODE_SIGN_IDENTITY="iPhone Distribution" \
             PROVISIONING_PROFILE=*****
             
	2. Generate ipa:
	
	xcodebuild -exportArchive \
             -archivePath ./build/Products/XXX.xcarchive \
             -exportOptionsPlist ./exportOptions-Release.plist \
             -exportPath ./build/Products/XXX.ipa     
             
其中需要对应的 exportOptions-Release.plist 文件，这个文件可以定义很多值，最简单的形式如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>teamID</key>
    <string>**********</string>
</dict>
</plist>             
```

至此，我们又可以愉快的在 Xcode 8 下生活了😜。

---
If you have any problems, send me email duyu1010@gmail.com or  [open an issue](https://github.com/bigyelow/Hblog/issues/new) please.
           
	
	



















