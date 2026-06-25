---
layout: post
title: "深入设备通信：WebSocket 长连接的工程化实践"
date: 2026-06-18 14:00:00 +0800
categories: 所思
tags: WebSocket 设备通信 长连接 心跳 鉴权 加密
---

这是[《局域网设备发现与通信：mDNS + WebSocket 的工程化方案》](/2026/06/18/react-native-mdns-websocket-device-discovery)的姊妹篇,专门把**第二阶段「设备通信」**讲透——它和讲发现层的[《深入设备发现》](/2026/06/18/mdns-device-discovery-register-resolve-lifecycle)对称。

前一阶段(mDNS 发现)已经帮我们拿到了对端的 `(host, port, 元数据)`。本文要回答的是:**「拿到地址之后,如何在两台设备间建立一条稳定、高效、可双向收发的长连接,并让它在不可信网络里长期可靠地工作?」**

和无连接的发现层不同,**通信层是连接导向的,这里才有严格的客户端 / 服务端(C/S)角色**。下面从建连一路讲到鉴权加密。

## 前置:WebSocket 是什么

正文默认你对 WebSocket 有基本认识。久不碰的话,这一节先把几个最容易含糊的底垫上。

### 基于 TCP,位于应用层

WebSocket(RFC 6455)**建立在一条 TCP 连接之上**,不是 UDP——它要的是「可靠、有序、双向」的字节流,这正是 TCP 提供的。按 TCP/IP 分层,**WebSocket 是应用层协议,和 HTTP 平级**:

{% highlight text %}
WebSocket   ← 应用层（和 HTTP 同层；且握手阶段借用 HTTP）
   │
  TCP       ← 传输层（可靠、有序的字节流）
   │
   IP       ← 网络层
   │
 链路层
{% endhighlight %}

它和 HTTP 不只是同层,还有「血缘」:WebSocket 连接是**借 HTTP 的 Upgrade 机制**握手升级出来的(见下),握手成功后就脱离 HTTP、改说自己的帧协议。

> 为什么有了 HTTP 还需要 WebSocket?它和 SSE、长轮询怎么选?HTTP/3 又为什么改用 QUIC/UDP?这条"实时通信选型"的来龙去脉单独成篇:《[Web 实时通信选型:HTTP、SSE、WebSocket 与 HTTP/3·QUIC](/2026/06/18/realtime-web-transport-http-sse-websocket-quic)》。

### 和 socket 的关系:双向性从哪来

「socket」是**操作系统层面的通用抽象**——网络通信的端点(`socket()`/`connect()`/`send()`/`recv()` 那套 BSD socket API)。一条 TCP 连接两端各是一个 TCP socket。

**WebSocket ≠ socket**,尽管名字像,它俩是**上下层**关系:WebSocket 是应用层协议,**底层用一个 TCP socket 收发字节**,在其上加了握手、帧、控制帧。取名「WebSocket」是个类比——给 Web 应用一个「像 socket 一样」的持久双向通道。

由此回答一个常见疑问:**双向通信是 TCP 本来就有的,不是 WebSocket 发明的。** TCP 天生全双工,TCP socket 天生双向。WebSocket 的贡献是把这层能力**重新暴露给应用层**——它的对照对象是 **HTTP**(请求-响应、服务端不能主动推),不是 socket。WebSocket 真正自己实现的是:

- **消息分帧(framing)**:裸 TCP 是**无边界字节流**,WebSocket 在其上加帧,变成**面向消息**的双向通道;
- **握手协议**(借 HTTP Upgrade)、**控制帧**(ping/pong/close)、浏览器安全 API。

> 一句话:**TCP/socket 给的是「字节流的双向」,WebSocket 做成「消息的双向」并带到了 Web。**

### 如何建连:connect → opened

WebSocket 的「开场握手」分几步,`readyState` 从 `CONNECTING` 走到 `OPEN`:

{% highlight text %}
① TCP 连接        客户端 ──SYN/SYN-ACK/ACK──▶ 服务端   （wss:// 再叠一层 TLS）

② HTTP 升级请求   客户端 ──▶ 服务端
   GET /path HTTP/1.1
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: <随机 16 字节的 base64>
   Sec-WebSocket-Version: 13

③ 101 切换协议    服务端 ──▶ 客户端
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: <由 Key 推导>       ← 客户端校验此值

④ OPEN           升级完成 → 触发 onopen，此后按 WebSocket「帧」收发，不再是 HTTP

⑤ 关闭           任一端发 Close 帧(0x8) → 对端回 Close → 关闭底层 TCP
{% endhighlight %}

几个关键点:

- **`Sec-WebSocket-Key` / `Accept` 不是为了加密**,而是**确认对端真懂 WebSocket**(防止把普通 HTTP 服务或缓存代理误当成 WebSocket)。服务端把 Key 拼上固定魔法串做 SHA-1 再 base64 得到 Accept,客户端验算一致才认。
- **`101 Switching Protocols`** 表示「这条 TCP 连接从此不再走 HTTP,改走 WebSocket」。
- **升级后是「帧」而非「报文」**:数据切成带 opcode 的帧;**客户端→服务端的帧必须加掩码(masking)**,反向不加——防御针对中间代理的缓存投毒。
- **`readyState` 生命周期**:`CONNECTING(0) → OPEN(1) → CLOSING(2) → CLOSED(3)`。后文「发送前先查是否 OPEN」「连接销毁要清理」,判的就是这个状态。

