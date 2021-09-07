# 实现一个微前端框架

主要功能：
- 如何进行路由劫持
- 如何渲染子应用
- 如何实现JS沙箱及样式隔离
- 提升体验性的功能

example：
- 主应用react
- 子应用vue

## 应用注册

在主应用主注册子应用的信息：
- name：子应用名称
- entry：子应用的资源入口
- container：主应用渲染子应用的节点
- activeRule：在哪些路由下渲染该子应用

其实这些信息和我们在项目中注册路由很像，entry 可以看做需要渲染的组件，container 可以看做路由渲染的节点，activeRule 可以看做如何匹配路由的规则。

## 路由劫持

监听路由的变化来判断渲染哪个子应用。

在SPA架构下，路由变化是不会引发页面刷新的，因此我们需要一个方式知晓路由的变化，从而判断是否需要切换子应用或者什么事都不干。

路由变化会涉及到两个事件：
- popstate
- hashchange

因此这两个事件我们是需要去监听的。除此之外，调用pushState以及replaceState也会造成路由变化，但不会触发事件，因此我们还需要去重写这两个函数。

主要功能总结：
- 重写pushState以及replaceState方法，在方法中调用原有方法后执行如何处理子应用的逻辑
- 监听hashchange以及popstate事件，事件触发后执行如何处理子应用的逻辑
- 重写监听/移除事件函数，如果应用监听了hashchange以及popstate事件就将回调函数保存起来以备后用。

## 应用生命周期

处理子应用加载资源以及挂载和卸载子应用。组件也同样需要处理这些事情，并且会暴露相应的生命周期给用户去干想干的事。

因此对于一个子应用来说，我们需要实现一套生命周期，既然子应用有生命周期，主应用肯定也有，而且也必然是相对应子应用生命周期的。

对主应用来说，分为一下三个生命周期：（当然如果想增加生命周期也是完全没有问题的。）
- beforeLoad：挂载子应用前
- mounted：挂载子应用后
- unmounted：卸载子应用

对子应用来说，通常也分为以下三个生命周期：
- bootstrap：首次应用加载触发，常用于配置子应用全局信息
- mount：应用挂载时触发，常用于渲染子应用
- unmount：应用卸载时触发，常用于销毁子应用

主要功能总结：
- 设置子应用状态，用于逻辑判断以及优化。比如说当一个应用状态为非NOT_LOADED时（每个应用初始都为NOT_LOADED状态），下次渲染该应用时就无需重复加载资源了
- 如需处理逻辑，比如说beforeLoad我们需要加载子应用资源
- 执行主/子应用生命周期，这里需要注意下执行顺序，可以参考父子组件的生命周期执行顺序

## 完美路由劫持

- 判断当前URL与之前的URL是否一致，如果一致则继续
- 利用当前URL去匹配相应的子应用，此时分为几种情况
  - 初次启动微前端，此时只需渲染匹配成功的子应用
  - 未切换子应用，此时无需处理子应用
  - 切换子应用，此时需要找出之前渲染过的子应用做卸载处理，然后渲染匹配成功的子应用
- 保存当前URL，用于下一次第一步判断

路由匹配的原则主要由两块组成：
- 嵌套关系
- 路径语法

path-to-regexp

```js
const fn = match('/user/:id', { decode: decodeURIComponent });
fn('/user/123');  // => { path: '/user/123', index: 0, params: { id: '123' } }
fn('/invalid'); // => false
fn('/user/caf%C3%A9');  // => { path: '/user/caf%C3%A9', index: 0, params: { id: 'café' } }
```

## 完善声明周期

加载子应用资源：回到 registerMicroApps 函数，我们最开始就给这个函数传入了 entry 参数，这就是子应用的资源入口。

资源入口其实分两种方案：
- JS Entry
- HTML Entry

JS Entry是single-spa中使用的一个方式。但是它限制有点多，需要用户将所有文件打包在一起，除非你的项目对性能无感，否则基本可以pass这个方案。
HTML Entry则要好得多，毕竟所有网站都是以HTML作为入口文件的。在这种方案里，我们基本无需改动打包方式，对用户开发几乎没有侵入性，只需要找出HTML中的静态资源加载并运行即可渲染子应用了，因此我们选择了这个这个方案。

### 加载资源

首先我们需要获取HTML的内容，这里我们只需调用原生 fetch 就能拿到东西了。

文件中有很多相对路径的静态资源URL，这些资源只有在自己的BaseURL下才能被正确加载到，如果是在主应用的BaseURL下肯定报404错误了。

我们需要先处理这些资源的路径，将相对路径拼接成正确的绝对路径，然后再去fetch

然后我们还需要注意一点：因为我们是在主应用的URL下加载子应用的资源，这很有可能会触发跨域的限制。因此在开发及生产环境大家务必注意跨域的处理。

import-html-entry

### 运行JS

- eval(js string)
- new Function(js string)()

注册子应用的时候设置了一个name属性，这个属性其实很重要，在之后的场景也会用到。给自已用设置name的时候别忘了还需要略微改动下打包的配置，将其中一个选项也设置为同样内容。

```js
// vue.config.js
module.exports = {
  configureWebpack: {
    output: {
      // 和主应用中注册的name一样
      library: `vue`
    },
  },
}

// 这样配置后，我们就能通过window.vue访问到应用的JS入口文件export出来的内容。

window.vue
// module {
//   bootstrap: f,
//   mount: f,
//   unmount: f,
// }

// 导出的这些函数都是这些子应用的生命周期，我们需要拿到这些函数去调用。
```

### JS沙箱

防止子应用直接修改window上的属性，又要访问window上的内容。

- 快照
- Proxy

快照：在挂载子应用前记录下当前window上的所有内容，然后接下来就随便让子应用去玩了，直到卸载子应用时回复挂载前的window即可，这种方案实现容易，唯一缺点就是性能慢点，qiankun是这种方案。

Proxy。

核心思路就是创建一个假的window出来，如果用户设置值的话就设置在fakeWindow上，这样就不会影响全局变量了。如果用户取值的话，就判断属性是存在于fakeWindow上还是window上。

一般快照和Proxy沙箱都是需要的，无非前者是后者的降级方案，毕竟不是所有浏览器都支持Proxy。

## 改善型功能

### prefetch

我们目前的做法是匹配一个子应用成功后才去加载子应用，这种方式其实不够高效。我们更希望用户在浏览当前子应用的时候就能把别的子应用资源也加载完毕，这样用户切换应用的时候就无需等待了。

window.requestIdleCallback()方法将在浏览器的空闲时段内调用的函数排队。这使开发者能够在主事件循环上执行后台和低优先级工作，而不会影响延迟关键事件，如动画和输入响应。

### 资源缓存机制

搞一个对象缓存下每次请求下来的文件内容，下次请求的时候先判断对象中存不存在值，存在的话直接拿出来用就行。

### 全局通信及状态

全局通信及状态实际上完全都可以看做是发布订阅模式的一种实现。