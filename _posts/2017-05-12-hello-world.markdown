---
layout: post
title:  "Vue 0.6.0 核心（observer，parse）"
date:   2017-05-12 00:12:17 +0800
categories: Vue
---

先看这一段代码，这是一个最简单的使用Vue编写的代码，在输入框输入值的时候，p标签里的值会跟着改变，这篇文件主要从源码角度来看看Vue是怎么实现这个功能的。

```
<div id="app">
    <p v-on="click:changeText">{{ message }}</p>
    <input v-model="message">
</div>
<script type="text/javascript">
    new Vue({
        el: '#app',
        scope: {
            message: 'Hello Vue.js!',
            changeText: function () {
                this.message = 'hello world!'
            }
        }
    });
</script>
```
通过浏览器打开这个运行这段代码，当页面加载完毕时，p标签中的内容为 Hello Vue.js!，输入框中的内容也是 Hello Vue.js!，说明了 html 中的 message 和 js 中 scope 对象的 message 属性互相关联了起来。当修改输入框中的值的时候，p 标签中的值也会跟着改变。

在初始化Vue的时候，其实是初始化了一个 ViewModel 对象，Vue 是 ViewModel 的别名。初始化 ViewModel 的过程很简单，就是把用户设置的属性原封不动的传入 Compiler 构造函数中，初始化一个 Compiler 对象。

在做核心工作之前，Vue做了这么几件事，初始化根元素，在这里即 id="app" 这个元素，其次把 scope 中的属性都拷贝到新创建的 ViewModel 对象上，在新创建的 Compiler 对象上设置了几个属性(bindings,dirs,observables)，用来保存即将会生成的一些对象。

准备工作做完之后，进入了第一个核心的方法：setupObserver。在这个方法中，先是创建了一个 事件发射器 对象，并将这个它设置在compiler对象上。这个 事件发射器 对象可以监听和发射事件，在这个方法中，它监听了3个事件并设置了相应的回调函数，这三个事件分别是get, set, mutate。

设置完了事件回调函数之后，就会遍历 viewModel 上所有用户设置的属性，并且为每一个属性 key 创建一个 binding 对象，将 key 的值存放在 binding.value 中，创建之后会把 binding 对象存储在 compiler.bindings 中，索引为 key。接下来通过 Object.defineProperty 这个函数，在 ViewModel 上设置 key 的 getter/setter 函数，设置完这些函数之后，对 vm.key 的读和写将会执行对应的 getter 函数和 setter 函数。上面 setupObserver 中设置了事件的回调函数，那这个事件是在哪些地方发射的呢？set 事件是在这里的 setter 函数中触发的。即要修改 vm 上的某个属性的值时，真正执行的函数是 setupObserver 中设置的回调函数。这里的逻辑理清楚之后，接下来要分析的就是，什么时候会去修改 vm 上面对应属性的值？

当所有的属性都创建好对应的 binding 之后，就开始对解析 DOM 元素了。在解析 DOM 元素的过程中，会创建一些必要的 binding，并且将 binding 和 directive 互相绑定。就拿上面的 demo 来举例，它会先执行 compiler.compile 方法来解析 id="app" 的元素，解析完该元素之后就会递归的解析它的子元素节点。这里的解析主要就是解析元素上面的属性，比如：
> <p v-on="click:changeText">{{ message }}</p>

这个 dom 中的 v-on="click:changeText" 就会被解析，通过 Directive.parse 来处理。parse 方法中，会拿到 on 这个字符串，然后去指令库中去寻找，是不是存在 on 这个指令，如果存在则拿到该值，不存在就会报错。拿到该值之后便会通过 new Directive 创建一个对象，该对象的 key 是 changeText，然后返回 Directive 对象。拿到这个 Directive 之后，会进行一步很关键的操作，执行 bindDirective 方法，参数是刚才拿到那个 Directive 对象。

在 bindDirective 中，会根据 Directive 的 key 在 bindings 中寻找，是否有对应的 binding，这里就会判断是否存在 key 为 changeText 的 binding，如果没有则创建一个新的 binding。然后通过

> binding.instances.push(directive)

> directive.binding = binding

这两行代码，将 binding 和 directive 互相关联起来。关联完毕之后会进行初始化动作，执行 Directive 的 bind 方法和 update 方法，update 方法的的参数为 binding 的 value，这里 binding 的 key 为 changeText，它的 value 就是用户设置的函数 this.message = 'hello world!'。在 update 方法中，会进行最后的事件绑定，就是将之前解析出来的 p 标签、changeText函数通过事件绑定起来。（不同的 Directive 具有不同的 bind 方法 和 update 方法）
```
// directives/on.js 
// 这里 event = 'click', 
// handler = function changeText(){...}
this.handler = function (e) {
    e.el = e.currentTarget
    e.vm = vm
    if (compiler.repeat) {
        e.item = vm[compiler.repeatPrefix]
    }
    handler.call(ownerVM, e)
}
this.el.addEventListener(event, this.handler)
```
在绑定事件回调函数的时候，使用了 Function.call 来保存 this 指针，这样在执行 this.message = 'hello world' 的时候，修改的是 viewModel 对象上 message 的值。在前面分析过，当修改 ViewModel 对象上的值的时候，会触发 defineProperty 设置的 setter 函数，在 setter 函数中发射了一个 set 事件，最终导致 setupObserver 中的回调函数执行了。
```
observer.on('set', function (key, val) {
    observer.emit('change:' + key, val)
    check(key)
    bindings[key].update(val)
})
```

在 set 事件的回调函数中，会执行 bindings[key].update(val) 方法，这个方法会首先更新自身存储的值，其次将所有与它相关的 directive 更新一遍，在前面解析 DOM 的时候，会将依赖 binding 的 directive 存储在 binding.instances 数组中，所以只要遍历整个数组，就可以更新页面上所有依赖 binding 的地方。

还有一处地方需要解析，那就是
> <input v-model="message">

解析流程和上面一样，解析属性拿到 v-model="message"，然后在 vm 上查找是否有 message 的属性，然后把 binding 和 directive 关联起来，然后执行 model 指令的 bind 方法和 update 方法。

在 bind 方法中会检查元素类型，因为 v-model 只能使用在部分表单元素上，如：input，checkbox等。这里根据不同的元素类型绑定不同的事件，这里是 input 标签，所以绑定了 input 事件：
```
self.event = (self.compiler.options.lazy ||
    el.tagName === 'SELECT' ||
    type === 'checkbox' ||
    type === 'radio')
        ? 'change'
        : 'input'
self.set = function () {
    // no filters, don't let it trigger update()
    self.lock = true
    self.vm.$set(self.key, el[attr])
    self.lock = false
}
el.addEventListener(self.event, self.set)
```
这个事件处理函数，就是把用户最新输入的值，更新到 ViewModel 上对应的 key 上，这会触发 setter 函数，后面就会触发所有的 directive 更新它们的值。

