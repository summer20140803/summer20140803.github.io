---
layout:     post
title:      iOS Pretty-Format API日志打印
date:       2017-07-21
author:     开不了口的猫
header-img: img/icon_pretty_format_api_logger_bg.jpg
catalog: true
tags:
    - iOS
    - API
---

## 前言
每个公司的App项目都有对应的网络层封装，而API日志输出既可以作为封装层中的一个子功能，也可以作为一个独立的、不与网络层耦合的功能来设计。我个人更偏向于后者，因为它更符合当今流行的组件化思想趋势。  

其实AFNetworking的作者mattt名下有一个开源库[AFNetworkActivityLogger](https://github.com/AFNetworking/AFNetworkActivityLogger)是专门帮助使用AFNetworking的开发者更快更好地完成API日志输出的功能。  

还有一种思路是通过自定义`NSURLProtocol`去拦截发出那些API请求，在对应的拦截方法中对API请求前和API着陆后的各种HTTP元素进行定制并格式化生成API日志，然后再输出，这样一个好处是可以不用依赖AFNetworking的网络库(但是现在市面上极少数oc项目是使用别的网络基础库的)。  

本文最终实现方案[Pretty-Format API日志打印](https://github.com/summer20140803/TDFAPILogger)，喜欢可以顺手点个star，谢啦~

## AFNetworkActivityLogger的缺陷
既然现在oc的项目基本离不开成熟的AFNetworking网络框架，那么我们何必舍近求远的使用NSURLProtocol实现一套API拦截呢。  

然后很自然地，我将注意力放到了`AFNetworkActivityLogger`的实现上，看完源码之后，发现AFNetworkActivityLogger的实现非常简单，因为AFNetworking在API起飞和API着陆后会对外发出两个通知名已经暴露出来的通知，而AFNetworkActivityLogger只需顺势注册这两个通知，实现回调方法，在回调中做API信息的定制即可。  
但是很快我发现AFNetworkActivityLogger的定制其实有点粗糙，输出的日志不规范且阅读性不是很好，并且由于作者matt是歪果仁，自然也不会对API请求体或者是响应体中出现的中文做处理，那么在Xcode控制台上这些中文字符只能变为碍眼的Unicode字符了。这里我截几张使用AFNetworkActivityLogger的控制台输出(我们的项目早期就是用的AFNetworkActivityLogger)。
![ctdpic](https://ws4.sinaimg.cn/large/006tNc79gy1fsbobbaw7gj30hn0msq72.jpg)
![ctdpic](https://ws2.sinaimg.cn/large/006tNc79gy1fsbnwseiryj317p09e7aj.jpg)
可以看到，API日志的输出结构不是很直观，如果在研发联调阶段，这可能会造成联调效率低下。

## TDFAPILogger的设计
于是我们决定重新设计一个新的API日志输出框架，鉴于AFNetworking已经为我们设计好了两个切面通知`AFNetworkingTaskDidResumeNotification`和`AFNetworkingTaskDidCompleteNotification`，所以TDFAPILogger会继续沿用AFNetworkActivityLogger的优点。  

除此之外，新的API日志输出框架还应该被设计为：
* 可以在运行时任意关闭或开启日志输出
* 根据API的状态拆分为请求日志、响应日志(甚至可以将异常日志也独立出来)
* 外界可以定制输出的元素集合，即输出有哪些内容，可以由外界自由拼接
* 可以支持对于API日志的筛选规则
* 支持拓展，将每次格式化后的API日志模型通知给外界已经注册过的监听者
* 一些相对耗时的格式化工作需要放在子线程执行，不能影响主线程处理事务的性能

其实还有一点，随着Xcode8禁止了绝大部分的民间插件，控制台颜色已经不能通过插件完成了，意味着我们的API日志结构必须更加直观，最好能有显眼的分割线(本人标准颜控🙃)。至此，TDFAPILogger才能有替代`AFNetworkActivityLogger`作为API日志输出框架的资本。  

依照上述的各种诉求，我们完成了TDFAPILogger的接口类的编写。
```objc
#import <Foundation/Foundation.h>
#import "TDFALRequestModel.h"
#import "TDFALResponseModel.h"

//======================================
// 运行期可通过LLDB `expr`命令实时修改的变量
//======================================

// 所有API日志的统一开关
CF_EXPORT BOOL  TDFAPILoggerEnabled;
// 请求日志的可定制格式符号
CF_EXPORT char *TDFAPILoggerRequestLogIcon;
// 响应日志的可定制格式符号
CF_EXPORT char *TDFAPILoggerResponseLogIcon;
// 异常日志的可定制格式符号
CF_EXPORT char *TDFAPILoggerErrorLogIcon;


//============================
//   可定制的API日志的组成元素
//============================

typedef NS_OPTIONS(NSUInteger, TDFAPILoggerRequestElement) {
    /** API起飞时间 */
    TDFAPILoggerRequestElementTakeOffTime        = 1 << 0,
    /** API请求方式 */
    TDFAPILoggerRequestElementMethod             = 1 << 1,
    /** API有效的请求路径 */
    TDFAPILoggerRequestElementVaildURL           = 1 << 2,
    /** API请求头字段 */
    TDFAPILoggerRequestElementHeaderFields       = 1 << 3,
    /** API请求体(一般是入参) */
    TDFAPILoggerRequestElementHTTPBody           = 1 << 4,
    /** API任务唯一标识 */
    TDFAPILoggerRequestElementTaskIdentifier     = 1 << 5,
};

typedef NS_OPTIONS(NSUInteger, TDFAPILoggerResponseElement) {
    /** API着陆时间 */
    TDFAPILoggerResponseElementLandTime          = 1 << 0,
    /** API请求-响应耗时 */
    TDFAPILoggerResponseElementTimeConsuming     = 1 << 1,
    /** API请求方式 */
    TDFAPILoggerResponseElementMethod            = 1 << 2,
    /** API有效的请求路径 */
    TDFAPILoggerResponseElementVaildURL          = 1 << 3,
    /** API响应头字段 */
    TDFAPILoggerResponseElementHeaderFields      = 1 << 4,
    /** API响应状态码 */
    TDFAPILoggerResponseElementStatusCode        = 1 << 5,
    /** API响应主体(或者异常) */
    TDFAPILoggerResponseElementResponse          = 1 << 6,
    /** API任务唯一标识 */
    TDFAPILoggerResponseElementTaskIdentifier    = 1 << 7,
};


@interface TDFAPILogger : NSObject

/**
 请求日志 可定制的组成元素
 */
@property (nonatomic, assign) TDFAPILoggerRequestElement  requestLoggerElements;

/**
 响应日志/异常日志 可定制的组成元素
 */
@property (nonatomic, assign) TDFAPILoggerResponseElement responseLoggerElements;

/**
 服务模块白名单，
 可用来在研发自己模块期间屏蔽其他模块的API日志，
 默认应用服务端全部模块
 */
@property (nonatomic, strong) NSArray<NSString *> *       serverModuleWhiteList;

/**
 AFN内部指定task的taskDescription的对象，
 一般是项目工程里直接接触和封装AFNetworking的单例对象，
 比如`TDFHTTPClient`单例对象
 */
@property (nonatomic, strong) id                          defaultTaskDescriptionObj;

/**
 API日志过滤器，
 在打印请求日志和响应日志之前都会通过这个block询问是否需要打印，
 block会将请求或者响应时获取的NSURLRequest子类实例返回给外部，
 外部通过request去作一些自定义的判定处理，
 return YES表示需要打印，return NO表示过滤这些日志
 */
@property (nonatomic,   copy) BOOL(^loggerFilter)(__kindof const NSURLRequest *request);

/**
 API请求日志汇报者，
 会在格式化后(不包括emoji)的请求描述模型通过这个block传给外部
 */
@property (nonatomic,   copy) void(^requestLogReporter)(TDFALRequestModel *requestLogDescription);

/**
 API响应日志汇报者，
 会在格式化后(不包括emoji)的响应描述模型通过这个block传给外部
 */
@property (nonatomic,   copy) void(^responseLogReporter)(TDFALResponseModel *responseLogDescription);


/**
 获取单例
 @return 单例
 */
+ (instancetype)sharedInstance;

/**
 开启API日志
 */
- (void)open;

/**
 关闭API日志
 (一般不需要这么做)
 */
- (void)close;

@end
```

## TDFAPILogger的实现
明确完框架标准和编写完接口类之后，剩余的工作就是依葫芦画瓢了。  

首先我们在`TDFAPILogger`的开启方法`- (void)load`中监听了AFNetworking的两个切面通知：
```objc
- (void)open {
#if DEBUG

    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(apiDidTakeOff:) name:AFNetworkingTaskDidResumeNotification object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(apiDidLand:) name:AFNetworkingTaskDidCompleteNotification object:nil];
#endif

}
```
然后我们在初始化方法中默认将`requestLoggerElements`和`responseLoggerElements`设为全部元素：
```objc
- (instancetype)init {
    if (self = [super init]) {
        // default settings..
        _requestLoggerElements =
        TDFAPILoggerRequestElementTakeOffTime |
        TDFAPILoggerRequestElementMethod |
        TDFAPILoggerRequestElementVaildURL |
        TDFAPILoggerRequestElementHeaderFields |
        TDFAPILoggerRequestElementHTTPBody |
        TDFAPILoggerRequestElementTaskIdentifier;
        _responseLoggerElements =
        TDFAPILoggerResponseElementLandTime |
        TDFAPILoggerResponseElementTimeConsuming |
        TDFAPILoggerResponseElementMethod |
        TDFAPILoggerResponseElementVaildURL |
        TDFAPILoggerResponseElementHeaderFields |
        TDFAPILoggerResponseElementStatusCode |
        TDFAPILoggerResponseElementResponse |
        TDFAPILoggerResponseElementTaskIdentifier;
    }
    return self;
}
```
一旦收到有请求已经起飞的通知，就会调用我们的`apiDidTakeOff:`方法，我们首先从通知中取出对应的NSURLRequest对象。
```objc
 NSURLRequest *request = TDFAPILoggerRequestFromAFNNotification(notification);
```
接着我们需要验证用户自定义的筛选规则，决定哪些请求需要输出具体的请求信息。
```objc
if (!request || !(self.requestLoggerElements | 0x00) || (self.loggerFilter && !self.loggerFilter(request))) return;
```
还会再次验证服务白名单，做到只会输出请求URL中包含白名单内字符串的请求信息日志，加入我只是在开发一个很小的子项目，那么这样做可以直接达到屏蔽其他模块请求日志的效果。
```objc
    // In addition，check whiteList for shielding some needless api log..
    if (self.serverModuleWhiteList && self.serverModuleWhiteList.count) {
        NSString *urlStr = [request.URL absoluteString];
        
        for (NSString *whiteModule in self.serverModuleWhiteList) {
            if (whiteModule &&
                [whiteModule isKindOfClass:[NSString class]] &&
                [whiteModule stringByReplacingOccurrencesOfString:@" " withString:@""].length) {
                
                NSString *serverModule = [NSString stringWithFormat:@"/%@/", whiteModule];
                if ([urlStr containsString:serverModule]) {
                    goto nextStep_Req;
                }
            }
        }
        return;
    }
```
接着就进入了为请求日志一次次加工的流程。因为我们拿到了NSURLRequest对象，所以这非常容易。
```objc
nextStep_Req:;
    objc_setAssociatedObject(notification.object, TDFAPILoggerTakeOffDate, [NSDate date], OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
    NSMutableString *frmtString = @"".mutableCopy;
    TDFALRequestModel *requestDescriptionModel = [[TDFALRequestModel alloc] init];
    
    if (self.requestLoggerElements & TDFAPILoggerRequestElementTaskIdentifier) {
        NSString *taskIdentifier = TDFAPILoggerTaskIdentifierFromAFNNotification(notification);
        NSString *taskIdentifierDes = [NSString stringWithFormat:@"\n<API序列号> %@", taskIdentifier];
        [frmtString appendString:taskIdentifierDes];
        requestDescriptionModel.taskIdentifier = taskIdentifierDes;
    }
    
    NSURLSessionTask *task = (NSURLSessionTask *)notification.object;
    NSUInteger taskDescLength = [task.taskDescription stringByReplacingOccurrencesOfString:@" " withString:@""].length;
    if (self.defaultTaskDescriptionObj) {
        NSString *taskDescriptionSetByAFN = [NSString stringWithFormat:@"%p", self.defaultTaskDescriptionObj];
        if (taskDescLength && ![task.taskDescription isEqualToString:taskDescriptionSetByAFN]) {
            NSString *apiTaskDes = [NSString stringWithFormat:@"\n<API描述>    %@", task.taskDescription];
            [frmtString appendString:apiTaskDes];
            requestDescriptionModel.taskDescription = apiTaskDes;
        }
    } else {
        if (taskDescLength) {
            NSString *apiTaskDes = [NSString stringWithFormat:@"\n<API描述>    %@", task.taskDescription];
            [frmtString appendString:apiTaskDes];
            requestDescriptionModel.taskDescription = apiTaskDes;
        }
    }
    
    if (self.requestLoggerElements & TDFAPILoggerRequestElementTakeOffTime) {
        NSDateFormatter * df = [[NSDateFormatter alloc] init];
        [df setDateFormat:@"yyyy-MM-dd HH:mm:ss"];
        NSString *timeStr = [df stringFromDate:objc_getAssociatedObject(notification.object, TDFAPILoggerTakeOffDate)];
        NSString *milestoneTimeDes = [NSString stringWithFormat:@"\n<起飞时间>  %@", timeStr];
        [frmtString appendString:milestoneTimeDes];
        requestDescriptionModel.milestoneTime = milestoneTimeDes;
    }
    
    if (self.requestLoggerElements & TDFAPILoggerRequestElementMethod) {
        NSString *methodDes = [NSString stringWithFormat:@"\n<请求方式>  %@", request.HTTPMethod];
        [frmtString appendString:methodDes];
        requestDescriptionModel.method = methodDes;
    }
    
    if (self.requestLoggerElements & TDFAPILoggerRequestElementVaildURL) {
        NSString *validURLDes = [NSString stringWithFormat:@"\n<请求地址>  %@", [request.URL absoluteString]];
        [frmtString appendString:validURLDes];
        requestDescriptionModel.validURL = validURLDes;
    }
    
    if (self.requestLoggerElements & TDFAPILoggerRequestElementHeaderFields) {
        NSDictionary *headerFields = request.allHTTPHeaderFields;
        NSMutableString *headerFieldFrmtStr = @"".mutableCopy;
        [headerFields enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull key, id  _Nonnull obj, BOOL * _Nonnull stop) {
            [headerFieldFrmtStr appendFormat:@"\n\t\"%@\" = \"%@\"", key, obj];
        }];
        NSString *headerFieldsDes = [NSString stringWithFormat:@"\n<HeaderFields>%@", headerFieldFrmtStr];
        [frmtString appendString:headerFieldsDes];
        requestDescriptionModel.headerFields = headerFieldsDes;
    }
    
    if (self.requestLoggerElements & TDFAPILoggerRequestElementHTTPBody) {
        __block id httpBody = nil;
        
        if ([request HTTPBody]) {
            httpBody = [[NSString alloc] initWithData:[request HTTPBody] encoding:NSUTF8StringEncoding];
        }
        // if a request does not set HTTPBody, so here it's need to check HTTPBodyStream
        else if ([request HTTPBodyStream]) {
            NSInputStream *httpBodyStream = request.HTTPBodyStream;
            
            __weak __typeof(self) w_self = self;
            TDFAPILoggerAsyncHttpBodyStreamParse(httpBodyStream, ^(NSData *streamData) {
                __strong __typeof(w_self) s_self = w_self;
                
                httpBody = streamData;
                NSString *httpBodyDes = [NSString stringWithFormat:@"\n<Body>\n\t%@", httpBody];
                [frmtString appendString:httpBodyDes];
                requestDescriptionModel.httpBody = httpBodyDes;
                
                NSString *logMsg = [frmtString copy];
                TDFAPILoggerShowRequest(logMsg);
                
                requestDescriptionModel.selfDescription = logMsg;
                
                !s_self.requestLogReporter ?: s_self.requestLogReporter(requestDescriptionModel);
            });
            return;
        }
        
        if ([httpBody isKindOfClass:[NSString class]] && [(NSString *)httpBody length]) {
            NSMutableString *httpBodyStr = @"".mutableCopy;
            
            NSArray *params = [httpBody componentsSeparatedByString:@"&"];
            [params enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
                NSArray *pair = [obj componentsSeparatedByString:@"="];
                
                NSString *key = nil;
                if ([pair.firstObject respondsToSelector:@selector(stringByRemovingPercentEncoding)]) {
                    key = [pair.firstObject stringByRemovingPercentEncoding];
                }else {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
                    key = [pair.firstObject stringByReplacingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
#pragma clang diagnostic pop
                }
                
                NSString *value = nil;
                if ([pair.lastObject respondsToSelector:@selector(stringByRemovingPercentEncoding)]) {
                    value = [pair.lastObject stringByRemovingPercentEncoding];
                }else {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
                    value = [pair.lastObject stringByReplacingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
#pragma clang diagnostic pop
                }
                value = [value stringByReplacingOccurrencesOfString:@"'" withString:@"\\'"];
                
                [httpBodyStr appendFormat:@"\n\t\"%@\" = \"%@\"", key, value];
            }];
            
            NSString *httpBodyDes = [NSString stringWithFormat:@"\n<Body>%@", httpBodyStr];
            [frmtString appendString:httpBodyDes];
            requestDescriptionModel.httpBody = httpBodyDes;
        }
    }
```
其中这部分代码需要稍微解释一下。
```objc
if (self.requestLoggerElements & TDFAPILoggerRequestElementHTTPBody) {
        __block id httpBody = nil;
        
        if ([request HTTPBody]) {
            httpBody = [[NSString alloc] initWithData:[request HTTPBody] encoding:NSUTF8StringEncoding];
        }
        // if a request does not set HTTPBody, so here it's need to check HTTPBodyStream
        else if ([request HTTPBodyStream]) {
            NSInputStream *httpBodyStream = request.HTTPBodyStream;
            
            __weak __typeof(self) w_self = self;
            TDFAPILoggerAsyncHttpBodyStreamParse(httpBodyStream, ^(NSData *streamData) {
                __strong __typeof(w_self) s_self = w_self;
                
                httpBody = streamData;
                NSString *httpBodyDes = [NSString stringWithFormat:@"\n<Body>\n\t%@", httpBody];
                [frmtString appendString:httpBodyDes];
                requestDescriptionModel.httpBody = httpBodyDes;
                
                NSString *logMsg = [frmtString copy];
                TDFAPILoggerShowRequest(logMsg);
                
                requestDescriptionModel.selfDescription = logMsg;
                
                !s_self.requestLogReporter ?: s_self.requestLogReporter(requestDescriptionModel);
            });
            return;
        }

        ...
            以下是一段格式化工作的代码
        ...
    }
```
因为一次Post请求有可能携带的是stream流的body(例如上传图片)，也有可能是Json串二进制(普通的数据请求)格式的body，所以我们需要先判断request的HTTPBody是否为空。如果不为空，则我们只需按照UTF8格式去将body的二进制数据encode为字符串即可。如果为空，则需要判断request的HTTPBodyStream，如果有值，那么我们会通过`TDFAPILoggerAsyncHttpBodyStreamParse`函数将stream流转换为二进制，再转换成字符串。  
我们看一下`TDFAPILoggerAsyncHttpBodyStreamParse`函数。
```objc
static void TDFAPILoggerAsyncHttpBodyStreamParse(NSInputStream *originBodyStream, tdfHttpBodyStreamParseBlock block) {
    
    // this is a bug may cause image can't upload when other thread read the same bodystream
    // copy origin body stream and use the new can avoid this issure
    NSInputStream *bodyStream = [originBodyStream copy];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [bodyStream scheduleInRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];
        [bodyStream open];
        
        uint8_t *buffer = NULL;
        NSMutableData *streamData = [NSMutableData data];
        
        while ([bodyStream hasBytesAvailable]) {
            buffer = (uint8_t *)malloc(sizeof(uint8_t) * 1024);
            NSInteger length = [bodyStream read:buffer maxLength:sizeof(uint8_t) * 1024];
            if (bodyStream.streamError || length <= 0) {
                break;
            }
            [streamData appendBytes:buffer length:length];
            free(buffer);
        }
        [bodyStream close];
        
        dispatch_async(dispatch_get_main_queue(), ^{
            !block ?: block([streamData copy]);
        });
    });
}
```
这里会出现一个小坑，因为该request的bodystream其本身就在其他线程中被随时读取，所以如果我们直接拿来做处理会导致程序崩溃，所以我们必须先对原始的bodystream进行拷贝操作。
紧接着我们通过GCD异步地将bodystream不断写入到新创建的streamData中去，最后将streamData返回给外部。  
至此，API请求的信息就被我们格式化完了。   

相对地，一旦收到有请求已经着陆的通知，就会调用我们的`apiDidLand:`方法，"流水线"跟`apiDidTakeOff`基本一致，在`apiDidLand`中，会调用`TDFAPILoggerAsyncJsonResponsePrettyFormat`函数，该函数会在该类内部创建的一个GCD队列`_tdfJsonResponseFormatQueue`中异步地将原始JSON串通过NSJSONSerialization类去加工成易于阅读的JSON串，并将Unicode转码成中文，然后返回给外部进行下一步的加工。因为`apiDidLand:`与`apiDidTakeOff`有蛮多类似的操作，所以一些细节这里就不重复了，有兴趣的可以直接阅读我的源码。  

## 最终效果
最后，让我们模拟几次请求，查看一下最终的日志输出效果。  

请求日志：
![ctdpic](https://ws1.sinaimg.cn/large/006tNc79gy1fsbt83j07lj311i0hgwja.jpg)
响应日志(成功)：
![ctdpic](https://ws4.sinaimg.cn/large/006tNc79gy1fsbt8yo24oj30ht0d3jtc.jpg)
响应日志(失败)：
![ctdpic](https://ws1.sinaimg.cn/large/006tNc79gy1fsbt98kpl7j30hi061my0.jpg)

## 参考
* [https://github.com/AFNetworking/AFNetworkActivityLogger](https://github.com/AFNetworking/AFNetworkActivityLogger)