### 各平台怎么用 WebSocket

选库之前,先分清**两个层次**——这是最容易选错的地方:

- **标准 WebSocket(RFC 6455)库**:说的是规范的 WebSocket 协议本身。`ws` / OkHttp / `URLSessionWebSocketTask` / 浏览器原生 `WebSocket` 都属于这层。
- **上层实时框架**:在 WebSocket 之上又包了一层**自有协议**,补上自动重连、房间/频道、ack、降级到长轮询等能力。代表是 **Socket.IO**(及 NestJS 的 WebSocket Gateway、Phoenix Channels 等)。

> **关键caveat:框架不是标准 WebSocket,两层不能混连。** Socket.IO 的服务端**只能**被 Socket.IO 客户端连上(它在 WS 之上加了自己的握手与帧),用浏览器原生 `WebSocket` 或 iOS `URLSessionWebSocketTask` 直连会失败。所以:
> - 如果你**两端都自己掌控、且要跨多种原生客户端互通**(正是设备通信这种场景)——**用标准 WebSocket**,别用 Socket.IO,否则各原生平台都得找对应的 socket.io 客户端,徒增耦合。
> - 如果是**纯 Web 前端 ↔ 自家后端**、想省掉重连/房间这些轮子——Socket.IO 这类框架更省事。

回到本方案:它是跨原生平台、自管两端的设备通信,**应走标准 WebSocket**。此时还有个硬约束——有一端要当**服务端(listen)**,而**浏览器、iOS 的 `URLSessionWebSocketTask` 只能当客户端**,服务端得选能 listen 的库:

| 平台 | 标准 WS 客户端(主流) | 服务端(能 listen) | 常见上层框架 |
|------|--------------------|------------------|------------|
| **Web** | 原生 `WebSocket` API | ❌ 浏览器只能当客户端 | Socket.IO-client |
| **Node.js** | `ws`;Node 22+ 有原生 `WebSocket` | **`ws`**(`WebSocketServer`,事实标准);NestJS Gateway 底层亦走它 | **Socket.IO** / NestJS Gateway |
| **Android** | **OkHttp**(多数 App 已内置 OkHttp) | Java-WebSocket / Ktor | Scarlet(Retrofit 风格)/ Ktor / socket.io-client-java |
| **iOS** | **`URLSessionWebSocketTask`**(iOS 13+,原生);**Starscream** 仍很常用 | Network framework(`NWListener`+`NWProtocolWebSocket`) | Starscream / socket.io-client-swift |

> 说明:真实项目里,Web/Node 之间做实时**常用 Socket.IO**(图它的重连与房间);但**一旦要和原生客户端互通,就回到标准 WS**。Android 客户端**主流是 OkHttp**(App 通常已经依赖它),复杂场景才上 Scarlet/Ktor 做协程封装;iOS 新项目用原生 `URLSessionWebSocketTask`,**Starscream** 因功能更全在存量项目里仍广泛使用。

**选服务端库时,优先平台原生——别背废弃的第三方库。** 这是个高频坑:很多存量 RN 项目的原生 WebSocket **服务端**,用的是早年的第三方 ObjC 库(iOS 上典型如 `SocketRocket` 做客户端、`PocketSocket` 做服务端)。这类库**多已停止维护甚至 archive**,还常被迫 **vendor(整份拷进仓库)** 才能打补丁——等于自己背着一堆没人管的废弃代码,是实打实的技术债。选型守则:

- **iOS**:服务端用 **Network framework**(`NWListener` + `NWProtocolWebSocket`,iOS 13+),客户端用 **`URLSessionWebSocketTask`**——**零第三方依赖、系统厂商持续维护、TLS 内建**,是替换老旧 ObjC WS 库的首选。
- **Android**:**Java-WebSocket**(纯 Java、能 listen)本身没问题,但**务必跟进版本**(存量项目常年钉在数年前的旧版,错过修复);要更现代就上 **Ktor**(Kotlin / 协程)。
- 通用口诀:**能用平台原生就别引第三方;非要第三方,先确认它「还在维护」**——一个 WS 库一旦停更,它的 TLS、协议兼容、安全修复就全停在了那一年。

最小标准 WS 用法(Web/Node 客户端 API 基本一致):

{% highlight javascript %}
const ws = new WebSocket('wss://host:443/path');
ws.onopen    = ()  => ws.send(JSON.stringify({ hello: 1 }));
ws.onmessage = (e) => handle(e.data);
ws.onclose   = ()  => reconnect();

// Node.js 服务端（ws 库）
import { WebSocketServer } from 'ws';
const wss = new WebSocketServer({ port: 0 });      // 端口 0 = 系统分配
wss.on('connection', (sock) => sock.on('message', (m) => sock.send(m)));
{% endhighlight %}

> 易踩坑:**iOS 的 `URLSessionWebSocketTask.receive` 是「一次性」的**——收到一条后要**再调一次 `receive`** 才能继续收,需自己递归续读;OkHttp / 浏览器是回调式的,自动持续推。

