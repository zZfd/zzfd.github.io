---
layout: post
title: "深入设备发现：mDNS 的注册、监听与异常 / 状态处理"
date: 2026-06-18 13:00:00 +0800
categories: 所思
tags: 设备发现 mDNS DNS-SD 组播 局域网
---

这是[《局域网设备发现与通信：mDNS + WebSocket 的工程化方案》](/2026/06/18/react-native-mdns-websocket-device-discovery)的姊妹篇,专门把**第一阶段「设备发现」**讲透。主文档负责串起发现 + 通信的全链路,本文则深入发现层一个个具体的工程问题:它依赖什么协议、怎么注册和监听、失败了怎么办、网络和应用状态变了又该怎么办。

> **前置知识**:本文会反复用到 DNS / PTR / SRV / 多播 / 广播域这些概念。久不碰的话,先花两三分钟读这份速查垫个底:《[计算机网络速查:读懂设备发现前要垫的几个底](/2026/06/18/network-basics-cheatsheet-for-device-discovery)》。一句话提要——**DNS 是按「名字 + 类型」查的命名数据库 → DNS-SD 复用 PTR/SRV/TXT 表达服务发现 → mDNS 把这套问答放到多播信道上做无中心广播 → 而多播信道受限于广播域,所以发现只能在同一网段内进行。**

## 一、mDNS 是什么:协议栈里的位置

**Multicast DNS,中文叫「多播 DNS」(也作组播 DNS)。** 它要解决的是:在**没有 DNS 服务器**的局域网里,把「域名/服务 → 地址」的解析能力,变成设备之间的**点对多点广播问答**。

它和它的搭档 **DNS-SD(DNS Service Discovery)** 常被合起来叫 Bonjour / Zeroconf——**Zeroconf 是这套零配置网络标准的通称,而 Bonjour 是 Apple 对它的实现与品牌名**(其开源核心 `mDNSResponder` 被广泛移植到各平台,可视为事实参考实现)。两者职责不同:

- **mDNS**:提供「在局域网内、无中心地解析名字」的能力(底层传输)。
- **DNS-SD**:在 mDNS 之上,约定**怎么用 PTR/SRV/TXT 记录来描述和枚举一个服务**(发现语义)。

关键认知:**mDNS 没有「连接」**。它不是 TCP 那样的会话,而是一问一答的无连接广播。这决定了它的失败模式、状态管理方式都和连接导向的通信层(WebSocket)完全不同。

## 二、底层依赖:它其实就是「跑在多播上的 DNS」

mDNS 并没有发明新协议,它是把**标准 DNS 报文格式**搬到了多播信道上。自底向上看它依赖的东西:

| 层次 | 依赖 | 说明 |
|------|------|------|
| 应用语义 | **DNS-SD** | 用 PTR/SRV/TXT 表达「服务有哪些实例、在哪、元数据是什么」 |
| 报文格式 | **DNS 报文** | 复用标准 DNS 的查询/应答报文结构 |
| 传输层 | **UDP** | 无连接、轻量;端口固定 **5353** |
| 网络层 | **IP 多播(IP Multicast)** | IPv4 组播地址 `224.0.0.251`、IPv6 `FF02::FB` |
| 链路层 | **IGMP / MLD** | IPv4 用 IGMP、IPv6 用 MLD 维护组播组成员关系 |

> 几个由此推出的工程后果:
> - **UDP 不可靠** → mDNS 自己靠「重复广播 + 缓存 + TTL」来对抗丢包,所以发现有**最终一致**的味道,不是即时强一致。
> - **多播是链路本地的** → 默认**不跨路由器/子网**。设备必须在同一广播域(同一 WiFi/VLAN)才能互相发现;企业网里的 AP 隔离、客户端隔离会直接让发现失效。
> - **依赖 IGMP/MLD** → 交换机若开了 IGMP snooping 且配置不当,组播可能被丢弃,这是排查「设备扫不到」时容易忽略的一环。

## 三、如何注册与监听

发现层有两类动作,分别对应主文档里的**发布方**和**浏览方**。

