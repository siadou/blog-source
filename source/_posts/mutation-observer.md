---
title: MutationObserver及其在富文本编辑器里的应用和Vue里的应用
date: 2020-10-10 17:55:57
tags: [前端技术, 浏览器]
---


`MutationObserver`接口提供了监视对 DOM 树所做更改的能力。它被设计为旧的 Mutation Events 功能的替代品，该功能是 DOM3 Events 规范的一部分。
本文将介绍它的用法和在富文本编辑器draft-js里的应用和在Vue的nextTick里的应用。

# MutationObserver API


## 构造函数
`MutationObserver()`

创建并返回一个新的 MutationObserver 它会在指定的DOM发生变化时被调用。

## 方法
`disconnect()`

阻止 MutationObserver 实例继续接收的通知，直到再次调用其observe()方法，该观察者对象包含的回调函数都不会再被调用。

`observe(target[, options])`

配置MutationObserver在DOM更改匹配给定选项时，通过其回调函数开始接收通知。

`takeRecords()`

从MutationObserver的通知队列中删除所有待处理的通知，并将它们返回到MutationRecord对象的新Array中。


```
const observer = new MutationObserver(callback);

// 以上述配置开始观察目标节点
observer.observe(targetNode, config);

// 之后，可停止观察
observer.disconnect();

```

# 在富文本编辑器框 draft-js里的应用
draft-js是facebook开发第一款基于react的开源富文本编辑器框架。为了记录用户对内容的修改，draft-js使用 `MutationObserver`对编辑器节点进行观察，记录用户对内容的改变操作。当onCompositionStart的时候开始观察，当`onCompositionEnd`事件触发(防止事件未处理完，源码此处做了延时)或者键盘操作的时候结束观察。


```
const DOM_OBSERVER_OPTIONS = {
  subtree: true,
  characterData: true,
  childList: true,
  characterDataOldValue: false,
  attributes: false,
};

// IE11 has very broken mutation observers, so we also listen to DOMCharacterDataModified
// 由于IE11的MutationObserver存在严重的问题，采用DOMCharacterDataModified监控变化
const USE_CHAR_DATA = UserAgent.isBrowser('IE <= 11');



class DOMObserver {
  observer: ?MutationObserver;
  container: HTMLElement;
  mutations: Map<string, string>;
  onCharData: ?({
    target: EventTarget,
    type: string,
    ...
  }) => void;

  constructor(container: HTMLElement) {
    this.container = container;
    this.mutations = Map();
    const containerWindow = getWindowForNode(container);
    if (containerWindow.MutationObserver && !USE_CHAR_DATA) {
      this.observer = new containerWindow.MutationObserver(mutations =>
        this.registerMutations(mutations),
      );
    } else {
      //此处采用DOMCharacterDataModified监控变化省略
    }
  }

  start(): void {
    if (this.observer) {
      this.observer.observe(this.container, DOM_OBSERVER_OPTIONS);
    } else {
      //此处采用DOMCharacterDataModified监控变化省略
    }
  }

  stopAndFlushMutations(): Map<string, string> {
    const {observer} = this;
    if (observer) {
      this.registerMutations(observer.takeRecords());
      observer.disconnect();
    } else {
      //此处采用DOMCharacterDataModified监控变化省略
    }
    const mutations = this.mutations;
    this.mutations = Map();
    return mutations;
  }

  registerMutations(mutations: Array<MutationRecord>): void {
    for (let i = 0; i < mutations.length; i++) {
      this.registerMutation(mutations[i]);
    }
  }
  
  registerMutation(mutation: MutationRecordT): void {
    const textContent = this.getMutationTextContent(mutation);
    if (textContent != null) {
      const offsetKey = nullthrows(findAncestorOffsetKey(mutation.target));
      this.mutations = this.mutations.set(offsetKey, textContent);
    }
  }

  // 对提交内容的处理
  getMutationTextContent(mutation: MutationRecordT): ?string {
    const {type, target, removedNodes} = mutation;
    if (type === 'characterData') {
      // When `textContent` is '', there is a race condition that makes
      // getting the offsetKey from the target not possible.
      // These events are also followed by a `childList`, which is the one
      // we are able to retrieve the offsetKey and apply the '' text.
      if (target.textContent !== '') {
        return target.textContent;
      }
    } else if (type === 'childList') {
      if (removedNodes && removedNodes.length) {
        // `characterData` events won't happen or are ignored when
        // removing the last character of a leaf node, what happens
        // instead is a `childList` event with a `removedNodes` array.
        // For this case the textContent should be '' and
        // `DraftModifier.replaceText` will make sure the content is
        // updated properly.
        return '';
      } else if (target.textContent !== '') {
        // Typing Chinese in an empty block in MS Edge results in a
        // `childList` event with non-empty textContent.
        // See https://github.com/facebook/draft-js/issues/2082
        return target.textContent;
      }
    }
    return null;
  }
}

```

