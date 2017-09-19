# MVC

​        框架的所有代码结构整合都是采用MVC的基础架构，这也是苹果iOS系统的基本架构。Controller作为关键角色负责View层和Data层的数据流交换，现在市面上流行很多架构优劣的争论（例如讨论比较多的MVVM），但是在我看来，**不论是那种架构都是以MVC为基础的，然后不断的划分集体职责，细化得来的。所以能够清楚的掌握职责划分，将视图、逻辑和数据三者联系起来，易用并方便维护，那就可以了，无所谓框架的优劣**。



# 代码库管理

​       在本框架的代码管理中使用的Cocoapod作为第三方依赖的统一管理模块。

在iOS的开发过程中会经常使用到第三方开源库，或者跟其他团队合作的时候有些公告的代码模块需要很几个团队使用的情况，更复杂的情况是某些第三方库还会再引用其他的第三方库，所以手动去一个个的下载并配置每个第三库的使用是一件十分费力不讨好的事情，而且还可能会出现各种的编译错误。另外如果项目中的类库需要更新，就需要手动把就的文件删除在重新下载新版本的类库然后再添加到工程中去，以上两种情况是类库管理中经常出现的问题，因此我们引入管理工具Cocoapod来简明清楚的管理框架中使用的第三方类库和自定义类库

