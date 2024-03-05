---
title: mvvm-resource-read
date: 2024-02-21 18:06:08
tags: mvvm
---
### 读react原理记录
为了深入复习一下react的原理，我特重新反向梳理react的源码,目的是为了剥丝抽茧，一点点弄清楚每一个方法到底是怎么执行的。


#### 先看看我们是怎么使用react的

```html
...... 省略其他，这里主要是建一个app盒子，用于后面js生成的dom插入进来。展示成完成的页面
<div id="app"></div>

```

```main.js
import React from 'mini-react'
....
React.render(<App />, document.getElementById('root'))

就从这一句开始分析，我们看是React的render方法接收了两个参数，一个 <App />， 一个是dom


1. 根据这个我们可以写出render函数类似下面

const render =  render(element, container) => {
  ...
}

2. 我们需要去了解 <App /> 是什么

那么跟着看我们找到了 App的相关代码类似如下

class App extends React.Component {
  constructor() {

  }
  ...

  render() {
    return (
      <div className="app-container">
        ......
      </div>
    )
  }
}


可以看到App实际返回的是 div 这里面是使用了jsx (语法糖，因为写的时候便捷，不用写为对象的形式， jsx会自动执行类里面的render函数) 的语法，便捷的讲 App 实例化为  <App /> 
jsx在实际运行的时候会转化为React.createElement语法也就是上面的实际执行的执行是的以下代码

<App/> = React.createElement('div', {className: 'app-container'}, ....)


简单点来说，<App /> 是一个虚拟dom的对象，而document.getElementById('root') 是获取的html的dom元素，虽然也是对象，但是是真实的。

3. 分析结构
那我们不妨大胆预估，其实render做的就是把<App /> 这个虚拟dom（本质上就是javascript的一个object，不过结构参考了DOM对象的形式，比如会按照类似下面的样子）挂载到了真实DOM(document.getElementById('root')) 里面。

虚拟dom结构猜测：
{
  type: 'div',
  className: 'app-container',
  ....
  child: [{
    type: 'div',
    ....
  }]
}

类似这样的一个对象结构，里面可以无限嵌套子节点。

4. React.render(<App />, document.getElementById('root')) 既然是文件的入口，且刚上面分析是将这个虚拟dom挂在到真实dom下，那如果虚拟dom内容改变了呢？

虚拟更换了，肯定是需要页面重新渲染的。也就是说react里面有一套机制会去刷新，完整版的react里面肯定是使用了算法，在应用的状态变化的情况下才去更新dom，而且更新dom也采取了合并多个相邻更新为一次和最小化dom更新（不会从根节点全部重新渲染）
那么我们的mini-react在这方面就写了个弱化版本的dom刷新机制，同时在dom节点循环阶段绑定了setState函数，为useState作为hooks可以导出页面刷新节点使用。


5. 那么当页面调用setState的时候本质上也就是对比dom节点变化，刷新页面
```

#### 总结，整个mini-react的内容本质上就是将页面内容转为虚拟dom，在虚拟dom中间对比变更节点，然后更新到页面上展示。当然完整版的react有更多更优化的方法和边界处理。


