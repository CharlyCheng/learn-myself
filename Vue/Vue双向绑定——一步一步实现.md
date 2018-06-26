---
title: Vue双向绑定原理及实现
date: 2018-04-20 15:00:16
categories:
- 前端开发
tags:
- vue
---
### 深入响应式原理

Vue 最独特的特性之一，是其非侵入性的响应式系统。数据模型仅仅是普通的 JavaScript 对象。而当你修改它们时，视图会进行更新。这使得状态管理非常简单直接，将进入一些 Vue 响应式系统的底层的细节。

#### 追踪变化

普通的 JavaScript 对象传给 Vue 实例的 data 选项，Vue 将遍历此对象所有的属性，并使用 Object.defineProperty 把这些属性全部转为 getter/setter，在内部它们让 Vue 追踪依赖，在属性被访问和修改时通知变化。每个组件实例都有相应的 watcher 实例对象，它会在组件渲染的过程中把属性记录为依赖，之后当依赖项的 setter 被调用时，会通知 watcher 重新计算，从而致使它关联的组件得以更新

[响应原理图](https://cn.vuejs.org/v2/guide/reactivity.html)


##### 简单实现双向绑定

基本实现：

```
var obj = {};
Object.defineProperty(obj, "hello", {
  get: function () {return sth},
  set: function (val) {do sth}
})
```

obj.hello // 可以像普通属性一样读取访问器属性
访问器属性的"值"比较特殊，读取或设置访问器属性的值，实际上是调用其内部特性：get和set函数。
obj.hello // 读取属性，就是调用get函数并返回get函数的返回值
obj.hello = "abc" // 为属性赋值，就是调用set函数，赋值其实是传参

此例实现的效果是：随文本框输入文字的变化，span 中会同步显示相同的文字内容；在js或控制台显式的修改 obj.hello 的值，视图会相应更新。这样就实现了 model => view 以及 view => model 的双向绑定。

```
<input type="text" id="a" />
<span id="b"></span>

<script>
    var obj = {}
    Object.defineProperty(obj, "hello", {
        get: function() {
            console.log("get方法被调用了")
        },
        set: function(newVal) {
            document.getElementById("a").value = newVal
            document.getElementById("b").innerHTML = newVal
        }
    })
    document.addEventListener("keyup", function (e){
        obj.hello = e.tartget.value
    })
</script>

```

##### 真正实现Vue的双向绑定
页面结构：

```
<div id="app">
    <form>
        <input type="text" v - model="number">
        < button type="button" v - click="increment">增加</ button>
    </form>
    <h3 v - bind="number"></h3>
</div>
```
最后会通过类似于vue的方式来使用我们的双向数据绑定，结合我们的数据结构添加注释

```
var app = new myvue({
    el: "#app",
    data: {
        number: 0
    },
    methods: {
        increment: function() {
            this.number++
        }
    }
})
```
首先定义一个myVue构造函数：
```
function MyVue(options){

}
```

初始化这个构造函数，添加_init 属性：
```
function myVue(options) {

}
myvue.prototype._init = function (options) {
    this.$options = options
    this.$el = document.querySelector(options.el)
    this.$data = options.data
    this.$methods = options.methods
}
```

接下来实现_obverse函数， 对data进行处理， 重写data的set和get函数， 并改造_init函数

```
myvue.prototype._obverse = function (obj) {
    var value
    for (key in obj) {
        if (obj.hasOwnproperty(key)){
            value = obj[key]
            if (typeof value === "object") {
                this._obverse(value)
            }
            Object.defineProperty(this.$data, key, {
                enumerable: true,
                configurable: true,
                get: function() {
                    console.log(`获取${value}`)
                    return value
                },
                set: function(newVal) {
                    console.log(`更新${newVal}`)
                    if (value !== newVal) {
                        value = newVal
                    }
                }
            })
        }
    }
}
myvue.prototype._init = function (options) {
    this.$options = options
    this.$el = document.querySelector(options.el)
    this.$data = options.data
    this.$methods = options.methods

    this._obverse(this.$data)
}
```

接下来我们写一个指令类Watcher，用来绑定更新函数，实现对DOM元素的更新：

```
function Watcher (name, el, vm, exp, attr) {
    this.name = name;
    this.el = el
    this.exp = exp
    this.attr = addEventListener
    this.updata()
}

Watcher.prototype.updata = function() {
    this.el[this.attr] = this.vm.$data[this.exp]
}
```

更新_init函数以及_obverse函数:

```
myvue.prototype._init = function (options) {
    this._binding = {}
}
myvue.prototype._obverse = function (obj) {
    var value
    for (key in obj) {
        if (obj.hasOwnproperty(key)){
            this._bind[key] = {
                _directives: []
            }
            var binding = this._binding[key];
            value = obj[key]
            if (typeof value === "object") {
                this._obverse(value)
            }
            Object.defineProperty(this.$data, key, {
                enumerable: true,
                configurable: true,
                get: function() {
                    console.log(`获取${value}`)
                    return value
                },
                set: function(newVal) {
                    console.log(`更新${newVal}`)
                    if (value !== newVal) {
                        value = newVal
                        binding._directives.forEach(function (item) {
                                item.updata()
                        })
                    }
                }
            })
        }
    }
}
```

那么如何将view与model进行绑定呢？接下来我们定义一个compile 函数，用来解析我们的指令（v-bind,v-model,v-clickde）等，并在这个过程中对view与model进行绑定。

```
myVue.prototype._init = function (options) {
    this._complie(this.$el)
}

myvue.prototype._complie = function(root) {
    var _this = this;
    var nodes = root.children
    for (var i =0; i < nodes.length; i++) {
        var node = nodes[i]
        if (node.children.length) {
            this._compile(node)
        }
        if (node.hasAttrbute('v-click')) {
            node.onclick = (function(){
                var attrVal = nodes[i].getAttribute["v-click"]
                return _this.$methods[attrVal].bind(_this.$data)
            })()
        }
        if (node.hasAttrbute('v-model') && (node.tagName == "INPUT" || node.tagName == "TEXTAREA")) {
            node.addEventListener('input', (function(key){
                var attrVal = node.getAttribute("v-model");
                _this._binding[attrVal]._directives.push(new Watcher(
                        "input",
                        node,
                        _this,
                        attrVal,
                        "value"
                ))
                return function () {
                    _this.$data[attrVal] = nodes[key],value
                }
            })(i))
        }
        if (node.hasAttrbute('v-bind')) {
            node.onclick = (function(){
                var attrVal = nodes[i].getAttribute["v-bind"]
                _this._binding[attrVal]._directives.push(new Watcher(
                        "text",
                        node,
                        _this,
                        attrVal,
                        "innerHTML"
                ))
            })()
        }
    }
}
```

至此，我们已经实现了一个简单vue的双向绑定功能，包括v-bind, v-model, v-click三个指令。效果如下图：

附上全部代码：
```
<!Doctype html>
<head>
    <meta charset="utf-8"/>
    <style>
        #app {
            text-align: center
        }
    </style>
</head>
<body>
   <div id="app">
        <input type="text" v-model = "number" />
        <button type="button" v-click = "increment">增加</button>
        <h3 v-bind="number"></h3>
   </div>
</body>
<script>
function myVue (options) {
    this . _init (options);
}
myVue . prototype . _init=function (options) {
    this . $options=options;
    this . $el=document . querySelector (options . el);
    this . $data=options . data;
    this . $methods=options . methods;
    this . _binding= {

    }
;
this . _obverse (this . $data);
    this . _complie (this . $el);
}
myVue . prototype . _obverse=function (obj) {
    var value;
    for (key in obj) {
        if (obj . hasOwnProperty (key)) {
            this . _binding[key]= {
                _directives :[]
            }
            ;
            value=obj[key];
            if (typeof value==='object') {
                this . _obverse (value);
            }
            var binding=this . _binding[key];
            Object . defineProperty (this . $data, key, {
                enumerable : true, configurable : true, get : function () {
                    console . log (`获取 $ {
                        value
                    }
                    `);
                    return value;
                }
                , set : function (newVal) {
                    console . log (`更新 $ {
                        newVal
                    }
                    `);
                    if (value !==newVal) {
                        value=newVal;
                        binding . _directives . forEach (function (item) {
                            item . update ();
                        }
                        )
                    }
                }
            }
            )
        }
    }
}
myVue . prototype . _complie=function (root) {
var _this=this;
var nodes=root . children;
for (var i=0;i < nodes . length;i ++) {
    var node=nodes[i];
    if (node . children . length) {
        this . _complie (node);
    }
    if (node . hasAttribute ('v-click')) {
        node . onclick=(function () {
            var attrVal=nodes[i]. getAttribute ('v-click');
            return _this . $methods[attrVal]. bind (_this . $data);
        }
        )();
    }
    if (node . hasAttribute ('v-model') && (node . tagName=='INPUT' || node . tagName=='TEXTAREA')) {
        node . addEventListener ('input', (function (key) {
            var attrVal=node . getAttribute ('v-model');
            _this . _binding[attrVal]. _directives . push (new Watcher ('input', node, _this, attrVal, 'value')) return function () {
                _this . $data[attrVal]=nodes[key]. value;
            }
        }
        )(i));
    }
    if (node . hasAttribute ('v-bind')) {
        var attrVal=node . getAttribute ('v-bind');
        _this . _binding[attrVal]. _directives . push (new Watcher ('text', node, _this, attrVal, 'innerHTML'))
    }
}
}
function Watcher (name, el, vm, exp, attr) {
    this . name=name;
    //指令名称，例如文本节点，该值设为"text" this . el=el;
    //指令对应的DOM元素 this . vm=vm;
    //指令所属myVue实例 this . exp=exp;
    //指令对应的值，本例如"number" this . attr=attr;
    //绑定的属性值，本例为"innerHTML" this . update ();
}
Watcher.prototype . update=function () {
    this.el[this.attr]=this.vm.$data[this . exp];
}
window.onload = function () {
    var app=new myVue ( {
        el :'#app',
        data : {
            number : 0
        },
        methods : {
            increment : function () {
                this . number ++;
            }
        }
    })
}


</script>
</html>
```

### 总结
至此梳理了一遍vue双向绑定实现，感觉对Js熟悉程度还是不够，以后还是要对原理性的东西深挖