## 一、通信层的 C/S 模型与动态端口

WebSocket 是一个明确的 C/S 协议:

- **WebSocket 服务端**:在某个端口上 `listen`,被动接受连接。
- **WebSocket 客户端**:用发现阶段得到的 `host:port` 主动 `connect` 拨入。

通常**由 WebSocket 服务端那台设备担任发现层的发布方**——它先把自己的服务端口广播出去,客户端发现后才能拨进来。这就把发现与通信两个阶段衔接了起来。

**服务端口不应写死**(如 `8080`),否则可能与设备上其它进程冲突。正确做法是把端口分配权交给操作系统:

1. 服务端 socket **绑定端口 `0`**;
2. 操作系统从 ephemeral 端口范围里**自动分配一个空闲端口**;
3. 通过 `getsockname` 把系统实际分配的端口**读回来**;
4. 把这个实际端口**写进 mDNS 广播**(SRV 的端口字段,或 TXT 元数据)——这正是发现阶段「服务端口」字段的来源。

{% highlight text %}
服务端 bind(0) → OS 分配 51843 → 写入 mDNS 广播 → 客户端发现后据此拨号
{% endhighlight %}

这样彻底规避了端口冲突,也不需要任何「约定端口」。发现与通信两个阶段的衔接闭环如下:

{% highlight text %}
  [发现层]  发布方 announce  ──组播──▶  浏览方 browse → 拿到 host:port
                                              │
  [通信层]  WS 服务端 listen ◀──connect───  WS 客户端 dial(host:port)
                              握手 → 一条全双工长连接，消息靠 ID 关联
{% endhighlight %}

## 二、消息信封：统一数据结构

在讨论"怎么收发"之前,先定死"消息长什么样"。这一层最容易乱,根源是把**两个正交维度**混在一起谈,先拆开:

- **序列化格式**:消息以什么编码上线(JSON?二进制 schema?)。
- **信封结构**:消息逻辑上的形状(怎么表达"这是什么消息、关联谁、装什么数据")。

下面先定信封结构,再定序列化格式。

### 一个信封,覆盖三种消息

目标:用**同一个信封**表达请求 / 响应 / 事件三种消息,且自描述、可序列化、可演进。

{% highlight json %}
{
  "v": 1,                     // 协议版本（整数，破坏性变更时自增）
  "id": "01H...",             // 本条消息唯一身份（即请求-响应里的 messageId）
  "type": "order.create",     // 消息种类：自描述字符串（命令名 / 事件名）
  "responseId": null,         // 非 null = 这是对某条消息的回应
  "ts": 1719300000000,        // 发送时间戳（可观测 / 辅助，不用于定序）
  "payload": { "table": 5 },  // 业务数据（成功响应也放这）
  "error": null               // 仅“失败响应”时非 null
}
{% endhighlight %}

字段取舍上有个值得对照的参照系——平台内置的线程消息原语(如 Android 的 `Message`:`what` / `arg` / `obj` / `replyTo` / `when`)。它的骨架(类型 + 负载 + 回复地址 + 顺序)同样适用,但**网络信封要反着选字段**:

| 概念 | 单机消息(如 Android Message) | 网络信封该选 | 为什么 |
|------|------------------------------|-------------|--------|
| 类型 | `what`(int 操作码) | **字符串 `type`** | 自描述、可读、跨语言对得上 |
| 负载 | `obj`/`Bundle`(任意对象) | **可序列化 `payload`** | 网络上传不了原生对象 |
| 关联 | `replyTo`(平台句柄) | **`responseId`(id 引用)** | 通用、不绑平台 |
| 版本 | (无) | **`v`** | 两端会异步升级,必须能协商兼容 |

### 三种消息 = 字段组合

| 消息 | `responseId` | `payload` | `error` | 发送方 | 接收方 |
|------|:---:|:---:|:---:|--------|--------|
| **请求 request** | `null` | 参数 | `null` | 登记 pending,等回应 | 处理后**回一条响应** |
| **响应 response** | =原请求 id | 成功结果 | 或错误 | — | 按 `responseId` 匹配挂起请求 |
| **事件 event** | `null` | 事件数据 | `null` | **不登记 pending** | **不回应** |

**请求与事件在线上都是 `responseId=null`,如何区分?** 用 **`type` 路由**,不用额外字段:接收方按 `type` 注册处理器,而处理器本身分两类——命令型(处理后必回响应)、事件型(不回)。发送方调 `request()`(登记 pending)还是 `emit()`(不登记)与之天然对齐。所以**不需要 `kind` 字段**,`type` 已隐含。

### 错误的表达

只在**失败响应**里出现 `error`,且与 `payload` **二选一**(借 JSON-RPC 的规矩);`code` 用字符串枚举而非裸数字:

{% highlight json %}
{ "v":1, "id":"...", "type":"order.create", "responseId":"<原id>",
  "error": { "code": "ORDER_CONFLICT", "message": "桌台已占用", "data": {} } }
{% endhighlight %}

### 可选扩展:定序与幂等

