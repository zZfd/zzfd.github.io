---
layout: post
title: "React 运行机制与性能优化：先搞懂 re-render，再谈优化"
date: 2026-07-15 14:00:00 +0800
categories: 所学
tags: React 渲染机制 性能优化 hooks re-render
---

深入 React，靠的不是记住更多 hook、认识更多 API。真正的分水岭是：对每一个机制，你能不能回答三件事——

1. 它背后到底发生了什么（why）；
2. 什么时候该用、什么时候不该用（trade-off）；
3. 它在什么情况下会失效（边界）。

停在「会调用」这一层，很多行为就只能靠背和试。往下沉一层，你会发现 React 的运行时其实只有很少几条主线，绝大多数「玄学」都能从它们推出来。这篇聚焦运行时的核心问题：**一个组件为什么会重新渲染，以及如何精准地控制和优化它**。更上层的架构决策和更底层的引擎（Fiber、Hooks 链表），放在下一篇 [React 架构与底层原理](https://zzfd.github.io/2026/07/15/react-architecture-and-internals)。

## 渲染的两个阶段：render 与 commit

React 的一次更新，分成两个**性质完全不同**的阶段。这个划分是后面一切的地基。

**Render 阶段——纯计算，可中断，无副作用。** React 调用你的组件函数，得到新的 element 树，和旧的 fiber 树做 diff，算出「哪些要变」。关键在于：**这个阶段只是算，不碰真实 DOM**；而且在并发模式下它**可以被打断、丢弃、重来**。正因如此，你的组件函数**必须是纯的**——同样的 props / state 给出同样的输出，不能在渲染过程中直接改外部变量、发请求。

**Commit 阶段——写 DOM，同步，不可中断。** React 把 diff 的结果一次性写进真实 DOM，然后按固定顺序执行：`useLayoutEffect` 的 setup（同步）→ 浏览器 paint → `useEffect` 的 setup（异步）。

把这两阶段记牢，一个高频困惑就有了答案——**「为什么组件函数一定要保持纯？」** 因为 render 阶段可能被并发调度**中断并重跑**；一旦函数里有副作用，就会执行多次、产生脏数据。纯，是「可以安全地丢弃重来」的前提。

## 一个组件为什么会 re-render：三个来源

组件重新执行（re-render），触发源**只有三个**：

1. **自身 state 变化**——`setState`，且新旧值经 `Object.is` 判定为不同；
2. **父组件 re-render**——父组件重渲，**默认所有子组件都跟着重渲**，哪怕它的 props 一个都没变；
3. **消费的 context value 变了**——`useContext` 订阅的那个 value 引用发生变化。

第 2 条最反直觉，也最多人不知道。很多人以为「props 变了才会重渲」——**错**。默认情况下是**「父渲我就渲」**，跟 props 变没变毫无关系。想切断这种向下传染，才需要 `React.memo`（浅比较 props，没变就跳过）。这条稍后在优化部分展开。

这里顺带澄清两个和「什么算变化」相关的点：

**为什么 state 要 immutable 更新。** React 靠 `Object.is` 比较新旧引用来决定要不要重渲。如果你原地 `list.push(x)`，引用没变，React 认为「没变」，不会重渲。所以必须 `setList([...list, x])` 造一个新引用。

**为什么连续三次 `setCount(count + 1)` 结果是 +1 而不是 +3。** 因为同一批更新里，`count` 是**同一次 render 的快照**（一个固定的旧值），三次算的都是「旧值 + 1」。要真正累加，得用**函数式更新** `setCount(c => c + 1)`——它拿到的是上一次的最新结果。这背后是「每次 render 是一次快照」的心智模型，下面还会专门讲。

## reconciliation：复用还是重建

拿到新的 element 树后，React 通过 diff（reconciliation）决定：一个节点是**复用**还是**重建**。

| 情况 | 结果 |
|---|---|
| 同一位置、type 相同（都是 `<div>`，或都是同一个组件） | **复用**该 fiber 实例，只更新变化的 props（re-render，不 remount） |
| type 变了（`<div>`→`<span>`，或 `<Foo>`→`<Bar>`） | **旧的整棵卸载、新的整棵挂载**，state 全丢 |
| 列表里 | 靠 **`key`** 判断「跨渲染的两个节点是不是同一个」 |

由此能推出两个经典结论：

- **为什么不能用数组 index 当 key。** 列表**增删 / 排序**时，index 会整体错位，导致 React 把「第 2 项的 fiber」复用给了「现在的第 2 项」——但它们其实是不同的数据。后果是受控输入框的值串位、组件 state 挂到了错误的项上。答得出「具体会出什么 bug」，才算真的理解 key。
- **`re-render ≠ remount`。** 同 type 复用时，React 只是重新执行函数、更新内容，**实例和 state 都还在**。这条贯穿全文。

## mount / unmount 与 re-render 的区别

这是最容易含糊的一组概念。

- **mount（挂载）**：组件实例**第一次被创建、并插入 DOM**，触发 effect 的 setup。一个实例一生只 mount 一次。
- **unmount（卸载）**：实例**从树里被移除、销毁**，React 清掉它的 state 和 DOM，触发 cleanup。

**最重要的分水岭：`re-render ≠ remount`。** state 变了、父组件重渲，导致组件**重新执行函数**——这叫 re-render，实例一直是 mounted 的，state 保留、DOM 复用，**不会**触发 mount/unmount 的 effect。（一个好的虚拟列表，滚动时是**复用同一批实例、只更新内容**，而不是 unmount + remount——后者会丢 state、effect 反复重跑，性能差得多。）

什么才会真正触发 unmount？只有三种：**条件渲染移除**（`{show && <Panel/>}` 变 false）、**从列表里被删掉**、**`key` 变了**。

其中 key 变化尤其值得记：

```tsx
<Profile key={userId} />   // userId 一变，旧 Profile 整个卸载、新的重新挂载
```

`key` 一变 = **强制卸载旧的 + 挂载新的**，内部 state 全部重置、effect 全部重跑。这正是「用 key 重置组件状态」这个技巧的原理——它和「列表用 index 当 key 会串位」是同一个机制的两面。

几个容易踩的点：

- **cleanup 不是只在 unmount 跑。** 依赖变化导致 effect 重跑前，也会**先执行上一次的 cleanup**。
- **开发环境的「双重挂载」。** `StrictMode` 下，开发环境会故意 `mount → 立刻 unmount → 再 mount`（生产不会），目的是逼你检查 effect 是否幂等、cleanup 是否写对。「为什么我的 effect 在开发环境跑了两次」就是这个。
- **精确表述：mount/unmount 跨越 render + commit 两个阶段。** 副作用（DOM mutation、effect setup/cleanup）确实都在 commit 执行；但「是否挂载/卸载」是 **render 阶段 diff 决定并打 tag** 的。因为 render 可中断——如果某次 render 算出「要挂载 X」，却被一个高优先级更新**丢弃**了，X 的挂载根本不会走到 commit，effect 一次都不会跑。这又回到了那句「render 必须纯」。

## 每次 render 都是一次快照：闭包陷阱

这是运行时最容易翻车的点。先看现象：

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // 永远打印 0！
    }, 1000);
    return () => clearInterval(id);
  }, []); // 空依赖
}
```

**为什么？** 每次 render，组件函数重跑，`count` 是**那一次 render 的快照**（一个固定的常量）。effect 因为 `[]` 只在 mount 跑一次，它内部的闭包**永久捕获了首次 render 时 `count = 0` 的那个值**。后续 `setCount` 触发新的 render、产生新的 `count`，但老的 interval 回调闭包里引用的，还是那个旧的 0。

核心心智模型只有一句：

> **每次 render 都是一次独立的快照**——props、state、以及基于它们的函数，都是那一帧冻结的值。

理解这句，闭包陷阱、`setCount(c => c+1)` 为什么要用函数式、依赖数组为什么要如实填，全通了。

三种解法，按场景选：

| 解法 | 说明 |
|---|---|
| **如实填依赖** `[count]` | effect 随 count 重建，闭包每次拿到新值（但 interval 会反复重建） |
| **函数式更新** `setCount(c => c + 1)` | 不依赖外部 count，直接绕开闭包 |
| **用 ref 存最新值** `countRef.current` | ref 是跨 render 共享的同一个盒子，读它永远是最新的（适合「要读最新值、但不想重建 effect」） |

顺带纠正一个常见误解：**依赖数组首先是「正确性」，其次才是性能。** 它的第一职责是防 stale closure；填错依赖是 **bug**，不是「慢」。

## effect 的执行时机：useEffect vs useLayoutEffect

两者都在 commit 之后，区别在于**相对「浏览器绘制（paint）」的时机**。一次更新里的完整时间线：

```
1. Render：调用组件函数，算出新 fiber 树、diff
2. Commit：
   ├─ 把变更写入真实 DOM（mutation）
   └─ ▶ useLayoutEffect 在这里【同步】执行 —— DOM 已更新，但屏幕还没画
