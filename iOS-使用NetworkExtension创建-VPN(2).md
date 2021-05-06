前一篇文章我们只是用了NEVPNManager，使用系统协议创建VPN，而没有使用NETunnelProvider扩展网络核心网络层实现非标准化的私有VPN，接下来我们分析一下这一部分。
# NETunnelProviderManager
----
>1、 NETunnelProviderManager 是NEVPNManager 的子类，所以功能上和NEVPNManager 其实相同，它和VPN是一一对应的，我们设置的manager设置成NETunnelProviderManager，而不是NEVPNManager。
2 、我们接下来针对manager设置VPN的配置，此时按照NEKit的shadowsock 的参数来配置manager的protocolConfiguration，和我们之前的配置manager的conf是一样的。
3、 接下来我们还会去保存VPN、监听状态。
4、 打开或者关闭VPN。
----

```
在打开或者关闭的时候，如果要使用私有协议的时候就用到了接下来的NEPacketTunnelProvider这个类
```

# NEPacketTunnelProvider
NEPacketTunnelProvider 是VPN的核心代码，项目中我们创建的PacketTunnelProvider是NEPacketTunnelProvider的子类，我们必须实现的两个方法：
```
- (void)startTunnelWithOptions:(nullable NSDictionary<NSString *,NSObject *> *)options completionHandler:(void (^)(NSError * __nullable error))completionHandler NS_AVAILABLE(10_11, 9_0);  

- (void)stopTunnelWithReason:(NEProviderStopReason)reason completionHandler:(void (^)(void))completionHandler NS_AVAILABLE(10_11, 9_0);
```
当manager（NETunnelProviderManager）对象调用方法`startVPNTunnelWithOptions:andReturnError:`时，控制器将会跳转到Extension的`startTunnelWithOptions`的方法中。

我们看`- (BOOL)startVPNTunnelWithOptions:(nullable NSDictionary<NSString *,NSObject *> *)options andReturnError:(NSError **)error NS_AVAILABLE(10_11, 9_0);`的参数，第一个参数options:开发者自己定义的信息，第二个参数'completionHandler'：block回调。

`- (void)stopTunnelWithReason:(NEProviderStopReason)reason completionHandler:(void (^)(void))completionHandler;`,`reason `表示VPN被关闭的理由，第二个参数'completionHandler':block回调。

### 那我们如何调用Network Extension？
我们在完成PacketTunnel的代码情况下，我们调用Extension的代码。
- 1、运行应用
- 2、停止运行
- 3、 Xcode菜单中 'Debug->attach to process by PID or name',填入'GameVPNPacket',然后'Attach'.
![image.png](http://upload-images.jianshu.io/upload_images/1756292-b26cf3fc0f310b9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

-4、在手机上再运行应用。

# 总结：
在使用`PacketTunnelProvider`中会有使用`NEKit`的情况，之后需要研究一下`NEKit`的使用，具体的使用步骤大体相似。

参考：博客[Bloodline's Blog 构建 NetworkExtension 应用](http://ibloodline.com/articles/2017/11/15/NetworkExtension-02.html)
github代码[lettleprince 的 QLadder 代码](https://github.com/lettleprince/QLadder.git)。