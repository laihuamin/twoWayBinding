学了vue已经有2个月了，项目做了不少，但是还停留在操作层面，最近开始对原理层面的东西做一些深入性的挖掘，写下来和大家共享，也让自己整理一遍之后，能有所收获。

<!--more-->
## 思路分析
先从MVVM框架说起，MVVM框架都是data和view通过一个viewModel进行双向绑定，data驱动视图更新，视图改变驱动data变化。
<br>
![MVVM框架view与data](http://upload-images.jianshu.io/upload_images/6018041-7f180404b25ea616.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</br>
那接下来我们在来说说data是怎么更新视图的，讲源码的时候会详细讲。这里就粗略一点，其实主要起作用的一个函数是Object.defineProperty()函数用的set函数，来知道data变化了，需要更新视图。
<br>
![data更新view](http://upload-images.jianshu.io/upload_images/6018041-bc0536105b2760f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</br>
## 实现过程
先看流程图：
<br>
![流程图](http://upload-images.jianshu.io/upload_images/6018041-0cc9d94a3f3c9af7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
</br>
首先，我们需要有：
一个观察者observer，用来劫持和监听所有属性，要是发生变化通知订阅者
一个订阅者watcher，可以收到属性变化并通知相应的执行函数，来更新视图
一个解析器complie，可以扫描和解析每一个节点的相关指令，并初始化模板数据和初始化相应的订阅者
具体流程：
1、先要有一个observer，来监听所有属性值，当其发生变化时通知订阅者watcher。
2、订阅者并非一个，而是有多个，需要有一个消息订阅者Dep来手机众多的订阅者，然后把他和observer进行统一的管理。
3、接下来，我们需要有个解析器complie，他会把每一个节点元素进行扫描分析，然后在将其初始化为一个订阅者，并且更新模板数据或者绑定相应的函数。
4、当订阅者接收到相应变化时，就会更新视图。

## 源码分析
![ViewModel实例](http://upload-images.jianshu.io/upload_images/6018041-29ab13665e9081f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
图中是一个最基本的vue实例，但是光有这些东西根本不能产生双向绑定，那Vue是怎么实现双向绑定的，我们一步一步来解析
```
<body>
    <div id="app">
        <input v-model="text">
        {{text}}
    </div>    
<script>
    //vue的实例挂载，相信大家都不陌生
    var vm = new Vue({
        el: 'app', // 挂载点
        data:{
            text: 'hello world' // 数据
        }
    })
</script>
</body>
```
<strong>首先我们先来看一下Vue这个函数</strong>
```
    function Vue (options) {
      this.data = options.data;
      var data = this.data;
      observe(data, this);
      var id = options.el;
      var dom = nodeToFragment(document.getElementById(id), this);
      document.getElementById(id).appendChild(dom); 
    }
```
想必大家和我有一样的疑问，observe这个函数又是干什么的，nodeToFragment这个函数又是干什么的。
<strong>看到这我也不明白，但我知道这个函数大致的作用：</strong>
1、他的数据从可以从options中获取，也就是我们自己定义的那个对象
```
// options就只这个对象
{
    el: 'app',
    data: {
        text: 'hello world'
    }
}
```
然后把data包装在一个变量里，传入到observe中
2、id也是如此，把他包装在id这个变量中，然后获取他的dom元素，传入到nodeToFragment中。
3、做完以上两项工作，获得了一个dom怎么个元素，然后把他插入到id节点元素的后面.
<strong>看完上面这个函数，那么我们先来解析一下observe这个函数</strong>
```
    function observe (obj, vm) {
      Object.keys(obj).forEach(function (key) {
        defineReactive(vm, key, obj[key]);
      });
    }
```
大家还记不记得刚刚传入了什么！！！没关系，忘记了我来帮大家回忆一下
```
observe(data, this);
```
第一个参数是data这个对象，第二个参数是this，当时指向的是Vue这个实例对象。
那么我们来看一下这个函数有什么作用！大家又会奇怪defineReactive这个函数是什么鬼，嘻嘻，我也不知道。
但这个函数是干什么的。
<strong>他是把obj（也就是我们传入的data）中的元素遍历出来，然后将实例+key（每一项所在的索引）+obj\[key](每一项的值)</strong>

<strong>接下来我们一条路走到黑，了解一下defineReactive这个函数</strong>
```
function defineReactive (obj, key, val) {
      var dep = new Dep();
      Object.defineProperty(obj, key, {
        get: function () {
          if (Dep.target) {
              dep.addSub(Dep.target);
          }
          return val
        },
        set: function (newVal) {
          if (newVal === val) {
              return
          }
          val = newVal;
          dep.notify();
        }
      });
    }
```
Object.defineProperty()这个方法是给对象定义一个新属性，或修改对象上的现有属性，并返回该对象。看懂了这个方法，我们来看一下整个函数。
<strong>我们先来做一个小例子，来理解一下这个函数</strong>
```
var _keke = {a: 'lai'};
var o = {};

Object.defineProperty(o, 'h',{
  get: function () {
    console.log('o.h has been got');
    return _keke.a;
  }
  set: function (newVal) {
    console.log('o.h has been assigned');
    _keke.a = newVal;
  }
})
console.log(o.h)    //输出结果 o.h has been got 和 lai
o.h = 'wanse'
console.log(_keke.a) //输出结果 o.h has been assigned 和 wanse
console.log(o.h)  // 输出结果 o.h has been assigned  o.h has been got 和 wanse
```
<strong>那看到这里我们可以大致明白defineReactive函数是干什么的。</strong>
```
// defineReactive函数的主体
function defineReactive (obj, key, val) {
     Object.defineProperty(obj, key, {
         ...
     }
}
```
这个函数里面有get和set方法，分别去修改obj[key]的值。
<strong>首先我们看到</strong>
```
var dep = new Dep();
```
dep是发布订阅者的一个容器。这里先了解一点
<strong>我们先来看set方法</strong>
```
set: function (newVal) {
  if (newVal === val) {
    return
  }
  val = newVal;
  dep.notify();
}
```
当值没有更新的时候，就直接返回，不走之后的流程。当值更新之后，先把data更新了，然后在出发dep的notify方法。
<strong>我们来看一下dep的构造函数Dep</strong>
```
    function Dep () {
      this.subs = []
    }
    Dep.prototype = {
      addSub: function(sub) {
        this.subs.push(sub);
      },
      notify: function() {
        this.subs.forEach(function(sub) {
          sub.update();
        });
      }
    };
```
Dep是一个函数，函数中有一个subs的空数组，Dep的原型有两个方法addSub和notify两个方法，Dep从他的原型继承这两个方法，dep是Dep的实例，也从Dep上继承这两个方法。<strong>sub其实就是一个订阅者，因为我前文说过dep是一个订阅者的一个容器，里面是存的很多个订阅者。</strong>看懂这一点很重要。然后set方法就是用来更新订阅者的。
```
get: function () {
  if (Dep.target) {
    dep.addSub(Dep.target);
  }
  return val
}
```
Dep.target看的有点懵逼，那我们先来看一下下面这个函数
```
    function Watcher (vm, node, name, nodeType) {
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
      //获取data的值
      get: function () {
        this.value = this.vm[this.name];
      }
    }
```
我们不看别的就看Dep.target等于的就是这个Watcher（订阅者），所以get方法是用来添加订阅者的。
其实看到这里可以来强行解释一波为什么sub是一个订阅者，因为Dep.target是一个Watcher，然后他用get方法的addsub添加到subs这个数组中。然后你们看懂了吧，没看懂在来回看几遍。
在之后我们来看你一下set里面的update方法，update的方法中的get方法是获取data的值，然后根据解析器初始化的模板数据，来更新响应的部分。
<strong>这条路已经走到解析器了，差不多要走另外一条路，nodeToFragment这个函数又是干什么的。</strong>
```
    function nodeToFragment (node, vm) {
      var flag = document.createDocumentFragment();
      var child;
      
      while (child = node.firstChild) {
        compile(child, vm);
        flag.appendChild(child); 
      }
      return flag;
    }
```
这个函数有有node和vm两个参数。
```
      var dom = nodeToFragment(document.getElementById(id), this);
```
首先传入的是一个节点元素和那个实例对象，返回的是一个节点元素。
1、创建一个flag，他是一个一个文档片段————主要就是效率的问题，片段的话，可以把所有的更新先插入进来，然后进行操作，可以减少回流和重绘，先了解这个得可以看这篇[重绘与回流(repaint and reflow) -- 前端性能(梁王的理论自习室)](http://www.jianshu.com/p/73fa7b5a2986)
2、定义一个child传入到complie函数中
3、就是理解这个循环，理解这个循环之前我们可以先实验一下appendChild这个方法
```
<body>
<ul id="myList1"><li>Coffee</li><li>Tea</li></ul>
<ul id="myList2"><li>Water</li><li>Milk</li></ul>
<p id="demo">请点击按钮把项目从一个列表移动到另一个列表中。</p>
<button onclick="myFunction()">亲自试一试</button>
<script>
  function myFunction(){
    var node=document.getElementById("myList2").lastChild;
    document.getElementById("myList1").appendChild(node);
  }
</script>
</body>
```
这个例子试一试就知道appendChild这个方法。他是把一个child移到另一个列表中，注意是<strong>移到！！！</strong>
那么我们来看一下这个循环。
```
 while (child = node.firstChild) {
  compile(child, vm);
  flag.appendChild(child); 
}
```
他的意思就是每次循环child是node的第一个，然后用compile解析一下，然后把他移到flag中，之后在重复之前的操作每一次操作node中的元素会减少一个，到最后child是空字符串，然后自转化成Boolean之后，为false。
<strong>接下来我们来看一下complie函数</strong>
```
    function compile (node, vm) {
      var reg = /\{\{(.*)\}\}/;
      if (node.nodeType === 1) {
        var attr = node.attributes;
        for (var i = 0; i < attr.length; i++) {
          if (attr[i].nodeName == 'v-model') {
            var name = attr[i].nodeValue;
            node.addEventListener('input', function (e) {
              vm[name] = e.target.value;
            });
            node.value = vm[name];
            node.removeAttribute('v-model');
          }
        };
        new Watcher(vm, node, name, 'input');
      }
      if (node.nodeType === 3) {
        if (reg.test(node.nodeValue)) {
          var name = RegExp.$1;
          name = name.trim();
          new Watcher(vm, node, name, 'text');
        }
      }
    }
```
这个函数也传入两个东西，第一个是child和实例对象。
<strong>接下来解决几个问题：</strong>
1、nodeType是什么？等于1和等于3又代表什么？
w3c上nodeType的constants值以及其代表内容：
![nodeType.png](http://upload-images.jianshu.io/upload_images/6018041-619780e1dc0e8b05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![节点元素类型](http://upload-images.jianshu.io/upload_images/6018041-f6ee47be8895d899.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过以上两张图，我们可以知道，1表示的是ELEMENT_NODE，也就是元素名，如< div>等。
```
if (node.nodeType === 1) {
  var attr = node.attributes;
    for (var i = 0; i < attr.length; i++) {
      if (attr[i].nodeName == 'v-model') {
        var name = attr[i].nodeValue;
        node.addEventListener('input', function (e) {
          vm[name] = e.target.value;
        });
        node.value = vm[name];
        node.removeAttribute('v-model');
      }
    };
  new Watcher(vm, node, name, 'input');
}
```
这个其实挺难看懂的，看了好久，其实可以从这里看
```
node.addEventListener('input', function (e) {
  vm[name] = e.target.value;
});
```
这个就是对node起到监听的作用，监听的是input标签，在真实的vueJs中会复杂很多，vm[name]是什么，vm是data对象，name是v-model的值，简而言之，就是当input发生变化的时候，把值赋值给data，就讲到这儿吧，基本都看懂了。

接下来就是Vue源码（待续）！！