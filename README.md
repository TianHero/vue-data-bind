# vue-data-bind
vue 双向数据绑定
# 原理
Vue.js 最显著的功能就是响应式系统，它是一个典型的 MVVM 框架，模型（Model）只是普通的 JavaScript 对象，修改它则视图（View）会自动更新。这种设计让状态管理变得非常简单而直观，不过理解它的原理也很重要，可以避免一些常见问题。下面让我们深挖 Vue.js 响应式系统的细节，来看一看 Vue.js 是如何把模型和视图建立起关联关系的。
- 通过 Observer 对 data 做监听，并且提供了订阅某个数据项变化的能力。
- 把 template 编译成一段 document fragment，然后解析其中的 Directive，得到每一个 Directive 所依赖的数据项和update方法。

- 通过Watcher把上述两部分结合起来，即把Directive中的数据依赖通过Watcher订阅在对应数据的 Observer 的 Dep 上。当数据变化时，就会触发 Observer 的 Dep 上的 notify 方法通知对应的 Watcher 的 update，进而触发 Directive 的 update 方法来更新 DOM 视图，最后达到模型和视图关联起来。

已经了解到vue是通过数据劫持的方式来做数据绑定的，其中最核心的方法便是通过Object.defineProperty()来实现对属性的劫持，达到监听数据变动的目的，无疑这个方法是本文中最重要、最基础的内容之一，如果不熟悉defineProperty [点这里](https://github.com/TianHero/defineProperty)  

整理了一下，要实现mvvm的双向绑定，就必须要实现以下几点：
- 1、实现一个数据监听器Observer，能够对数据对象的所有属性进行监听，如有变动可拿到最新值并通知订阅者
- 2、实现一个指令解析器Compile，对每个元素节点的指令进行扫描和解析，根据指令模板替换数据，以及绑定相应的更新函数
- 3、实现一个Watcher，作为连接Observer和Compile的桥梁，能够订阅并收到每个属性变动的通知，执行指令绑定的相应回调函数，从而更新视图

```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <title>Two-way-data-binding</title>
</head>

<body>

  <div id="app">
    <input type="text" v-model="text"> {{ text }}
  </div>

  <script>
    function observe(obj, vm) {
      Object.keys(obj).forEach(function (key) {
        defineReactive(vm, key, obj[key]);
      })
    }
    function defineReactive(obj, key, val) {
      var dep = new Dep();
      Object.defineProperty(obj, key, {
        get: function () {
          // 添加订阅者 watcher 到主题对象 Dep
          if (Dep.target) dep.addSub(Dep.target);
          return val
        },
        set: function (newVal) {
          if (newVal === val) return
          val = newVal;
          // 作为发布者发出通知
          dep.notify();
        }
      });
    }
    function nodeToFragment(node, vm) {
      var flag = document.createDocumentFragment();
      var child;
      // 许多同学反应看不懂这一段，这里有必要解释一下
      // 首先，所有表达式必然会返回一个值，赋值表达式亦不例外
      // 理解了上面这一点，就能理解 while (child = node.firstChild) 这种用法
      // 其次，appendChild 方法有个隐蔽的地方，就是调用以后 child 会从原来 DOM 中移除
      // 所以，第二次循环时，node.firstChild 已经不再是之前的第一个子元素了
      while (child = node.firstChild) {
        compile(child, vm);
        flag.appendChild(child); // 将子节点劫持到文档片段中
      }
      return flag
    }
    function compile(node, vm) {
      var reg = /\{\{(.*)\}\}/;
      // 节点类型为元素
      if (node.nodeType === 1) {
        var attr = node.attributes;
        // 解析属性
        for (var i = 0; i < attr.length; i++) {
          if (attr[i].nodeName == 'v-model') {
            var name = attr[i].nodeValue; // 获取 v-model 绑定的属性名
            node.addEventListener('input', function (e) {
              // 给相应的 data 属性赋值，进而触发该属性的 set 方法
              vm[name] = e.target.value;
            });
            node.value = vm[name]; // 将 data 的值赋给该 node
            node.removeAttribute('v-model');
          }
        };
        new Watcher(vm, node, name, 'input');
      }
      // 节点类型为 text
      if (node.nodeType === 3) {
        if (reg.test(node.nodeValue)) {
          var name = RegExp.$1; // 获取匹配到的字符串
          name = name.trim();
          new Watcher(vm, node, name, 'text');
        }
      }
    }
    function Watcher(vm, node, name, nodeType) {
      Dep.target = this;
      this.name = name;
      this.node = node;
      this.vm = vm;
      this.nodeType = nodeType;
      this.update();
      Dep.target = null;
    }
    Watcher.prototype = {
      update: function () {
        this.get();
        if (this.nodeType == 'text') {
          this.node.nodeValue = this.value;
        }
        if (this.nodeType == 'input') {
          this.node.value = this.value;
        }
      },
      // 获取 data 中的属性值
      get: function () {
        this.value = this.vm[this.name]; // 触发相应属性的 get
      }
    }
    function Dep() {
      this.subs = []
    }
    Dep.prototype = {
      addSub: function (sub) {
        this.subs.push(sub);
      },
      notify: function () {
        this.subs.forEach(function (sub) {
          sub.update();
        });
      }
    }
    function Vue(options) {
      this.data = options.data;
      var data = this.data;
      observe(data, this);
      var id = options.el;
      var dom = nodeToFragment(document.getElementById(id), this);
      // 编译完成后，将 dom 返回到 app 中
      document.getElementById(id).appendChild(dom);
    }
    var vm = new Vue({
      el: 'app',
      data: {
        text: 'hello world'
      }
    })
  </script>
</body>

</html>
```
