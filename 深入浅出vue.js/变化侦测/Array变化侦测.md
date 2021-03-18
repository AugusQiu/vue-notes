## Array的变化侦测
````js
this.list.push(1)

/*
  我们使用push方法向list中新增了数字1
  对于数组，用push方法改变了数组，并不会触发getter/setter
*/
````
Object的变化是靠setter来追踪的，只要属性发生变化，就会触发setter  
顺着这个思路，当用push来改变数组的内容，是不是只要在用户使用push操作数组的时候得到通知，就能达到同样的目的   
然而，在ES6之前，js并没有提供拦截原型方法的能力，尤大就**重写了数组的那些方法，用自定义的方法去覆盖原生的原型方法**  
<img src="../images/shuzulanjie.png" width="400" height="240">

如图，我们可以用一个拦截器覆盖Array.prototype, 之后，当**使用Array原型上的方法操作数组时，其实执行的都是拦截器中提供的方法，然后再在拦截器中使用原生的数组原型方法去操作数组**
### 拦截器
拦截器就是用来帮我们追踪到Array的变化  
拦截器就是一个和Array.prototype几乎一样的Object, 差别在于其中，某些可以改变数组自身内容的方法我们处理过  
在Array原型中可以改变数组自身内容的方法有7个：push、pop、shift、unshift、splice、sort、reverse
````js
const arrayProto = Array.prototype

// arrayMethods.__proto__ = Array.prototype 相当于创建了个数组实例
export const arrayMethods = Object.create(arrayProto)

['push','pop','shift','unshift','splice','sort','reverse'].forEach(function(method){
    const original = arrayProto[method] // 缓存原始方法
    Object.defineProperty(arrayMethods,method,{
       value: function mutator(...args){
          return original.apply(this,args)
       },
       enumerable: false,
       writable: true,
       configurable: true
    })
})


/*
  变量arrayMethods, 它继承自Array.prototype，具备其所有功能，未来，我们要用其去覆盖Array.prototype
  在arrayMethods上使用 Object.defineProperty方法将那些可以改变数组自身内容的方法进行封装
  所以，当使用push方法的时候，其实调用的是arrayMethods.push, 而arrayMethods.push就是函数mutator, 实际上执行的也就是mutator函数  
  最后，在mutator中执行original(它是原生Array.prototype上的方法，比如Array.prototype.push)来做它该做的(push功能)
  当然，我们可以再对这个mutator函数中做一些其他的事，比如说发送变化通知
*/
````
## 使用拦截器覆盖Array原型
有了拦截器，想要让它生效，就需要使用它去覆盖Array.prototype，但是我们又不能直接覆盖，因为这样会污染全局的Array  
我们只是希望拦截操作只针对那些被侦测了变化的数据生效，也就是说**期望拦截器只是覆盖那些响应式数据的原型**  
而将一个数据转换成响应式的，需要通过Observer
````js
export class Observer{
    constructor(value){
        this.value = value
        if(Array.isArray(value)){
            value.__proto__ = arrayMethods
        }else{
            this.walk(value)
        }
    }
}
````
<img src="../images/shuzulanjie2.png" width="400" height="280">

### 如何收集依赖
目前只有一个拦截器，但还是什么事情都做不了，**我们创建拦截器，是为了得到一种能力，一种当数组的内容发生变化时得到通知的能力**  
现在，我们利用拦截器知道了数组内容何时发生改变，但是又该去通知谁呢？前面介绍Object时，答案肯定是通知Dep中的Watcher, 那要如何收集数组的依赖？  
> Object的依赖，是在defineReactive中的getter里使用Dep收集的，每个key都会有一个对应的Dep列表来存储依赖

其实**数组也是在getter中收集依赖**

### 依赖列表存在哪儿
````js
export class Observer{
    constructor(value){
        this.value = value
        this.dep = new Dep() // 新增dep

        /*
           为什么数组的dep(依赖)要保存在Observer实例上呢？
           数组在getter中收集依赖，在拦截器中触发依赖，这个依赖保存的位置就很关键，它必须在getter和拦截器中都可以访问到
        */

        if(Array.isArray(value)){
            value.__proto__ = arrayMethods
        }else{
            this.walk(value) // 是对象，则递归调用defineReactive()方法，在defineReact方法里为每个key创建对应的dep列表
        }
    }
}
````
### 收集依赖
把Dep实例保存在Observer的属性上之后，我们可以在getter中像下面这样访问并收集依赖
````js
function defineReactive(data,key,val) {
   let childOb = observe(val) //新增
   let dep = new Dep()
   Object.defineProperty(data,key,{
       enumerable: true,
       configurable: true,
       get: function(){
          dep.depend()
           
          // 新增
          if(childOb){
              childOb.dep.depend()
          }
        
          return val
       },
       set: function(newVal){
          if(val === newVal) return
           dep.notify()
           val = newVal
       }
   })
}



/*
  尝试为value创建一个Observer实例
  如果创建成功，直接返回新创建的Observer实例
  如果value已经存在一个Observer实例，则直接返回它
*/
export function observe(value, asRootData){
    if(!isObject(value)){
        return
    }

    let ob
    if(hasOwn(value, '__ob__') && value.__ob__ instanceof Observer){
        ob = value.__ob__
    } else {

        /*
         Observer类的构造函数对于数组、对象做了区分处理
        */
        ob = new Observer(value) 
    }
    return ob
}
````
新增了函数observe, 它尝试创建一个Observer实例，如果value已经是响应式数据，不需要再次创建Observer实例，直接返回已经创建的Observer实例即可，避免了重复侦测value变化的问题  
在defineReactive函数中调用了observe，它把val当作参数传了进去并拿到一个返回值，那就是数组的Observer实例 （childOb) , 最后通过childOb的dep执行depend方法来收集依赖  
### 在拦截器中获取Observer实例&向数组的依赖发送通知
因为Array拦截器是对原型的一种封装，所以可以在拦截器中访问到this(当前正在被操作的数组)  
而dep保存在Observer中，所以需要在this上读到Observer的实例
````js
// 工具函数
function def(obj,key,val, enumerable){
    Object.defineProperty(obj,key,{
        value: val,
        enumerable:!!enumerable,
        writable: true,
        configurable: true
    })
}

