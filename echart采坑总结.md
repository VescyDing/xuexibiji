Echarts作为一款流行全球的前度数据可视化工具，其基本使用原理非常简单，即`做好数据，做好容器，用数据初始化echarts canvas图形，放入容器`，简直就跟`如何把大象放入冰箱？1.打开冰箱,2.把大象放进去,3.关闭冰箱`一样简单。但是实际使用是否真的有那么简单呢？其实看使用场景。

如果你在没有任何约束的纯HTML页面中书写Echarts，那么其实与上述过程其实并无二致。但是如果你是在系统化的框架中应用Echarts，那么将大象放入冰箱的过程将会更加的规范化，其中避免不了一些坑。~~当然这些坑的本质原因还是自身对框架的理解不足，而不是Echarts的！~~

当在vue中使用Echarts时，你要做的第一步是异步请求数据，等待请求完成，处理拼装数据。如何拼装视情况而定，但是要对Echarts约定的数据结构严格遵守。这是最基本的。

之后就是准备容器，初始化，放入容器。在vue中，如果你恰好用到了v-if来控制页面视图，那么很不幸，如果这个页面视图一开始恰好是不显示**即不存在**的，你准备的dom容器是获取不到的。而这个时候，你可能会想到切换成v-show，可是这里仍然有一个问题。按照vue官方文档的说明：

>## [`v-show`](https://cn.vuejs.org/v2/guide/conditional.html#v-show)
>
>另一个用于根据条件展示元素的选项是 `v-show` 指令。用法大致一样：
>
>```
><h1 v-show="ok">Hello!</h1>
>```
>
>不同的是带有 `v-show` 的元素始终会被渲染并保留在 DOM 中。`v-show` 只是简单地切换元素的 CSS 属性 `display`。

虽然说dom元素确确实实的被保留在了document中，但是在display的状态下，dom元素是没有宽高的。你调用的`vue.$echarts.init() `方法是获取不到正确的容器大小，无论你怎样更改css样式，你最后得到的Echarts视图大小只会是100px*100px（默认大小）。这个时候就涉及到一个平时不是很常用的函数（至少菜如我之前没有用到过）:

> # [vue nexttick的理解和使用场景](https://www.cnblogs.com/fozero/p/10863667.html)
>
> ### 应用场景
>
> 需要在视图更新之后，基于新的视图进行操作
>
> ### 文档说明
>
> 在下次 DOM 更新循环结束之后执行延迟回调。在修改数据之后立即使用这个方法，获取更新后的 DOM
>
> ### nextTick原理
>
> 1、异步说明
> Vue 实现响应式并不是数据发生变化之后 DOM 立即变化，而是按一定的策略进行 DOM 的更新
> 2、事件循环说明
> 简单来说，Vue 在修改数据后，视图不会立刻更新，而是等同一事件循环中的所有数据变化完成之后，再统一进行视图更新。
>
> ### created、mounted
>
> 在 created 和 mounted 阶段，如果需要操作渲染后的试图，也要使用 nextTick 方法。
> 注意 mounted 不会承诺所有的子组件也都一起被挂载。如果你希望等到整个视图都渲染完毕，可以用 vm.$nextTick 替换掉 mounted

只需要保证，你的`vue.$echarts.init()`方法是在此函数的回调中执行，那么这个时候就能在正确的钩子时机上得到正确的容器大小。这样做还有一个好处，就是如果你页面的所有dom（包括你给的Echarts容器）大小都是按窗口大小计算出来时，只需要把整个渲染流程封装放入vue的`watch`进行监控，就可以实现容器大小随动。



接下来就说说代码优化。

我最开时的init方法大概写成是这样：

```javascript
let sysResPie = this.$echarts.init(this.$refs.sysResPie) 
sysResPie.setOption({
	...//各种option
})
sysResPie.resize()

let sysResStaPie = this.$echarts.init(this.$refs.sysResStaPie) 
sysResStaPie.setOption({
	...//各种option
})
sysResStaPie.resize()

let sysUserCalc = this.$echarts.init(this.$refs.sysUserCalc) 
sysUserCalc.setOption({
	...//各种option
})
	//后面还有很多这种结构的代码，这里为了优化代码结构，可以做的一个最简单最直观的操作就是抽离。可是抽离的话，其实也并不好抽离。
```

我们其实可以比较优雅的写成这样：

```javascript
const chartLoading = (echart, dom, option)=>{
                echart = this.$echarts.init(dom)
                echart.setOption(option)
                echart.resize()
            }

```

这样我们调用的时候就可以这样：

```javascript
let sysResPie;
let sysResStaPie;
let sysUserCalc;

this.chartLoading(sysResPie, this.$refs.sysResPie, {...options})
this.chartLoading(sysResStaPie, this.$refs.sysResStaPie, {...options})
this.chartLoading(sysUserCalc, this.$refs.sysUserCalc, {...options})

	...

// 当然，目前写的是静态数据，之后通过接口获取option肯定要写成一个对象。
// 这个地方只是最最最基础的封装，本来我想把创建对象和查询dom也一起写入，弄成工厂模式，但是还是略显复杂，等我能力到了再说吧。
```

