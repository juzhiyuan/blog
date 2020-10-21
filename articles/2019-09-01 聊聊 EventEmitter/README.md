---
title: 聊聊 EventEmitter
labels: EventEmitter, JavaScript
---
## 场景

存在如下场景：当音频播放器分别处于加载资源、播放音频、播放结束等不同状态时，通过某种机制执行不同操作（如更新 UI 状态）。即通过监听某些对象，当其状态发生变化时，触发不同事件。

上述场景可通过 EventEmitter 实现，该方法至少可对应两种模式：观察者模式与发布/订阅模式。我们先来了解这两种模式：

### 观察者模式

观察者模式（Observer Pattern）定义了 1:n 关系，使 n 个观察者对象监听某个被观察者对象（主题对象），当被观察者对象状态发生变化时，它将主动通知所有观察者对象这个消息，不同的观察者对象将产生不同动作。

这种行为型模式关注的是观察者与被观察者之间的通讯，发布/订阅模式由它衍生而来。

![https://blog-static-cdn.shaoyaoju.org/2019/10/26f49447-eca8-42de-b5f5-71aeeeffe3eb-image.png](https://blog-static-cdn.shaoyaoju.org/2019/10/26f49447-eca8-42de-b5f5-71aeeeffe3eb-image.png)

#### 代码实现

```js
class Subject {
  constructor() {
    this.observers = [];
  }
  
  add(observer) {
    this.observers.push(observer);
  }
    
  notify(...args) {
    this.observers.forEach(observer => observer.update(...args));
  }
}

class Observer {
  update(...args) {
    console.log(...args);
  }
}
```

#### 使用方式

```js
// 创建 2 个观察者 observer1、observer2
const observer1 = new Observer();
const observer2 = new Observer();

// 创建 1 个目标 subject
const subject = new Subject();

// 目标 subject 主动添加 2 个观察者，建立 subject 与 observer1、observer2 依赖关系
subject.add(observer1);
subject.add(observer2);

doSomething();
// 目标触发某些事件、主动通知观察者
subject.notify('subject fired a event');
```

由于观察者模式没有调度中心，观察者必须将自身添加到被观察者中进行管理；被观察者触发事件后，将主动通知每个观察者。

### 发布/订阅模式

发布订阅模式（Pub/Sub Pattern）与观察者模式主要区别是：前者具有事件调度中心，它将过滤发布者发布的所有消息并分发给订阅者，发布者与订阅者无需关心对方是否存在。

```js
https://blog-static-cdn.shaoyaoju.org/2019/10/36d43573-d79b-4e86-8104-0ce9a3a021ac-image.png
```

#### 代码实现

```js
class PubSub {
  constructor() {
  	this.subscribers = [];
  }
  
  subscribe(topic, callback) {
  	const callbacks = this.subscribers[topic];
    if (callbacks) {
      callbacks.push(callback);
    } else {
      this.subscribers[topic] = [callback];
    }
  }
  
  publish(topic, ...args) {
  	const callbacks = this.subscribers[topic] || [];
    callbacks.forEach(callback => callback(...args));
  }
}
```

#### 使用方式

```js
// 创建事件调度中心，为发布者与订阅者提供调度服务
const pubSub = new PubSub();

// A 系统订阅了 SMS 事件，不关心谁将发布这个事件
pubSub.subscribe('lovelyEvent', console.log);
// B 系统订阅了 SMS 事件，不关心谁将发布这个事件
pubSub.subscribe('lovelyEvent', console.log);

// C 系统发布了 SMS 事件，不关心谁会订阅这个事件
pubSub.publish('lovelyEvent', 'I just published a lovely event');
```

两种模式相比，发布/订阅模式有利于系统间解耦，适合处理跨系统消息事件。

## EventEmitter

EventEmitter 是 Node.js [events](https://github.com/Gozala/events/blob/master/events.js) 模块中的类，用于对 Node.js 中事件进行统一管理。通过查看其源码实现，我们可以尝试构造自己的 EventEmitter 来满足文首场景，以下实现为发布/订阅模式的体现。

EventEmitter 实例具有如下几个主要方法：

![https://blog-static-cdn.shaoyaoju.org/2019/10/0f047a8a-e3f5-4fb2-bd47-cb19236173d3-image.png](https://blog-static-cdn.shaoyaoju.org/2019/10/0f047a8a-e3f5-4fb2-bd47-cb19236173d3-image.png)

#### 代码实现

```js
class EventEmitter {
  constructor() {
    this._events = Object.create(null);
  }

  /**
   * 为 EventEmitter 实例添加事件
   */
  addListener(eventName, listener) {
    if (this._events[eventName]) {
      this._events[eventName].push(listener);
    } else {
      this._events[eventName] = [listener];
    }
    return this;
  }

  /**
   * 移除某个事件
   * 调用该方法时，仅移除一个 Listener，参考下方 Case#7
   */
  removeListener(eventName, listener) {
    if (this._events[eventName]) {
      for (let i = 0; i < this._events[eventName].length; i++) {
        if (this._events[eventName][i] === listener) {
          this._events[eventName].splice(i, 1);

          // NOTE: 或调用 spliceOne 方法，参见文末
          // spliceOne(this._events[eventName], i);

          break;
        }
      }
    }
    return this;
  }

  /**
   * 移除指定事件名中的事件或全部事件
   */
  removeAllListeners(eventName) {
    if (this._events[eventName]) {
      this._events[eventName] = [];
    } else {
      this._events = Object.create(null);
    }
  }

  /**
   * 是 addListener 方法别名
   */
  on(eventName, listener) {
    return this.addListener(eventName, listener);
  }

  /**
   * 类似 on 方法，但通过 once 方法注册的事件被多次调用时只执行一次
   */
  once(eventName, listener) {
    let fired = false;
    let onceWrapperListener = (...args) => {
      this.off(eventName, onceWrapperListener);
      if (!fired) {
        fired = true;
        listener.apply(this, args);
      }
    }
    return this.on(eventName, onceWrapperListener);
  }

  /**
   * 是 removeListener 方法别名
   */
  off(eventName, listener) {
    return this.removeListener(eventName, listener);
  }

  /**
   * 主动触发事件
   */
  emit(eventName, ...args) {
    if (this._events[eventName]) {
      for (let i = 0; i < this._events[eventName].length; i++) {
        this._events[eventName][i].apply(this, args);
      }
    }
    return this;
  }
}
```

### 使用方法

```js
const emitter = new EventEmitter();

const fn1 = name => console.log(`fn1 ${name}`);
const fn2 = name => console.log(`fn2 ${name}`);

/**
 * Case 1
 */
emitter.on('event1', fn1);
emitter.on('event1', fn2);

emitter.emit('event1', 'ju');
// Expected Output:
// fn1 ju
// fn2 ju

/**
 * Case 2
 */
emitter.once('event1', fn1);

emitter.emit('event1', 'ju');
emitter.emit('event1', 'ju');
emitter.emit('event1', 'ju');

// Expected Output:
// fn1 ju

/**
 * Case 3
 */
emitter.on('event1', fn1);
emitter.on('event1', fn1);
emitter.off('event1', fn1);
emitter.on('event1', fn1);

emitter.emit('event1', 'ju');

// Expected Output:
// fn1 ju
// fn1 ju

/**
 * Case 4
 */
emitter.on('event1', fn1);
emitter.on('event1', fn2);
emitter.on('event2', fn2);
// emitter.removeAllListeners('event1');

emitter.emit('event1', 'ju');
emitter.emit('event2', 'zhiyuan');

// Expected Output:
// fn1 ju
// fn2 ju
// fn2 zhiyuan

/**
 * Case 5
 */
emitter.on('event1', fn1);
emitter.on('event1', fn2);
emitter.on('event2', fn2);
emitter.removeAllListeners('event1');

emitter.emit('event1', 'ju');
emitter.emit('event2', 'zhiyuan');

// Expected Output:
// fn2 zhiyuan

/**
 * Case 6
 */
emitter.on('event1', fn1);
emitter.on('event1', fn2);
emitter.on('event2', fn2);
emitter.removeAllListeners();

emitter.emit('event1', 'ju');
emitter.emit('event2', 'zhiyuan');

// Expected Output:
// (empty output)

/**
 * Case 7
 */
emitter.on('event1', fn1);
emitter.on('event1', fn1);
emitter.off('event1', fn1);
emitter.on('event1', fn2);

emitter.emit('event1', 'ju');
// Expected Output:
// fn1 ju
// fn2 ju
```

需要注意的是，在调用 off 方法时，其传入的 Listener 函数应与 on 方法中传入的 Listener 函数一致，请勿传递匿名函数，否则 off 方法无效。这是由于在 JavaScript 中，（函数）对象是按引用传递的，两个逻辑一样的函数却并不相等。因此，我们先将函数定义好，再将该函数传入 on/off 方法。

另外，当实例触发一个事件时，所有与其相关的 Listener 将被同步调用（请查看 emit 方法实现：循环数组、依次被调用）。

## 其它

### spliceOne 方法

在 events 官方实现中，调用 removeListener 方法移除某一个 Listener 函数时，调用了自定义的 spliceOne 方法，该方法执行效率优于 splice。实现如下：

```js
function spliceOne(list, index) {
  for (; index + 1 < list.length; index++) {
    list[index] = list[index + 1];
  }

  list.pop();
}
```

### return this

return this 返回了当前对象实例，在本文 EventEmitter 实现中，this 指代 emitter，这样可以实现链式调用。

```js
const emitter = new EventEmitter();

const fn = () => console.log('Hello');

emiter.on('event', fn).emit('event');
// or
emiter.once('event', fn).emit('event');
```

## 参考
1. [Observer vs Pub-Sub pattern](https://hackernoon.com/observer-vs-pub-sub-pattern-50d3b27f838c)
2. [基于观察者模式实现一个 EventEmitter 类](https://www.xiabingbao.com/post/design/observer-eventemitter.html)
3. [浅谈 JavaScript 设计模式之发布订阅者模式](https://www.xiabingbao.com/post/design/js-design-pattern-pubsub.html)
4. [events.js](https://github.com/Gozala/events/blob/master/events.js)
5. [EventEmitterTrait](https://github.com/peridot-php/event-emitter/blob/master/src/EventEmitterTrait.php)
6. [观察者模式与订阅发布模式的区别](https://www.cnblogs.com/onepixel/p/10806891.html)