**注册(发布服务,Advertise / Register):** 把自己提供的服务登记到系统 mDNS 责任方。一次注册通常包含:服务类型(`_<service>._tcp`)、实例名、端口、TXT 元数据。系统拿到后会走两步:

1. **探测(Probing)**:先在网络里问一圈「这个实例名有没有被占用」,做**名称冲突检测**。
2. **公告(Announcing)**:确认无冲突后,组播公告该服务上线,并周期性重发以对抗丢包。

**监听(浏览服务,Browse):** 针对某个服务类型发起组播查询,系统通过回调持续吐出发现结果。注意「浏览」和「解析」是**两步**:

1. **Browse**:发现「有哪些实例」(PTR),得到实例名 → 触发 `serviceFound` / `serviceLost`。
2. **Resolve**:对某个实例做解析,拿到真正的 `host:port`(SRV)、IP(A/AAAA)和 TXT。**只有 resolve 成功,才有资格进入通信阶段去建连接。**

### Register 的具体实现

无论底层是哪套系统框架(Apple 的 Bonjour、Android 的 NsdManager),注册都是**异步、回调驱动**的:你提交一个服务描述,系统在后台跑探测+公告,再通过回调告诉你成功或失败。把它实现成一个**带状态的、可停可重来的管理器**,而不是「调一次就不管」:

{% highlight javascript %}
// 注册管理器(示意,封装平台差异)
class Advertiser {
  state = 'idle'; // idle | registering | registered

  register({ serviceType, port, instanceName, txt }) {
    // 幂等:无论当前什么状态,先拆掉旧的再重来,避免叠加
    if (this.state !== 'idle') this.unregister();

    this.state = 'registering';
    // 1) 组装服务描述:类型 + 实例名 + 端口 + TXT 元数据
    const service = nativeBuildService({ serviceType, port, instanceName, txt });

    // 2) 提交注册;系统内部完成 Probing(冲突检测)→ Announcing(组播公告)
    nativeRegister(service, {
      onRegistered: (registered) => {
        // 关键:系统可能因冲突自动改名,以"回调里返回的最终名"为准
        this.state = 'registered';
        this.actualName = registered.name;
      },
      onFailed: (err) => {
        this.state = 'idle';
        this.handleFailure(err); // 见第四节:按错误类型决定重试 / 改名 / 提示授权
      },
    });
  }

  unregister() {
    if (this.state === 'idle') return;
    // 注销会让系统组播一个 TTL=0 的 goodbye,通知对端"我下线了"
    nativeUnregister();
    this.state = 'idle';
  }
}
{% endhighlight %}

上面的 `nativeXxx` 在两个平台落到具体的系统 API 如下。**注意 Probing/Announcing 都不是这些 API「自己做」的,而是它们背后的系统 mDNS 守护进程(Apple 的 `mDNSResponder` / Android 的系统 mDNS 服务)做的**——你只管调用、等回调:

| 抽象动作 | Android(`NsdManager`) | iOS / macOS(Bonjour) |
|----------|----------------------|----------------------|
| 注册 | `registerService(info, PROTOCOL_DNS_SD, regListener)` | `NSNetService.publish(options:)`,或 Network framework 的 `NWListener`(广播 `service`) |
| 注册成功回调 | `RegistrationListener.onServiceRegistered(info)` | `NSNetServiceDelegate.netServiceDidPublish(_:)` |
| 注册失败回调 | `RegistrationListener.onRegistrationFailed(info, errorCode)` | `netService(_:didNotPublish:)` |
| 注销(发 goodbye) | `unregisterService(regListener)` | `NSNetService.stop()` |

> iOS 上「走 Bonjour」通常指 **`NSNetService` 那套 Foundation API**(底层连到系统 `mDNSResponder`);新项目也可用 **Network framework** 的 `NWListener`。两者都是 Bonjour 的不同封装,行为一致。

实现要点:

