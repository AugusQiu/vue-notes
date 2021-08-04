initMixin、stateMixin、eventsMixin、lifecycleMixin、renderMixin这5个函数的参数是Vue构造函数
````js
// 以函数initMixin为例，当函数被调用时，就会向Vue构造函数的prototype属性添加_init方法，执行 new Vue的时候，会调用_init方法，该方法实现了一系列初始化操作，包括整个生命周期的流程以及响应式系统流程的启动等
export function initMixin(Vue){
    Vue.prototype._init = function(options){
        // ......
    }
}
````
## 数据相关的实例方法
数据相关的api有3个，vm.$watch、vm.$set、vm.$delete，它们是在stateMixin中挂载到Vue的原型上的  
````js
import {
   set,
   del
} from '../observer/index'
export function stateMixin(Vue){
    Vue.prototype.$set    = set
    Vue.prototype.$delete = del
    Vue.prototype.$watch  = function(expOrFn,cb, options){}
}
````
## 事件相关的实例方法
vm.$on、vm.$once、vm.$off、vm.$emit，这4个方法是在eventsMixin方法中挂载的 
````js
export function eventsMixin(Vue){
    Vue.prototype.$on   = function(event,fn){}
    Vue.prototype.$once = function(event,fn){}
    Vue.prototype.$off  = function(event,fn){}
    Vue.prototype.$emit = function(event,fn){}
}
````
### vm.$on
用于监听当前实例上的自定义事件，事件可以由vm.$emit触发，回调函数会接收所有传入事件所触发的函数的额外参数  
````js
// 用法示例
vm.$on('test',function(msg){
    console.log(msg)
})
vm.$emit('test','hi')

// 最常见的还是 用于父子组件通信 @符号就是vm.$on的语法糖
<child @update-data="handleUpdate">
````
````js
// 实现原理
Vue.prototype.$on = function(event,fn){
    const vm = this
    if (Array.isArray(event)) {
        for(let i = 0; i < event.length; i++){
            // 递归
            this.$on(event[i],fn)
        }
    } else {
       (vm._events[event] || (vm._events[event] = [])).push(fn)
    }
    return vm
}
````
### vm.$off
移除自定义事件监听器  
* 如果没有提供参数，就移除所有的事件监听器
* 如果只提供了事件，则移除该事件所有的监听器
* 如果同时提供了事件与回调，则只移除这个回调的监听器  
````js
Vue.prototype.$off = function(event,fn){
    const vm = this
    // 移除所有事件监听器
    if (!arguments.length){
        vm._events = Object.create(null)
        return vm
    }
    // event支持数组
    if (Array.isArray(event)){
        for(let i = 0;i < event.length; i++){
            this.$off(event[i],fn)
        }
        return vm
    }

    const cbs = vm._events[event]
    if (!cbs) return vm

    // 移除该事件的所有监听器
    if (arguments.length === 1){
        vm._events[event] = null
        return vm
    }

    // 只移除与fn相同的监听器
    if (fn){
        const cbs = vm._events[event]
        let cb
        let i = cbs.length
        // 有个细节，循环是从后向前遍历，这样在列表中移除当前位置的监听器时，不会影响列表中未遍历到的监听器的位置
        // 如果是从前向后遍历，那么当从列表中移除一个监听器时，后面的会自动向前移动一个位置，这会导致下一轮循环时跳过一个元素
        while (i--){
            cb = cbs[i]
            if (cb === fn || cb.fn === fn){
                cbs.splice(i,1)
                break
            }
        }
    }
    return vm
}
````
### vm.$once
监听一个自定义事件，但是只触发一次，在第一次触发之后就移除监听器  
实现的思路：在vm.$once中调用vm.$on来实现监听自定义事件的功能，执行完，就将监听器移除
````js
Vue.prototype.$once = function(event,fn){
    const vm = this
    function on(){
        vm.$off(event,on)
        fn.apply(vm,arguments)
    }
    on.fn = fn
    vm.$on(event,on)
    return vm
}
````
### vue.$emit
触发当前实例上的事件，附加参数传给监听器回调  
````js
Vue.prototype.$emit = function(event){
    const vm = this
    let cbs = vm._events[event]
    if (cbs) {
        // 类数组转数组，1代表参数截取的位置，也就是从第二个参数开始，把event参数去掉了
        const args = toArray(arguments,1)
        for(let i = 0, i < cbs.length; i++){
            cbs[i].apply(vm, args)
        }
    }
    return vm
}
````
### 生命周期相关的实例方法
vm.$mount、vm.$forceUpdate、vm.$nextTick和vm.$destroy  
````js
export function lifecycleMixin(Vue){
    Vue.prototype.$forceUpdate = function(){
        ....
    }
    Vue.prototype.$destroy = function(){

    }
}

