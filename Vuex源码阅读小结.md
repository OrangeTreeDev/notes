### Vuex源码阅读小结

------

Vuex的核心包括三个类：Store，ModuleCollection和Module。![Vuex UML](https://github.com/OrangeTreeDev/notes/Vuex_UML.png)

#### Store类

构造Store实例主要包括如下工作：

1. 初始化实例属性_mudules。根据传入的option，构造ModuleCollection实例；

   ![初始化实例属性_mudules](https://github.com/OrangeTreeDev/notes/registerModule.png)

2. 安装模块。通过this._modules遍历所有的Module对象，依次将这些Module的action、mutation、getter和state挂载到Store实例的实例属性 _actions、 _mutation、 _wrappedGetters 和局部对象state上。![安装模块](https://github.com/OrangeTreeDev/notes/installModule.png)

3. 初始化store vm。把state作为data，和_wrappedGetters作为computed，构建一个vue实例，帮助store实现响应式。

   ![初始化store vm](https://github.com/OrangeTreeDev/notes/resetStoreVM.png)

