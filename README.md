# 俄罗斯方块小游戏 —— Vue的diff过程

## 补充概念

### Dom树

![MacDown logo](https://img-blog.csdnimg.cn/201812261441271.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L29wZW5ib3gyMDA4,size_16,color_FFFFFF,t_70)


### 真实DOM的解析过程

![MacDown logo](https://upload-images.jianshu.io/upload_images/4345378-b7ccad3bc808783f.png)


##  Vue基本原理

### Vue内部流程

![MacDown logo](https://segmentfault.com/img/bV51sZ?w=1752&h=1216)

### Vue.js异步更新及nextTick

https://blog.csdn.net/sinat_17775997/article/details/82183435

https://github.com/tuzilingdang/learnVue/blob/master/docs/Vue.js%E5%BC%82%E6%AD%A5%E6%9B%B4%E6%96%B0DOM%E7%AD%96%E7%95%A5%E5%8F%8AnextTick.MarkDown

### VNode结点

```

export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  functionalContext: Component | void; // only for functional component root nodes
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions
  ) {
    /*当前节点的标签名*/
    this.tag = tag
    /*当前节点对应的对象，包含了具体的一些数据信息，是一个VNodeData类型，可以参考VNodeData类型中的数据信息*/
    this.data = data
    /*当前节点的子节点，是一个数组*/
    this.children = children
    /*当前节点的文本*/
    this.text = text
    /*当前虚拟节点对应的真实dom节点*/
    this.elm = elm
    /*当前节点的名字空间*/
    this.ns = undefined
    /*编译作用域*/
    this.context = context
    /*函数化组件作用域*/
    this.functionalContext = undefined
    /*节点的key属性，被当作节点的标志，用以优化*/
    this.key = data && data.key
    /*组件的option选项*/
    this.componentOptions = componentOptions
    /*当前节点对应的组件的实例*/
    this.componentInstance = undefined
    /*当前节点的父节点*/
    this.parent = undefined
    /*简而言之就是是否为原生HTML或只是普通文本，innerHTML的时候为true，textContent的时候为false*/
    this.raw = false
    /*静态节点标志*/
    this.isStatic = false
    /*是否作为根节点插入*/
    this.isRootInsert = true
    /*是否为注释节点*/
    this.isComment = false
    /*是否为克隆节点*/
    this.isCloned = false
    /*是否有v-once指令*/
    this.isOnce = false
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next */
  get child (): Component | void {
    return this.componentInstance
  }
}
```

**简单的Vnode树**

```
{
    tag: 'div'
    data: {
        class: 'test'
    },
    children: [
        {
            tag: 'span',
            data: {
                class: 'demo'
            }
            text: 'hello,VNode'
        }
    ]
}

```

**对应的HTML**

```
<div class="test">
    <span class="demo">hello,VNode</span>
</div>
```

##  Vue的Diff算法

### 1. 新老节点的比较 

#### patch比较过程

![MacDown logo](https://i.loli.net/2017/08/27/59a23cfca50f3.png)


![MacDown logo](https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=3465143140,4129223592&fm=15&gp=0.jpg)

#### 相同节点 -- sameVNode

```

/*
  判断两个VNode节点是否是同一个节点，需要满足以下条件
  key相同
  tag（当前节点的标签名）相同
  isComment（是否为注释节点）相同
  是否data（当前节点对应的对象，包含了具体的一些数据信息，是一个VNodeData类型，可以参考VNodeData类型中的数据信息）都有定义
  当标签是<input>的时候，type必须相同
*/
function sameVnode (a, b) {
  return (
    a.key === b.key &&
    a.tag === b.tag &&
    a.isComment === b.isComment &&
    isDef(a.data) === isDef(b.data) &&
    sameInputType(a, b)
  )
}

```

### 2. 相同节点进行 patchVNode比较

* 如果新旧VNode都是静态的，同时它们的key相同（代表同一节点），并且新的VNode是clone或者是标记了once（标记v-once属性，只渲染一次），那么只需要替换elm以及componentInstance即可。

* 新老节点均有children子节点，则对子节点进行diff操作，调用updateChildren，这个updateChildren也是diff的核心。

* 如果老节点没有子节点而新节点存在子节点，先清空老节点DOM的文本内容，然后为当前DOM节点加入子节点。

* 当新节点没有子节点而老节点有子节点的时候，则移除该DOM节点的所有子节点。

* 当新老节点都无子节点的时候，只是文本的替换。


### 3. Diff的核心内容 —— updateChildren

#### 循环

(1) 四个指针向内靠拢， 两两比对共4次

![MacDown logo](https://i.loli.net/2017/08/28/59a4015bb2765.png)

![MacDown logo](https://user-gold-cdn.xitu.io/2018/1/2/160b70ecf5967f0a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)


* 第1种比对

![MacDown logo](https://i.loli.net/2017/08/28/59a40c12c1655.png)

* 第2种比对

![MacDown logo](https://i.loli.net/2017/08/29/59a4c70685d12.png)

* 第3种比对

![MacDown logo](https://github.com/tuzilingdang/img-temp/blob/master/vnode.png)

* 第4种比对


* 


 (2) 以上4中比较不满足

通过createKeyToOldIdx会得到一个oldKeyToIdx，里面存放了一个key为旧的VNode，value为对应index序列的哈希表。从这个哈希表中可以找到是否有与newStartVnode一致key旧的VNode节点，如果同时满足sameVnode，patchVnode的同时会将这个真实DOM（elmToMove）移动到oldStartVnode对应的真实DOM的前面


![MacDown logo](https://i.loli.net/2017/08/29/59a4d7552d299.png)

 newStartVnode在旧的VNode节点找不到一致的key，或者是即便key相同却不是sameVnode，这个时候会调用createElm创建一个新的DOM节点
 
 
![MacDown logo](https://i.loli.net/2017/08/29/59a4de0fa4dba.png)

#### 循环结束

![MacDown logo](https://i.loli.net/2017/08/29/59a509f0d1788.png)


![MacDown logo](https://i.loli.net/2017/08/29/59a4f389b98cb.png)




##  小游戏的性能问题

```
    left(matrix) {
        if (this.pos.y - 1 < 0) return false

        for(let i = this.pos.x; i < this.pos.x  + this.shape.length; i++) {
            if(matrix[i][this.pos.y - 1] == 1) return false
        }
    
        this.pos.y--;

        for (let i = 0; i < this.shape.length; i++) {
            matrix[this.pos.x + i] && matrix[this.pos.x + i].splice(this.pos.y + this.shape[0].length, 1, 0)
        }

        for (let i = 0; i < this.shape.length; i++) {
            for (let j = 0; j < this.shape[i].length; j++) {
                matrix[this.pos.x + i] && matrix[this.pos.x + i].splice(this.pos.y + j, 1, this.shape[i][j])
            }
        }

        return true
    }

```

```
    down(matrix, accRowsList, clearRows) {
        const shapeHeight = this.shape.length
        const shapeWidth = this.shape[0].length
        const shape = this.shape
        const posY = this.pos.y
        const matLength = matrix[0].length
        let shapeAcc = []
        let crashType = ''

        for (let j = 0; j < shapeWidth; j++) {
            const attatchBottom = (this.pos.x + this.shape.length) >= matrix.length
            const blockCrash = matrix[this.pos.x + this.shape.length] && matrix[this.pos.x + this.shape.length][this.pos.y + j] && this.shape[shapeHeight - 1][j]
            if (attatchBottom || blockCrash) {
                for (let i = 0; i < shapeHeight; i++) {
                    // 检测是否行满
                    if (this.pos.x + i > -1) {
                        let checkLine = matrix[this.pos.x + i] && matrix[this.pos.x + i].filter(value => {
                            return value == 1
                        })
                        if (checkLine.length == matLength) clearRows.push(this.pos.x + i)
                    }
                }
                crashType = blockCrash ? 'blockCrash' : 'attatchBottom'
            }
        }
        if (crashType) {
            for (let j = 0; j < shapeWidth; j++) {
                for (let i = 0; i < shapeHeight; i++) {
                    if (!shapeAcc[j] && shape[i][j]) shapeAcc[j] = shapeHeight - i
                }
                accRowsList[posY + j] = crashType == 'blockCrash' ? matrix.length - this.pos.x - shapeHeight + shapeAcc[j] : accRowsList[posY + j] + shapeAcc[j]
            }
            return false
        }

        this.pos.x++;
        if (this.pos.x - 1 >= 0) {
            for (let i = 0; i < this.shape[0].length; i++)
                matrix[this.pos.x - 1].splice(this.pos.y + i, 1, 0)
        }
        for (let i = 0; i < this.shape.length; i++) {
            for (let j = 0; j < this.shape[i].length; j++) {
                const newVal = i == this.shape.length - 1 ? this.shape[i][j] || matrix[this.pos.x + i][this.pos.y + j] : this.shape[i][j]
                matrix[this.pos.x + i] && matrix[this.pos.x + i].splice(this.pos.y + j, 1, newVal)
            }
        }

        return true
    }

```
```
<div class="screen-grid-area" :class="!isGameOn ? 'hidden': ''">
       <div class="square"  :class="!gameOver && matrix[parseInt((n-1)/columnNum)][(n-1)%columnNum ] ? 'black':''"
                    v-for="n in columnNum*rowNum" :id="`${parseInt((n-1)/columnNum)}-${(n-1)%columnNum }`" v-bind:key="n">
             <div class="square-inner"></div>
       </div>
</div>

```



每次掉落都会触发一次diff， 每秒 diff 的最大复杂度  O（12 * 22 ）， 一个周期 22 * O（12 * 22 ）


合理划分组件后：  一个周期触发一次diff  O（12 * 22 ）









