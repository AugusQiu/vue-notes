## diff过程
* 同层节点vnode比对，oldNode === newNode或者是静态节点，退出diff，不做更新 
* oldNode!==null && newNode!==null &&  oldNode !== newNode，以newNode为准替换更新
* oldNode和newNode都有子节点而且不一样，调用updateChildren比对更新
* 只有newNode有子节点，调用createEle创建节点
* newNode没有子节点、oldNode有子节点，直接删除老节点  

updateChildren:新旧节点数组首尾元素两两四次匹配比对，在根据key值查找比对
