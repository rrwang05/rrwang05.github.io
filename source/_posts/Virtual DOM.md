---
title: Virtual DOM
---

### 什么是虚拟DOM
虚拟DOM是一个变成概念。JS对象，通过ReactDOM等类库使之与真实的DOM同步，这个过程叫做协调（recilication）。
往下会说协调。

### 什么是 React Fiber
使Virrual DOM可以进行增量更新。
https://github.com/acdlite/react-fiber-architecture 以后看

### 如何创建渲染虚拟DOM
```
let demoNode = ({
    tagName: 'ul',
    props: {'class': 'list'},
    children: [
        ({tagName: 'li', children: ['douyin']}),
        ({tagName: 'li', children: ['toutiao']})
    ]
});
// 构建一个render函数，将demoNode对象渲染为以下dom
<ul class="list">
    <li>douyin</li>
    <li>toutiao</li>
</ul>
// 构建虚拟DOM
var elem = createElement(
    'ul',
    {'class': 'list'},
    [
        createElement('li', {}, ['douyin']),
        createElement('li', {}, ['toutiao'])
    ]
);
function Element(tagName, props, children) {
    this.tagName = tagName;
    this.props = props || {};
    this.children = children || [];
};
function createElement(tagName, props, children) {
    return new Element(tagName, props, children);
};
// 通过Element我们可以任意地构建虚拟DOM树了。如何将DOM树呈现到页面中去呢
// 通过遍历，逐个节点地创建真实DOM节点
1) createElement
2) createTextNode
// 如何遍历呢？因为这是一棵树，对于树形结构无外乎两种遍历
1) 深度优先遍历(DFS)
2) 广度优先遍历(BFS)
// 实际情况中我们需要采用DFS，因为我们需要将子节点append到父节点中去
采用DFS实现render函数
// 模拟渲染函数渲染虚拟DOM
Element.prototype.render = function() {
    var el = document.createElement(this.tagName),
        props = this.props,
        propName,
        propValue
    ;
    for (propName in props) {
        propValue = props[propName];
        el.setAttribute(propName, propValue);
    }
    this.children.forEach(function(child) {
        var childEl = null;
        if (child instanceof Element) {
            childEl = child.render();
        } else {
            childEl = document.createTextNode(child);
        }
        el.appendChild(childEl);
    });
    return el;
};
var root = elem.render();
document.querySelector('body').appendChild(elem.render());
```

### 协调
虚拟DOM和真实DOM之间的协调，关键是React的diff算法中，它的设计决策以保证组件满足更新具有可预测性，以及在繁杂业务下依然保持应用的高性能性。
#### 设计动力
某一个时间点，调用render方法，生成一颗由 React 元素组成的树。在下一次state 或 props 更新的时候，相同的render方法会返回一颗新的由 React 元素组成的树。React 需要基于这两棵树之间的差别来判断如何有效率的更新 UI 以保证当前 UI 与最新的树保持同步。
React在以下两个假设基础上提出了一套O(n)的启发算法：
1.  两个不同类型的元素会产生出不同的树
2.  开发者可以通过key props来暗示哪些子元素在不同的渲染下能保持稳定。
#### Diff 算法
对比