#### 附上源码（Zack大佬的）
```
let wipRoot = null;
let nextUnitOfWork = null;
let currentRoot = null;
let deletions = [];
let wipFiber;
let hookIndex = 0;
// Support React.Fragment syntax.
const Fragment = Symbol.for("react.fragment");

// Enhanced requestIdleCallback.
((global) => {
  const id = 1;
  const fps = 1e3 / 60;
  let frameDeadline;
  let pendingCallback;
  const channel = new MessageChannel();
  const timeRemaining = () => frameDeadline - window.performance.now();

  const deadline = {
    didTimeout: false,
    timeRemaining
  };

  channel.port2.onmessage = () => {
    if (typeof pendingCallback === "function") {
      pendingCallback(deadline);
    }
  };

  global.requestIdleCallback = (callback) => {
    global.requestAnimationFrame((frameTime) => {
      frameDeadline = frameTime + fps;
      pendingCallback = callback;
      channel.port1.postMessage(null);
    });
    return id;
  };
})(window);

const isDef = (param) => param !== void 0 && param !== null;

const isPlainObject = (val) =>
  Object.prototype.toString.call(val) === "[object Object]" &&
  [Object.prototype, null].includes(Object.getPrototypeOf(val));

// Simple judgment of virtual elements.
const isVirtualElement = (e) => typeof e === "object";

// Text elements require special handling.
const createTextElement = (text) => ({
  type: "TEXT",
  props: {
    nodeValue: text
  }
});

// Create custom JavaScript data structures.
const createElement = (type, props = {}, ...child) => {
  const children = child.map((c) =>
    isVirtualElement(c) ? c : createTextElement(String(c))
  );

  return {
    type,
    props: {
      ...props,
      children
    }
  };
};

// Update DOM properties.
// For simplicity, we remove all the previous properties and add next properties.
const updateDOM = (DOM, prevProps, nextProps) => {
  const defaultPropKeys = "children";

  for (const [removePropKey, removePropValue] of Object.entries(prevProps)) {
    if (removePropKey.startsWith("on")) {
      DOM.removeEventListener(
        removePropKey.slice(2).toLowerCase(),
        removePropValue
      );
    } else if (removePropKey !== defaultPropKeys) {
      DOM[removePropKey] = "";
    }
  }

  for (const [addPropKey, addPropValue] of Object.entries(nextProps)) {
    if (addPropKey.startsWith("on")) {
      DOM.addEventListener(addPropKey.slice(2).toLowerCase(), addPropValue);
    } else if (addPropKey !== defaultPropKeys) {
      DOM[addPropKey] = addPropValue;
    }
  }
};

// Create DOM based on node type.
const createDOM = (fiberNode) => {
  const { type, props } = fiberNode;
  let DOM = null;

  if (type === "TEXT") {
    DOM = document.createTextNode("");
  } else if (typeof type === "string") {
    DOM = document.createElement(type);
  }

  // Update properties based on props after creation.
  if (DOM !== null) {
    updateDOM(DOM, {}, props);
  }

  return DOM;
};

// Change the DOM based on fiber node changes.
// Note that we must complete the comparison of all fiber nodes before commitRoot.
// The comparison of fiber nodes can be interrupted, but the commitRoot cannot be interrupted.
const commitRoot = () => {
  const findParentFiber = (fiberNode) => {
    if (fiberNode) {
      let parentFiber = fiberNode.return;
      while (parentFiber && !parentFiber.dom) {
        parentFiber = parentFiber.return;
      }
      return parentFiber;
    }

    return null;
  };

  const commitDeletion = (parentDOM, DOM) => {
    if (isDef(parentDOM)) {
      parentDOM.removeChild(DOM);
    }
  };

  const commitReplacement = (parentDOM, DOM) => {
    if (isDef(parentDOM)) {
      parentDOM.appendChild(DOM);
    }
  };

  const commitWork = (fiberNode) => {
    if (fiberNode) {
      if (fiberNode.dom) {
        const parentFiber = findParentFiber(fiberNode);
        const parentDOM = parentFiber?.dom;
        switch (fiberNode.effectTag) {
          case "REPLACEMENT":
            commitReplacement(parentDOM, fiberNode.dom);
            break;
          case "UPDATE":
            updateDOM(
              fiberNode.dom,
              fiberNode.alternate ? fiberNode.alternate.props : {},
              fiberNode.props
            );
            break;
          default:
            break;
        }
      }

      commitWork(fiberNode.child);
      commitWork(fiberNode.sibling);
    }
  };

  for (const deletion of deletions) {
    if (deletion.dom) {
      const parentFiber = findParentFiber(deletion);
      commitDeletion(parentFiber?.dom, deletion.dom);
    }
  }

  if (wipRoot !== null) {
    commitWork(wipRoot.child);
    currentRoot = wipRoot;
  }

  wipRoot = null;
};

// Reconcile the fiber nodes before and after, compare and record the differences.
const reconcileChildren = (fiberNode, elements = []) => {
  let index = 0;
  let oldFiberNode = void 0;
  let prevSibling = void 0;
  const virtualElements = elements.flat(Infinity);

  if (fiberNode.alternate && fiberNode.alternate.child) {
    oldFiberNode = fiberNode.alternate.child;
  }

  while (
    index < virtualElements.length ||
    typeof oldFiberNode !== "undefined"
  ) {
    const virtualElement = virtualElements[index];
    let newFiber = void 0;
    const isSameType = Boolean(
      oldFiberNode &&
        virtualElement &&
        oldFiberNode.type === virtualElement.type
    );

    if (isSameType && oldFiberNode) {
      newFiber = {
        type: oldFiberNode.type,
        dom: oldFiberNode.dom,
        alternate: oldFiberNode,
        props: virtualElement.props,
        return: fiberNode,
        effectTag: "UPDATE"
      };
    }

    if (!isSameType && Boolean(virtualElement)) {
      newFiber = {
        type: virtualElement.type,
        dom: null,
        alternate: null,
        props: virtualElement.props,
        return: fiberNode,
        effectTag: "REPLACEMENT"
      };
    }

    if (!isSameType && oldFiberNode) {
      deletions.push(oldFiberNode);
    }

    if (oldFiberNode) {
      oldFiberNode = oldFiberNode.sibling;
    }

    if (index === 0) {
      fiberNode.child = newFiber;
    } else if (typeof prevSibling !== "undefined") {
      prevSibling.sibling = newFiber;
    }

    prevSibling = newFiber;
    index += 1;
  }
};

// Execute each unit task and return to the next unit task.
// Different processing according to the type of fiber node.
const performUnitOfWork = (fiberNode) => {
  const { type } = fiberNode;

  switch (typeof type) {
    case "function": {
      wipFiber = fiberNode;
      wipFiber.hooks = [];
      hookIndex = 0;
      let children;

      if (typeof Object.getPrototypeOf(type).REACT_COMPONENT !== "undefined") {
        const C = type;
        const component = new C(fiberNode.props);
        // eslint-disable-next-line react-hooks/rules-of-hooks
        const [state, setState] = useState(component.state);
        component.props = fiberNode.props;
        component.state = state;
        component.setState = setState;
        children = component.render?.bind(component)();
      } else {
        children = type(fiberNode.props);
      }
      reconcileChildren(fiberNode, [
        isVirtualElement(children)
          ? children
          : createTextElement(String(children))
      ]);
      break;
    }

    case "number":
    case "string":
      if (!fiberNode.dom) {
        fiberNode.dom = createDOM(fiberNode);
      }
      reconcileChildren(fiberNode, fiberNode.props.children);
      break;

    case "symbol":
      if (type === Fragment) {
        reconcileChildren(fiberNode, fiberNode.props.children);
      }
      break;

    default:
      if (typeof fiberNode.props !== "undefined") {
        reconcileChildren(fiberNode, fiberNode.props.children);
      }
      break;
  }

  if (fiberNode.child) {
    return fiberNode.child;
  }

  let nextFiberNode = fiberNode;

  while (typeof nextFiberNode !== "undefined") {
    if (nextFiberNode.sibling) {
      return nextFiberNode.sibling;
    }

    nextFiberNode = nextFiberNode.return;
  }

  return null;
};

// Use requestIdleCallback to query whether there is currently a unit task
// and determine whether the DOM needs to be updated.
const workLoop = (deadline) => {
  while (nextUnitOfWork && deadline.timeRemaining() > 1) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }

  if (!nextUnitOfWork && wipRoot) {
    commitRoot();
  }

  window.requestIdleCallback(workLoop);
};

// Initial or reset.
const render = (element, container) => {
  currentRoot = null;
  wipRoot = {
    type: "div",
    dom: container,
    props: {
      children: [
        {
          ...element
        }
      ]
    },
    alternate: currentRoot
  };
  nextUnitOfWork = wipRoot;
  deletions = [];
};

// Associate the hook with the fiber node.
function useState(initState) {
  const hook = wipFiber?.alternate?.hooks
    ? wipFiber.alternate.hooks[hookIndex]
    : {
        state: initState,
        queue: []
      };

  while (hook.queue.length) {
    let newState = hook.queue.shift();
    if (isPlainObject(hook.state) && isPlainObject(newState)) {
      newState = { ...hook.state, ...newState };
    }
    hook.state = newState;
  }

  if (typeof wipFiber.hooks === "undefined") {
    wipFiber.hooks = [];
  }

  wipFiber.hooks.push(hook);
  hookIndex += 1;

  const setState = (value) => {
    hook.queue.push(value);

    if (currentRoot) {
      wipRoot = {
        type: currentRoot.type,
        dom: currentRoot.dom,
        props: currentRoot.props,
        alternate: currentRoot
      };
      nextUnitOfWork = wipRoot;
      deletions = [];
      currentRoot = null;
    }
  };

  return [hook.state, setState];
}

class Component {
  props;

  constructor(props) {
    this.props = props;
  }

  // Identify Component.
  static REACT_COMPONENT = true;
}

// Start the engine!
void (function main() {
  window.requestIdleCallback(workLoop);
})();

export default {
  createElement,
  render,
  useState,
  Component,
  Fragment
};

```


### 读vue原理记录
...wait...



