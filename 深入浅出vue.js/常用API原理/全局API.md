### Vue.extend(options)
使用基础Vue构造器创建一个“子类”
````js
<div id = "mount-point"></div>

// 创建构造器
var Profile = Vue.extend({
    template: '<p>{{firstName}} {{ lastName }}</p>',
    // 注意，这里data必须是函数
    data: function(){
        return{
            firstName:'Qiu',
            lastName: 'augus',
        }
    }
})

// 创建Profile实例，并挂载到一个元素上
new Profile().$mount('#mount-point')
````
全局API和实例方法不同，后者是在Vue的原型上挂载方法，也就是在Vue.prototype上挂载方法，而前者是直接在Vue上挂载方法
````js
Vue.extend = function(extendOptions){
    ......
}
````
出于性能考虑，在Vue.extend方法内首先增加了缓存策略。反复调用Vue.extend其实应该返回同一个结果，只要返回结果是固定的，就可以将计算结果缓存，再次调用extend方法，只需要从缓存中取出结果即可  
````js
let cid = 1
Vue.extend = function(extendOptions){
    extendOptions = extendOptions || {}
    const Super = this
    const SuperId = Super.cid
    const cachedCtors = extendOptions._Ctor || ( extendOptions._Ctor = {} )
    if(cachedCtors[SuperId]){
        return cachedCtors[SuperId]
    }

    const Sub  = function VueComponent(options){
        this._init(options)
    }

    // 缓存构造函数
    cachedCtors[SuperId] = Sub
    return Sub 
}

// 代码中使用父类的id作为缓存的key，将子类缓存在cachedCtors中，代码最后返回Sub，但是此时它还不具备Vue的能力，因为没有写继承的逻辑
````
````js
// 继承原型的各个实例方法：$nextTick、$on、$emit...
Sub.prototype = Object.create(Super.prototype)
Sub.prototype.constructor = Sub
Sub.cid = cid++

// 合并父类选项和子类选项的逻辑，并将父类添加到子类的super属性中
Sub.options = mergeOptions(
    Super.options,
    extendOptions
)
Sub['super'] = Super

// 如果选项中存在props属性，则初始化，初始化props的作用是将key代理到_props中，例如 vm.name 实际上可以访问到的是 Sub.prototype._props.name
if (Sub.options.props){
    initProps(Sub)
}

function initProps(Comp){
    const props = Comp.options.props
    for (const key in props){
        proxy(Comp.prototype, '_props',key)
    }
}

function proxy(target, sourceKey, key){
    sharedPropertyDefinition.get = function proxyGetter(){
        return this[sourceKey][key]
    }
    sharedPropertyDefinition.set = function proxySetter(val){
        this[sourceKey][key] = val
    }
    Object.defineProperty(target, key, sharedPropertyDefinition)
}

// 如果选项中存在computed，则对它进行初始化，将computed对象遍历一遍，并将里面的每一项都定义一遍
function initComputed(Comp){
    const computed = Comp.options.computed
    for(const key in computed){
        definedComputed(Comp.prototype,key,computed[key])
    }
}

// 最后 将父类中存在的一些属性依次复制到子类中
Sub.extend = Super.extend
Sub.mixin  = Super.mixin
Sub.use    = Super.use

// ASSET_TYPES = ['component', 'directive', 'filter']
ASSET_TYPES.forEach(function(type){
    Sub[type] = Super[type]
})
Sub.superOptions = Super.options
Sub.extendOptions = extendOptions
Sub.sealedOptions = extend({}, Sub.options)
````
### Vue.directive
注册或获取全局指令  
除了核心功能默认内置的指令：v-model、v-show之外，Vue.js也允许注册自定义指令，虽然代码复用和抽象的主要形式是组件，但是有些情况下，仍然需要对普通DOM元素进行底层操作  
需要强调的是 Vue.directive 方法的作用是注册或获取全局指令，而不是让指令生效。**区别是注册指令需要做的事是将指令保存在某个位置，而让指令生效是从某个位置拿出来执行它**
````js
// 用于保存指令的位置
Vue.options = Object.create(null)
Vue.options['directives'] = Object.create(null)

Vue.directive = function(id, definition){
    if (!definition) {
       // 获取操作
       return this.options['directives'][id]
    } else {
       // 注册操作
       if(typeof definition === 'function'){
           definition = { bind:definition, update:definition }
       }
       this.options['directives'][id] = directives
       return definition
    }
}
````
### Vue.filter
用于一些常见的文本格式化，过滤器可以用在两个地方，一个是template的{{ }} 插值，一个是v-bind表达式  
````js
{{ message | formatMsg }}

<div v-bind:id="rawId | formatId"></div>
````
### Vue.component
````js
// 注册组件，传入一个扩展过的构造器
Vue.component('my-component', Vue.extend({...}))

// 注册组件，传入一个选项对象（自动调用Vue.extend)
Vue.component('my-component', {...})

// 获取注册的组件（始终返回构造器）
var MyCompoent = Vue.component('my-component')
````
组件其实是一个构造函数，是使用Vue.extend生成的子类
````js
Vue.options['components'] = Object.create(null)

Vue.component = function(id,definition){
    if (!definition) {
        return this.options['components'][id]
    } else {
        if (isPlainObject(definition)) {
            definition.name = definition.name || id
            // 如果是纯对象，就自动调用extend方法把它变成Vue的子类
            definition = Vue.extend(definition)
        }
        this.options['components'][id] = definition
        return definition
    }
}
````
### Vue.use{plugin}
Vue.use的作用是注册插件。plugin如果是一个对象，必须提供install方法，如果是一个函数，它会被作为install方法，调用install方法，会将Vue作为参数传入。install方法被同一个插件多次调用时，插件也只会被安装一次  
````js
Vue.use = function(plugin){
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1){
        return this
    }

    // 其他参数
    const args = toArray(arguments,1)
    args.unshift(this) // this就是Vue，Vue现在是第一个参数
    if (typeof plugin.install === 'function'){
        plugin.install.apply(plugin,args)
    } else if (typeof plugin === 'function'){
        plugin.apply(null,args)
    }
    installedPlugins.push(plugin)
    return this
}
````  