export function renderMixin(Vue){
    Vue.prototype.$nextTick = function(fn){
        ....
    }
}
````
### vm.$forceUpdate
使得Vue.js 实例重新渲染，仅仅影响实例本身以及插入插槽内容的子组件，而不是所有子组件  
只要执行实例watcher的update方法，就可以让实例重新渲染，Vue.js的每一个实例都有一个watcher.之前介绍虚拟dom时提到，当状态发生变化时，会通知到组件级别。事实上，组件就是Vue.js实例  
````js
Vue.prototype.$forceUpdate = function(){
    const vm = this
    if (vm._watcher){
        vm._watcher.update()
    }
}
````
vm._watcher就是Vue.js实例的watcher, 每当组件内依赖的数据发生变化时，都会自动触发实例中_watcher的update方法  
### vm.$destroy
完全销毁一个实例，它会清理该实例与其他实例的连接，并解绑其全部指令及监听器，同时触发beforeDestroy和destroyed的钩子函数  
````js
Vue.prototype.$destroy = function(){
    const vm = this
    // 防止反复执行，为true说明正在被销毁
    if (vm._isBeingDestroyed){
        return
    }
    callHook(vm,'beforeDestroy')
    vm._isBeingDestroyed = true

    // 删除自己与父级之间的连接
    /*
      一个组件可以被多个组件引入，为什么代码只从一个父组件的$children列表中移除了子组件？
      子组件在不同父组件中是不同的Vue.js实例，所以一个子组件实例（注意是实例）的父级只有一个
    */
    const parent = vm.$parent
    if (parent && !parent._isBeingDestroyed && !vm.$options.abstract){
        remove(parent.$children,vm)
    }
    
    // 从watcher监听的所有状态的依赖列表中移除watcher
    // _watcher是vue实例自动创建的，组件粒度，监听这个组件中用到的所有状态
    if (vm._watcher){
        vm._watcher.teardown()
    }
    // _watchers是保存用户所创建的watcher实例
    let i = vm._watchers.length
    while(i--){
      vm._watchers[i].teardown()
    }
    vm._isDestroyed = true
    
    // 在vnode树上触发destroyed钩子函数解绑指令
    vm.__patch__(vm._vnode,null)
    callHook(vm,'destroyed')
    // 移除所有的事件监听器
    vm.$off()
}
````
### vm.$nextTick
nextTick接收一个回调函数作为参数，它的作用是将回调延迟到下次DOM更新周期后执行  
使用场景：当更新了状态（数据）后，需要对新DOM做一些操作，但是这时我们还获取不到更新后的DOM，因为还没有重新渲染
````js
new Vue({
    //...
    methods:{
        //...
        update:function(){
            this.message = 'changed' // 修改数据
            this.$nextTick(function(){
                // DOM现在更新了
                ...
            })
        }
    }
})
````
**下次DOM更新周期之后执行，具体是什么时候？**  
在Vue.js中，当状态改变时，watcher会得到通知，然后触发虚拟dom的渲染流程，而watcher触发渲染这个操作是异步的，Vue.js中有一个队列，每当需要渲染时，会将watcher推送到这个队列中，在下一次事件循环时，才触发渲染  
**为什么要用异步更新队列？**
组件内所有状态的变化都会通知到同一个watcher，然后虚拟DOM会对整个组件进行diff操作。也就是说如果同一轮事件循环中 有两个数据发生了变化，那么组件的watcher会收到两份通知，从而触发两次渲染。事实上，并不需要渲染两次，粒度已经是组件级别了，虚拟DOM会对整个组件进行渲染，所以只需要等到所有状态都修改完毕后，一次性渲染最新的就是  
所以，实现方式就是：先将收到通知的watcher实例，添加到队列中缓存起来，并且在添加到队列之前检查其中是否已经存在相同的watcher，不存在才会添加。然后在下一次事件循环（event loop)中，Vue.js会让队列中的watcher触发流程并清空队列  
**什么是执行栈**  
当我们执行一个方法时，js会生成一个与该方法对应的执行环境（context)，又叫执行上下文。这个执行环境中有这个方法的私有作用域（私有作用域中定义的变量、this对象）、上层作用域的指向、参数（arguments)，这个执行环境会被添加到一个栈中，这个栈就是执行栈  
如果在这个方法的代码中，执行到了一行函数调用语句，那么js会生成这个函数的执行环境并将其添加到执行栈中，进入到这个执行栈执行代码，执行完毕从栈中销毁这个新压入的执行环境，再回到上一个方法的执行环境，这个过程反复进行，知道所有代码全部执行完毕  