基础信封保持精简;需要**严格定序或恰好一次**的命令,再加两个**可选**字段(普通事件/幂等查询不带):

{% highlight json %}
{ "...": "...",
  "seq": 42,                 // 每连接单调递增 → 服务端只认 last+1、忽略 ≤last
  "idempotencyKey": "uuid"   // 幂等键 → 重连重发不会被执行两次
}
{% endhighlight %}

定序靠 `seq`,**不靠 `ts`、更不靠到达顺序**(TCP 只保单连接的到达序,保不了完成序与跨连接顺序);幂等靠 `idempotencyKey`,重连重放未确认命令时供服务端去重。

### 序列化格式(另一个维度)

| 格式 | 优 | 劣 | 何时用 |
|------|----|----|--------|
| **JSON** | 可读、通用、零依赖、调试友好 | 体积大、解析慢、无强类型 | **默认** |
| **MessagePack / CBOR** | 比 JSON 小/快,结构兼容 | 不可读 | 带宽/性能敏感 |
| **Protobuf / FlatBuffers** | 最小最快、强类型 + 版本演进 | 需 schema、编译、不可读 | 高吞吐、强类型、多端 |

两维度**正交**:信封结构不变,带宽/类型成为瓶颈时**只换序列化格式**(JSON → Protobuf),上层逻辑不动。

> 收口:**一个信封 = `v / id / type / responseId / ts / payload / error`(可选 `seq / idempotencyKey`)**;三种消息靠字段组合 + `type` 路由区分;定序靠 `seq`、幂等靠 key、版本靠 `v`。后面的连接管理、请求-响应关联、可靠性,全都建立在这个信封之上。

## 三、连接管理：请求-响应关联与连接表

建立连接后要管理好它。这里有两个常被讲糊的点,先把概念厘清。

### 1. 不是「多路复用」,而是请求-响应关联

WebSocket 本身就是**全双工**的——两端随时都能向对方推消息,这是协议自带的能力,不需要我们做任何「多路复用」。所以把业务消息说成「在一条连接上多路复用」是不准确的。

真正需要我们做的是两件事:

1. **复用一条长连接**:两个节点之间只维持**一条**持久 WebSocket,所有消息都走它,而不是「每次请求开一条新连接」。省掉反复握手的开销。
2. **请求-响应关联(correlation)**:因为通道是**异步、双向**的,响应回来时你无法天然知道「它是对哪条请求的回应」。解决办法是给每条消息一个 `messageId`;当某条消息是**对另一条消息的回应**时,它带上 `responseId = 原请求的 messageId`。请求方据此把回应对上号。

{% highlight javascript %}
// 请求方：发出去时记下 messageId，按它等待对应回应（示意）
const messageId = uuid();
send({ messageId, type: 'REQUEST', payload });           // 不含 responseId

const reply = await race({
  response: waitForMessage((m) => m.responseId === messageId), // 按 responseId 对上号
  timeout:  delay(REQUEST_TIMEOUT),                       // 应用层超时，避免无限等待
});

// 响应方：回应时把对方的 messageId 原样回填进 responseId（示意）
function onRequest(req) {
  send({ messageId: uuid(), responseId: req.messageId, type: 'RESPONSE', payload });
}
{% endhighlight %}

> 关键区别:`messageId` 是「这条消息的身份」,`responseId` 是「我在回应哪条消息」。有了这对 ID,**多个异步请求-响应可以在同一条双向连接上并发地交错进行而不串台**——靠的是 ID 关联,不是「多路复用」。没有 `responseId` 的消息就是一条单向通知(不需要回应)。

### 2. 连接表解决什么问题

连接表是一张「**当前在线设备 → 它的活连接**」的注册表,**以对端稳定的 `deviceId` 为键**(而不是易变的 socket 句柄或 IP)。它要解决三个具体问题:

- **按设备寻址**:业务要「给某台设备发消息」时,得用稳定的 `deviceId` 找到它当前那条 socket——地址会变、socket 会换,只有 deviceId 稳定。
- **一设备一连接(去重)**:同一台设备因重连/重复发现可能多次连进来。用 deviceId 作键,保证它在表里**只占一个槽位**,新连接覆盖旧的。
- **在线状态管理**:表里有 = 在线,移除 = 离线,对外发设备上下线事件的依据。

### 3. 重连时:如何正确替换旧连接、回收内存

这是连接表**最容易出错、最容易漏内存**的地方。设备重连时,表里那条旧连接不能简单地被新连接「覆盖」就完事——**旧的 socket 对象、它身上挂的事件监听、心跳定时器,如果不显式拆掉,会一直被引用、无法被 GC 回收**,既漏内存,旧心跳还可能继续乱发。

正确的替换要**先彻底拆除旧连接,再装入新连接**:

{% highlight javascript %}
// 重连：先销毁旧连接，再登记新连接（示意）
function onClientConnected(deviceId, newConn) {
  const old = connections.get(deviceId);
  if (old && old !== newConn) destroyConnection(old); // 关键：先拆旧的
  connections.set(deviceId, newConn);
}

