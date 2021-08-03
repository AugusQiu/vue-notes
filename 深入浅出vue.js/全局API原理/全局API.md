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


