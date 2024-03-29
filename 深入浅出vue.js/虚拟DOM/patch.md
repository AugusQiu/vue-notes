虚拟dom最核心的就是patch，它将vnode渲染成真实的dom，它不是暴力覆盖原dom，而是比对新旧vnode之间有哪些不同，然后根据对比结果找出需要更新的节点进行更新，可以看出patch本身就有补丁、修补的意思  
因为dom操作的执行速度远不如js的运算速度快，我们应该最大限度地减少DOM操作
## patch介绍
对比两个vnode之间的差异只是patch的一部分，这是手段，而不是目的，patch的目的其实是修改dom节点来渲染视图  
patch的流程其实就是在做三件事：
* 创建新增的节点
* 删除已经废弃的节点
* 修改需要更新的节点
### 新增节点
* 事实上，只有那些因为状态的改变而新增的节点，在DOM中又不存在时，才需要创建一个节点并插入到DOM中，这通常会发生在首次渲染中，因为首次渲染时，DOM中不存在任何节点，所以oldVnode是不存在的  
* 当oldVnode和vnode都存在但并不是同一个节点，使用vnode创建的DOM元素替换旧的DOM元素
* 但是还有一个很常见的场景，新旧两个节点是同一个节点，这个时候需要更详细的对比操作，比如：新旧节点皆是同一个文本节点，只是文本内容变了，那我们只要重新设置oldVnode在视图中所对应的真实dom节点的文本
## 创建节点
只有三种类型的节点会被创建并插入到DOM中：元素节点、注释节点、文本节点  
* 判断vnode是否是元素节点，只需要判断它是否具有tag属性即可，接着调用createElement方法来创建真实的元素节点  
* 当一个元素节点被创建后，接着就是将它插入到指定的父节点中
* 元素渲染到视图，就是调用appendChild方法，parentNode.appendChild就是将一个元素插入到指定的父节点中  
* 元素节点通常都会有子节点(children)，创建子节点的过程就是一个递归的过程

>如果vnode的tag属性不存在，那么就是注释节点和文本节点，注释节点有个唯一的标识属性isComment，文本节点调用当前环境下的createTextNode方法，注释节点调用createComment方法  
## 删除节点
removeNode用于删除视图中的单个节点，removeVnodes用于删除一组指定的节点
````js
const nodeOps = {
    removeChild(node,child){
        node.removeChild(child)
    }
}

function removeNode(el){
    const parent = nodeOps.parentNode(el)
    if(isDef(parent)){
        nodeOps.removeChild(parent,el)
    }
}
````
## 更新节点
回顾一下，vnode的不同类型：
````js
// 注释节点
{
    text:"注释节点",
    isComment: true
}

// 文本节点
{
    text:"文本"
}
````
**注意！！！p标签、span标签这些都还是元素节点，文本节点指的是标签里的纯文本内容**

只有两个节点是同一个节点时，才需要更新元素节点，更新节点也不是暴力地使用新节点覆盖旧节点，而是通过比对找出新旧两个节点不一样的地方，针对那些不一样的地方进行更新  
### 静态节点
静态节点指的是那些一旦渲染到界面上，状态不会改变，不会发生任何变化的节点  
````js
<p>我是静态节点，我不需要发生变化</p>
````
### 新虚拟节点有文本属性
当新旧两个虚拟节点（vnode和oldVnode)不是静态节点，并且有不同属性时，根据新节点vnode是否有text属性，更新节点可以分为两种不同的情况：
* 新vnode有text属性，那么不论之前旧节点的子节点是什么，直接调用setTextConent方法（在浏览器环境下是node.textContent方法）来将视图中DOM节点的内容改为虚拟节点（vnode)的text属性所保存的文字