export class Observer{
    constructor(value){
        this.value = value
        this.dep = new Dep()
        def(value,'__ob__',this)  // 新增

        if(Array.isArray(value)){
            value.__proto__ = arrayMethods
        }else{
            this.walk(value)
        }
    }
}

/*
  新增的这行代码，可以在value中新增一个不可枚举的属性__ob__,这个属性的值就是当前Observer的实例  
  这样我们就可以通过数组数据的 __ob__属性拿到 Observer实例，然后就可以拿到 __ob__ 上的dep  
  __ob__的作用不但可以在拦截器中 访问Observer实例这么简单，当然也可以用来标记当前value 是否已经被Observer转换成了响应式的数据
  也就是说，所有被侦测了变化的数据身上都会有 一个__ob__属性来表示它们是响应式的  
  当value身上被标记了 __ob__之后，就可以通过value.__ob__来访问Observer实例，如果是Array拦截器，因为拦截器是原型方法，所以可以直接通过 this.__ob__来访问Observer实例
*/
````
````js
[
 'push',
 'pop',
 'shift',
 'unshift',
 'splice',
 'sort',
 'reverse'
].forEach(function(method){
    // 缓存原始方法
    const original = arrayProto[method]
    Object.defineProperty(arrayMethods,method,{
       def(arrayMethods,method,function mutator(...args){
           const result = original.apply(this.args)
           const ob  = this.__ob__
           ob.dep.notify() //想依赖发送消息
           return result
    })
})
````
### 侦测数组中元素的变化
前面侦测数组的变化，指的是数组自身的变化，比如是否新增、删除一个元素  
另外一些变化也是需要侦测的，比如当数组元素中的一个object身上某个属性的值发生变化，也需要发送通知  
此外，如果用户使用了push往数组中新增元素，这个新增元素的变化也需要侦测  
>Observer的作用就是将Object的所有属性转换为getter/setter的形式来侦测变化，现在Observer类不光能处理Object类型的数据，还可以处理Array类型的数据
````js
export class Observer{
    constructor(value){
        this.value = value
        def(value,'__ob__',this)

        // 新增
        if(Array.isArray(value)){
            this.observeArray(value)
        }else{
            this.walk(value)
        }
    }

    // 侦测Array中的每一项
    observeArray(items){
        for(let i=0, l=items.length;i<l;i++){
            observe(items[i])
        }
    }
    ......
}
````
### 侦测新增元素的变化
获取新增的元素，并且也使用Observer来侦测它们
### 获取新增元素
需要在拦截器中对数组方法的类型进行判断，如果操作数组的方法是push、unshift和splice(可以新增数组元素的方法)，则把参数中新增的元素拿过来，用Observer来侦测
````js
[
    'push',
    'pop',
    'shift',
    'unshift',
    'splice',
    'sort',
    'reverse'
].forEach(function(method){
    // 缓存原始方法
    const original = arrayProto[method]
    def(arrayMethods,method,function mutator(...args){
        const result = original.apply(this,args)
        const ob = this.__ob__
        let inserted
        switch(method){
            case 'push':
            case 'unshift':
               inserted = args
               break
            case 'splice':
               /*
                   splice(0,-1,2)  splice第二个参数指定为-1表示新增，第三个参数就是向数组中新增的元素2
                */
               inserted = args.slice(2) 
               break    
        }
        if(inserted) ob.observeArray(inserted)
        ob.dep.notify()
        return result
    })
})
````
从args中将新增元素取出来，暂存在inserted中，接下来，使用Observer把inserted中的元素转换成响应式的
### 使用Observer侦测新增元素
Observer会将自身的实例附加到value的__ob__属性上，**所有被侦测了变化的数据都有一个__ob__属性，数组元素也不例外** 
可以在拦截器中通过this访问到__ob__, 然后调用__ob__上的observeArray方法就可以了(代码见上面示例)

### 关于Array的问题
因为Array的变化侦测是通过拦截原型的方式实现的，有些数组操作Vue.js是拦截不到的
````js
// 根据索引项改变数组元素
this.list[0] = 2

// 设置length,置空数组
this.list.length = 0
````
上面这两种改变数组的方式，都不会触发re-render或watch

### 小结
* Array追踪变化的方式和Object不同，因为它是通过数组方法来改变内容，所有我们通过创建拦截器去覆盖数组原型的方式来追踪变化  
* 为了不污染全局Array.prototype，我们在Observer中只针对那些需要侦测变化的数组使用__proto__来覆盖原型方法
* Array收集依赖的方式跟Object一样，都是在getter中收集，但是由于使用依赖的位置不同，数组要在拦截器中向依赖发消息，所以不能像Object那样保存在defineReactive中，而是把依赖保存在了Observer实例上  
* 在Observer中，我们对每个侦测了变化的数据都标上**__ob__**印记，两个作用：1.标记数据是否被侦测了变化（保证同一个数据只被侦测一次）2. 通过数据取到__ob__, 从而拿到Observer实例上保存的依赖，当拦截到数组发生变化，向依赖发生通知















