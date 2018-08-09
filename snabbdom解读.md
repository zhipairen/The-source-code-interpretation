### 虚拟dom的实现及snabbdom源码解读
[How to write your own Virtual DOM
](https://medium.com/@deathmood/how-to-write-your-own-virtual-dom-ee74acc13060) 

[Write your Virtual DOM 2: Props & Events](https://medium.com/@deathmood/write-your-virtual-dom-2-props-events-a957608f5c76)

[snabbdom](https://github.com/snabbdom/snabbdom/blob/master/src/snabbdom.ts)

**两个重要概念**：
1.  Virtual DOM 是 real DOM的一种表现/映射
2.  通过old and new Virtual tree的比较之后，再将重要的最小变化映射到real DOM

首先简单的dom映射为js对象：
``` js
<ul class="a">
    <li>item1</li>
    <li>item2</li>
</ul>
转为：
{ type: ‘ul’, props: { ‘class’: ‘list’ }, children: [
  { type: ‘li’, props: {}, children: [‘item 1’] },
  { type: ‘li’, props: {}, children: [‘item 2’] }
] }

like:
{ type: ‘…’, props: { … }, children: [ … ] }
```
> 注意：real DOM node(elements, text nodes)以'$'开头

**1. 转为dom表示**
```js
/**
* 由虚拟dom创建真实元素
* createElement
*/
function createElement(node){
    if(typeof node === 'string'){ // textNodes
        return document.createTextNode(node)
    }
    const $el = document.createElement(node.type);
    setProps($el, node.prop);
    addEventListeners($el, node.props);
    node.children
    .map(createElement)
    .forEach($el.appendChild.bind($el));
    return $el
}
```
**2. handing change**
```js
/**
* appendChild() 增加新节点
* removeChild() 移除旧节点
* replaceChild(new,old) 替换节点
* childNodes[index] 子节点集合
*/
function updateElement($parant, newNode, oldNode, index = 0){
    if(!oldNoe){ 
        $parant.appentChild(createElement(newNode))
    }else if(!newNode){
        $parant.removeChild($parant.childNodes[index])
    }else if(change(newNode, oldNode)){
        $parant.replaceChild(createElement(newNode),$parant.childNodes[index])
    }else if(newNode.type){ // elemnet children更新
    updateProps(
    $parant.childNodes[index], 
    newNode.prop, 
    oldNode.prop);
    const newLength = newNode.children.length;
    const oldLength = oldNode.children.length;
    for(let i = 0; i < newLength || i < oldLength; i++){
    // 子节点递归更新
        updateElement(
        $parant.childNodes[index],
        newNode.children[i],
        oldNode.children[i],
        i
        )
    }
    }
}
/**
* 比较两节点 element textNodes 文本 element tag
*/
function change(node1, node2){
    return typeof node1 !== typeof node2 ||
    (typeof node1 === 'string' && node1 !== node2)
    || node1.type !== node2.type
}
```
**3. setting and diffing props**
```js
/**
* 设置布尔值的属性prop
*/
function setBoolenProp($target,name,value){
    if(value){
        $target.setAttribute(name, value);
        $target[name] = true;
    }else{
        $target[name] = false;
    }
}
/**
* 移除布尔属性
*/
function removeBoolenProp($target,name){
    $target.removeAttribute(name);
    $target[name] = false;
}
/**
* 判断是否为自定义属性
*/
function isCustomProp(name){
    
}
/**
* 设置prop: 自定义属性，布尔属性，class属性，其他
*/
function setProp($target,name,value){
    if(isCustomProp(name)){
        return;
    }else if(name === "className"){
        $target.setAttribute('class', value);
    }else if(typeof value === 'boolen'){
        setBoolenProp($target,name,value)
    }else{
        $target.setAttribute(name,value)
    }
}
/**
* 移除prop: removeAttribute
*/
function removeProp($target, name, value){
    if(isCustomProp){
        return;
    }else if(name === "className"){
        $target.removeAttribute("class")
    }else if(typeof value === "boolen"){
        removeBoolenProp($target,name)
    }else{
        $target.removeAttribute(name)
    }
}

function setProps($target, props){
    Object.keys(props)
    .forEach( name => {
        setProp($target, name, props[name])
    })
}
/**
* 更新prop: newVal不存在，移除旧属性值；旧属性不存
* 在或者新旧属性值不同，都直接重新设置新属性值
*/
function updateProp($target, name, newVal, oldVal){
    if(!newVal){
        removeProp($target, name, oldVal)
    }else if(!oldVal || newVal !== oldVal){
        setProp($target, name, newVal)
    }
}
/**
* diff new | old 之后,更新属性
*/
function updateProps($target, newProps, oldProps = {}){
    const props = Object.assign({},newProps,oldProps); // 合并新旧属性
    Object.keys(props).forEach( name => {
        updateProp($target, name, newProps[name], oldProps[name])
    })
}
```
**4. Event**
```js
/**
* 判断是否是事件
*/
function isEventProp(name){
   return /^on/.test(name);
}
/**
* 添加事件监听
*/
function addEventListeners($target,props){
    Object.keys(props).forEach(name => {
        if(isEventProp(name)){
            $target.addEventListener(name.slice(2).toLowerCase(),props[name])
        }
    })
}
```

**snabbdom源码阅读**
[简书](https://www.jianshu.com/p/b461657e49c0)

各部分解读
- h.ts
```ts
/**
* 该函数功能为 h(sel,data,children) => 转为 vnode对象 
* vnode={sel,data:VnodeData,children,text,elm....}在vnode.ts中查找结构
*/
function h(sel:any,b?:any,c?:any){
    // vnode对象组装
    // addNs 组装svg表示的vnode
}
```
- modules
```ts
/**
* 该文件下分别对 attr,class,dataset,event,props,style进行了patch（补丁）
* diff oldNode和newNode的属性等，update元素(elm)的属性等
* 以attr为例:(其他处理请看具体代码)
* oldAttr存在，newAttr不存在，则 elm.removeAttribute(oldAttr),
* 否则直接 elm.setAttribute(newAttr)
*/
export const attributesModule = {create: updateAttrs, update: updateAttrs} as Module;
*重点：将upAttrs操作放在了create,update(hook钩子)上，利用钩子巧妙的将Vnode上的属性等进行了更新
```
- hook.ts, vnode.ts, htmldomapi.ts
```ts
lifecycle hook 生命周期钩子
interface Hooks {
    pre?: PreHook;
    init?: 初始化钩子;
    create?: 创建;
    insert?: 插入vnode队列钩子;
    prepatch?: patch之前钩子;
    update?:...;
    postpatch?:...;
    destroy?:...;
    remove?:...;
    post?:...;
}
vnode 对象定义
htmldomapi: dom方法的包装，方便之后兼容元素方法
```
- 核心 snabbdom.ts
```ts
/**
* 只需要判断key或者sel确定是否为同一vnode
*/
function sameVnode(vnode1, vnode2){
    return vnode1.key === vnode2.key || vnode1.sel === vnode2.sel
}
/**
* 初始化中,将关于attr,props等的modules中对应的生命周期的cb收集到cbs
* cbs {
*  create: [updateAttr, updateProps....],
*  update: [updateAttr, updateProps....]
*  ...
* }
*/
function init(modules,domApi?){
let i: number, j: number, cbs = ({} as ModuleHooks);
     for (i = 0; i < hooks.length; ++i) {
        cbs[hooks[i]] = [];
        for (j = 0; j < modules.length; ++j) {
          const hook = modules[j][hooks[i]];// 取出关于attr,prop,style等module对应的hook回调函数
          if (hook !== undefined) {
            (cbs[hooks[i]] as Array<any>).push(hook); // 存入对应的生命周期中 eg: create:[cb1,cb2....]
          }
        }
}
```
**createElm** (vnode转为real dom) 函数分析:
```ts
function createElm(vnode, insertdVnodeQueue){
    ...
     if (isDef(i = data.hook) && isDef(i = i.init)) {
        i(vnode); // 自定义生命周期init执行
        data = vnode.data;
      }
      ...
      if(sel === '!'){
          // 生成注释
          vnode.elm = api.createComment(vnode.text as string);
      }else if(sel !== undefined){
          // 解析选择器
          // 依据sel 创建createElement(tag)
          // setAttribute(id || class)
          ...
          if(is.array(children)){
              // 递归子vnode,并创建元素，再追加到父节点下
          for (i = 0; i < children.length; ++i) {
          const ch = children[i];
          if (ch != null) {
            api.appendChild(elm, createElm(ch as VNode, insertedVnodeQueue));
              }
            }
          }else if(is.primitive(vnode.text)){
            // textNode 无子节点，直接创建文本节点追加
             api.appendChild(elm, api.createTextNode(vnode.text));
          }
          // vnode.data.hook 自定义hook执行
          i = (vnode.data as VNodeData).hook;      
           if (isDef(i)) {
            if (i.create) i.create(emptyNode, vnode);
            if (i.insert) insertedVnodeQueue.push(vnode);
      }
      }else{ // sel 为空或未定义，
        // 说明为文本节点  
      }
      return vnode.elm;
}
```
**updateChildren**(比较子节点并更新)重点分析：

旧子节点与新子节点两组数组比较分析：
 1. 首首比较，尾尾比较是否为sameVnode
 2. 首尾比较, 尾首比较是否为sameVnode
 3. 中间比较，为sameVnode需要patchVnode
 4. 注意新节点向前插入的位置，以及新旧节点的位置移动
 5. 对于key,通过createKeyToOldIdx函数生成映射，具体看代码分析
 ```ts
 function updateChildren(parentElm, oldCH, newCh, insertedVnodeQueue){
     ....
     // 新旧子节点循环比较开始
     while( oldStartId <= oldEndIdx && newStartIdx <= newEndIdx){
         ...
         // 首首vnode比较
         // patchVnode函数主要作用类似的新旧节点打补丁，具体作用见下分析
         else if(sameVnode(oldStartVnode,newStartVnode){
            patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue);
            oldStartVnode = oldCh[++oldStartIdx];// 旧节点位置前移
            newStartVnode = newCh[++newStartIdx]; 
         }else if(sameVnode(oldEndVnode, newEndVnode)){
             // 尾尾节点对比
         }else if(sameVnode(oldStartVnode, newEndVnode)){// 首尾对比
            patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue);
            api.insertBefore(parentElm, oldStartVnode.elm as Node, api.nextSibling(oldEndVnode.elm as Node)); // 因为新旧vnode相同，所以可复用 真实dom元素，且需要把旧dom节点插入到父节点下子节点末尾
            oldStartVnode = oldCh[++oldStartIdx];
            newEndVnode = newCh[--newEndIdx];
         }else if(sameVnode(oldEndVnode, newStartVnode)){
             // 尾首比较，vnode moved left
         }else{
             if (oldKeyToIdx === undefined) {
                oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx); // 创建key
          }
          idxInOld = oldKeyToIdx[newStartVnode.key as string];
          if(isUndef(idxInOld)){ // 新节点
              // 插入到父元素下的子节点最前面并移动新节点数组开始位置
          }else{
              // key相同
              elmToMove = oldCh[idxInOld];
              if(elmToMove.sel !== newStartVnode.sel){
                  // 子节点key相同，但是元素不同，重新创建元素并插入到前面
              }else{
                  // key sel 都相同，patch新旧节点补丁
                  patchVnode(elmToMove, newStartVnode, insertedVnodeQueue);
                  ....
              }
           }
         }
     }
     if(oldStartIdx <= oldEndIdx || newStartIdx <= newEndIdx){ 
         if (oldStartIdx > oldEndIdx) {
             // 新子节点存在但是旧子节点不存在，直接 addVnodes 新增新节点即可
         }else{
             // 新子节点不存在但旧子节点存在，直接removeVnodes 移除旧节点即可
         }
     }
 }
 ```
 **patchVnode** 新旧vnode打补丁函数分析（进入此函数说明新旧vnode的ele元素是相同的）：
 1. 先执行update生命周期(包括module定义的和自定义的update hook),此时会更新成功新旧vnode的attr,style,prop,event等module中定义的
 2. 再细分有无子节点的情况，具体分析如下
 ```ts
 function patchVnode(oldVnode, Vnode, insertedVnodeQueue){
    ...
    // prepatch hook
     if(oldVnode === Vnode) return; // 两Vnode指针完全相同
     ...
     // update hook
     if(isUndef(Vnode.text){ // 新vnode无text
        if(isDef(oldCh) && isDef(ch){
            // 都有子节点，updateChildren
        }else if( isDef(ch)){
            // 有子节点，但无旧子节点
            if (isDef(oldVnode.text)) api.setTextContent(elm, ''); // 更新vnode的text为空
            addVnodes(...) //新增节点
        }else if(isDef(oldCh)){
            // 有旧节点，但是无新节点
            removeVnodes
        }else if(isDef(oldVnode.text)){
            api.setTextContent(elm, '');
        }
     }else if (oldVnode.text !== vnode.text) {
      api.setTextContent(elm, vnode.text as string);
    }
    ...
    // 如果有 postpatch hook执行
 }
 ```
 **patch**函数分析
```ts
function patch(oldVnode: VNode | Element, vnode){
    ...
    // 执行 pre hook
    ...
    if(sameVnode(oldVnode,vnode)){
        patchVnode(oldVnode, vnode, insertedVnodeQueu)
    }else {
        // vnode不同 创建新vnode元素，然后插入到父元素下的原旧节点的下个位置，再移除旧节点oldVnode
        elm = oldVnode.elm as Node;
      parent = api.parentNode(elm);

      createElm(vnode, insertedVnodeQueue);

      if (parent !== null) {
        api.insertBefore(parent, vnode.elm as Node, api.nextSibling(elm));
        removeVnodes(parent, [oldVnode], 0, 0);
      }
    }
    ...
    // 处理 insert hook
    // 处理 post hook
}
```