3. Paint：浏览器真正把像素画到屏幕
4. ▶ useEffect 在这里【异步】执行 —— 用户已经看到画面之后
```

一句话：**useLayoutEffect 卡在「DOM 改完、还没画」这个缝里同步执行，会阻塞 paint；useEffect 等画完了才异步跑。**（对应的 cleanup 时机也一样：layout 的 cleanup 同步、在 paint 前；effect 的 cleanup 异步、在 paint 后。）

**什么时候需要 useLayoutEffect？** 当你要「**读取 DOM 布局 → 据此再改一次，且不想让用户看到中间的闪烁**」时。经典场景是 tooltip / 弹层定位：

```tsx
useLayoutEffect(() => {
  const rect = ref.current.getBoundingClientRect(); // 读真实布局
  setPos(computePosition(rect));                    // 立刻重排
}, [deps]);
```

因为它在 paint **之前**同步跑完并触发重渲，用户直接看到最终位置，没有「先闪一下错位、再跳正」的抖动。换成 `useEffect`，浏览器会先把错位的那一帧画出来 → 再 setState → 又画一帧正确的 → 用户看到闪烁。

原则：**默认用 `useEffect`，只有涉及「读布局 + 同步重排防闪烁」才升级到 `useLayoutEffect`**。后者会阻塞绘制，里面别放重活或请求；而且它在服务端渲染时会报 warning（server 没有 DOM）。

## 可中断渲染与并发

默认情况下，所有更新都是**紧急（urgent）、同步优先级**的，不可被打断。但 render 阶段本身是**可中断**的——React 靠 Fiber 架构把渲染拆成一个个工作单元，配合时间切片和优先级调度实现（这个引擎细节放在下一篇的 Fiber 部分）。要让 React 有机会去中断，你得主动把某些更新**降级为 transition（过渡，非紧急）**。

两个 API：

```tsx
const [isPending, startTransition] = useTransition();

