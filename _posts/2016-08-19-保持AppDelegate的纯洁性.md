---
layout:     post
title:      保持AppDelegate的纯洁性
date:       2016-08-19
author:     开不了口的猫
header-img: img/icon_保持AppDelegate的纯洁性_bg.png
catalog: true
tags:
    - iOS
---

## 前言
随着现在主流APP功能设计越来越重，一个APP可能需要涉及三方登录、地图、支付、统计、推送等各种三方平台SDK的接入，哪怕你的APP不包含以上功能，但是总会有一些自制组件，如果用Cocoapods的话，一些工具型Pod组件的注册和注入总也无可厚非。更有甚者，不同组件的注入需要有依赖关系，需要先后触发开启。总而言之，繁杂的业务需求决定了，如果我们只是每次粗暴地将这些功能的注入代码全部扔在了AppDelegate中，随着APP功能不断的扩展和APP需求的迭代，迟早有一天，AppDelegate会不堪重负。  

不得不承认的是，AppDelegate的`- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)options`方法确实是一个对于注册功能组件来说不可多得的最佳时机以及最佳位置，因为此时所有类都已经被load进runtime，且主页面还未在keyWindow上显示出来，那么是否存在可替代的方案呢？  
## 

## 不明智的方案
有人会立马想到method swizzling，可是我得说method swizzling并不总是组件自动化的上乘之选，就拿这个topic的情景来说，我们现在需要在AppDelegate的`- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)options`方法中注册/开启N个组件的功能，然后我们按照method swizzling的思路，将AppDelegate拆分成N个分类，然后在分类的`+ load`方法中进行`- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)options`方法的调剂，将N个组件的注册/开启方法放到对应的N个AppDelegate分类中，然后回头一看，AppDelegate本类确实如我们所愿，变得十分干净。  

可是这个方案存在太多的限制和弊处。  

首先，如果使用Cocoapods的话，作为工具组件Pod，如果把AppDelegate的分类放在Pod内，那就要求整个项目工程的AppDelegate类文件必须也放在一个Pod内，但如果不那么做，这个工具组件Pod将lint失败。但如果选择将AppDelegate分类放在AppDelegate本类旁边，则要求AppDelegate本类必须放在寄主工程根目录中，否则其所在的Pod需要依赖的依赖项将是所有工具Pod组件的集合，这明显也是不被推荐的。
其次，大量的method swizzling将会导致程序出错时难以排查，且无法解决多个组件之间的先后顺序依赖问题。且大量的method swizzling本身需要消耗一些性能，放在load中，可能会导致`热启动时间`不必要的增加。

## 必要的排查
在思索AppDelegate的解耦方案之前，我们其实更应该先去确定下来具体的`解耦对象清单`。为什么这么说呢，因为一百个组件可能会有一百种应用场景，而一个组件在多个APP上也可以有多个触发(注册)时机。  

举个栗子，一个APP需要引入定位SDK，但是它的首页其实并不需要定位功能，而可能在某个子级页面才需要用到，那么我们可以认为，这个定位SDK的注册时机完全可以延后到启动后的某一个结点甚至是某个需要定位功能的子级页面被动触发。可能这个栗子举的不太恰当(因为定位SDK的注册操作通常不需要大量复杂的操作)，我想说的是，某些生效复杂且完全没必要在程序启动时就触发/注册的功能组件，我们是否可以考虑延后去触发/注册呢。

