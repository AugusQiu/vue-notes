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
    ......
}
````
### Watcher 与 Dep的关系
如果Wacher中的expOrFn参数是一个表达式，那么肯定只收集一个Dep，大部分时候都是这样；但是expOrFn是一个函数，若该函数中使用了多个数据，那么这时Watcher就要收集多个Dep了
````js
this.$watch(function(){
    return this.name + this.age
},function(newValue, oldValue){
    console.log(newValue,oldValue)
})
````
上例，我们的表达式是一个函数，并且在函数中访问了name和age两个数据，这种情况下Watcher内部会收集两个Dep：name的Dep和age的Dep，同时这两个Dep中也会收集Watcher, 所以age和name中的任意一个数据发生变化时，Watcher都会收到通知  
当我们已经在Watcher中记录自己都订阅了哪些Dep之后，就可以在Watcher中新增teardown方法来通知这些订阅的Dep，让它们把自己从依赖列表中移除掉
````js
/*
  从所有依赖项的Dep列表中将自己移除
*/
teardown(){
    let i = this.deps.length
    while(i--){
        this.deps[i].removeSub(this)
    }
}

export default class Dep{
    ...
    removeSub(sub){
        const index = this.subs.indexOf(sub)
        if(index > -1){
            return this.subs.splice(index,1)
        }
    }
    ...
}
````
### deep参数的实现原理
收集依赖和触发依赖，Watcher想监听某个数据，就会触发某个数据收集依赖的逻辑，将自己收集进去，然后当它发生变化时，就会通知Watcher  
**deep原理要把当前监听的这个值在内的所有子值都触发一遍依赖收集**
````js
export default class Watcher{
    constructor(vm,expOrFn,cb,options){
        this.vm = vm
        // 新增
        if(options){
            this.deep = !!options.deep
        }else{
            this.deep = false
        }

        this.deps = []
        this.depIds = new Set()
        this.getter = parsePath(expOrFn)
        this.cb = cb
        this.value = this.get()
    }

    get(){
        window.target = this
        let value = this.getter.call(vm)

        // 新增
        if(this.deep){
            traverse(value)
        }
        window.target = undefined
        return value
    }
    ......
}
````
>注意：一定要在window.target = undefined 之前去触发子值的收集依赖逻辑，这样才能保证子集收集的依赖是当前这个Watcher

````js
// 递归value的所有子值来触发它们收集依赖的功能
const  seenObjects = new Set()
export function traverse(val){
   _traverse(val,seenObjects)
   seenObjects.clear()
}

function _traverse(val,seen){
    let i, keys
    const isA = Array.isArray(val)
    if((!isA && !isObject(val)) || Object.isFrozen(val)){
        return
    }

    if(val.__ob__){
        const depId = val.__ob__.dep.id
        if(seen.has(depId)){
            return
        }
        seen.add(depId)
    }

    if(isA){
        i = val.length
        while(i--) _traverse(val[i],seen)
    }else{
        keys = Object.keys(val)
        i = keys.length
        while(i--) _traverse(val[keys[i]],seen)
    }
}
````
## vm.$set
用法：在object上设置一个属性，如果object是响应式的，vue.js会保证属性被创建后也是响应式的，并且触发视图更新，**这个方法主要是用来避开Vue.js不能侦测属性被添加的限制（只有已经存在的属性变化会被追踪到，新增的属性无法被追踪到），在ES6之前，js并没有提供元编程的能力，所以无法侦测object什么时候被添加了一个新属性**  
````js
// 使用vm.$set就可以为object新增属性，然后Vue.js就可以将这个新增属性转换成响应式的
var vm = new Vue({
    el: '#el',
    template: '#demo-template',
    data:{
        obj:{}
    },
    methods:{
        action(){
            this.obj.name = 'qgq'
        }
    }
})
// 当action方法被调用时，会为obj新增一个name属性，vue.js并不会得到任何通知，新增的这个属性也不是响应式的，vue.js根本不知道这个obj新增了属性，就像Vue.js无法知道我们使用array.length = 0 清空了数组一样
````
### Array的处理
````js
export function set(target,key,val){
    if(Array.isArray(target) && isValidArrayIndex(key)){
        target.length = Math.max(target.length,key)
        target.splice(key,1,val)
        return val
    }
}

// 通过splice方法把val设置到target中的指定位置，触发数组拦截器，从而自动帮我们把这个新增的val转换成响应式的
````


