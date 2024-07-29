首先 React 必须可少的就是 jsx 语法糖，jsx 会被 babel 或者 tsc 等编译器编译成 render function，也就是类似 React.createElement

所以在之前的 react 中每个 react 组件都需要在顶部写明`import * as React from 'react'` 就是因为编译的时候会用到 react 的 api

后来 babel、tsx 等编译工具会自动引入一个 react/jsc-runtime 的包，也不需要特意引入 react 了

jsx -->> render function -->> vdom

vdom（react Element） 是一个通过 children 串联起来的树

之后 react 会把 react element 树转换为 fiber 结构（一个链表）

React Element 只有 children 属性来链接父子节点，但是转为 fiber 之后就有了 child、sibling、return 属性来关联父子节点、兄弟节点。

## fiber 结构看起来也是一棵树，为什么叫做链表那？

因为按照 child、sibling、return、sibling、return 之类的遍历顺序，可以把整个 vdom 变成线性的链表结构，这样一次循环就可以处理完。。

具体遍历顺序：先访问当前节点，然后访问其子节点，子节点访问完再访问兄弟节点，兄弟节点访问完再返回父节点并访问下一个兄弟节点。

## 链表遍历的好处

链表式的遍历有几个显著的优势：

可中断和恢复：

React 的协调过程（即重新渲染）可以被中断，以便处理更高优先级的任务（例如用户输入）。中断后可以从上次中断的地方恢复，继续遍历和更新。
这在传统的树结构中难以实现，因为树的遍历通常是递归的，不容易中断和恢复。
增量渲染：

通过将更新任务分散到多个帧中，React 可以逐步进行渲染，而不是一次性完成所有更新。这提高了应用的响应速度，使得用户界面更加流畅。

## 并发

react 在处理 fiber 链表的时候通过一个叫 workInProgress 的指针指向当前节点，记录下当前位置

而 react 能够实现并发，就是基于 fiber 的聊表结构

因为之前的 React element 树里只有 children，没有 parent、sibling 信息，这样只能一次性处理完，不然中断了就找不到他的 parent 和 sibling 节点了。

但是 fiber 不停，他额外保存了 return、sibling 节点，这样就算打断了也可以找到下一个节点继续处理

所以现在完全可以先处理这个 fiber 树的某几个节点，然后展厅，处理其他的 fiber 树，之后在回来继续处理

这就是 react 所谓的并发。

## 时间分片

浏览器通过 Event loop 跑一个个 task
如果某一个任务过长的话，就会阻塞渲染导致丢帧，也就是页面卡顿。

之前 react element 递归渲染的时候容易计算量过多，导致页面卡段。

而改成 fiber 之后，可以在每次渲染 fiber 节点之前判断是否超过一定的时间间隔，是的话就放到下一个任务里跑，这样就不会阻塞渲染。

### 怎么实现的呢

很简单，fiber 链表是可以打断的，每次处理一个节点。

```js
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

然后再处理下一个节点之前判断下当前时间片还有没有空余时间，有的话继续 performUnitOfWork 处理下一个 fiber 节点

否则放到下一次任务里跑

这个时间切片的判断就是通过，当前时间和任务开始时间点的差值

```js
function shouldYieldToHost() {
  var timeElapsed = exports.unstable_now() - startTime;
  if (timeElapsed < frameInterval) {
    return false;
  }
  return true;
}
```

## fiber 架构的好处

**通过记录 parent、sibling 信息，让树变成链表，可以打断。每次处理一个 fiber 节点，处理每个 fiber 节点钱前断是否到了固定的时间间隔，也就是时间分片，通过时间分片把处理 fiber 的过程放到多个任务礼炮，这样页面内容多了也不会卡顿**

时间分片机制可以直接使用浏览器的 requestldleCallback 的 api 做

## react 的渲染流程

render function：`React.createElement`被称为"渲染函数”（render function）,因为它描述了渲染 UI 所需的元素数结构。这个函数不是一个真正意义上的函数调用，而是一种概念，表示生成 React 元素树的过程。

当 React 需要更新 UI 时，它会将 React Element 树转换成 Fiber 结构，这个过程称为协调（reconciliation）。Fiber 是一种比 React Element 更加细粒度的数据结构，它允许 React 进行更加细致的调度和中断，从而实现更高效的更新和渲染。

协调过程（Reconciliation）
协调过程主要包括以下几个步骤：

创建 Fiber 树：React 通过递归地遍历 React Element 树，创建相应的 Fiber 节点。每个 Fiber 节点对应一个 React Element 节点，但包含更多的元数据，以便进行细粒度的更新和调度。

比较新旧树：React 比较新旧 Fiber 树，确定需要更新的部分。这部分工作由协调器（reconciler）完成。协调器会标记哪些节点需要更新、添加或删除。

提交更新：标记完成后，React 将这些变化提交给渲染器（renderer），更新实际的 DOM 树。

## react 的渲染流程

JSX --->render function （react.createElement）--->Vdom--->fiber

JSX 通过 babel、tsc 等编译成 render function，执行后变成 React Element 的树。

然后 React ELement 转成 fiber 结构，这个过程叫 reconcile（协调）

之后会根据 fiber 的类型做不同的处理

比如：function 组件、provider、lazy 组件等类型的 fiber 节点，都会做相应的处理。

当然 reconcile 并不只是创建新的 fiber 节点，当更新的时候，还会和之前的 fiber 节点做 diff，判断是新增、修改、还是删除，然后打上对应的标记

reconcile 完之后，fiber 链表也就构建好了，并且在每个 fiber 节点上保存了当前的一些额外的信息

比如 function 组件要执行的 effect 函数。

之后会再次遍历构建好的这个 fiber 链表，处理其中的 effect，根据增删改的标记来更新 dom，这个阶段叫做 commit。

这样，React 的渲染流程就结束了。

## 整体阶段

render 阶段：React Element （VDOM）reconcile 为 fiber 树，由 Scheduler 负责调度，通过时间分片来吧计算分到多个任务离去

commit 阶段：reconcile 结束之后就有了完整的 fiber 链表，再次遍历 fiber 链表，执行其中的 effect、增删改 dom 等

其实 commit 阶段也分成了三个小阶段：

before mutation：操作 DOM 之前
mutation：操作 DOM
layout：操作 DOM 之后

比如 useEffect 的 effect 函数会在 before mutation 前异步调度执行，而 useLayoutEffect 的 effect 函数是在 layout 阶段同步执行。

React 团队按照操作 dom 前后来分了三个小阶段，更清晰了一点。

再就是 ref，在 mutation 阶段更新了 dom，所以在 layout 阶段就可以拿到 ref 了。

## 总结

这节我们简单分析了下 React 的渲染流程。

JSX 通过 babel、tsc 等编译器编译成 render function，然后执行后产生 React Element 的树。

React Element 的树会转成 fiber 链表，这个过程叫做 reconcile，由 React 的 Scheduler 调度。

reconcile 每次只处理一个 fiber 节点，通过时间分片把 reconcile 的过程分到多个任务跑，这样 fiber 树再大也不会阻塞渲染。

reconcile + schedule 这个过程叫做 render 阶段，而之后会进入 commit 阶段。

commit 阶段会遍历构建好的 fiber 链表，执行 dom 操作，还有函数组件的 effect 等。

它按照更新 dom 前后，分了 before mutation、mutation、layout 三个小阶段。

这就是 React 的 fiber 架构的好处和渲染流程，下节我们按照这个流程来写下 Mini React。