- **端口来自已成功 listen 的通信服务**——先把 WebSocket 服务起好、拿到实际端口(见主文档的动态端口),再来注册,顺序别反。
- **实例名从一开始就唯一**(如基名 + 设备唯一 ID),从源头规避冲突;但仍要**以 `onRegistered` 回调返回的最终名为准**(Android 读 `NsdServiceInfo.serviceName`,iOS 读 `NSNetService.name`),因为系统在 Probing 阶段检测到冲突时会**自动改名**(如 `MyDevice (2)`)。记下这个 `actualName` 的用途见下。
- **`register` 做成幂等**(内部先 `unregister`),这样网络变更/重连时直接再调一次即可,不会叠加出多个广播者。

> **为什么必须记 `actualName`?** 因为「你申请的名字」≠「网络上实际生效的名字」——冲突改名让两者可能不同,而这个最终名只在注册成功回调里给你。记住它有三个实打实的用途:① **正确注销/管理**(按名操作时不能拿申请名);② **自我过滤**(浏览时会发现自己广播的服务,要用 `actualName` 比对才能把"自己"从设备列表里剔除);③ **展示与日志关联**。

### Browse + Resolve 的具体实现

浏览这边有个**最容易踩的认知坑**:`serviceFound` 回调给你的**只有实例名和类型,没有地址**。要拿到 `host:port` 必须**对每个实例再做一次 resolve**。而且 resolve 是有代价的异步操作,**通常需要串行排队**(很多系统的解析器一次只能稳妥地处理一个请求),不能对一批实例并发猛发:

{% highlight javascript %}
// 浏览管理器:found 只给名字,必须二次 resolve(示意)
class Browser {
  state = 'idle';
  resolveQueue = [];   // 串行化:解析逐个来,避免并发解析互相打架
  resolving = false;

  browse(serviceType) {
    if (this.state !== 'idle') this.stop(); // 幂等:先停后起
    this.state = 'browsing';

    nativeStartDiscovery(serviceType, {
      onFound: (instance) => this.enqueueResolve(instance), // 只有 name+type!
      onLost:  (instance) => this.markOffline(instance),    // 实例消失 → 标记离线
      onFailed: (err) => { this.state = 'idle'; this.handleFailure(err); },
    });
  }

  enqueueResolve(instance) {
    this.resolveQueue.push(instance);
    this.drainQueue();
  }

  drainQueue() {
    if (this.resolving || this.resolveQueue.length === 0) return;
    this.resolving = true;
    const instance = this.resolveQueue.shift();

    nativeResolve(instance, {
      onResolved: ({ host, port, txt }) => {
        // 拿到真实地址 + 元数据,这才是可交给通信层的"成品"
        this.upsertDevice({ id: txt.id, host, port, txt });
        this.resolving = false;
        this.drainQueue(); // 处理下一个
      },
      onResolveFailed: (err) => {
        // 见第五节:超时/陈旧/已下线 → 标 stale,不要硬连;按需重试
        this.resolving = false;
        this.drainQueue();
      },
    });
  }
}
{% endhighlight %}

浏览与解析在两个平台的具体 API:

| 抽象动作 | Android(`NsdManager`) | iOS / macOS(Bonjour) |
|----------|----------------------|----------------------|
| 开始浏览 | `discoverServices(type, PROTOCOL_DNS_SD, discListener)` | `NSNetServiceBrowser.searchForServices(ofType:inDomain:)`,或 `NWBrowser` |
| 发现/消失回调 | `DiscoveryListener.onServiceFound(info)` / `onServiceLost(info)` | `netServiceBrowser(_:didFind:)` / `didRemove:` |
| 解析单个实例 | `resolveService(info, resolveListener)` | `NSNetService.resolve(withTimeout:)` |
| 解析结果回调 | `ResolveListener.onServiceResolved(info)`(从 `info` 取 host/port/TXT) | `netServiceDidResolveAddress(_:)`(读 `addresses`/`port`/TXT) / `didNotResolve:` |
| 停止浏览 | `stopServiceDiscovery(discListener)` | `NSNetServiceBrowser.stop()` |