# 在Vue的nextTick中的应用

`nextTick`函数在Vue框架中会在下次 DOM 更新循环结束之后执行延迟回调。在修改数据之后立即使用这个方法，可以获取到更新后的 DOM。

在Vue中的nextTick函数中，MutationObserver用法和普通的用法有着很大的差别的自身功能被淡化仅作为一个异步的API被调用，。MutationObserver的作用是作为异步函数将本周期应该执行的事件推迟到下个周期里。
代码详见[这里](https://github.com/vuejs/vue/blob/d7d8ff06b70cf1a2345e3839c503fdb08d75ba49/src/core/util/next-tick.js#L1)

核心代码如下：

```
let timerFunc
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Technically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```


由以上代码可知，在nextTick里面一共完成了以下任务：

1. 将目标执行函数推入异步队列中
2. 执行timerFunc（使异步队列里面的所有内容在下一个周期里执行）
3. 返回一个Promise对象。当最后一个没有callback函数执行的空的nextTick执行的时候，Promise会被resolve。
4. 如果同一个同步周期里有多次NextTick的调用，第二次调用只会将目标函数推到队列中。（防止队列里的内容多次执行）

timerFunc是将待执行事件推迟到下一个事件循环里面执行的函数。
源代码里面按照不同优先级一共用了4种方法：

实际上MutationObserver 有更高的支持度。然而在iOS> = 9.3.3中的UIWebView中存在严重的bug。在触发几次之后会失效。所以还是优先采用Promise的方法

由于浏览器对macro task和micro task的处理的方式有着微妙的差别，micro task触发新的micro task后会直接加在原有的队尾执行再进行浏览器的UI线程渲染，而macro task则会先触发渲染再执行新的micro task，如果此时有数据修改又会触发重复渲染，导致采用macro task会存在一些抖动问题，性能相比也差一些。（详见参考文献6）。timerFunc优先采用micro task(微任务)，fallback采用macro task (宏任务)，最终以以下优先级实现：

1. 优先采用Promise.resolve
创建一个Promise对象，在timerFunc里通过resolve来在下一个eventloop里处理队列里面的任务。
2. MutationObserver
创建一个文本节点，并创建一个MutationObserver监听文本节点里面的内容改变。在timerFunc里面改变文本节点的内容。由于在IE和iOS> = 9.3.3中的UIWebView中存在bug，作为Promise的备选项。
3. setImmediate
相比于setTimeout可以直接运行，而setTimeout需要js引擎和系统的时钟同步，同步的频率在4ms，所以优先选择前者。
4. setTimeout



# 参考文献：

1. [MutationObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)
2. [Mutation Observer API](https://javascript.ruanyifeng.com/dom/mutationobserver.html)
3. [Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
4. [Tasks, microtasks, queues and schedules（译）](https://segmentfault.com/a/1190000014812771?utm_source=channel-hottest)
5. [vue.js 中nextTick 和setTimeout有什么区别](https://github.com/sevenCon/blog-github/issues/14)
6. [draft-js DOMObserver](https://github.com/facebook/draft-js/blob/master/src/component/handlers/composition/DOMObserver.js#L120)