function onChange(e) {
  setInput(e.target.value);        // urgent：输入框立刻响应，不可打断
  startTransition(() => {
    setResults(filter(e.target.value)); // transition：低优先级、可被下次输入打断
  });
}
```

效果：用户狂敲键盘时，input 每次都即时更新；而沉重的结果列表渲染被标记为可中断——**新一次输入进来，上一次还没算完的列表 render 直接被丢弃重来**，主线程不卡。

另一个是 `useDeferredValue`，作用在**值**上（当你只有值、拿不到 setter 时用，比如值来自 props）：

```tsx
const deferredQuery = useDeferredValue(query); // query 的「滞后版本」
// 用 deferredQuery 去渲染重列表，这次重渲是低优先级、可被打断的
```

几个高频混淆要分清：

- **`AbortController` ≠ 中断 render**——它中止的是 fetch / 请求，和 render 调度是两码事；
- **`Suspense` ≠ 中断 render**——它管的是「暂停提交、等数据 / 代码就绪」，是「等」，不是「打断计算」；
- **`startTransition` 不会让代码更快**——它**不减少总工作量**，只是**重排优先级**。

回收一下开头那句：正因为一次 render 随时可能被**丢弃并重跑**，组件函数才**必须纯**。

---

前半篇讲清了「组件为什么会重渲、渲染怎么运作」。有了这套机制模型，性能优化就不再是「无脑加 memo」，而是有据可依的诊断。

## 性能优化第一步：先分类

React 的运行时性能问题只有两类，**先分清是哪一类，再谈方案**：

| 类型 | 症状 | 解法方向 |
|---|---|---|
| **1. render 太频繁** | 不必要的 re-render | `memo` 家族 / 细粒度订阅 —— 切断重渲 |
| **2. 单次 render 太慢** | 渲染的东西太多 / 算得太重 | 虚拟化 / `useMemo` 缓存计算 / Web Worker 挪走 |

**虚拟列表治的是第 2 类，memo 治的是第 1 类**——别拿虚拟列表当万能药。优化的第一动作永远是**诊断分类**，而不是上来堆 API。

## memo 家族与它们的失效边界

三个 API，缓存的东西不同：

| API | 缓存什么 | 目的 |
|---|---|---|
| `React.memo(Comp)` | 组件 | props 浅比较没变 → **跳过这个子组件的 re-render** |
| `useMemo(fn, deps)` | 一个**值** | deps 没变 → 复用上次算出的值（缓存重计算） |
| `useCallback(fn, deps)` | 一个**函数** | deps 没变 → 复用同一个函数引用（本质是 `useMemo(() => fn)`） |

**它们其实是一条链。** `React.memo` 靠浅比较 props 生效——但如果你给子组件传了**对象 / 数组 / 函数**，父组件每次 render 都会造一个**新引用**，浅比较判定「变了」，**memo 直接失效**。于是需要 `useMemo`（稳住对象/数组引用）和 `useCallback`（稳住函数引用）来喂给 memo：

{% raw %}
```tsx
const Child = React.memo(ChildImpl);