**下次DOM更新周期** 就是下次微任务执行时更新DOM，vm.$nextTick其实也就是把回调添加到微任务中（只有在特殊情况下才会降级成宏任务）  
````js
// 如果是先使用vm.$nextTick注册回调，然后修改数据，则在微任务队列中因为先后顺序，就无法得到最新的DOM
new Vue({
    //...
    methods:{
        //...
        update:function(){
            // 先使用nextTick注册回调
            this.$nextTick(()=>{
                // DOM没有更新
            },0)
            // 然后修改数据，更新dom
            this.message = 'changed'
        }
    }
})

new Vue({
    //...
    methods:{
        //...
        update:function(){
            // 先使用setTimeout向宏任务中注册回调
            // 即使先注册了宏任务，但是因为微任务先执行，所以这里也能获取到更新后的DOM
            setTimeout(()=>{
                // DOM现在更新了
            },0)
            // 修改数据向微任务队列中注册回调
            this.message = 'changed'
        }
    }
})
````
vm.$nextTick和全局方法Vue.nextTick是相同的，所以nextTick的具体实现并不是在Vue原型上的$nextTick方法中，而是抽象成了nextTick方法给大家共用  
````js
import { nextTick } from '../util/index'
Vue.prototype.$nextTick = function(fn){
    return nextTick(fn,this)
}
````
````js
// 实现原理
const callbacks = []
// 标记是否已经向任务队列中添加了一个任务，每当向任务队列中 插入任务时，设为true, 每当任务被执行时，设为false
let pending = false 

function flushCallbacks(){
    pending = false
    const copies = callbacks.slice(0)
    callbacks.length = 0 // 清空原数组
    for(let i = 0; i < copies.length; i++){
        copies[i]()
    }
}

let microTimeFunc
const p = Promise.resolve()
microTimerFunc = () =>{
    p.then(flushCallbacks)
}

let useMacroTask = false
export function withMacroTask(fn){
    return fn._withTask || (fn._withTask = function(){
        useMacroTask = true
        const  res = fn.apply(null,arguments)
        useMacroTask = false
        return res
    })
}

export function nextTick(cb,ctx){
    callbacks.push(()=>{
        if(cb){
            cb.call(ctx)
        }
    })

    if(!pending){
        pending = true
        if (useMacroTask) {
            macroTimerFunc()
        } else {
            microTimerFunc()
        }
    }
}
````
### vm.$mount
这个方法不常用，如果在实例化Vue.js时设置了el选项，就会自动把Vue实例挂载到DOM元素上。无论没有设置el选项，那么它处于“未挂载”状态，没有关联的DOM元素，就可以使用vm.$mount手动挂载一个未挂载的实例  
````js
var MyComponent = Vue.extend({
    template: '<div>hello!!!</div>'
})

// 创建并挂载到#app(会替换#app)
new MyComponent({ el:'#app' })

// 创建并挂载到#app(会替换#app)
new MyComponent().$mount('#app')

// 或者，在文档之外渲染并且随后挂载
var component = new MyComponent().$mount()
document.getElementById('app').appendChild(component.$el)
````