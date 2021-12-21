## 计算属性原理 
````js
computed:{
   sum(){
       return this.count + 1
   } 
}
````
* 1. computed对象里的每个属性都会初始化一个computed watcher
* 2. 模板中读取 {{ sum }}前，此时全局的 Dep.target是render wather，targetStack是[render watcher]
* 3. 读取 {{ sum }}时，如果发现是脏数据（dirty:true，代表需要重新计算，最开始肯定是为true的），触发计算属性sum的get求值
* 4. get求值，先执行pushTarget，即Dep.target变成了computed watcher，targetStack是[render watcher, computed watcher]
````js
// sum的get劫持
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    value = this.getter.call(vm, vm) // getter函数即为用户传入的sum函数
  } finally {
    popTarget()
  }
  return value
}
````
* 5. getter函数执行的时候，读取到了this.count，count本身是个响应式数据，所以又会触发count的get劫持  
````js
// count的get劫持 

// 在闭包中，会保留为count所定义的dep
const dep = new Dep()

// 闭包中也会保留上一次set函数所设置的 val
let val
Object.defineProperty(obj, key, {
  get: function reactiveGetter () {
    const value = val
    // Dep.target 此时就是computed watcher
    if (Dep.target) {
      // 收集依赖
      dep.depend()
    }
    return value
  },
})
````
````js
// dep.depend()
depend(){
    if (Dep.target) {
       Dep.target.addDep(this)
    }
}
````
* 6. 可以看出，count会收集computed 作为依赖, 调用Dep.target.addDep，又绕回到是调用 computed watcher的addDep函数上  
````js
addDep(dep:Dep){
    // vue内部做了一些去重的优化，这里省略掉
    this.deps.push(dep) // 把count的dep存在watcher自身的deps上
    dep.addSub(this) // 这里又把watcher自身作为参数 传回到dep的add.sub函数了
}
````
````js
class Dep{
    subs = []
    addSub(sub:Watcher){
        // computed watcher作为依赖被count的dep保存了
        this.subs.push(sub)
    }
}
````
````js
// 到了这步，sum的计算watcher 状态
{
    deps: [count的dep],
    dirty: false, // 已求值完，所以是false
    value: 1, // 1 + 1 =2
    getter: f sum(),
    lazy: true
}

// count的dep
{
    subs: [sum的计算watcher]
}
````
* 7. sum get求值结束，接下来执行popTarget，Dep.target又变更为render watcher，最后return value为2，
* 8. 但是此时sum属性的get访问是还没结束的
````js
Object.defineProperty(vm, 'sum', { 
    get() {
          // 此时函数执行到了这里, Dep.target是render watcher
          if (Dep.target) {
            watcher.depend()
          }
          return watcher.value
        }
    }
})


// watcher.depend
depend () {
  let i = this.deps.length
  while (i--) {
    this.deps[i].depend()
  }
}
````
* 9. computed watcher的deps里保存了count的dep，所以又会调用count 上的 dep.depend()，这次是把render watcher存进自身的subs里
````js
{
    subs: [ sum的计算watcher，渲染watcher ]
}
````
* 10. 最后依次触发subs里watcher的update方法
````js
// count的响应式get劫持 

// 在闭包中，会保留为count所定义的dep
const dep = new Dep()

// 闭包中也会保留上一次set函数所设置的 val
let val
Object.defineProperty(obj, key, {
  get: function reactiveGetter () {
    const value = val
    // Dep.target 此时就是computed watcher
    if (Dep.target) {
      // 收集依赖
      dep.depend()
    }
    return value
  },
  set: function reactiveSetter (newVal) {
      val = newVal
      // 触发 count 的 dep 的 notify
      dep.notify()
    }
  })
})
````
````js
class Dep {
  subs = []
  notify () {
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
/*
  1. 调用 computed watcher的update, 仅仅是把dirty属性置为true，等待下次读取计算即可
*/
update () {
  if (this.lazy) {
    this.dirty = true 
  }
}

/* 
  2. 调用watcher的update, 其实就是调用 vm._update(vm._render()) 这个函数，重新根据render函数生成的vnode去渲染视图了
*/
````