#逆向 iOS HomeKit 接口（非Accessory端）
PS:题目取为“逆向”实在是夸大了，只是在别人的基础上摸索出HomeKit端的接口,大神勿喷

最近公司有涉及智能家居的项目，所以就学习了HomeKit相关的内容。具体可以了解之前的两篇预研文章：[Apple HomeKit](http://blog.csdn.net/u013883974/article/details/53204676)和[Google Smart Home](http://blog.csdn.net/u013883974/article/details/53097168).

Apple继续发扬闭环的生态系统精神，HomeKit被打包成framework提供开发者使用，而且对于Accessory要求MFI认证。蛋疼的是MFI只对公司认证,个人开发者木有申请的资格。幸好有大神逆向了Accessory端的server代码[KhaosT](https://github.com/KhaosT)的[HAP-NodeJS](https://github.com/KhaosT/HAP-NodeJS)。这位大神在Apple实习过，正好是在HomeKit部门，你懂得😏。

首先是把HAP-NodeJS下载下来run一下，Mac下配置NodeJS环境和Python过程就略过了，网上百度很多教程。

代码主要是Accessory端的，如何同iOS device配对，接收device的指令，发送Accessory的状态参数给device。他其实是把HAP协议逆向出来了，工程里的Accessory都是用代码模拟出来的。可以与iOS的Home APP交互。

后来发现有点不对尽，只有Accessory端模拟HAP的代码，可以说只能是设备被动地去接收iOS的指令或者查询。

<div align=center >
Accessory ----------> iOS device
</div>
<div align=center >
Accessory <----X----- iOS device
</div>





           
通过HAP协议，可以将不支持HAP协议的智能硬件挂到Home App下，但是，如果需要你自己做一个平台向下能兼容HAP Accessory和 Non HAP Accessory，向上可以对接iOS平台和安卓平台，这个时候就需要该平台有两个线程，一个作为HAP Accessory的bridge，另外一个作为 Non HAP Accessory的bridge。

大致如图所示
<div align=center >
<img src="http://ww4.sinaimg.cn/mw690/7cafd2d5gw1f9vbl640k8j20j60j6acl.jpg"/>
</div>

在platform和phone之间需要自定义协议，而platform需要兼容两类Accessory。再回头看HAP-NodeJS的代码，只实现了Accessory端的服务，而作为自定义的platform，没有HomeKit framework的支持，不是太容易和HAP Accessory进行通信。最暴力的办法是模拟出HomeKit framework的功能。幸好Accessory端的代码有，至少知道Accessory是怎么接收数据，处理数据，然后返回的。

**先从最简单做起：局域网内设备Pair**

工具：wireshark、一台Mac（Windows也行，用于跑HAP-NodeJS服务）、XCode、sublime（用于修改和阅读NodeJS代码）

HAP底层是基于[Bonjour](https://developer.apple.com/bonjour/)零配置互相发现协议，而Apple的Bonjour又是基于mDNS来实现局域网内互相发现的.Object-C提供了[NSNetService](https://developer.apple.com/reference/foundation/netservice)和[NSNetServiceBrowser](https://developer.apple.com/reference/foundation/netservicebrowser)两个类，可以在上层发布和查找mDNS类的服务。所以，为了省事，在XCode上新建一个工程，用手机APP来测试。在这之前，你需要将HAP-NodeJS run起来，并且保证手机和HAP-NodeJS在同一局域网内。通过NSNetServiceBrowser来发现NSNetService并且解析HAP-NodeJS在局域网内发布服务的ip和端口。

~~~objective-c
- (void)findService{
    browser = [[NSNetServiceBrowser alloc] init];
    browser.delegate = self;
    NSRunLoop *mainRunLoop = [NSRunLoop currentRunLoop];
    [browser scheduleInRunLoop:mainRunLoop forMode:NSRunLoopCommonModes];
    [browser searchForServicesOfType:@"_hap._tcp" inDomain:@"local."];
    [mainRunLoop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:30]];
}
~~~
这里service type和Domain是怎么来，之前说的wireshark派上用场了，请移步下面wireshark截图，Host:Node\032Bridge._hap._tcp.local，就是这么来的😂
~~~objective-c
//发现服务
- (void)netServiceBrowser:(NSNetServiceBrowser *)browser didFindService:(NSNetService *)service moreComing:(BOOL)moreComing {
    NSLog(@"didFindService---------=%@  =%@  =%@",service.name,service.addresses,service.hostName);
    aNetService = service;
    [aNetService scheduleInRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];
    aNetService.delegate = self;
    [aNetService resolveWithTimeout:6];
    CFRunLoopRun();
}
//解析服务
- (void)netServiceDidResolveAddress:(NSNetService *)sender {
    NSLog(@"netServiceDidResolveAddress---------=%@  =%@  =%@",aNetService.name,aNetService.addresses,aNetService.hostName);
    NSArray *arr = [aNetService addressesAndPorts];
    NSLog(@"%@\n",arr);
    [self pullRequest:[arr objectAtIndex:0]];
}
~~~
将NSData解析成ip和端口的category

~~~objective-c
//
//  NSNetService+Util.h
//  HomeKitReverse
//
//  Created by SunCheng on 2016/11/17.
//  Copyright © 2016年 Channe. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface NSNetService (Util)

- (NSArray*)addressesAndPorts;

@end


@interface AddressAndPort : NSObject

@property (nonatomic, assign) int port;
@property (nonatomic, strong)  NSString *address;

@end
~~~

~~~objective-c
//
//  NSNetService+Util.m
//  HomeKitReverse
//
//  Created by SunCheng on 2016/11/17.
//  Copyright © 2016年 Channe. All rights reserved.
//

#import "NSNetService+Util.h"
#include <arpa/inet.h>

@implementation NSNetService (Util)

- (NSArray*)addressesAndPorts {
    
    // this came from http://stackoverflow.com/a/4976808/8047
    NSMutableArray *retVal = [NSMutableArray array];
    char addressBuffer[INET6_ADDRSTRLEN];
    
    for (NSData *data in self.addresses)
    {
        memset(addressBuffer, 0, INET6_ADDRSTRLEN);
        
        typedef union {
            struct sockaddr sa;
            struct sockaddr_in ipv4;
            struct sockaddr_in6 ipv6;
        } ip_socket_address;
        
        ip_socket_address *socketAddress = (ip_socket_address *)[data bytes];
        
        if (socketAddress && (socketAddress->sa.sa_family == AF_INET || socketAddress->sa.sa_family == AF_INET6))
        {
            const char *addressStr = inet_ntop(
                                               socketAddress->sa.sa_family,
                                               (socketAddress->sa.sa_family == AF_INET ? (void *)&(socketAddress->ipv4.sin_addr) : (void *)&(socketAddress->ipv6.sin6_addr)),
                                               addressBuffer,
                                               sizeof(addressBuffer));
            
            int port = ntohs(socketAddress->sa.sa_family == AF_INET ? socketAddress->ipv4.sin_port : socketAddress->ipv6.sin6_port);
            
            if (addressStr && port)
            {
                AddressAndPort *aAndP = [[AddressAndPort alloc] init];
                aAndP.address = [NSString stringWithCString:addressStr encoding:kCFStringEncodingUTF8];
                aAndP.port = port;
                [retVal addObject:aAndP];
            }
        }
    }
    return retVal;
    
}

@end

@implementation AddressAndPort
@end
~~~
为了更清楚地了解pair过程，可以用iPhone来走一遍流程，在Mac终端上用DEBUG=* node BridgedCore.js命令打开HAP NodeJS的debug功能，在一些关键性的步骤上加上debug命令可以看到终端的输出情况，如下图所示：
<div align=center >
<img src="http://ww2.sinaimg.cn/mw690/7cafd2d5gw1f9vc592jz0j20fu0a6q6r.jpg"/>
<img src="http://ww3.sinaimg.cn/mw690/7cafd2d5gw1f9vc69ffi6j20fu0a6mzx.jpg"/>
</div>
可以看到在发现服务后用HTTP来pair，**{ '0': \<Buffer 00>, '6': \<Buffer 01> }**就是我们需传的pair data。在HAPServer.js这个文件里面有各种type对应的编码，需要仔细看下pair是属于哪一类type，需要几步完成。

然后辅助wireshark，进一步知道HTTP body为[tlv](http://www.360doc.com/content/15/0716/15/16410669_485283089.shtml)格式的数据,在HAP NodeJS中也有tlv 的encode和decode的方法，可以按照他的流程移到XCode工程中，模拟一个HTTP pair的请求。
<div align=center >
<img src="http://ww3.sinaimg.cn/mw690/7cafd2d5gw1f9vc6xut6lj20np0kf0wr.jpg"/>
</div>
最后模拟一个pair请求，看看是否work

~~~objective-c
- (void)pullRequest:(AddressAndPort *)sender{
    NSURLSessionConfiguration *sessionConfiguration = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession *session = [NSURLSession sessionWithConfiguration:sessionConfiguration];
    NSURL *url = [NSURL URLWithString:[NSString stringWithFormat:@"http://%@:%d/pair-setup",sender.address,sender.port]];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    [request setValue:@"application/pairing+tlv8" forHTTPHeaderField:@"Content-Type"];
    [request setValue:@"6" forHTTPHeaderField:@"Content-Length"];
    YAUserRegisterReq *req = [[YAUserRegisterReq alloc]init];
    req.sequence = @"01";
    YATlv *reqTlv = [req buildTlvPacket];
    request.HTTPBody=reqTlv.tlvData;
    NSURLSessionDataTask *postDataTask = [session dataTaskWithRequest:request completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        NSLog(@"--response--%@",response);
        // The server answers with an error because it doesn't receive the params
    }];
    [postDataTask resume];
}
~~~
可以看到模拟成功了
<div align=center >
<img src="http://ww2.sinaimg.cn/mw690/7cafd2d5gw1f9vdg4lgyzj20sc064mzh.jpg"/>
<img src="http://ww1.sinaimg.cn/mw690/7cafd2d5gw1f9vdhu3h82j20fu0a6dij.jpg"/>
</div>
其他的接口也可以按照类似的方法，在HAP NodeJS的基础逆向出来😁