> 一个具体坑:在 Android 上 `resolveService` 的 `ResolveListener` 历史上**一次只能可靠地处理一个解析请求**,所以上面代码里的「串行队列」不是洁癖,而是必要的——这也是为什么要把 resolve 排队而非并发。

实现要点:

- **found ≠ 有地址**:`onServiceFound` / `didFind` 只是「看见一个名字」,**必须 resolve** 才有 `host:port`。这是新手最常忽略的一步。
- **resolve 串行排队**:把解析请求放进队列逐个处理,避免并发解析互相干扰、或压垮系统解析器(尤见 Android)。
- **以 resolve 的新鲜结果落库**:用 `txt` 里的稳定标识(如 `id`)作为设备表的键做 upsert,天然对重复 found 去重(呼应第七节)。
- **found / lost 要对称处理**:`onServiceLost` / `didRemove` 到了就把对应设备标记离线,别让陈旧实例一直挂在列表里。

> 一句话概括两端:**注册 = 提交描述 → 系统 Probing/Announcing → 回调确认(可能改名)**;**浏览 = 发现名字 → 逐个 Resolve → 拿到 host:port 才算数**。两者都要做成「带状态、幂等、可停可重建」的管理器,才扛得住后面网络与应用状态的反复变化。

## 四、注册失败:原因与处理

注册不是「调一次就成」,要把它当成一个**可能失败、需要重试**的操作。常见失败原因与对策:

| 失败原因 | 处理策略 |
|----------|----------|
| **名称冲突**(实例名已被占用) | 自动改名重试——给实例名加唯一后缀(设备唯一 ID / 递增序号),让多台同型设备不撞名。系统多数会自动改名,但**应用层最好用一开始就唯一的实例名**从源头规避 |
| **网络不可用**(无 WiFi / 接口未就绪) | 不要立即判死。监听网络就绪事件,网络可用后**指数退避重试**注册 |
| **端口/资源问题**(通信服务没起来) | 注册端口应来自「已成功 listen 的通信服务」。**先把通信服务起好拿到实际端口,再注册**,顺序别反 |
| **权限缺失**(如移动平台的「本地网络」权限) | 这是移动端高频坑。捕获权限错误,引导用户授权;未授权时明确暴露「发现不可用」状态而非静默失败 |

> 通用原则:**注册要幂等且可重试**,失败要有明确的对外状态(而不是吞掉),并区分「暂时性失败(重试)」与「永久性失败(需用户介入,如授权)」。

## 五、解析失败:何时发生、如何处理

「Browse 到了实例」**不等于**「能解析出可用地址」。resolve 这一步随时可能失败,典型场景:

- **实例刚下线**:PTR 缓存里还在,但设备已离开,SRV/A 解析不到。
- **陈旧缓存(stale)**:拿到的是 TTL 内尚未过期、但实际已失效的旧记录。
- **地址已变**:设备 IP 变了(DHCP 续租、网络切换),旧地址解析失败或解析到错的 IP。
- **跨网段 / AP 漫游**:设备漫游到另一个 AP 后,原广播域里的记录失效。
- **goodbye 丢失**:设备非正常下线,TTL=0 的告别包没送达,留下「幽灵实例」。

处理策略:

1. **resolve 设超时 + 有限次退避重试**,不要无限等。
2. **失败即降级**:把该实例标记为 stale 或从候选列表剔除,**等下一轮 browse 刷新**,而不是拿失败的地址硬连。
3. **永远不要跳过 resolve**:不要用 browse 阶段的陈旧信息直接建连接——以 resolve 的新鲜结果为准。
4. **解析成功才进连接阶段**:把「resolve 成功」作为进入通信层的硬门槛。

## 六、网络变更:必须重新注册与监听

**网络接口一旦变化,发现层的状态基本全部失效,必须重建。** 因为:多播组成员关系、socket 绑定的接口/地址、对端可达性,都是和「当前网络接口」绑定的。

会触发重建的事件:WiFi 切换(换 SSID)、IP 变更(DHCP 续租)、AP 漫游、WiFi ↔ 蜂窝切换、接口上下线。

