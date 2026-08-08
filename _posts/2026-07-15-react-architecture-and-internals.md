---
layout: post
title: "React 架构与底层原理：状态归属、组件组合与 Fiber / Hooks"
date: 2026-07-15 15:00:00 +0800
categories: 所学
tags: React 架构 状态管理 Fiber Hooks
---

上一篇 [React 运行机制与性能优化](https://zzfd.github.io/2026/07/15/react-rendering-mechanism-and-performance) 讲清了运行时：组件为什么会重渲、怎么控制和优化它。这一篇往两头走——

- **往上**是**架构决策**：状态放在哪、组件怎么拼、逻辑怎么复用；
- **往下**是支撑这一切的**引擎**：Fiber、Hooks 链表、Reconciler。

这两层看似无关，其实是一条线：上层每一个「最佳实践」，几乎都能在下层找到物理原因。把它们缝起来看，React 才从「会用」变成「懂」。

## 一、状态归属：state 到底放在哪

这是架构里最核心、也最考验判断力的决策。有一条清晰的**优先级阶梯**，从最优先到最后手段：

1. **Colocation（就近原则）**：状态默认放在**用到它的最小范围**里。只有一个组件用 → 放它自己的 `useState`，别往上提；
2. **Lifting up（状态提升）**：多个兄弟组件要共享 → 提到**最近的公共父级**；
3. **全局 store / Context**：跨越很多层、很多模块都要用 → 才上全局。

最常见的 anti-pattern 是**一上来就把所有状态塞进全局 store**。后果是：任何一个字段变化，牵连范围都过大，耦合高、难维护。好的架构是**「能就近就不提升，能提升就不全局」**。

### server state vs client state：最重要的区分

比「放哪一层」更关键的，是先分清你手上的是哪**一类**状态：

| | Server State | Client State |
|---|---|---|
| **是什么** | 来自后端、你并不拥有的数据（列表、报表、行情） | 纯 UI 本地状态（弹窗开关、表单输入、选中项） |
| **特点** | 异步、会过期，要缓存 / 重取 / 失效 | 同步、你独占、随时可丢 |
| **该用什么** | 数据请求层（React Query / SWR / RTK Query 等） | `useState`，或真正的全局 UI store |

最关键的认知是：

> **大多数人以为的「全局状态」，其实是 server state。** 它不该塞进全局 store 手动维护，而该交给带缓存 / 失效 / 重取的**数据请求层**。真正需要放进全局 client store 的东西其实很少——登录态、主题、跨页共享的 UI 态。

把 server data 手动搬进全局 store，等于要自己重新实现一遍缓存、失效、重取、并发去重——这些请求层已经做好了。分清这两类，能消掉很大一部分「状态管理很复杂」的错觉。

### 什么时候不该用 useState

反直觉的一点：**不是所有「会变的值」都该进 `useState`。** 判断标准只有一条——**这个变化，需不需要驱动一次 re-render？** 不需要，就别放 state。三种典型：

1. **不驱动 UI 的可变值 → 用 `ref`**：定时器 id、上一次的值、DOM 引用、长连接实例。放进 state 只会触发无谓的 re-render；
2. **需要订阅外部数据源 → 用 `useSyncExternalStore`**：这是 React 18 给「外部 store（Redux、浏览器 API、自建 pub/sub）」接入 React 的**官方入口**，能正确处理并发下的 **tearing**（同一次渲染里读到不一致的快照）。Redux / Zustand 底层就是用它；
3. **极端高频、纯展示 → 直接绕过 React**（`ref` + 命令式改 DOM）：数据 → ref → `el.textContent = ...`，完全不走 render。性能上限最高，但可维护性最差，**只在局部热点用**。

## 二、组件设计：组合优于配置

一个组件为了适配各种场景，props 越加越多——`showHeader`、`headerType`、`hideFooter`、`variant`……这叫 **props 爆炸 / 配置地狱**。

解法是用**组合（composition）**把「控制权反转（IoC）」给调用方：

```tsx
// ❌ 配置地狱
<Card title="x" showAvatar avatarUrl="..." footerButtons={[...]} variant="compact" />

// ✅ 组合：结构交给调用方，Card 只管布局壳
<Card>
  <Card.Header><Avatar src="..." /></Card.Header>
  <Card.Body>...</Card.Body>
  <Card.Footer><Button /></Card.Footer>
</Card>
```

一句话判断：**当一个组件的 props 开始出现大量 `showXxx` / `variant` 这类布尔和枚举时，就该考虑改成组合**，把结构决定权交还给调用方。原则是——**配置适合封闭、有限的变化；组合适合开放、不可预知的变化。**

下面是两种最常用的组合模式。

### 模式 A：children 当 slot

最基础的组合。组件只负责「壳」（布局、样式、行为），**不写死里面装什么**，内容由调用方通过 `children` 塞进来：

```tsx
// Modal 只管遮罩、居中、关闭逻辑、ESC —— 它不关心里面是什么
function Modal({ open, onClose, children }: {
  open: boolean; onClose: () => void; children: React.ReactNode;
}) {
  if (!open) return null;
  return (
    <div className="overlay" onClick={onClose}>
      <div className="panel" onClick={(e) => e.stopPropagation()}>
        <button className="close" onClick={onClose}>×</button>
        {children}  {/* ← slot：装什么由调用方决定 */}
      </div>
    </div>
  );
}

<Modal open={x} onClose={close}>
  <h2>确认删除？</h2>
  <Button onClick={handleDelete}>删除</Button>
</Modal>
```

它替代的烂做法是 `<Modal title="..." content="..." buttons={[...]} />`——一旦有人要在 modal 里放个表单、放个图表，props 就不够用了。**slot 把「内容」这个无限变化的维度，交还给了调用方。** 需要多个插槽时，用**具名 props 传 `ReactNode`**：

```tsx
function PageLayout({ header, sidebar, children }: {
  header: React.ReactNode; sidebar: React.ReactNode; children: React.ReactNode;
}) {
  return (
    <div className="layout">
      <header>{header}</header>
      <aside>{sidebar}</aside>
      <main>{children}</main>
    </div>
  );
}
<PageLayout header={<TopBar />} sidebar={<Nav />}><Content /></PageLayout>
```

### 模式 B：Compound Components（复合组件）

当一组组件需要**联动**、又想让调用方**自由拼结构**时用它。特征是：一组组件对外像一个整体，**父组件统一管状态**，**子组件通过 context 读状态**，但结构和顺序由调用方决定。以 Tabs 为例：

{% raw %}
```tsx
// 1. 父组件持有状态，用 context 下发
const TabsContext = createContext<{
  active: string; setActive: (id: string) => void;
} | null>(null);

function Tabs({ defaultTab, children }: {
  defaultTab: string; children: React.ReactNode;
}) {
  const [active, setActive] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ active, setActive }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

// 2. 子组件从 context 读 / 写状态
function Tab({ id, children }: { id: string; children: React.ReactNode }) {
  const ctx = useContext(TabsContext)!;
  return (
    <button
      className={ctx.active === id ? 'tab active' : 'tab'}
      onClick={() => ctx.setActive(id)}
    >{children}</button>
  );
}

function TabPanel({ id, children }: { id: string; children: React.ReactNode }) {
  const ctx = useContext(TabsContext)!;
  return ctx.active === id ? <div className="panel">{children}</div> : null;
}

// 3. 挂成命名空间，对外像一个整体
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// 调用方：结构自由拼，选中态自动联动，无需自己管 active
<Tabs defaultTab="overview">
  <Tabs.Tab id="overview">概览</Tabs.Tab>
  <Tabs.Tab id="detail">明细</Tabs.Tab>
  <Tabs.Panel id="overview"><OverviewView /></Tabs.Panel>
  <Tabs.Panel id="detail"><DetailView /></Tabs.Panel>
</Tabs>
```
{% endraw %}

它比配置式（`<Tabs items={[{id, label, content}, ...]} />`）强在哪？一旦你想在某个 tab 标签里塞个 badge、想让 panel 顺序和 tab 顺序不一样、想在中间插个分隔线，配置对象就撑不住了。**compound 把「怎么排」交给调用方，把「选中态」留给自己**——这就是控制反转。（子组件靠 context 拿共享状态，所以它们必须包在 `<Tabs>` 里才有意义；`Tabs.Tab = Tab` 只是挂命名空间，纯为 API 好看，不是必须。）

### 两种模式的关系

- **children slot**：把「内容」这个维度交出去，**无共享状态**；
- **compound components**：是「slot + 共享状态」，父管状态、context 下发。

核心是同一句话：**控制反转——组件别什么都自己扛，把该交出去的决定权交给调用方。**

## 三、逻辑复用：custom hook

组件复用的是 UI，而 **custom hook 复用的是「有状态的逻辑（stateful logic）」**——这是它和组件复用的本质区别，也是它比 HOC / render props 更干净的地方。

抽象边界怎么切，是判断力所在：

- 一个 hook 应该**围绕一个关注点**：`usePagination`、`useDebounce`、`useConnection`；
- **别造万能 hook**（一个 `useEverything` 塞十件事）——和「别造万能组件」同理；
- **抽象要等到有第二个真实用例再抽**。过早抽象（premature abstraction）比重复更糟，容易抽出一个谁都不好用的错误边界。

---

上面是「怎么组织状态与组件」。接下来往下钻一层，看支撑这些行为的引擎——它会反过来解释很多上层现象。

## 四、Fiber：render 为什么能被中断

**Fiber 是 React 内部为每个组件实例维护的一份「可暂停的工作记录」。**

第一个反直觉的点：**Fiber 不是「带方法的类」，而是一个普通对象。** React 源码里确实有个 `FiberNode` 构造函数，但它造出来的本质是一个**纯数据结构**——只有属性，几乎没有方法。逻辑不在 fiber 上，而在 reconciler 的外部函数里（`beginWork` / `completeWork` / `commitWork` 这些函数**接收 fiber 作为参数**去操作它）。这种「数据与行为分离」是刻意设计——只有这样，调度器才能统一地遍历、暂停、丢弃这些 fiber。

fiber 上存了什么？挑最值得记的：

- **`return` / `child` / `sibling` 三个指针**——把 fiber 树组织成一个**可迭代遍历的链式结构**。这正是「render 可中断」的物理前提：React **不用递归**（递归压栈没法中途停下），而是用这三个指针做**循环遍历**，做完一个 fiber 就能停下、记住位置、下次接着走。**「可暂停」就是靠指针 + 循环，换掉了递归。**
- **`memoizedState`**——函数组件里，它就是 **hooks 链表的头节点**（下一节展开）；
- **`flags`**（Placement / Update / Deletion）——该做什么副作用，commit 阶段照它执行；
- **`lanes`**——优先级，决定它能否被高优先级更新打断。

第二个关键机制是**双缓冲**：屏幕上正在用的是 `current` 树，React 在旁边构建一棵 `workInProgress` 树；**算好了才 commit、一次性切换**。如果没算完就被高优先级更新打断，**直接丢弃 `workInProgress`，`current` 纹丝不动**——所以你永远不会在屏幕上看到「渲染了一半」的界面。因此，一个组件实例在更新期间其实**同时对应 current / workInProgress 一对 fiber**，两者用 `alternate` 指针互指、复用以减少 GC。

它为什么叫 "Fiber"？因为 "fiber" 是计算机科学里的既有概念：一种比线程更轻量的执行单元，采用**协作式调度**——不会被系统强行抢占，而是自己主动「让出（yield）」控制权。React 借这个名字，正是因为**每个 fiber 就代表一个「可以主动暂停、让出、之后恢复」的最小工作单元**。这也是 React 16 那次架构重写（Stack Reconciler → Fiber Reconciler）的核心：**把「不可中断的递归」换成「可中断的循环遍历」。**

> 顺带厘清一个易混点：**element ≠ fiber**。element 是你 JSX 产出的轻量描述对象（不可变、用完即弃）；fiber 是 React 内部为它维护的、**跨渲染持久存在**的工作单元。

这也回收了上一篇那句「render 可中断，所以组件函数必须纯」——因为一次 render 的进度都存在可被丢弃的 `workInProgress` 上，**纯，才是可以安全丢弃重来的前提**。

## 五、Hooks 链表：为什么不能在条件里调用

这是「会用 hook」和「懂 hook 怎么实现」的分水岭，而答案极其干净。

**根因：hook 靠「调用顺序」匹配，不靠名字。** 每个 fiber 上挂着一条 **hook 链表**，按你调用 hook 的先后顺序依次排列：

```
第1次 useState  →  第2次 useState  →  第3次 useEffect  →  ...
   (node1)          (node2)            (node3)
```

React **不知道**每个 hook「叫什么」、对应哪个变量。它只知道：「这次 render 里第 1 个被调用的 hook，去链表第 1 个节点取状态；第 2 个调用的，取第 2 个节点……」**完全靠位置（index）对齐。**

所以每次 render，**hook 的调用顺序必须完全一致**，链表才对得上。一旦你写：

```tsx
if (someCondition) {
  const [a, setA] = useState(0);  // ❌ 条件为 false 时这个 hook 被跳过
}
const [b, setB] = useState('');    // 顺序错位：b 拿到了本该属于 a 的节点
```

条件为 false 时第一个 hook 没被调用 → 后面所有 hook 的 index **整体前移一位** → `b` 读到了 `a` 的状态槽位 → **state 全部串位、错乱**。这就是 **Rules of Hooks（只在顶层调用，不在条件 / 循环 / 嵌套函数里调用）的真正原因**——它不是「React 团队的规定」，而是**链表按序匹配的机制决定的**。

再看一个细节，**两层 `memoizedState`**：

```
fiber.memoizedState  ──►  hook1  ──►  hook2  ──►  hook3  ──► null
(指向第一个 hook)         │next      │next      │next
                          每个 hook 自己也有：
                            hook.memoizedState  ← 这个 hook 存的实际值（如某个 useState 的 state）
                            hook.queue          ← 这个 hook 的更新队列
                            hook.next           ← 指向下一个 hook
```

- **fiber 上的 `memoizedState`** = 指向第一个 hook 节点；
- **每个 hook 节点上的 `memoizedState`** = 这个 hook 自己存的值；
- hook 之间靠 `next` 指针串成**单向链表**，顺序 = 你调用 hook 的顺序。

配合上一节的双缓冲：current / workInProgress 是两棵 fiber，**每棵 fiber 各有自己一条 hook 链**。render 时 React 从 current fiber 的 hook 链读旧状态，同时在 workInProgress 上克隆 / 构建新的一条。

这顺带解释了一个基础却少人深究的问题——**hook 的状态凭什么能「跨 render 保留」？** 因为它一直挂在**持久存在的 fiber** 上，而不是随组件函数每次执行而消失。

## 六、Reconciler 与 Renderer：一套 React 跑多端

最后一块，解释一个很多人用了多年却没想过的事：为什么同一套 React，既能渲染网页，又能渲染原生 App？

先厘清译名：**reconcile = 使一致 / 协调**，**reconciler 通译「协调器」**。它协调的是**「你想要的 UI」**（render 返回的新 element 树）和**「当前实际的 UI」**（旧 fiber 树 / 真实界面）这两者，算出「要改哪里」让它们一致。

关键在于 **reconciler 和 renderer 是分开的**：

| | 职责 |
|---|---|
| **Reconciler（协调器）** | **平台无关**，只管「算出差异」这套逻辑：diff、fiber 遍历、调度 |
| **Renderer（渲染器）** | **平台相关**，负责把差异**落到具体平台**：浏览器 DOM、原生视图、3D 场景…… |

> **这就是「为什么同一套 React 能跑 Web 和 Native」的答案——共用一个 reconciler，换不同的 renderer。** `react-dom` 把差异落到浏览器 DOM，面向原生的 renderer 落到原生视图，还有 renderer 能落到 3D 场景。上层组件模型和调度逻辑完全复用，只有「最后怎么画」这一层是平台相关的。

---

把上下两层缝起来，才是这篇的意图：

- 「render 可中断，所以组件要纯」——因为 **Fiber 用指针链 + 双缓冲**，让一次渲染可以被安全丢弃重来；
- 「hook 不能条件调用」——因为 **hook 状态是 fiber 上一条按序匹配的链表**；
- 「hook 状态能跨 render 保留」——因为它挂在**持久存在的 fiber** 上；
- 「一套 React 跑多端」——因为 **reconciler 与 renderer 分离**。

上层的架构判断（状态放哪、组件怎么拼）让代码好维护，下层的引擎原理让这些判断有了物理解释。两者贯通，React 才算真正「懂」了。

（本文没有展开服务端渲染的那条线——RSC / App Router 的渲染模型、`'use client'` 边界、hydration 与 streaming SSR——它们自成体系，留待后续单独成篇。）

原文地址：<a href="https://zzfd.github.io/2026/07/15/react-architecture-and-internals">React 架构与底层原理：状态归属、组件组合与 Fiber / Hooks</a>
