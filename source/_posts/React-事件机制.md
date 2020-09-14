---
title: React 事件机制
---

## React事件处理
React元素的事件处理与DOM元素的事件很类似，React使用时，不需要使用addEventListener为已创建的DOM元素添加监听器，事实上React是在初次渲染时添加监听器。下面列一下React事件处理要注意的点
1.  React事件的命名采用的是小驼峰
2.  使用JSX语法时需要传入的是一个函数作为事件处理函数，而不是一个字符串
3.  组织默认事件的执行，不能直接return false，而是要用preventDefault阻止。
   ```
   jumpfalse(e){ //e是一个合成事件，后面我们会对react的合成事件做一些介绍
       e.preventDefault(); //阻止默认的跳转事件
       console.log("react 事件");
   }
    <a href="#" onClick={this.jumpfalse}></a>

   ```
   ```
    <a href="#" onclick="console.log("dom事件，阻止默认跳转事件"); return false;"></a>
   ```
4.  事件处理程序传递的参数, 我们看下下面这两种传参
    ```
    <button onClick={(e) => this.deleteRow(id, e)}>Delete Row</button>
    <button onClick={this.deleteRow.bind(this, id)}>Delete Row</button>
    ```
    在这两种情况下，React的事件对象e会被当做第二个参数传递。如果通过箭头函数，事件对象必须显示的进行传递。通过bind的方式，事件对象以及更多的参数会被隐式的进行传递。

### 箭头函数 和 Function.prototype.bind
箭头函数的表达比函数表达式更为简单，它没有this, arguments, super 或 new.target。
适用于本来需要匿名函数的地方，并且它不能用作构造函数。
关于箭头函数的一些语法可以查阅 https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/Arrow_functions

bind()方法创建一个新函数，在bind()被调用时，这个新函数的this被指定为bind()的第一个参数，而其余参数将被作为新函数的参数，供调用时使用。
Plofill:
```
if (!Function.prototype.bind) (function() {
    var slice = Array.prototype.slice;
    Function.prototype.bind = function(){
        var thatFunc = this, ThatArgs = arguments[0];
        var args = slice.call(arguments, 1);
        if (typeof thatFunc !== "function") {
            throw new TypeError('error');
        };
        return function(){
            var funcArgs = args.concat(slice.call(arguments));
            return thatFunc.apply(thatArgs, funcArgs);
        }
    };
})();
```

## React 合成事件
合成事件是由React事件系统一部分的SyntheticEvent包装器。
SyntheticEvent 实例将被传递给你的事件处理函数，它是浏览器的原生事件的跨浏览器包装器。除兼容所有浏览器外，它还保留了浏览器原生的事件接口，如stopPropagation() 和 preventDefault()。
每个SyntheticEvent对象都包含以下属性：
```
boolean bubbles
boolean cancelable
DOMEventTarget currentTarget
boolean defaultPrevented
number eventPhase
boolean isTrusted
DOMEvent nativeEvent
void preventDefault()
boolean isDefaultPrevented()
void stopPropagation()
boolean isPropagationStopped()
void persist()
DOMEventTarget target
number timeStamp
string type
```
其中 DOMEvent nativeEvent 是原生对象，需要使用浏览器的底层事件时，通过 nativeEvent 属性能获取到。

##### React 使用 SyntheticEvent 的两个目的
1.  兼容各个浏览器事件对象之间的差异
2.  避免垃圾回收。事件对象可能会被频繁创建和回收， 所以React搞了一个事件池，从这个池中获取或释放事件对象。换句话说，就是事件对象不会被释放掉，而是存在一个数组中，一旦有事件触发，就从这个数组中弹出来，这样可避免频繁地创建和销毁。
SyntheticEvent的函数可查看 https://github.com/facebook/react/blob/75ab53b9e1de662121e68dabb010655943d28d11/packages/events/SyntheticEvent.js#L62

源码中介绍两个函数属性： isPersistent 和 persist
persist() 是不要释放。 isPersiste()判断是否是持久的，是就不会被释放掉。

在异步场景下或者想持久地引用SyntheticEvent对象，可以通过调用 persist()方法。或者是引用 nativeEvent 。



#### React 支持的事件
React 通过 normalize 事件让他们在不同浏览器中拥有一致的属性。
详细可查阅官方文档


### React 事件注册
React 事件注册过程主要做了两件事：
1.  document上注册
    在组件挂载阶段，根据组件内部声明的事件类型(onclick、 onchange)等，在document上使用addEventListener注册事件，并指定统一的回调函数 dispatchEvent 。因为这一事件委托机制，具有同样的回调函数 dispatchEvent ， 所以对同一事件类型，不论在document上注册了几次，最终也只会保留一个有效的实例，这能减少内存开销。
    ```
    // JSX语法
    handleClick1() {
        // ...
    }
    handleClick2() {
        // ...
    }
    render() {
        return(
            <div>
                <button onClick={this.handleClick1}></button>
                <button onClick={this.handleClick2}></button>
            </div>
        )
    }
    // 两个事件类型都是click，由于React的事件委托机制，会统一制定到回调函数 dispatchEvent ，所以最终只会在document上保留一个click事件
    ```

2.  存储事件回调
    React为了在触发事件时可以查找到对象的回调函数执行，会把组件内的所有时间统一放到一个对象( listenBank )中. 存储方式会根据事件类型，如click事件相关的统一存储在一个对象中， 回调函数采用（value/key）的方式存储在对象中，key是唯一标识id，value 对应的是事件回调函数。