// 彻底销毁一条连接，让它能被回收
function destroyConnection(conn) {
  conn.clearHeartbeat();           // 1) 清掉心跳/超时定时器（否则定时器一直持有 conn）
  conn.socket.removeAllListeners(); // 2) 摘掉所有事件监听（否则闭包一直引用 conn）
  try { conn.socket.close(); } catch (_) {} // 3) 关闭底层 socket，释放 fd
  conn.socket = null;              // 4) 断开强引用，交给 GC
}

// 设备离线：从表里移除并销毁
function onClientGone(deviceId) {
  const conn = connections.get(deviceId);
  if (!conn) return;
  connections.delete(deviceId);    // 先移出表，避免别处再拿到它
  destroyConnection(conn);
}
{% endhighlight %}

要点 / 易踩的坑:

- **定时器和监听器是头号泄漏源**:只 `close()` socket 而不清心跳定时器、不摘事件监听,`conn` 仍被定时器回调和监听闭包引用,GC 收不走。三者必须一起拆。
- **先 `delete` 再 `destroy`**:先把条目移出表,确保业务不会再从表里取到这条将死的连接。
- **用连接身份防「旧连接的迟到事件」误删新连接**:旧 socket 关闭后,它的 `close` 事件可能**晚于**新连接登记才触发。`close` 回调里要判断「**表里当前这条是不是我自己**」(按连接实例 / 一个唯一的 connId 比对),是我才从表里移除——否则会把刚装上的新连接误删:

  {% highlight javascript %}
  socket.on('close', () => {
    // 仅当表里仍是“我”这条连接时才移除，避免误删重连后的新连接
    if (connections.get(deviceId) === thisConn) onClientGone(deviceId);
  });
  {% endhighlight %}

- **清理待响应的请求**:连接销毁时,所有还在 `waitForMessage` 等待的请求应被立即以「连接已断」拒绝(reject),否则它们只能干等到超时,既拖慢上层、其 Promise 也悬挂着持有引用。

### 4. 重复发现的去重

发现阶段的 mDNS 广播会重复到达,同一服务会被多次「发现」。浏览方在发起连接前需按稳定标识去重,判定依据应是「**是否已有活连接**」(查连接表),而非「是否见过这个 ID」——否则设备掉线重连会被错误拦截。

## 四、消息可靠性：超时、发送失败与重试

WebSocket 给你的是一条**字节管道**,它**不保证应用语义上的「送达」**:`send()` 返回了不代表对端收到了、更不代表对端处理了。在不可信网络里,消息会超时、会发不出去、连接会在半路断掉。要让上层用得放心,得在应用层补上一套可靠性机制。

### 1. 三种要分开处理的失败

| 失败 | 现象 | 该怎么判定 |
|------|------|-----------|
| **发送失败** | `send()` 当场抛错 / 连接不在 OPEN 态 | 同步可知:发之前先查连接状态,发不出去立即失败 |
| **响应超时** | 发出去了,但约定时间内没等到 `responseId` 对应的回应 | 靠**请求级超时定时器**(见下),到点判超时 |
| **连接中断** | 等待回应期间连接断了 | 连接销毁时**立即 reject 所有挂起请求**(见上一节),不要干等到超时 |

把这三者分开,上层才能拿到**准确的失败原因**,而不是笼统一个「失败」。

### 2. 每个请求一个超时,且要能被多种方式叫醒

请求-响应是异步的,所以**每一条期待回应的请求都要挂一个独立的超时**,并且这个等待应当能被「超时 / 收到回应 / 连接断开」三者中**任何一个**结束:

{% highlight javascript %}
// 发送一个需要回应的请求,统一处理三种结局(示意)
function request(deviceId, payload, { timeout = REQUEST_TIMEOUT } = {}) {
  const conn = connections.get(deviceId);
  // ① 发送失败:连接不可用,当场失败,不进入等待
  if (!conn || conn.socket.readyState !== OPEN) {
    return Promise.reject(new SendError('connection not open'));
  }

  const messageId = uuid();
  return new Promise((resolve, reject) => {
    // ② 响应超时:到点 reject,并清掉登记,避免泄漏
    const timer = setTimeout(() => {
      conn.pending.delete(messageId);
      reject(new TimeoutError(messageId));
    }, timeout);

    // 登记“在等这条的回应”;连接销毁时会遍历 pending 逐个 reject(③ 连接中断)
    conn.pending.set(messageId, { resolve, reject, timer });

    try {
      conn.socket.send(serialize({ messageId, payload }));
    } catch (e) {
      // 发送过程本身抛错:回滚登记,立即失败
      clearTimeout(timer);
      conn.pending.delete(messageId);
      reject(new SendError(e));
    }
  });
}

// 收到回应:按 responseId 找到挂起请求,结束它
function onResponse(msg) {
  const p = conn.pending.get(msg.responseId);
  if (!p) return;            // 没有对应挂起请求 → 多半是迟到的超时回应,丢弃
  clearTimeout(p.timer);     // 关键:清掉超时定时器,否则泄漏
  conn.pending.delete(msg.responseId);
  p.resolve(msg);
}
{% endhighlight %}

要点:**无论哪种结局,都要 `clearTimeout` + 从 `pending` 移除**——这和上一节讲的内存回收是一回事,挂起请求的定时器和 Promise 回调是另一个高发泄漏点。