### 新虚拟节点无文本属性（元素节点）
如果新创建的虚拟节点没有text属性，那么它就是一个元素节点，元素节点通常会有子节点（children)属性，但也可能没有子节点，所以又分两种情况讨论
* 新创建的虚拟节点有children属性，旧虚拟节点也有children属性，那么我们要对新旧两个虚拟节点的children进行一个更加详细的对比并更新；旧虚拟节点没有children属性，那说明旧虚拟节点是一个空标签，要么是有文本的文本标签（所以元素节点往往可以包含文本节点），包含文本节点，就先清空文本再插入  
* 新创建的虚拟节点没有children属性，当新创建的虚拟节点既没有text属性也没有children属性，那么这就是个空节点，它下面没有文本内容也没有子节点，那oldVnode有什么就删什么，最后达到视图中是空标签的目的

<img src="https://pic1.zhimg.com/80/v2-f9d6f677e54a61da09aa8d1ceba1f4bc_1440w.jpg" width="320" height="400">

### [更新节点的优化策略](https://zhuanlan.zhihu.com/p/99015402)
新旧vnode树 diff更新时，有些节点只是位置需要移动，vue给出了高效的节点更新方案  
优化策略总结为三步：
* 1. 跳过undefined节点。移动节点时，真正移动的是真实DOM节点，新旧vnode树
还要经过一轮又一轮的比对，所以它们肯定是维持原样的，但是为了防止后续重复处理同一个节点，旧的虚拟子节点就会被设置为undefined。这里只要记住如果旧开始节点为undefined,就后移一位；如果旧结束节点为undefined，就前移一位
* 2. 快捷查找 四种比对方式
* 3. key值查找

<img src="https://pic3.zhimg.com/80/v2-3c18762eb69bd3996b2d33b29ffd0646_1440w.jpg" width="400" height="300">
  
初始化状态如上

<img src="https://pic3.zhimg.com/80/v2-9252a9c37c7c4192fb676bb4113e6e6e_1440w.jpg" width="400" height="300">

首先进行的是两两四次的快捷比对，四次比对都没找到，再通过旧节点的key值列表，也没有找到，说明E是新增的节点，创建对应的真实dom节点，插入到旧节点里start对应真实dom的前面，也就是A的前面，已经处理完了一个，新start位置后移一位

<img src="https://pic4.zhimg.com/80/v2-2abe2f515a5c09cedee8ea59daf097db_1440w.jpg" width="400" height="300">

接着开始处理第二个，还是首先进行快捷查找，没有后进行key值列表查找，发现是已有的节点，只是位置不对，那么进行插入操作，参考节点还是A节点，将原来旧节点C设置为undefined， 之后就会跳过它，又处理完了一个节点，新start后移一位

<img src="https://pic2.zhimg.com/80/v2-e270d02def5c091c545ffc483c14bbb1_1440w.jpg" width="400" height="300">

再处理第三个节点，通过快捷查找找到了，是新start节点对应旧start节点，dom位置是对的，新start和旧start都后移一位

<img src="https://pic3.zhimg.com/80/v2-b6cc8bcfe1e25096676959efa9c0dd92_1440w.jpg" width="400" height="300"> 

接着处理的第四个节点，通过快捷查找，这个时候先满足了旧start节点和新end节点的匹配，但是dom位置是不对的， 插入节点到最后位置，最后将新end前移一位，旧start后移一位  

<img src="https://pic3.zhimg.com/80/v2-c7401096a5f8948b187ab62846564942_1440w.jpg" width="400" height="300"> 

处理最后一个节点，首先会执行跳过undefined的逻辑，然后再开始快捷比对，匹配到的是新start节点和旧start节点，它们各自start后移一位，这个时候就会跳出循环了  

最后收尾工作  
<img src="https://pic2.zhimg.com/80/v2-a2df50dd8ba117791f375aaf4897fbfd_1440w.jpg" width="400" height="300">

>在更新子节点时，需要在oldChildren中循环去找一个节点，但是如果在模板中渲染列表时，为子节点设置了属性key，那么会建立key与index索引的对应关系，就是一个key对应着一个节点下标这样一个对象，就可以直接通过key拿到下表，这样我们就不需要通过循环来查找节点