典型症状:**不重建的话,你的服务还广播在旧接口上,而对端在新网络里怎么也扫不到**;或者你 browse 到的全是旧网络的陈旧实例。

正确做法——监听系统的网络变化事件,做一次彻底的**先停后起**:

{% highlight javascript %}
// 网络变更 → 重建发现层(示意)
onNetworkChange(() => {
  stopAdvertising();   // 拆掉旧接口上的注册
  stopBrowsing();
  clearDiscoveryCache(); // 清空陈旧实例,避免拿旧地址连接

  // 等新接口就绪后重新来过(可加短延迟 + 退避)
  register(serviceType, port, txtMeta);
  browse(serviceType);
});
{% endhighlight %}

## 七、幂等性:如何避免重复注册与重复监听

重连、网络变更、应用状态变化都会触发「再来一次注册/监听」,**如果不做幂等,就会叠加出多个广播者和多个浏览者**——既浪费资源,又制造重复的 `serviceFound` 事件和混乱的状态。

几条防重复的工程手段:

- **集中到单一管理器**:所有 `register` / `browse` 只能经由一个单例的发现管理器,杜绝多处各自开。
- **维护显式状态标志**:`isAdvertising` / `isBrowsing`。start 前先判断,已在进行就不再重复开。
- **start 实现成「先 stop 再 start」**:让启动天然幂等——无论当前是什么状态,都先清掉旧的再建新的,而不是叠加。
- **重建用「unregister → register」,不要「register 之上再 register」**:网络变更/重连时先彻底拆,再重建。
- **发现结果按稳定 ID 去重**:即便底层重复广播,上层也以「设备唯一标识 + 是否已有活连接」来去重(这点和通信层连接表的去重一脉相承,见主文档)。

## 八、应用前后台切换:要不要重新注册与监听?

**取决于设备形态**——这和主文档「生命周期管理」一节是同一个权衡轴,但落到发现层有更具体的动作:

- **手持移动设备**(频繁切后台 / 锁屏):进后台时操作系统往往会挂起网络活动,继续持有注册/浏览既耗电又会留下陈旧状态。**建议进后台 stop 掉 advertise / browse,回前台再重新 register / browse。**
- **常驻在岗设备**(常亮、几乎不切后台):可以保持,避免无谓的断开重建开销。

但有一个**对所有形态都成立**的要点:**回到前台时务必做一次「对账」**。因为后台期间网络可能已经变过、设备列表可能已经过期。回前台应:

1. 重新 `browse` 一遍,**刷新**当前在线设备列表;
2. 清理掉后台期间失效的陈旧实例与死连接;
3. 必要时重新 `register`(尤其当后台期间发生过网络变更)。

> 一句话:**前后台切换本质上要当成「一次可能的网络变更」来对待**——回前台先重建/对账,再信任手里的设备列表。

## 九、小结:把发现层做成一个受控的状态机

把上面这些问题归拢,会发现「设备发现」远不止「调个 API 广播一下」。它是一个需要**显式状态管理**的子系统,要同时应对:不可靠的 UDP 多播、随时失效的解析、频繁变动的网络与应用状态。

可迁移的经验:

- **认清它的本质**:跑在 UDP 多播上的 DNS,无连接、最终一致、链路本地,失败是常态而非异常。
- **注册与监听都要幂等可重试**:start 即「先停后起」,集中到单例管理器,杜绝重复叠加。
- **resolve 是硬门槛**:browse 到 ≠ 能连上,必须 resolve 成功且新鲜,失败就降级刷新。
- **网络变更 = 全量重建**,**前后台切换 ≈ 一次可能的网络变更**,回前台先对账。
- **失败要可观测**:注册失败、解析失败、权限缺失都应暴露成明确状态,而不是静默吞掉。

把这些收敛成一个清晰的状态机(未注册 / 注册中 / 已广播 / 浏览中 / 重建中),发现层才能在真实的、抖动的网络里稳定工作——这也是它能把可靠的 `(host, port, 元数据)` 交给通信层的前提。