## 通过NSNotification解耦
我们发现在程序启动触发几个APP启动方法的时候，会在对应代理方法执行结束后发送出对应的通知给整个应用。以下通知直接摘录于UIApplication.h。
``` objc
// These notifications are sent out after the equivalent delegate message is called
UIKIT_EXTERN NSNotificationName const UIApplicationDidEnterBackgroundNotification       NS_AVAILABLE_IOS(4_0);
UIKIT_EXTERN NSNotificationName const UIApplicationWillEnterForegroundNotification      NS_AVAILABLE_IOS(4_0);
UIKIT_EXTERN NSNotificationName const UIApplicationDidFinishLaunchingNotification;
UIKIT_EXTERN NSNotificationName const UIApplicationDidBecomeActiveNotification;
UIKIT_EXTERN NSNotificationName const UIApplicationWillResignActiveNotification;
UIKIT_EXTERN NSNotificationName const UIApplicationDidReceiveMemoryWarningNotification;
UIKIT_EXTERN NSNotificationName const UIApplicationWillTerminateNotification;
UIKIT_EXTERN NSNotificationName const UIApplicationSignificantTimeChangeNotification;
UIKIT_EXTERN NSNotificationName const UIApplicationWillChangeStatusBarOrientationNotification __TVOS_PROHIBITED; // userInfo contains NSNumber with new orientation
UIKIT_EXTERN NSNotificationName const UIApplicationDidChangeStatusBarOrientationNotification __TVOS_PROHIBITED;  // userInfo contains NSNumber with old orientation
UIKIT_EXTERN NSString *const UIApplicationStatusBarOrientationUserInfoKey __TVOS_PROHIBITED;            // userInfo dictionary key for status bar orientation
UIKIT_EXTERN NSNotificationName const UIApplicationWillChangeStatusBarFrameNotification __TVOS_PROHIBITED;       // userInfo contains NSValue with new frame
UIKIT_EXTERN NSNotificationName const UIApplicationDidChangeStatusBarFrameNotification __TVOS_PROHIBITED;        // userInfo contains NSValue with old frame
UIKIT_EXTERN NSString *const UIApplicationStatusBarFrameUserInfoKey __TVOS_PROHIBITED;                  // userInfo dictionary key for status bar frame
UIKIT_EXTERN NSNotificationName const UIApplicationBackgroundRefreshStatusDidChangeNotification API_AVAILABLE(ios(7.0), tvos(11.0));

UIKIT_EXTERN NSNotificationName const UIApplicationProtectedDataWillBecomeUnavailable    NS_AVAILABLE_IOS(4_0);
UIKIT_EXTERN NSNotificationName const UIApplicationProtectedDataDidBecomeAvailable       NS_AVAILABLE_IOS(4_0);
```
`UIApplicationDidFinishLaunchingNotification`通知常量显然是我们需要的答案，然后我们辅以NSNotificationCenter的
``` objc
- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (^)(NSNotification *note))block API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));
```
方法就可以帮助我们轻松地完成AppDelegate解耦的第一步。  
我们可以在每个功能组件的主类或者新建一个launcher类，通过重载`+ load`方法注册`UIApplicationDidFinishLaunchingNotification`通知回调的block，在block中实现对应功能的注册和注入工作。  
因为公司项目中的支付有微信、支付宝、银联等多种支付方式，所以我制作了一个支付通用组件，以实现多种支付参数、调用方式及回调处理标准化，在这个组件Pod中，因为避免直接import AppDelegate本类，所以我新建了一个launcher类专门用来完成支付组件的注册和回调注入工作。  
``` objc
@implementation TDFPaymentLauncher

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
        __weak NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
        __block id noti_observer = [[NSNotificationCenter defaultCenter] addObserverForName:UIApplicationDidFinishLaunchingNotification object:nil queue:nil usingBlock:^(NSNotification * _Nonnull note) {
            
            NSDictionary *plistConfiguration = [NSDictionary dictionaryWithContentsOfFile:[[NSBundle mainBundle] pathForResource:TDF_PAYMENT_CONFIGURATION_RESOURCE_NAME ofType:@"plist"]];
            
            NSUInteger configurations = kTDFPaymentConfigurationTypeNone;
            for (NSString *sdk_key in plistConfiguration.allKeys) {
                if ([sdk_key isEqualToString:TDF_PAYMENT_CONFIGURATION_WX]) {
                    configurations = configurations | kTDFPaymentConfigurationTypeWeixin;
                    continue;
                }
                if ([sdk_key isEqualToString:TDF_PAYMENT_CONFIGURATION_ALI]) {
                    configurations = configurations | kTDFPaymentConfigurationTypeAli;
                    continue;
                }
                if ([sdk_key isEqualToString:TDF_PAYMENT_CONFIGURATION_UNION]) {
                    configurations = configurations | kTDFPaymentConfigurationTypeUnion;
                    continue;
                }
                if ([sdk_key isEqualToString:TDF_PAYMENT_CONFIGURATION_APPLE]) {
                    configurations = configurations | kTDFPaymentConfigurationTypeApple;
                    continue;
                }
            }
            
            [[TDFPaymentManager manager] setupPaymentConfigurations:configurations];
            
            [center removeObserver:noti_observer];
        }];
        
        // swizzle appDelegate method.
        Class class = NSClassFromString(@"AppDelegate");
        assert(class != NULL);
        
        //- (BOOL)application:(UIApplication *)application handleOpenURL:(NSURL *)url
        tdf_payment_swizzle(class, NSSelectorFromString(@"application:handleOpenURL:"), NSSelectorFromString(@"tdf_payment_application:handleOpenURL:"));
        
        //- (BOOL)application:(UIApplication *)application openURL:(nonnull NSURL *)url sourceApplication:(nullable NSString *)sourceApplication annotation:(nonnull id)annotation
        tdf_payment_swizzle(class, NSSelectorFromString(@"application:openURL:sourceApplication:annotation:"), NSSelectorFromString(@"tdf_payment_application:openURL:sourceApplication:annotation:"));
        
        //- (BOOL)application:(UIApplication *)app openURL:(NSURL *)url options:(NSDictionary<UIApplicationOpenURLOptionsKey, id> *)options
        tdf_payment_swizzle(class, NSSelectorFromString(@"application:openURL:options:"), NSSelectorFromString(@"tdf_payment_application:openURL:options:"));
    });
}
```
可以看到，这样就完成了支付功能的注入。不过需要注意以下几点：
* block 对 observer 对象的捕获早于函数的返回，所以若不加__block，会捕获到 nil
* 记得在block代码块结尾移除这个observer
* 最容易忽视的一点，记得weak化这个NSNotificationCenter实例，因为NSNotificationCenter是单例模式且center实例不weak化的话会与center的回调block出现循环引用，导致这个block在observer被移除后仍然得不到释放。

