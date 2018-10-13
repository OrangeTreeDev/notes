### 阅读Vuex源码引起关于class  this 的思考

#### Vuex基础知识

[Vuex](https://vuex.vuejs.org/)采用集中式存储状态，使用Vuex的核心步骤就是用类`Vuex.Strore`创建一个实例`store`，用`store`来管理应用的状态state，同时约定只能使用`store.commit`方法触发mutation来修改`store.state`。如下一段代码演示Vuex的具体使用。

```javascript
import Vue from 'Vue';
import Vuex from 'vuex';

Vue.use(Vuex);

const store = new Vuex.Store({
    state: {
        count: 0,
    },
    mutations: {
        increment(state) {
            state.count++;
        },
    },
}); 

new Vue ({
    store，
});
```

接下来，调用store.commit触发状态的改变。

```javascript
store.commit('increment');
console.log(store.state.count); // 1
```



#### 阅读源码产生的疑问

Vuex源码使用ES6语法编写的，`Vuex.Store` 是一个class。阅读源码时，发现一段有趣的代码，来来来，群来们一起来吃瓜。

```javascript
// bind commit and dispatch to self
const { dispatch, commit } = this;
const store = this;
this.dispatch = function boundDispatch (type, playload) {
    return dispatch.call(store, type, playload);
};
this.commit = function boundCommit (type, playload, options) {
    return commit.call(store, type, playload, options);
};
```

看到这里，我就走不动了，为什么要手动把`dispatch`和`commit`的`this`指向`Strore`实例了？如果我们删除这段手动绑定的代码一切会正常运行吗？带着好奇心，写了段简单的测试代码，大家一起感受下。

```javascript
class Store {
  constructor(option = {}) {
    this._mutations = Object.create(null);
    this.state = option.state;

    const store = this;
    const rawMutations = option.mutations || {};
    Object.keys(rawMutations).forEach((key) => {
      store._registerMutaton(store, key, rawMutations[key]);
    });
  }

  commit (type, playload) {
    const entry = this._mutations[type];
    entry(playload);
  }

  _registerMutaton(store, type, handle) {
    this._mutations[type] = function wrappedMutation (playload) {
      handle.call(store, store.state, playload);
    }
  }
}
```

接下来使用两种方式调用`commit`。

```
// with store instance
store.commit('increment');
console.log(store.state.count); // 1

// without store instance
const commit = store.commit
commit('increment');
console.log(store.state.count); // Uncaught TypeError: Cannot read property '_mutations' of undefined
```

如果不带上`store`实例直接调用`commit`，会报错`this`为`undefined`。原因大家都是知道的，就是因为没有指明`this`。另人想不通的是，平时的时候不是这样的啊，`this`应该指向全局对象（示例代码是在`window`全局环境下写的，因此，这里`this`应该指向`window`）。这个时候就要用依赖MDN了，查了下class 的语法特性，果不其然，遇见真理。下面摘自MDN的关于[class](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)的描述

> When a static or prototype method is called without a value for *this*, the *this* value will be `undefined` inside the method. This behavior will be the same even if the `"use strict"` directive isn't present, because code within the `class` body's syntactic boundary is always executed in strict mode. 
>
> If the above is written using traditional function-based syntax, then autoboxing in method calls will happen in non–strict mode based on the initial *this* value. If the initial value is `undefined`, *this* will be set to the global object.

 我大致解释下，class 是强制采用严格模式的，在严格模式下，如果`this`没有定义，那么就等于`undefined`，但是，如果是非严格模式下，就会做自动装箱操作（关于装箱和拆箱的知识，可以参考[You Don't Know JS: Coercion](https://github.com/getify/You-Dont-Know-JS/blob/master/types%20%26%20grammar/ch4.md))，如果`this`等于`undefined`，那么`this`就会指向全局对象。

事情到这里，也就告一段落了！