### 3. 重试:幂等是前提

超时/发送失败后要不要重试?**取决于该操作是否幂等。**

- **幂等操作**(查询、状态上报、可去重的写):可以安全重试。**重试时复用同一个业务 id**(而非每次换新 id),让对端能识别「这是同一条的重发」并去重,避免「超时其实成功了,重试又做一遍」。
- **非幂等操作**(扣款、开闸这类副作用):**不能盲目重试**。要么把它设计成幂等(带唯一业务流水号 + 对端去重),要么对超时采取「**先查询后决定**」——查对端到底执行没执行,再决定补发还是放弃。

> 一句话:**「超时」不等于「失败」**——它只代表「我没等到回应」,对端可能成功了、也可能没收到。这个不确定性必须靠**幂等 + 去重**来消化,而不是假设超时就是没做。

### 4. 顺序与背压

- **顺序**:WebSocket 单连接内,消息是**按发送顺序到达**的(底层 TCP 保证)。但**应用层的异步处理会打乱完成顺序**——若业务依赖严格顺序,需要自己用 sequence 号定序,别假设「先发的一定先处理完」。
- **背压(backpressure)**:对端或网络慢时,只管 `send()` 会让发送缓冲无限堆积、吃光内存。要关注发送缓冲水位(如 `bufferedAmount`),超过阈值就**暂缓发送 / 丢弃可丢的低优先级消息**,而不是一味往里塞。

## 五、活性检测与重连

这是通信层**最难**的部分。在不可信网络里，「准确地知道一个连接死了」远比「建立连接」难。可靠的活性检测应当**多层兜底**：

1. **应用层心跳（主力）**：在长连接上定时收发 ping/pong。约定一个**心跳间隔**和一个**超时阈值**，在阈值内收不到回应即判定连接已死。心跳逻辑放在贴近连接的底层（如 native 层）比放在 JS 层更可靠，不受上层事件循环阻塞影响。

   {% highlight text %}
   ping 间隔 :  每 N 秒发一次心跳
   pong 超时 :  M 秒（M > N）内无任何回应 → 判定掉线
   典型取值 :  间隔 5s / 超时 10s 量级（按网络质量调整）
   {% endhighlight %}

2. **发现层 goodbye**：设备**主动**下线时，mDNS 会广播一个 `TTL=0` 的 goodbye 记录，让对端立刻知道「我要走了」，无需苦等心跳超时。多数系统 mDNS 栈在 `unregisterService` / `stop` 时会自动发出，无需手写。

3. **传输层兜底**：TCP keepalive 可作为最底层的探活，但它触发慢、粒度粗，**只能兜底，不能当主力**——应用层心跳才是核心。

检测到掉线后，**重连不能莽撞**。一群设备同时掉线又同时疯狂重连，会形成**重连风暴**，把刚恢复的网络二次打垮。标准做法是**指数退避 + 随机抖动（jitter）**：

{% highlight javascript %}
// 指数退避 + 抖动，错开重连时刻（示意）
function nextRetryDelay(attempt) {
  const base   = Math.min(maxDelay, baseDelay * 2 ** attempt);
  const jitter = Math.random() * base * 0.3; // 加随机，避免同步重连
  return base + jitter;
}
{% endhighlight %}

还有一个常被忽略的判断：**区分「网络抖动」与「真掉线」**。短暂抖动应快速恢复原连接；真掉线才彻底清理连接表、对外发「设备离线」事件。两者混为一谈，会导致 UI 上设备状态疯狂闪烁。

## 六、生命周期管理：场景决定策略

移动操作系统会在应用进入后台时回收 socket，所以连接的生命周期必须挂到应用与系统的生命周期上。但**具体策略取决于设备的使用形态**，这是一个需要显式权衡的设计轴：

- **手持移动设备**（频繁切后台、锁屏）：进入后台 / 锁屏时应**主动停广播 + 优雅断连**（与其让系统粗暴回收，不如自己先 goodbye），回到前台再重新广播 + 重连。否则会出现「系统已杀连接、应用却以为还在线」的幽灵状态。
- **常驻在岗设备**（长时间常亮、几乎不切后台）：反而应当**在后台保持服务存活**，避免不必要的断连重连开销。

> 这里没有放之四海皆准的答案。**关键是认清目标设备的形态，并把生命周期策略作为一个有意识的决策**，而不是照搬「进后台就断连」的教条。

### 网络变更：服务端要不要停掉重启?

网络变化(DHCP 续租、换 WiFi、接口上下线)时,一个常见的过度反应是无脑 `stopServer()/startServer()`。**多数情况并不需要**——要不要重启,取决于**监听 socket 绑的是什么地址**;而且要把"重启 server"和"重广播 mDNS"两件事分清。

**先把两层拆开:** 服务端在网络上挂着两样东西——**TCP 监听 socket** 和 **mDNS 广播**。网络变更时,它们的处理完全不同:

| 绑定方式 | IP 变化时,监听 socket | 结论 |
|----------|----------------------|------|
| **绑 `0.0.0.0`(INADDR_ANY,推荐)** | 不认死某个 IP,在所有接口上 accept,IP 变了通常**照常工作** | **无需重启 server** |
| **绑了具体 IP** | 该 IP 一消失,socket 绑在不存在的地址上,accept 失效 | **必须 rebind**(关掉重绑到新 IP 或 `0.0.0.0`) |