这种方案依然有它的弊处。
* 首先它解决不了多个注册方法之间的依赖问题，加入组件A需要以组件B的注册为前提，那么我们便很难通过优雅的方式去控制注册的顺序，因为我们都是通过`+ load`中增加UIApplication生命周期方法的监听回调来实现的。
* 如果功能组件需要在除`UIApplicationDidFinishLaunchingNotification`外的其他UIApplication的生命周期方法中注入自己的处理代码时，我们需要监听多个生命周期方法通知，但还有一些生命周期方法，例如上述支付组件示例中的`- (BOOL)application:(UIApplication *)application handleOpenURL:(NSURL *)url`方法，系统并没有设计或暴露其对应的通知常量以供外界监听，我们只能通过额外的meethod swizzling去实现，同样不是特别优雅。

## 组件化思想的解决方案(于2018.06.13补充)
随着现在公司规模和体量慢慢变大，组件化思想日趋成熟。就在前段时间，一个关系比较好的公司同事(青木)设计出了一个适用于体量大且非常优雅的解决方案。  

具体方案和对应实现我都仔细看了一遍，确实很好的解决了我提出的前一个方案的痛点，并且使用上非常灵活，所以我也没必要再造轮子了。大家可以点击[这里](http://triplecc.github.io/blog/2017-10-25-zu-jian-sheng-ming-zhou-qi/)阅读解决方案。

## 参考
* [https://developer.apple.com/documentation/foundation/nsnotificationcenter/1411723-addobserverforname](https://developer.apple.com/documentation/foundation/nsnotificationcenter/1411723-addobserverforname)
* [http://triplecc.github.io/blog/2017-10-25-zu-jian-sheng-ming-zhou-qi/](http://triplecc.github.io/blog/2017-10-25-zu-jian-sheng-ming-zhou-qi/)
