## vm.$watch
### 用法
用于观察一个表达式或computed函数在Vue.js实例上的变化，回调函数触发的时候，会从参数得到 new value和 old value  
表达式只接受以点分割的路径，例如a.b.c，如果是一个比较复杂的表达式，可以用函数代替表达式
````js
vm.$watch('a.b.c',function(newVal, oldVal){
    ......
})

// vm.$watch 返回一个取消观察函数，用来停止触发回调
var unwatch = vm.$watch('a', (newVal,oldVal)=>{})
// 之后取消观察
unwatch()
````
此外还有两个选项，**deep**和**immediate**
### deep 发现对象内部值的变化
````js
vm.$watch('someObject',callback,{
    deep:true
})
vm.someObject.nestedValue = 123 // 回调函数将被触发
````
监听数组的变动不需要这么做
### immediate 立即以表达式的当前值触发回调
````js
vm.$watch('a',callback,{
    immediate: true
})
// 立即以 'a'的当前值触发回调
````
### watch的内部原理
vm.$watch其实是对Watcher的一种封装，通过Watcher完全可以使用vm.$watch的功能，但vm.$watch中的参数deep和immeriate是Watcher中所没有的
````js
Vue.prototype.$watch = function(expOrFn,cb,options){
    const vm      = this
       options    = options || {}
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if(options.immediate){
        cb.call(vm, watcher.value)
    }
    return function unwatchFn(){
        watcher.teardown()
    }
}
````
先执new Watcher 来实现 vm.$watch 的基本功能，expOrFn是支持函数的，之前没有提到这点，需要对Watcher类进行一个简单的修改
````js
export default class Watcher{
    constructor(vm, expOrFn, cb){
        this.vm = vm
        // expOrFn 参数支持函数
        if(typeof expOrFn === 'function'){
            this.getter = expOrFn
        } else {

            /*
              使用parsePath 函数来读取 keyPath中的数据，这里keypath指的是属性路径，
              例如 a.b.c.d 是一个keypath, 说明从vm.a.b.c.d中读取数据
            */
            this.getter = parsePath(expOrFn)
        }
        this.cb = cb
        this.value = this.get()
    }
    ......
}
````
当expOrFn是函数时，它不只可以动态返回数据，其中读取的所有数据也都会被Watcher观察，也就是说，如果函数从Vue实例上读取了两个响应式数据，那么Watcher会同时观察这两个数据的变化，当其中任意一个发生变化，Watcher都会得到通知  
最后返回一个函数unwatchFn，它的作用是取消观察数据，实际上也就是执行watcher.teardown来观察数据，**本质上是把watcher实例从当前正在观察的状态的依赖列表中移除**  

**unwatch的功能如何实现？**
首先，需要在Watcher中记录自己都订阅了谁，即watcher实例被收集进了哪些Dep里，然后当Watcher不想继续订阅这些Dep时，循环自己记录的订阅列表来通知它们（Dep)将自己从它们(Dep) 的依赖列表中移除掉
````js
// 需要在Watcher中添加addDep方法，该方法的作用是在Watcher中记录自己都订阅过哪些Dep
export default class Watcher{
    constructor(vm,expOrFn, cb){
       this.vm = vm
       this.deps = []  // 新增
       this.depIds = new Set() // 新增
       this.getter = parsePath(expOrFn)
       this.cb = cb
       this.value = this.get()
    }
    .....
    addDep(dep){
        const id = dep.id
        if(!this.depIds.has(id)){
            this.depIds.add(id)
            this.deps.push(dep)
            dep.addSub(this)
        }
    }
}
````
我们使用depIds来判断当前Watcher已经订阅了Dep，就不会重复订阅，接着执行 this.depIds.add 来记录当前Watcher 已经订阅了这个Dep，然后执行 this.deps.push(dep) 记录自己都订阅了哪些Dep，最后触发，dep.addSub(this) 来将自己订阅到Dep中  
在Watcher中新增addDep方法后，Dep中收集依赖的逻辑也需要有所改变
````js
let uid = 0 // 新增

export default class Dep{
    constructor(){
        this.id   = uid++ // 新增
        this.subs = []
    }
    ...
    depend(){
        if(window.target){
           // this.addSub(window.target) 废弃
           window.target.addDep(this) // 新增
        }
    }
}
````

 





