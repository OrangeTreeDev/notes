### Vuex源码阅读小结

------

`Vuex`的核心包括三个类：`Store`，`ModuleCollection`和`Module`。![Vuex UML](https://github.com/OrangeTreeDev/notes/blob/master/Vuex_UML.png)

#### `Store`类

构造`Store`实例主要包括如下工作：

1. 初始化实例属性`_mudules`。根据传入的option，构造`ModuleCollection`实例；

   ![初始化实例属性_mudules](https://github.com/OrangeTreeDev/notes/blob/master/registerModule.png)

2. 安装模块。通过`this._modules`遍历所有的`Module`对象，依次将这些Module的`action`、`mutation`、`getter`和`state`挂载到`Store`实例的实例属性 `_actions`、 `_mutation`、 `_wrappedGetters` 和局部对象`state`上。![安装模块](https://github.com/OrangeTreeDev/notes/blob/master/installModule.png)

   ![创建局部context](https://github.com/OrangeTreeDev/notes/blob/master/makeLocalContext.png)

3. 初始化`store vm`。把`state`作为`data`，和`_wrappedGetters`作为`computed`，构建一个`vue`实例，帮助`store`实现响应式。

   ![初始化store vm](https://github.com/OrangeTreeDev/notes/blob/master/resetStoreVM.png)