CocoaPod资源  [https://github.com/CocoaPods/CocoaPods](https://github.com/CocoaPods/CocoaPods)



# 网络核心模块

​       在本框架中网络核心模块对`AFNetworking`**进行了抽象的封装，抽取**`AFNetworking`的常用的POST和GET请求，并在此基础上添加网络机密功能。

主要包块以下几个主要的功能：

1、表单数据上传功能，主要用于上传富文本文件

```objective-c
/**
 *  上传
 *
 *  @param datas 可以同时上传多个，datas为NSDictionary的集合，NSDictionary中必需包含data, name, fileName, mimeType
 */
-(void)requestWithMultiDatas:(NSArray*)datas params:(NSDictionary*)params;
```

2、非加密POST请求

```objective-c
/* 请求非加密 - 参数类型只能是字符串或者数据字典 */
-(void)requestWithParams:(id)templates;
```

3、加密POST请求，报文体采用AES加密的方式，密钥由服务端提供

```objective-c
/* 请求加密 */
-(void)requestWithAESEncrpytParams:(NSDictionary*)params token:(NSString*)token;
```

4、 Get请求(Get请求均不加密)

```objective-c
/* 请求加密 */
-(void)requestWithAESEncrpytParams:(NSDictionary*)params token:(NSString*)token;
```

​        网络请求模块作为BaseViewController的基础功能，因此在实际使用中由Controller来处理HTTP请求，同时对数据进行处理和封装，然后展示给View层。



# 资源加载

​        资源加载在这里特指App读取和加载资源图片，在实际开发中需要大量使用图片来显示App的视觉效果，但是根据硬件的物理尺寸不同，在读取资源的时候分为@2x和@3x两种图片。前者针对非5.5英寸的屏幕尺寸，后者是针对5.5英寸的屏幕尺寸。

​        理论上说每个App都应真多不同的屏幕尺寸分别存在一套@2x和@3x的图片，但是在实际使用中两套图片的存在会大大的增加App的体积这是不可取的，因此此框架在资源加载额时候统一使用4.7英寸**(@2x图)**作为标准，向上长宽分别扩大1.1倍来适配5.5英寸的屏幕尺寸，向下缩小1.08倍来适配4英寸的屏幕尺寸。

- 把图片资源根据需要加载的模块放在根目录的res文件夹下统一管理
- 通过打包脚本build_plist.sh将res下的资源名称组织成plist文件，其中图片索引跟世界@2x图片名称一一对应。

脚本文件build_plist.sh

```shell
#  根据res下文件结构，构建config.plist

/usr/libexec/PlistBuddy -c "Delete :res" res/resconfig.plist
find res | grep @2x.png$ | while read line
do
if [ "$line" ]
then
path1=${line#res/}
path2=${path1%@2x.png}
path3=${path2//\//_}

image_path=${line%@2x.png}

/usr/libexec/PlistBuddy -c "Add :res:$path3 string \"/$image_path.png\""  res/resconfig.plist

fi
done

#/usr/libexec/PlistBuddy -c "Print" res/resconfig.plist
```

通过资源加载UIImageManager类的imageName方法来根据图片索引加载实际对应的图片资源

```objective-c
 /*  @param imageKey imageKey description
 *
 *  @return UIImage
 */
+ (UIImage *)imageNamed:(NSString *)imageKey
{
    NSString *imagePath = [[QHResManager sharedInstance] pathForImage:imageKey];
    UIImage *tmpImage = [UIImage imageWithContentsOfFile:imagePath];
    if (tmpImage == nil)
    {
        imagePath = [imagePath stringByReplacingOccurrencesOfString:@".png" withString:@"@2x.png"];
        tmpImage = [UIImage imageWithContentsOfFile:imagePath];
        
    }
    return tmpImage;
}
```

*资源加载模块的优势：*

> 1、图片资源统一管理，资源分类清晰
>
> 2、通过图片索引加载图片，避免频繁书写@2x和@3x，减少代码冗余
>
> 3、自动适应屏幕的尺寸，免除在使用中需要手动计算图片大小做屏幕自适应



# 日志Log模块

日志系统简单来说就是在程序运行中通过NSLog来打印出来，上下文中关键的信息，方便进行程序调试和问题的追踪，但是如果仅仅是简单的打印一句话在实际的使用中是无法目标大型应用的要求的，一个成熟的Log系统需要如下功能：

- 可以设定 Log 等级
- 可以积攒到一定量的 log 后，一次性发送给服务器，绝对不能打一个 Log 就发一次
- 可以一定时间后，将未发送的 log 发送到服务器
- 可以在 App 切入后台时将未发送的 log 发送到服务器

如果要实现上述的要求，在框架中采用CocoaLumberjack来辅助搭建自己的日志系统。

CocoaLumberjack 最早是由 [Robbie Hanson ](https://github.com/robbiehanson)开发的日志库，可以在 iOS 和 MacOSX 开发上使用。其简单，快读，强大又不失灵活。它自带了几种log方式，分别是:

- DDASLLogger 将 log 发送给苹果服务器，之后在 Console.app 中可以查看
- DDTTYLogger 将 log 发送给 Xcode 的控制台
- DDFileLogger 讲 log 写入本地文件

CocoaLumberjack 打一个 log 的流程大概就是这样的：

![001.png](http://cc.cocimg.com/api/uploads/20150311/1426054062244229.png)

所有的 log 都会发给 DDLog 对象，其运行在自己的一个GCD队列(GlobalLoggingQueue)，之后，DDLog 会将 log 分发给其下注册的一个或多个 Logger，这步在多核下是并发的，效率很高。每个 Logger 处理收到的 log 也是在它们自己的 GCD队列下（loggingQueue）做的，它们询问其下的 Formatter，获取 Log 消息格式，然后最终根据 Logger 的逻辑，将 log 消息分发到不同的地方。

因为一个 DDLog 可以把 log 分发到所有其下注册的 Logger 下，也就是说一个 log 可以同时打到控制台，打到远程服务器，打到本地文件，相当灵活。

CocoaLumberjack 支持 Log 等级：

```objective-c
typedef NS_OPTIONS(NSUInteger, DDLogFlag) {
    DDLogFlagError      = (1 << 0), // 0...00001
    DDLogFlagWarning    = (1 << 1), // 0...00010
    DDLogFlagInfo       = (1 << 2), // 0...00100
    DDLogFlagDebug      = (1 << 3), // 0...01000
    DDLogFlagVerbose    = (1 << 4)  // 0...10000
};
typedef NS_ENUM(NSUInteger, DDLogLevel) {
    DDLogLevelOff       = 0,
    DDLogLevelError     = (DDLogFlagError),                       // 0...00001
    DDLogLevelWarning   = (DDLogLevelError   | DDLogFlagWarning), // 0...00011
    DDLogLevelInfo      = (DDLogLevelWarning | DDLogFlagInfo),    // 0...00111
    DDLogLevelDebug     = (DDLogLevelInfo    | DDLogFlagDebug),   // 0...01111
    DDLogLevelVerbose   = (DDLogLevelDebug   | DDLogFlagVerbose), // 0...11111
    DDLogLevelAll       = NSUIntegerMax                           // 1111....11111 (DDLogLevelVerbose plus any other flags)
};
```

DDLogLevel 定义了全局的 log 等级，DDLogFlag 是我们打 log 时设定的 log 等级，CocoaLumberjack 会比较两者，如果 flag 低于 level，则不会打 log。

DDLogger 协议定义了 logger 对象需要遵从的方法和变量，为了方便使用，其提供了 DDAbstractLogger 对象，我们只需要继承该对象就可以自定义自己的 logger。对于第二点和第三点需求，我们可以利用 DDAbstractDatabaseLogger，其也是继承自 DDAbstractLogger，并在其上定义了 saveThreshold, saveInterval 等控制参数。这个 logger 本身是针对写入数据库的 log 设计的，我们也可以利用它这几个参数，实现我们上面所提的需求的第二和第三点。

对于第二点，设定 _saveThreshold 值即可，比如如果希望积攒1000条 log 再一次性发送，就赋值 1000.

对于第三点，设定 _saveInterval，比如如果希望每分钟发送一次，就设定 60.

由此，CocoaLumberjack 已经实现了需求中的 1、2、3 点，我们要做的无非是自定义 Logger 和 Formatter，将 log 的最终去处改为发送到我们自己的服务器中。

而第四点，我们可以监听 UIApplicationWillResignActiveNotification 事件，当触发时，手动调用 logger 的 db_save 方法，发送数据给服务器。



# 工具处理Util

工具类处理主要包括图片处理、文件系统、日期、加解密、时间、字符串等常用的工具函数，具体的使用可以参考对应的头文件里边的函数注释。

QHFileUtil   文件处理

QHImageUtil  图片处理

QHDateUtil  日期处理

QHTimerUtil 时间处理

/Encrypt 目录提供AES加解密算法、RSA加解密算法、Base64编码、MD5编码、SHA1编码等算法



# BaseUI

框架的baseUI主要是指的三个提供基础服务的组件，分别是控制器基类BaseViewController、导航控制器基类BaseNavigationController和WebView控制器BaseWebViewController。



## BaseViewController

该控件主要对controller的基础服务进行封装，在实际使用过程中所有的业务逻辑的controller都应该从此类进行继承。它主要封装如下功能供子类使用：

> - controller样式定义
> - 网络模块的调用，以及网络成功、失败、处理中回调函数的处理
> - 页面切换返回操作pop
> - 公告遮罩的显示、隐藏和等待动画
> - 多线程
> - 切换手势操作

以上功能在基类中给出详细的实现，且这些功能都是子类很频繁使用的业务逻辑，将这些业务逻辑在基类中实现可以更好实现代码的复用，减少程序冗余

## BaseNavigationController

BsaeNavigationController继承自UINavigationController，除了提供必要的页面跳转逻辑之外，本框架对导航控制器进行了定制，主要添加如下功能：

> - 对iOS上的interactivePopGestureRecognizer进行定制，可以方便打开或者关闭手势交互
>
>
> - 对导航控制器的Push和Pop进行功能定制，使用范围更广

实现UINavigationControllerDelegate的方式，对Push和Pop时的controller的变换进行跟踪，方便后续的调试

```objective-c
- (id<UIViewControllerAnimatedTransitioning> )navigationController:(UINavigationController *)navigationController animationControllerForOperation:(UINavigationControllerOperation)operation fromViewController:(UIViewController *)fromVC toViewController:(UIViewController *)toVC
{
    if (operation == UINavigationControllerOperationPush) {
        DDLogDebug(@"navigationController will push to %@", [toVC class]);
    }else if(operation == UINavigationControllerOperationPop){
        DDLogDebug(@"navigationController will pop to %@", [toVC class]);
    }
    return nil;
}

```

## BaseWebViewController

BaseWebViewController是本框架提供出来的加载WebView的控制器，可以认为是一个带有定制化功能的浏览器。它采用桥接的模式是WebViewController具有了Javascript交互的能力，具体是指我们可以再浏览器中执行JavaScript的语句获取数据，同时JavaScript也可以执行注入在Controller中的OC的函数，获取OC函数执行的数据，在Javascript运行时中使用，**简单来说就是可以实现JS和Native的双向数据传输**。具体的使用可以参考如下的部分的内容。



# 桥接模块的使用

桥接模块的主要在iOS端实现（js -> native interface）,安卓的js调用native的方式非常简单明了，不禁想如果iOS端也有如此实现的话，这样同时即保证安卓，iOS，h5的统一性也能让开发者只用关心交互的接口即可。因此框架便引如了`EasyJSWebView`的第三方的框架（基于说明2设计），我们不在此过多的论述EasyJSWebView的桥接原理，只说明在框架中如何使用。



虽然在iOS7以后iOS提供了`JavaScriptCore Framework`来进行JS的控制，但是却存在：

> - iOS端虽然也可以通过`JSContext`注入全局的方法但是达不到与安卓端统一
> - iOS端可以通过拦截h5请求的url，通过url的格式区分类或方法，但是这样不够直观，也达不到与安卓端统一

因此我们在本框架中采用EasyJSWebView来实现桥接，具体步骤简单描述如下：

- 在JSInterface类中定义Native方法供JS调用
- 将JSInterface类作为接口同addInterface注入到EasyJSWebView类中
- 将EasyJSWebView的对象作为SubView添加到BaseWebViewController中

这样就可以再当前BaseWebViewController加载的JS上下文环境中调用Native的方法了。

EasyJSWebView的代码参考： https://github.com/dukeland/EasyJSWebView

# WebSocket

​       WebSocket消息就是指的App跟服务端的异步通知消息，客户端通过WebSocket协议同远程服务器之间建立一个长连接，通过此连接，服务器可以向客户端主动推送消息，客户端亦可以向服务器发送请求，通过WebSocket连接，客户端可以在请求发出后，等待服务器端发送异步处理的结果消息，避免主动查询，减少等待时间，根据异步返回的消息来进行下一步处理。

Websocket连接建立、销毁

异步消息通知的基础是WebSocket连接的建立，但是由于移动客户端的诸多限制，一直维持WebSocket连接的耗费太大，包括电量、流量等，因此websocket连接的建立和销毁仅限以下场景（WebSocket连接的建立、销毁对于用户是无感知的）：

-  用户登录、登出

用户登录包括用户名密码登陆、手势密码登陆、一账通登陆。在未登陆前，客户端同服务器是没有WebSocket连接的，因此不会有任何消息提示。在登陆后，客户端会主动去创建WebSocket连接。相应的，用户主动登出后，WebSocket连接被销毁。

- 应用切换

应用切换是指应用进入后台，或者从后台唤醒，包括锁屏、点击home键(iOS)等，当应用进入后台时，连接被销毁，当应用唤醒时，如果已登陆，创建连接。

- 连接建立失败处理

当WebSocket连接首次创建失败时，客户端会进行重试，重试的最大次数为3，每次重试之间的间隔为15秒。

本框架对第三方框架`SRWebSocket`()进行的定制化的封装，通过QHWebSocket类初始化创建连接对象，

连接服务端

```objective-c
-(void)connect;
```

发送心跳

```objective-c
-(void)startHeartBeat;
```

发送数据

```objective-c
-(void)sendText:(NSString*)text;
-(void)sendData:(NSData*)data;
```

断开服务器连接并销毁

```objective-c
-(void)destroy;
```

WebSocket跟服务器的反馈，以及服务端下行通道的数据获取通过对应代理来实现：

```objective-c
@protocol QHWebSocketDelegate <NSObject>

- (void)webSocket:(QHWebSocket *)webSocket didReceiveTextMessage:(NSString*)message;
- (void)webSocket:(QHWebSocket *)webSocket didReceiveBinaryMessage:(NSData*)message;

@optional

- (void)webSocketDidOpen:(QHWebSocket *)webSocket;
- (void)webSocket:(QHWebSocket *)webSocket didFailWithError:(NSError *)error;
- (void)webSocket:(QHWebSocket *)webSocket didCloseWithCode:(NSInteger)code reason:(NSString *)reason wasClean:(BOOL)wasClean;
- (void)webSocket:(QHWebSocket *)webSocket didReceivePong:(NSData *)pongPayload;
- (void)applicationDidEnterBackground;
- (void)applicationWillEnterForeground;

@end
```

服务端通过WebSocket下行通知我们可以在客户端App实现一些比较特殊的控制功能；例如：

- 登陆互踢

​        登陆互踢，是当用户在一台设备A上已经登陆，然后该账号在另一台设备B上又登陆了，这时，如果websocket连接保持连接状态，服务器会向A设备发送一条登陆互踢消息，然后A设备上应用就会提示用户账号在其他设备登陆。

![Image text](http://www.openlo.cn:8083/lo-view/ios/blob/b4e4f6b7c45fb2fe3c2e728664b2e81da3513012/img/1.png)

- 会话超时

​       会话超时，是在客户端建立websocket连接时，服务器可能返回的消息之一，譬如当应用切换至后台时，websocket断开连接，然后一段时间后，重新切回应用，此时客户端重新建立websocket连接，但是会话已超时，服务器端就会返回会话超时消息，客户端接收到消息后就会弹出手势密码（如果有）重新登陆，或者跳往未登录前首页。

# UI组件

本框架封装了很多iOS常用的UI组件，可以方便的修改和复用

公告弹框----AlertView

手势密码组件----GesturePassword

输入框组件-----InputField（包括文字、纯数字、密码等等）

Toast组件-----MessageToast

轮播组件-----RecycleScrollView(常用于轮播广告)

密保键盘------SecurityKeyboard

下拉刷新组件-----QHTableViewRefresher（列表下拉刷新的封装）

公告等待层遮罩------WaitView

App启动的广告Splash-----SplashView

公共的控件工厂类-----ViewFactory(根据样式定义公共的Button、Label、Navbar等基础UI)

ViewFactory的部分工厂方法的示例：

```objective-c
// button类型
typedef NS_ENUM(NSUInteger, BNButtonType) {
    BNButtonTypeDefault,
    BNButtonTypeNavBarDarkBack,
    BNButtonTypeNavBarDarkClose
};

// NavBar类型
typedef NS_ENUM(NSUInteger, BNNavigationBarType) {
    BNNavigationBarTypeDefault
};

//label类型
typedef NS_ENUM(NSUInteger, BNLabelType) {
    BNLabelTypeDefault,
    BNLabelTypeWhite,
    BNLabelTypeDark
};

+ (UIImageView *)navigationBarWithType:(BNNavigationBarType)type;
+ (UILabel *)labelWithType:(BNLabelType)type;
+ (UIButton *)buttonWithType:(BNButtonType)type;
```

# React Native支持

React Native是一种利用Javascript和React的技术来描述Native页面布局的开发方式，我们简要的描述一下在现有的iOS工程添加React Native支持的方式，由于该开发对H5的开发要求较高，如果你对React技术不熟悉建议先参考相关文档。

方式一：直接创建React Native的工程

例用faceBook提供的React Native开发通过react-native init可以创建一个纯使用react开发的工程，但是在实际中我们不可能完全用React来进行App的开发，因此此方式适用范围比较窄。

关于环境的搭建请参考：

http://reactnative.cn/docs/0.40/getting-started.html

方式二：在现有的iOS工程中添加React Native的支持

1、在工程文件根目录创建package.json

2、在使用过程中可以根据实际需要修改配置文件package.json，本文示例只添加了react和react native两个版本的node依赖包。

```json
{
  "dependencies":{
    "react":"15.4.1",
    "react-native": "0.42.0"
  }
}
```

3、使用npm（node包管理器，Node package manager）来安装React和React Native模块。这些模块会被安装到项目根目录下的`node_modules/`目录中。 在包含有package.json文件的目录（一般也就是项目根目录）中运行下列命令来安装：

```
$ npm install

```

在安装成功后，可以再根目录看到node_module的文件夹，这个就是react native所有的依赖库文件。

4、按照如下图在项目工程中添加react native的第三方库文件,当完成如下图所示的工程依赖并编译通过就表示当前的工程已经开始使用React Native进行开发了。

![Image text](http://www.openlo.cn:8083/lo-view/ios/raw/master/img/2.png)

5、编译工程文件，解决库依赖文件，这样支持React Native的文件就搭建完毕。

然后在根目录创建React Native的入口文件index.ios.js（名称不能改动），就可以进行相关开发了，

关于React Native的语法和文档可以参考如下网站：

http://reactnative.cn/

http://nav.react-china.org/

https://github.com/reactnativecn/react-native-guide

index.ios.js的示例，在RCTView中用React显示一个Label文字

```javascript
"use strict"
import React from 'react'
import {
    AppRegistery,
    StyleSheet,
    Text,
    View,
    NavigatorIOS
} from 'react-native'

class RNView extends React.Component{
    render() {
        var param = this.props.param;
        return (
            <View style={styles.container}>
                <Text style={styles.labelText}>Text {{param}}</Text>
            </View>
        )
    },

    var styles = StyleSheet.create({
        container:{
            flex:1,
            justifyContent:'center',
            alignItems:'center'
            backgroundColor:'#FFFFFF'
        }
        labelText:{
            fontSize:10
            textAlign:'center'
            color:'#333333'
        }
    })
}
AppRegistery.registerComponent('RNView', () => lo-view-ios);
```

