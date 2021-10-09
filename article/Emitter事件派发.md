# Emitter事件派发---mitt和tiny-emitter源码分析
在我们学习js过程中，用户点击了某个按钮，然后网页进行某个交互，那么这是如何做到的呢？在js中提供了`addEventListener`和`removeEventListener`两个API来监听和移除事件，用户点击某个节点后，该DOM元素产生了一个click事件并派发出去，`addEventListener`监听到事件后执行某个操作。那么我们该如何实现对应的事件注册、派发、移除等功能呢？接下来我们来看看 mitt 和 tiny-emitter 这两个库吧。

## mitt 分析
[mitt 详细使用](https://github.com/developit/mitt)
```js
import mitt from 'mitt'

const emitter = mitt()

// listen to an event
emitter.on('foo', e => console.log('foo', e) )

// listen to all events
emitter.on('*', (type, e) => console.log(type, e) )

// fire an event
emitter.emit('foo', { a: 'b' })

// clearing all events
emitter.all.clear()

// working with handler references:
function onFoo() {}
emitter.on('foo', onFoo)   // listen
emitter.off('foo', onFoo)  // unlisten
```
在 mitt 这个库中，整体分析起来比较简单，就是 导出了一个 `mitt([all])`函数，调用该函数返回一个 `emitter`对象，该对象包含`all`、`on(type, handler)`、`off(type, [handler])`和`emit(type, [evt])`这几个属性。
主要实现就是：
1. `all = all || new Map()` mitt 支持传入 all 参数用来存储事件类型和事件处理函数的映射Map，如果不传，就 `new Map()`赋值给 all 
2. `on(type, handler)`定义函数 `on`来注册事件，以`type`为属性，`[handler]`为属性值，存储在 all 中，属性值为数组的原因是可能存在监听一个事件，多个处理程序
3. `off(type, [handler])`来取消某个事件的某个处理函数，根据 type 找到对应的事件处理数组，对比 handler 是否相等，相等则删除该处理函数，不传则删除该事件的全部处理函数
4. `emit(type, [evt])`来派发事件，根据 type 找到对应的事件处理数组并依次执行，传入参数 evt(对象最好，传多个参数只会取到第一个)

具体的可参考源码实现
```js
// 导出函数 mitt ，调用该函数返回一个 对象，该对象包含 all、on、off、emit 方法
export default function mitt<Events extends Record<EventType, unknown>>(
	all?: EventHandlerMap<Events>
): Emitter<Events> {
	type GenericEventHandler =
		| Handler<Events[keyof Events]>
		| WildcardHandler<Events>;
	all = all || new Map();

	return {

		all,

		on<Key extends keyof Events>(type: Key, handler: GenericEventHandler) {
			// 获取 type 对应的的 事件处理函数数组
			const handlers: Array<GenericEventHandler> | undefined = all!.get(type);
			// 存在，说明设置过，handlers 已经是数组，直接 push 进去
			if (handlers) {
				handlers.push(handler);
			}
			else {
				// 不存在，说明这是第一次 注册该事件，那么 设置 type 的属性值为 [handler]
				all!.set(type, [handler] as EventHandlerList<Events[keyof Events]>);
			}
		},

		off<Key extends keyof Events>(type: Key, handler?: GenericEventHandler) {
			// 找到 type 事件的 事件处理函数数组
			const handlers: Array<GenericEventHandler> | undefined = all!.get(type);
			if (handlers) {
				if (handler) {
					// 删除指定的 handler处理函数, 找到了 idx >>> 0 就是idx对应的索引，
					// 没找到 -1 变为 4294967295，原数组不会改变
					// >>> 无符号右移 正数不会变
					handlers.splice(handlers.indexOf(handler) >>> 0, 1);
				}
				else {
					// 说明删除全部
					all!.set(type, []);
				}
			}
		},

		emit<Key extends keyof Events>(type: Key, evt?: Events[Key]) {
			// 找到 type 事件的 事件处理函数数组
			let handlers = all!.get(type);
			if (handlers) {
				// 依次执行每个函数，并传入相应的参数
				(handlers as EventHandlerList<Events[keyof Events]>)
					.slice()
					.map((handler) => {
						handler(evt!);
					});
			}

			// 是否监听了全部事件
			handlers = all!.get('*');
			if (handlers) {
				// 依次执行每个函数，并传入相应的参数
				(handlers as WildCardEventHandlerList<Events>)
					.slice()
					.map((handler) => {
						handler(type, evt!);
					});
			}
		}
	};
}
```

缺点：
1. mitt 支持传入 all 参数，如果 all = {} 而不是 new Map() 那么会报错，不能正常使用，在ts中当然会提醒你，但是如果在js中使用这个库就没有提示，运行时会报错（当然可能是我用法错了，我直接在html文件中引用）
2. `emit(type, [evt])`中只能接受一个参数，要是传多个参数需要将多个参数合并成对象传入，然后在事件处理函数中解构

## tiny-emitter 分析
[tiny-emitter 详细使用](https://github.com/scottcorgan/tiny-emitter)

在 tiny-emitter 这个库中，定义了函数`E`，修改`E.prototype`，在原型对象中引入了`on(name, callback, ctx)`、`once(name, callback, ctx)`、`emit(name)`、`off(name, callback)`四个方法，整体功能和 mitt 大同小异，直接看源代码吧
```js
function E () {
  // Keep this empty so it's easier to inherit from
  // (via https://github.com/lipsmack from https://github.com/scottcorgan/tiny-emitter/issues/3)
}

// return this 支持链式调用
E.prototype = {
  on: function (name, callback, ctx) {
    // 获取事件处理函数和事件类型的映射对象，第一次不存在则赋值为空对象
    var e = this.e || (this.e = {});

    // 获取 name 类型的事件处理函数，如果不存在先赋值为 [],在push进去，保持事件处理函数是一个数组
    (e[name] || (e[name] = [])).push({
      fn: callback,
      ctx: ctx
    });

    return this;
  },

  once: function (name, callback, ctx) {
    var self = this; // this 丢失问题
    function listener () {
      // 派发执行后，注销该事件
      self.off(name, listener);
      callback.apply(ctx, arguments);
    };

    listener._ = callback
    return this.on(name, listener, ctx);
  },

  emit: function (name) {
    // 获取传入的参数，可以多个
    var data = [].slice.call(arguments, 1);
    // 获取 name 类型的事件处理函数数组
    var evtArr = ((this.e || (this.e = {}))[name] || []).slice();
    var i = 0;
    var len = evtArr.length;

    // 遍历执行事件处理函数
    for (i; i < len; i++) {
      // evtArr[i] ==>  { fn: callback, ctx: ctx }, data 是传入的参数
      evtArr[i].fn.apply(evtArr[i].ctx, data);
    }

    return this;
  },

  off: function (name, callback) {
    var e = this.e || (this.e = {});
    var evts = e[name]; // 事件处理函数数组
    var liveEvents = []; // 注销某事件的一个处理函数后的剩余函数数组

    if (evts && callback) {
      for (var i = 0, len = evts.length; i < len; i++) {
        // 排除 callback 这个函数，之后在派发该事件就不执行 这个处理函数
        // evts[i].fn._ 对应 once 的时候，此时 evts[i].fn 是 listener 函数， evts[i].fn._  === callback
        if (evts[i].fn !== callback && evts[i].fn._ !== callback)
          liveEvents.push(evts[i]);
      }
    }

    // Remove event from queue to prevent memory leak
    // Suggested by https://github.com/lazd
    // Ref: https://github.com/scottcorgan/tiny-emitter/commit/c6ebfaa9bc973b33d110a84a307742b7cf94c953#commitcomment-5024910

    // 注销事件后，还存在其他事件就进行赋值，否则删除
    (liveEvents.length)
      ? e[name] = liveEvents
      : delete e[name];

    return this;
  }
};
```

## mitt 和 tiny-emitter 对比分析

### 共同点
1. 都支持`on(type, handler)`、`off(type, [handler])`和`emit(type, [evt])`三个方法来注册、注销、派发事件

### 不同点
- emit
    1. 有 all 属性，可以拿到对应的事件类型和事件处理函数的映射对象，是一个Map不是{} 
    2. 支持监听'*'事件，可以调用`emitter.all.clear()`清除所有事件
    3. 返回的是一个对象，对象存在上面的属性
- tiny-emitter
    1. 支持链式调用, 通过e属性可以拿到所有事件（需要看代码才知道）
    2. 多一个 `once` 方法 并且 支持设置 `this`(指定上下文 ctx)
    3. 返回的一个函数实例，通过修改该函数原型对象来实现的