3. 源码
   enqueuePutListener方法是整个React事件机制的核心方法。这个方法是注册事件到document上，存储回调函数到EventPluginHub中。
   ```
   /*
    @param id: 事件绑定的当前元素的id
    @param registrationName： 绑定的事件名称，onClick。
    @param listener： onClick后的那个回调函数
    @param transaction： 一个事务
    */ 
    function enqueuePutListener(id, registrationName, listener, transaction) {
    // id是当前元素的id，就是你绑定的事件，比如onClick绑定在哪个元素上，那个元素的data-reactid属性的值。
    var container = ReactMount.findReactContainerForID(id);
    // container就是包裹所有React组件的那个元素。单页面应用一般只有一个 就是我们在HTML中定义的那个 <div id="app"></div>
    if (container) {
        var doc = container.nodeType === ELEMENT_NODE_TYPE ? container.ownerDocument : container;
        // doc是一个document
        listenTo(registrationName, doc);
        // 调用listenTo方法注册事件。注意，这里只是将事件注册在了document上，而没有给我们定义的回到函数。
        // listenTo方法是ReactBrowserEventEmitter模块的listenTo方法。
    }
    // 使用事务，将listenr存储起来。
    transaction.getReactMountReady().enqueue(putListener, {
        id: id,
        registrationName: registrationName,
        listener: listener
    });
    }
   ```
    enqueuePutListener 有两个核心方法，listenTo 注册到document上； enqueue 是存储回调函数
    ```
    listenTo: function (registrationName, contentDocumentHandle) {
        <!--listenTo方法，考虑了太多的可能性，我们删除一些无用的代码，保留核心代码-->
        <!--mountAt就是事件注册的document（页面中可以有多个document）-->
        var mountAt = contentDocumentHandle;
        <!--// isListening就是当前Document---mountAt，对应的一个唯一的对象。-->
        var isListening = getListeningForDocument(mountAt);
        <!--定义我们写入的事件与top事件的对应。 onAbort:['topAbort'],dependencies对应的是['topAbort']-->
        <!--这里的top事件就是EventConstants模块中定义的，这些top事件是我们原始的事件类型，在顶层的一种表示方法。只是换了个名字而已。就像是click事件，我们绑定的时候是写作onClick的，但是，React要使用顶层来代理这些事件，所以就写作了topClick。-->
        var dependencies = EventPluginRegistry.registrationNameDependencies[registrationName];
        <!--EventPluginRegistry模块的registrationNameDependencies属性是一个对象，主要是保存了我们写作的事件与顶层事件的一一对应关系。就像是
        {
            onAbort:['topAbort'],onClick:['topClick']
        }
        -->
        ReactBrowserEventEmitter.ReactEventListener.trapBubbledEvent(dependency, topEventMapping[dependency], mountAt);
        <!--这里的核心调用了trapBubbledEvent方法，也有一些情况是使用 trapCapturedEvent方法顾名思义，他们是在将事件注册在不同的事件流中（捕获事件流或者是冒泡事件流），下面我们以冒泡事件流为例说明。-->
    }
    ```
    ```
     //  @param topLevelType：topAbort,topClick等
     //  @param handlerBaseName：abort,click等
     //  @param handle：一个document
     trapBubbledEvent: function (topLevelType, handlerBaseName, handle) {
        var element = handle;
        if (!element) {
        return null;
        }
        /*
        注意：
        listen方法主要是调用了addEventListener方法。而addEventListener方法接受三个参数。
        //  @param eventType：就是click，abort等。
        //  @param callback ：就是事件触发的时候的回调函数。
        //  @param Boolean值： 表示在什么时候触发，冒泡阶段，还是捕获阶段默认是false，冒泡阶段。
        而到了这里，我们没有拿到一个回调函数callback，你看给的三个参数是：topLevelType, handlerBaseName, handle

        这里React给了一个函数 ReactEventListener.dispatchEvent.bind(null, topLevelType)来做回调函数。
        这个回调函数，和我们绑定在页面上的回调函数不是一回事。
        我们在页面上绑定的回调函数，最终是存储在了EventPluginHub的listenerBank中了。
        */ 
        <!--核心还是调用了EventListener模块的listen方法-->
        return EventListener.listen(element, handlerBaseName, ReactEventListener.dispatchEvent.bind(null, topLevelType));
    },
    ```
    ```
    <!-- 该方法就是调用了注册方法的API。抹平了不同浏览器的差异。在Target上注册了一个事件类型为eventType的事件
    target是上边传过来的element，也就是一个document。
    eventType：是一个事件类型，比如click，change等。
    callback:我们之前说过，我们再组件内写的方法是没有注册在这的。
    这里的callback是React提供的一个分发函数。ReactEventListener.dispatchEvent（）。而我们在组件中写的方法最终是存储到了EventPluginHub中。-->
        listen: function (target, eventType, callback) {
            if (target.addEventListener) {
            target.addEventListener(eventType, callback, false);
            return {
                remove: function () {
                target.removeEventListener(eventType, callback, false);
                }
            };
            } else if (target.attachEvent) {
            target.attachEvent('on' + eventType, callback);
            return {
                remove: function () {
                target.detachEvent('on' + eventType, callback);
                }
            };
        }
    },
    ```