所以**第一原则:监听 socket 一开始就绑 `0.0.0.0`**,就能避免绝大多数"网络变更要重启"的情况。

**网络变更时真正必做的,是这两件(都不是重启 TCP server):**

1. **重新广播 mDNS**:A 记录指向的是旧 IP / 旧主机名,地址变了必须重新 announce,客户端才能发现新地址。这是**发现层**的职责(详见[《深入设备发现》的"网络变更"一节](/2026/06/18/mdns-device-discovery-register-resolve-lifecycle))——和 TCP 监听 socket 是两码事,别混。
2. **reap 失效的旧连接**:已建立的客户端连接按四元组绑定,IP 一变就死了;但这靠**心跳判死 + 客户端重连**收拾,**不需要重启 server**。

恢复路径是:**server 重新广播 → 客户端重新发现 + 重连**,监听 socket 完全可以不动。

> 若确实被迫 rebind(绑了具体 IP,或防御性重建),注意**动态端口的坑**:`bind(0)` 重启会拿到**新端口**→ 必须重新广播;想保持稳定就**显式绑回原端口**。
>
> 一句话:**绑 `0.0.0.0` 时,网络变更 = 重广播 mDNS + reap 死连接,而不是重启 server**;无脑 `stop/start` 反而会换端口、踢掉本可保留的监听 socket,制造一次不必要的全量重连。

### 网络变更：客户端必须重连

和服务端相反——**客户端 IP 一变,连接必死,重连不可避免**。原因在于客户端持有的是一条**已建立的出站连接**,被钉死在四元组上(`客户端IP:端口 → 服务端IP:端口`);客户端 IP 一变(换 WiFi、WiFi↔蜂窝、换租约),旧四元组的源 IP 就没了,**这条 TCP 连接无法迁移、直接失效**。

| | 服务端监听 socket | 客户端已建立连接 |
|---|---|---|
| 绑定 | 可绑 `0.0.0.0`,不认死 IP | **必然钉死在源 IP 的四元组上** |
| IP 变化 | 通常照常 accept | **连接死亡,无法迁移** |
| 结论 | 多数不用重启 | **必须重连** |

客户端**没有"绑 `0.0.0.0` 就能扛住"那一招**。但有三点要把握:

1. **别一有网络事件就盲目重连**。网络变更事件 ≠ IP 一定变了:同 AP 抖一下、DHCP 续同一个 IP,连接多半还活着;但 NAT 重置 / AP 漫游也可能悄悄断。所以把网络事件当成「**立刻核实**」的信号——**确知 IP 变了就直接重连**(比等心跳超时快),**拿不准就发一次 ping 探**,通的别动、不通才重连。
2. **重连常要连"重新发现"一起做**。若客户端切到了**另一个网段**,旧的服务端地址多半已不可达(不在同一广播域),不能拿旧 `host:port` 硬连——要**重新 mDNS 浏览**拿到新地址再拨号。
3. **重连 ≠ 丢会话**。传输层连接虽必须重建,但应用层会话可无缝续上:重连 → 重新鉴权 → **恢复**(outbox 重放未 ack 命令 + 幂等去重、`seq`/`lastEventId` 补拉漏掉的事件)。**传输要重连,状态不必丢。**

> 唯一能"换网不断连"的是 **QUIC / HTTP3 的连接迁移**(靠独立于 IP 的 Connection ID,正是 HTTP/3 改用 UDP 的好处之一,见[《Web 实时通信选型》](/2026/06/18/realtime-web-transport-http-sse-websocket-quic))。但只要用标准 WebSocket(TCP),**客户端换网就必须重连,没有例外**。

## 七、安全：连接鉴权与消息加密

局域网设备通信的安全,要**按信任边界来权衡**,而非简单的「加密 / 不加密」二选一。它由两个正交的问题组成:**鉴权**(你是谁、是不是我该信的设备)和**加密**(传输内容会不会被看到/篡改)。下面分开讲怎么落地。

### 1. 连接鉴权:确保只有合法设备能建会话

鉴权应当**在握手阶段、业务消息之前**完成——一条没通过鉴权的连接,根本不该进入连接表、不该能收发业务消息。常见做法从弱到强:

| 方案 | 做法 | 适用 |
|------|------|------|
| **共享密钥/令牌校验** | 握手时带上一个双方约定的 token / 对凭证签名,服务端校验通过才接受连接 | 封闭、设备可信的局域网底线方案 |
| **质询-应答(challenge-response)** | 服务端发一个随机 nonce,客户端用密钥对它签名/加密后回传,服务端验证 | 比"直接发 token"强:**密钥不上网**,且 nonce 一次性,可防重放 |
| **双向证书(mTLS)** | 两端各持证书,在 TLS 握手层互验身份 | 信任要求高、有证书签发能力时 |

一个稳妥的握手鉴权流程(质询-应答)大致是:

{% highlight text %}
客户端 connect ──▶ 服务端
服务端 ──▶ 发随机 nonce(质询)
客户端 ──▶ 回 sign(nonce, key) + 自己的身份信息(deviceId 等)
服务端 ── 验签通过? ──┬─ 是 ─▶ 接受连接、登记进连接表、放行业务消息
                     └─ 否 ─▶ 立即 close,不进连接表
{% endhighlight %}

鉴权要点:

- **鉴权未过 ≠ 半开连接**:验不过就立刻 `close`,**绝不**让它停在「连上了但没鉴权」的中间态占用资源。
- **加超时**:握手鉴权本身要有超时——连上却迟迟不完成鉴权的,按攻击/异常处理,踢掉。
- **防重放**:用一次性 nonce / 时间戳,避免攻击者录下一次合法握手后重放。
- **身份与连接表绑定**:鉴权拿到的 `deviceId` 才是连接表的键来源——**先鉴权、拿到可信身份,再登记**。

### 2. 消息加密:传输层 vs 应用层

加密有两个层次,解决的问题不同,可单用也可叠加:

- **传输层加密(TLS / `wss://`)**:在 socket 层把**整条连接**加密,自动覆盖所有消息,还顺带防篡改、防中间人。**首选**——只要能接受证书的成本。难点在局域网内的**证书**:公网 CA 签不了内网设备,得自建 CA / 自签名 + 分发信任,这套分发和轮换是主要复杂度。
- **应用层加密**:在业务消息这一层,用对称密钥(如 AES-GCM)加密 payload 再 `send`。它**不依赖证书**,实现门槛低;代价是要自己管密钥、自己保证完整性,且**只加密了 payload,连接元信息仍暴露**。

{% highlight javascript %}
// 应用层加密 payload(示意,选 AEAD 算法以同时保证机密性+完整性)
function sendSecure(conn, payload) {
  const iv = randomBytes(12);                       // 每条消息独立、不重用的 IV
  const { ciphertext, tag } = aesGcmEncrypt(sessionKey, iv, serialize(payload));
  conn.socket.send(serialize({ iv, tag, ciphertext }));
}
{% endhighlight %}

加密要点:

- **优先选 AEAD 算法**(如 AES-GCM):一步同时保证**机密性 + 完整性**,避免只加密不校验导致的篡改漏洞。
- **IV / nonce 绝不重用**:每条消息用独立随机 IV,重用会直接击穿 GCM 这类算法的安全性。
- **能用 TLS 就别自己造**:应用层手搓加密容易在 IV 管理、完整性校验、密钥协商上踩坑;TLS 是被反复审计过的成熟方案。

### 3. 密钥管理:整个安全的真正软肋

无论鉴权还是加密,**最终都落到「密钥/凭证怎么来、怎么换」**,这往往才是真正的薄弱点:

- **硬编码共享密钥**:实现最简单,但**不可轮换、一旦泄露即全网失守**,且反编译就能拿到。它只在「设备已物理可信的封闭网络」这个强假设下才勉强成立。
- **更稳的方向**:密钥**每设备一份**(而非全网一把)、由可信渠道(如配网流程 / 后台下发)分发、支持**轮换与吊销**;会话用**临时会话密钥**(握手时协商,如 ECDH),即便泄露也只影响一次会话。

> 诚实地说,很多局域网方案出于成本,会停在「明文传输 + 应用层身份校验 + 静态共享密钥」这一档。**这不是值得吹嘘的安全设计,但在明确的信任假设下能工作。** 工程上更重要的是:**讲清楚你的信任边界和假设是什么**(谁能进这个网?设备物理可信吗?),并据此选择鉴权与加密的强度——而不是假装做到了端到端加密。

## 八、小结：把通信层收敛成一台状态机

通信层真正的难点不在「连上」,而在「连上之后,在抖动的网络里长期可靠地工作」。把这一层的设计点归拢:

- **建连**:C/S 模型,服务端动态端口(bind 0)经 mDNS 广播,客户端据此拨入。
- **连接管理**:复用一条全双工长连接 + `messageId`/`responseId` 关联请求-响应(不是多路复用);连接表以 `deviceId` 为键;重连时旧连接要连同定时器、监听器**彻底销毁**以回收内存。
- **消息可靠性**:区分发送失败 / 响应超时 / 连接中断;每请求独立超时;**超时 ≠ 失败**,靠幂等 + 去重消化不确定性;关注顺序与背压。
- **活性检测**:多层兜底,以应用层心跳为主;重连用指数退避 + jitter 防风暴;区分抖动与真掉线。
- **生命周期**:按设备形态权衡前后台策略。
- **安全**:拆成鉴权 + 加密两个正交问题;握手先鉴权(质询-应答 + 防重放)再放行;传输优先 TLS;密钥每设备一份、可轮换。

一以贯之的思路:**把整条连接收敛成一个显式状态机**(连接中 / 已鉴权 / 心跳中 / 重连中 / 已断开),让每一个状态迁移都有明确的入口、出口和清理动作——这是它能在真实网络里稳定服务的根本。

---

回到全链路总览:《[局域网设备发现与通信：mDNS + WebSocket 的工程化方案](/2026/06/18/react-native-mdns-websocket-device-discovery)》。
