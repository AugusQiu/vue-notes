## setup()函数
vue3新增的、十分关键的生命周期函数，Composition API新特性的统一入口，vue3中，**定义methods、watch、computed、data数据等都放在了setup()函数中**
### 执行时机
setup()执行前于beforeCreate
### 接收参数
````js
setup(props,context){
    console.log(props.name)  // 组件接收到的props，通过第一个参数获取
    console.log(context) // 第二个参数context来访问vue的实例this
    console.log(this) // 直接打印this, 是undefined
}
````
````js
export default{
    props:{
        title:String
    },
    setup(props){
        console.log(props.title)
    }
}

/*
官网注解：props是响应式的，不能使用ES6解构，会消除props的响应性
首先，props和data、computed在vue都是响应式(Object.defineProperty setter/getter)，props父组件传递过来的数据，适用于树状关系;
data组件内部私有
*/
````
````js
const obj = {
    count:1
}
const proxy = new Proxy(obj,{
    get(target,key,receiver){
        console.log('get...')
        return Reflect.get(target,key,receiver)
    },
    set(target,key,value,receiver){
        console.log('set...')
        return Reflect.set(target,key,value,receiver)
    }
})

console.log(proxy) //  Proxy { count:1 }
console.log(proxy.count) // get...  1

proxy.count = 2  // set... 2
let { count } = proxy // get...  (相当于 let count = proxy.count)
count = 2  // 没有打印set... 失去响应
````
>结论：**解构赋值直接跳过了proxy get/set的代理，直接copy了值
实际上ES6的解构赋值对于引用类型，是浅拷贝，基本类型是深拷贝**
### 为什么用setup()，而不是之前组件的选项？
data、computed、methods、watch等选项在大多数情况下组织逻辑都有效，但是当我们的组件变得很大时，逻辑书写的列表会变得巨长，代码阅读困难，通过setup()可以抽离出函数
