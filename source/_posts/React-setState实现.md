---
title: setState
---

### setState 实际上做了什么

对组件的setState对象安排了一次更新。当state改变了，该组件就会渲染。

### state 和 props 有什么区别

定义上不同，props 和 state 都是普通的对象，props 是传递给组件的，而 state 是组件内部自己管理的状态

### setState 两个例子

```
class Button extends React.Component {
  constructor() {
    super();
    this.state = {
      count: 0,
    };
  }
  updateCount() {
    this.setState({
      count: this.state.count + 1 
    });
  }
  count() {
    this.updateCount()
    this.updateCount()
    this.updateCount()
    this.updateCount()
    this.updateCount()
  }
  render() {
    return (<button
              onClick={() => this.count()}
            >
              Clicked {this.state.count} times
            </button>);
  }
}
React.render(<Button />, document.getElementById('app'));
```
执行1次click事件，state.count 值为 1
并没有得到我们想要的5。 因为updateCount 每次是取的 this.state.count, 这个值初始是0， 不会更新，直到组件被重新渲染才会更新。
React 在重新渲染之前会等待setState操作，只去渲染一次

基于上面的问题，我们可以做下面的修改：
```
    updateCount() {
        this.setState((prevState, props) => {
            return { count: prevState.count + 1 }
        });
    }
```
依赖当前state修改state
可以得到我们想要的5

### setState 函数的实现
setState(updater, [callback])
setState 将对组件 state 的更改排入队列，并告知React 使用新的state 去渲染此组件和子组件。
setState 不是立即更新组件，会批量推迟更新。如果有些操作是要在更新完成后处理，可以使用 componentDidUpdate 或者 setState(updater, [callback]) 的callback事件执行
。
如果像上例中，需要依赖上一次的state，需要了解下updater函数.

### updater 函数
```
    updater = (prevState, props) => stateChange
```
updater 接收的参数是最新的state 和 props。 updater 的返回值会与state进行 浅合并。

### setState 何时以及为什么会批量执行
在一个React event handler 内，无论执行多少个setState 在多少个组件，都只会在事件结束之前执行一次 re-render。更新也是按照setState发生的顺序，进行一个浅层的合并。
React 16 或更早的版本，没有对react事件处理程序之外的setState做批处理。如下
```
    promise.then(()=>{
        // 不在一个react event handler中，他们是格子渲染的
        this.setState({a: true}); // 1
        this.setState({b: true}); // 2
        this.props.setParentState(); // 渲染父组件
    })
```
才意识到，是由你当前是否在一个react 事件处理程序中。
react16或以下版本可以采用黑魔法，如下：
```
    promise.then(()=>{
        ReactDOM.unstable_batchedUpdates(() => {
            this.setState({a: true}); // no re-render
            this.setState({b: true}); // no re-render
            this.props.setParentState(); // no re-render
        });
        当unstable_batchUpdates执行结束后，re-render 一次。
    })
```
### setState 为什么是异步的
保证内部一致性，props和state的一致
启用并发更新
    我们一直在说的异步更新是 React会根据setState来自不同的调用区分事件优先级，如一个事件处理程序，一个网络响应， 一个动画等。

大家看一下原文的回答吧。https://github.com/facebook/react/issues/11527