function Parent() {
  // ❌ 每次 render 都是新对象 / 新函数 → Child 的 memo 形同虚设
  return <Child config={{ a: 1 }} onClick={() => doSomething()} />;

  // ✅ 稳住引用，memo 才真正生效
  const config = useMemo(() => ({ a: 1 }), []);
  const onClick = useCallback(() => doSomething(), []);
  return <Child config={config} onClick={onClick} />;
}
```
{% endraw %}

由此引出那个最能区分深浅的问题——**`useCallback` 什么时候是白写的？**

> **当接收这个函数的子组件没有被 `React.memo` 包裹时。** 子组件反正每次都会重渲，你稳住函数引用毫无意义，反而多了一层缓存开销。`useCallback` 只有在和 `React.memo`（或作为其它 hook 的依赖）配合时才有价值。

更一般地说，判断标准只有一条：**有没有谁在乎这个函数的引用是否变？** 只有两种情况有人在乎：

1. **传给了被 `React.memo` 包裹的子组件**——引用变会击穿 memo；
2. **被用作 `useEffect` / `useMemo` / 另一个 `useCallback` 的依赖**——引用变会导致 effect 反复重跑。

这两种都不沾边，就**不需要** `useCallback`。特别提醒一个高频误区：**传给原生 DOM 元素的函数不算**——

```tsx
// 每次 render 都是新函数，但完全不用 useCallback
<button onClick={() => handleClick()}>提交</button>
```

`<button>` / `<div>` 这些原生元素不是被 memo 的组件，React 走事件委托，你每次给它新函数它毫不介意。「传给子组件才要 useCallback」特指传给**你自己写的、且被 `React.memo` 包住的**组件，不是传给 HTML 标签。

总原则：**别默认到处包 `useMemo` / `useCallback`。** 它们本身有成本（存缓存、比较依赖），滥用会让代码更慢更难读。**先测量、定位到真正的热点，再针对性加。**

## Context 的 re-render 陷阱

回收前面「re-render 三来源」之一：**消费的 context value 一变，所有 consumer 全部重渲**——哪怕它只用了 value 里的一个字段。

{% raw %}
```tsx
// ❌ 把高频变的和低频变的塞进同一个 context
<AppContext.Provider value={{ user, theme, cartCount }}>
// cartCount 一变，只用 theme 的组件也跟着重渲
```
{% endraw %}

两种解法：

1. **拆 context**：按变化频率分开（`UserContext` / `ThemeContext` / `CartContext`），consumer 只订阅自己要的；
2. **上带 selector 的外部 store**（如 Redux / Zustand）：`useSelector(s => s.theme)` 只有 theme 变才重渲。

这其实回答了「**为什么大型应用常用外部 store，而不是纯 Context 做全局状态**」：Context 没有细粒度订阅，外部 store 有 selector。

## React 编译器：自动 memo

React 19 引入的编译器（React Compiler）**会自动帮你做 memo**：编译期分析依赖、自动缓存组件和值——理想情况下，你不用再手写 `useMemo` / `useCallback` / `React.memo`。

那还要不要理解手动 memo？我的判断是：

> **方向上，它让手动 memo 变得基本不必要**——编译器覆盖了绝大多数场景。但它①需要代码遵守 Rules of Hooks、组件保持纯，才能安全优化；②是渐进采用，老代码里仍有手写 memo；③特殊热点仍可能需要人工干预。所以务实的态度是：**新代码可以依赖编译器，但理解 memo 的原理仍是前提**——因为你得知道它在背后做了什么、以及什么时候它没帮上忙。

## 两条正交的优化路径

最后收束成一张认知图，避免把手段混为一谈：

- **memo 家族 = 减少工作量**——跳过不必要的重渲 / 重算；
- **并发特性（`useTransition` / `useDeferredValue`）= 重排优先级**——不减少总工作，只保证紧急交互不被沉重渲染阻塞。

这是**两条正交的路**。诊断出「工作太多」就用前者，诊断出「工作阻塞了交互」就用后者，它们可以叠加，但解决的不是同一个问题。

---

把这篇串起来：React 的运行时只有几条主线——render 是可中断的纯计算、commit 才写 DOM；re-render 有 self-state / parent-render / context 三个来源，其中「父渲子必渲」最反直觉；reconciliation 靠 type 和 key 决定复用还是重建；每次 render 是独立快照，这解释了闭包陷阱和函数式更新。而性能优化，本质就是围绕这几条主线做「减少重渲」和「重排优先级」两件事，前提是先诊断问题属于哪一类。

再往下一层——render 凭什么能被中断、hook 状态凭什么能跨 render 保留、一套 React 凭什么能同时跑 Web 和 Native——这些引擎问题，连同状态归属与组件组合的架构决策，都在下一篇：[React 架构与底层原理：状态归属、组件组合与 Fiber / Hooks](https://zzfd.github.io/2026/07/15/react-architecture-and-internals)。

原文地址：<a href="https://zzfd.github.io/2026/07/15/react-rendering-mechanism-and-performance">React 运行机制与性能优化：先搞懂 re-render，再谈优化</a>
