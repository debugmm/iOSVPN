# 1、背景知识
iOS 8开始，apple 才开放新的框架NetworkExtension。iOS中的VPN分成个人VPN和非个人VPN开发。个人VPN开发简单，直接使用系统的IPSec、IKEv2协议来进行VPN连接。而iOS9之后，apple 开放新的api，开发者开发自己私密协议的VPN。
我们先看一下，apple关于[NetworkExtension的介绍](https://developer.apple.com/videos/play/wwdc2015/717/)的视频介绍.

### 主要介绍的内容是：
![image.png](http://upload-images.jianshu.io/upload_images/1756292-be7fdf5cda8f7063.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中主要用到的VPN的NEVPNManager和NETunnelProvider这两个类，其中NEVPNManager是比较简单的跟人VPN，而NETunnelProvider是实现企业VPN远程访问的方式，需要使用这个类。

# 2、开发前提
------
首先，需要在账号中，创建bundleID同时添加Network Extension和Personal VPN。
![image.png](http://upload-images.jianshu.io/upload_images/1756292-32d16eccdccfa340.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其次，需要在Xcode中使用新建target中添加
![image.png](http://upload-images.jianshu.io/upload_images/1756292-d34d9788f3b4796d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
但是未知原因苹果在mac OS 10.12中删除了这个文件，因此我们需要从10.11系统中提取或[下载](https://www.dropbox.com/s/6f1rxfjkk0ha89n/NEProviderTargetTemplates.pkg?dl=0)。

# 3、开发设置
添加相应的设置：
![image.png](http://upload-images.jianshu.io/upload_images/1756292-7899867e31ab5331.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
之后会在文件夹中生成后缀是`entitlements`的文件，我们查看会发现：
![image.png](http://upload-images.jianshu.io/upload_images/1756292-3afaee120dc0bc6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 4、开发个人VPN
我们在开发个人VPN的时候其实并没有用到我们在开始添加的`PacketTunnelProvider`文件，我们看一下具体的步骤：

----
## - 1、初始化一些信息
```
    //初始化一些信息
    self.serverName = @"com.alexYang.vpnServerName";
    self.vpmPasswordIdentifier = @"xxxxxxx"; //password 密码
    self.vpnPrivateKeyIdentifier = @"xxxxxxxxx"; //IPSec PSK
    [KeyChainHelper save:@"vpnPwd" data:self.vpmPasswordIdentifier];//将pwd放入钥匙串，因为我们读取密码的时候需要从钥匙串中读出
    [KeyChainHelper save:@"IPSecSharedPwd" data:self.vpnPrivateKeyIdentifier];//将PSK放入钥匙串
    
```
## - 2、创建VPN配置
```
 [self.manager loadFromPreferencesWithCompletionHandler:^(NSError * _Nullable error) {
        if (error) {
            NSLog(@"load error");
        }else{
            NEVPNProtocolIPSec *conf = [[NEVPNProtocolIPSec alloc] init];
            conf.serverAddress = @"xxx.xxx.xxx.xxx";
            conf.username = @"vpnuser";
            conf.authenticationMethod = NEVPNIKEAuthenticationMethodSharedSecret;//共享密钥方式
            conf.sharedSecretReference =  [[KeyChainHelper load:@"IPSecSharedPwd"] dataUsingEncoding:NSUTF8StringEncoding];//从keychain中获取共享密钥
            conf.passwordReference = [[KeyChainHelper load:@"vpnPwd"] dataUsingEncoding:NSUTF8StringEncoding];//从keychain中获取密码
            //本地id
            conf.localIdentifier = @"";
            conf.remoteIdentifier = @"xxx.xxx.xxx.xxx";//远程服务器的ID，该参数可以在自己服务器的VPN配置文件查询得到
            conf.useExtendedAuthentication = YES;
            conf.disconnectOnSleep = NO;//进入后台时是否断开VPN连接

            //按需连接，仅在wifi情况下连接，可以设置多种连接规则
            NSMutableArray *rules = [[NSMutableArray alloc] init];
            NEOnDemandRuleConnect *connectRule = [[NEOnDemandRuleConnect alloc] init];
            connectRule.interfaceTypeMatch = NEOnDemandRuleInterfaceTypeWiFi;
            [rules addObject:connectRule];
            self.manager.onDemandRules = rules;

            //self.manager.onDemandEnabled = NO;//按需连接不可用

            [self.manager setProtocolConfiguration:conf];
            [self.manager setOnDemandEnabled:conf];
            self.manager.localizedDescription = @"alexYang";
            self.manager.enabled = true;
            
            ///保存VPN配置
        }
    }];
```

## - 3、保存VPN配置
注意在保存VPN设置的时候，就需要我们确认是否允许我们进行VPN设置。
```
[self.manager saveToPreferencesWithCompletionHandler:^(NSError * _Nullable error) {
                if (error) {
                    NSLog(@"save error: %@", error);
                }else{
                    NSLog(@"save");
                }
            }];
```
执行这段代码之后请求用户授权，允许VPN的配置。

## - 4、VPN的开启或关闭
```
[self.manager loadFromPreferencesWithCompletionHandler:^(NSError * _Nullable error) {
        NSError *startError;

        [self.manager.connection startVPNTunnelAndReturnError:&startError];
        if (startError) {
            NSLog(@"start error %@", error.localizedDescription);
        }else{
            NSLog(@"Connection established");
        }
    }];
```
## - 5、监听VPN状态
```
//添加VPN状态变化通知
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(onVpnStateChange:) name:NEVPNStatusDidChangeNotification object:nil];


-(void)onVpnStateChange:(NSNotification *)Notification{
    NEVPNStatus state = self.manager.connection.status;
    
    switch (state) {
        case NEVPNStatusInvalid:
            NSLog(@"链接无效");
            break;
        case NEVPNStatusDisconnected:
            NSLog(@"未连接");
            break;
        case NEVPNStatusConnecting:
            NSLog(@"正在连接");
            break;
        case NEVPNStatusConnected:
            NSLog(@"已连接");
            break;
        case NEVPNStatusDisconnecting:
            NSLog(@"断开连接");
            break;
            
        default:
            break;
    }
}
```
---
# 5 总结其中问题：
---
>1、服务端搭建的协议要和我们使用的协议一样，ios 系统自带支持的协议时IPSec和IKEv2的方式，其他的方式像L2TP的方式似乎不支持或者需要自己去实现对应的协议。
>2、我们在开发这个简单的个人VPN，似乎没有用到PacketTunnelProvider 这个文件。
3、如果要使用自定义协议的VPN就要使用到PacketTunnelProvider这个文件。
4、每一个NEVPNManager 对应每一个VPN的设置。
---

# 6 IKEv2方式实现的代码
```
[self.manager loadFromPreferencesWithCompletionHandler:^(NSError * _Nullable error) {
        if (error) {
            NSLog(@"load error");
        }else{
//            NEVPNProtocolIPSec *conf = [[NEVPNProtocolIPSec alloc] init];
            NEVPNProtocolIKEv2 *conf = [[NEVPNProtocolIKEv2 alloc] init];
            conf.serverAddress = @"xxx.xxx.xxx.xxx";
            conf.username = @"vpnuser";
            conf.authenticationMethod = NEVPNIKEAuthenticationMethodSharedSecret;//共享密钥方式
            conf.sharedSecretReference =  [[KeyChainHelper load:@"IPSecSharedPwd"] dataUsingEncoding:NSUTF8StringEncoding];//从keychain中获取共享密钥
            conf.passwordReference = [[KeyChainHelper load:@"vpnPwd"] dataUsingEncoding:NSUTF8StringEncoding];//从keychain中获取密码
            //本地id
            conf.localIdentifier = @"";
            conf.remoteIdentifier = @"xxx.xxx.xxx.xxx";//远程服务器的ID，该参数可以在自己服务器的VPN配置文件查询得到,这两个值没有看到是必须设置的
            conf.useExtendedAuthentication = YES;
            conf.disconnectOnSleep = NO;

            //按需连接
            NSMutableArray *rules = [[NSMutableArray alloc] init];
            NEOnDemandRuleConnect *connectRule = [[NEOnDemandRuleConnect alloc] init];
            connectRule.interfaceTypeMatch = NEOnDemandRuleInterfaceTypeWiFi;
            [rules addObject:connectRule];
            self.manager.onDemandRules = rules;

            [self.manager setProtocolConfiguration:conf];
            [self.manager setOnDemandEnabled:conf];
            self.manager.localizedDescription = @"alexYang";
            self.manager.enabled = true;

            [self.manager saveToPreferencesWithCompletionHandler:^(NSError * _Nullable error) {
                if (error) {
                    NSLog(@"save error: %@", error);
                }else{
                    NSLog(@"save");
                }
            }];
            
        }
    }];
    
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(onVpnStateChange:) name:NEVPNStatusDidChangeNotification object:nil];
```
其实我们可以看出，其实IPSec 和 IKEv2 只是设置配置上的改变，其他的并无改变。
其中[NetworkExtension framework的具体介绍](https://developer.apple.com/documentation/networkextension?language=objc).

保存在钥匙串中的代码：
.h中
```
@interface KeyChainHelper : NSObject
+ (OSStatus)save:(NSString *)service data:(id)data;
+ (id)load:(NSString *)service;
+ (OSStatus)delete:(NSString *)service;
@end
```
.m
```
@implementation KeyChainHelper

+ (NSMutableDictionary *)getKeychainQuery:(NSString *)service
{
    return [NSMutableDictionary dictionaryWithObjectsAndKeys:
            (id) CFBridgingRelease(kSecClassGenericPassword), (id) CFBridgingRelease(kSecClass),
            service, (id) CFBridgingRelease(kSecAttrService),
            service, (id) CFBridgingRelease(kSecAttrAccount),
            (id) CFBridgingRelease(kSecAttrAccessibleAfterFirstUnlock), (id) CFBridgingRelease(kSecAttrAccessible),
            nil];
}

+ (OSStatus)save:(NSString *)service data:(id)data
{
    //Get search dictionary
    NSMutableDictionary *keychainQuery = [self getKeychainQuery:service];
    //Delete old item before add new item
    SecItemDelete((__bridge CFDictionaryRef) keychainQuery);
    //Add new object to search dictionary(Attention:the data format)
    [keychainQuery setObject:[NSKeyedArchiver archivedDataWithRootObject:data] forKey:(id) CFBridgingRelease(kSecValueData)];
    //Add item to keychain with the search dictionary
    return SecItemAdd((__bridge CFDictionaryRef) keychainQuery, NULL);
}

+ (id)load:(NSString *)service
{
    id ret = nil;
    NSMutableDictionary *keychainQuery = [self getKeychainQuery:service];
    //Configure the search setting
    //Since in our simple case we are expecting only a single attribute to be returned (the password) we can set the attribute kSecReturnData to kCFBooleanTrue
    [keychainQuery setObject:(id) kCFBooleanTrue forKey:(id) CFBridgingRelease(kSecReturnData)];
    [keychainQuery setObject:(id) CFBridgingRelease(kSecMatchLimitOne) forKey:(id) CFBridgingRelease(kSecMatchLimit)];
[keychainQuery setObject:@YES forKey:(__bridge id)kSecReturnPersistentRef];
    CFDataRef keyData = NULL;
    if (SecItemCopyMatching((__bridge CFDictionaryRef) keychainQuery,
                            (CFTypeRef *) &keyData) == noErr) {
        @try {
            ret = [NSKeyedUnarchiver unarchiveObjectWithData:(__bridge
                                                              NSData *) keyData];
        } @catch (NSException *e) {
            NSLog(@"Unarchive of %@ failed: %@", service, e);
        } @finally {
        }
    }
    if (keyData)
        CFRelease(keyData);
    return ret;
}
+ (OSStatus)delete:(NSString *)service
{
    NSMutableDictionary *keychainQuery = [self getKeychainQuery:service];
    return SecItemDelete((__bridge CFDictionaryRef) keychainQuery);
}
@end
```

个人总结，如有其他的不严谨之处请指出，会及时修改。


常见错误：
1、与服务器协议错误（IPSec）
IPSec PSK 密钥错误或者conf.localIdentifier 和 conf.remoteIdentifier设置错误。
2、未提供任何VPN共享秘钥（IPSec）
共享密钥认证属性设置错误或者IPSce协议中的密码和共享密钥没有使用keyChain中密码的永久引用，需要将kSecReturnPersistentRef设置成YES。
3、VPN服务器未响应。
服务器地址错误，或者服务器本身有错误。