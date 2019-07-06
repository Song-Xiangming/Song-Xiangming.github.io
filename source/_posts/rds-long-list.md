---
title: 长列表性能优化
date: 2019-02-7 10:29:06
tags: 
    - js
    - 性能优化
comments: true
---

> 阅读本文可以了解：
1. 使用chrome performance进行性能分析的方式
2. 前端长列表渲染的解决方案

## 背景

本文是记录了一个页面性能问题的排查和解决过程。起因是在做公司项目时，在一个页面中，发现编辑器中快速输入出现卡顿，影响用户体验。如图：
![rds_SQL编辑器卡顿](https://ws4.sinaimg.cn/large/006tNc79ly1fzxmfhkj3cj31h20u0aba.jpg)

## 问题排查

### 问题定位

- 在SQL编辑器卡顿状态下，尝试在编辑器下方备注栏快速输入，也有卡顿现象。
- 清除页面中其他元素，只保留SQL编辑器，再次输入发现没有卡顿。
- 由以上两点可排除编辑器（项目中使用的codemirror）自身问题。
- 本项目使用vue技术栈。该页面中deep watch了一些复杂对象，怀疑是它影响了性能，将其优化为不使用deep，发现编辑器依然卡顿，说明watch deep不是核心问题
- 发现左侧sql table列表节点过多（300+），将该列表节点减少，发现卡顿有明显好转。猜想因为DOM节点过多，并且有大量监听器绑定，影响vue渲染速度。

### 页面性能分析

接着使用chrome performance对页面性能分析，在record期间进行了如下操作：  
`打开空页面 -> 选中一个包含300+个表的db（左侧渲染出大量table节点） -> 尝试快速输入，发现卡顿 -> 切换到一个空db（清空table节点）-> 尝试快速输入，不再卡顿 -> 再次切换回包含大量表的db -> 快速输入，再次出现卡顿。`  
得到如下分析图：
![rds_性能分析记录1](https://ws1.sinaimg.cn/large/006tNc79gy1fzgabthgkcj31hc0qego6.jpg)

图中包含大量信息，这里主要关注红框部分：
- 标记1：选择db，页面渲染300+table节点的list，nodes, listeners, js heap骤增（可以看到节点数高达7000+而不是300左右，因为每个列表item是一个组件，内部还包含众多dom节点，且该记录包含整个页面所有dom节点）
- 标记2：在sql中快速输入字符，发现keypress的handler了处理缓慢，每次时长达120+ms。下图是放大标记2的内容，事件右上的红色三角标记，代表chrome认为这个function处理速度过慢
![rds_性能分析记录2](https://ws1.sinaimg.cn/large/006tNc79ly1fzxnjwk1s2j30v40dsaan.jpg)
- 标记3：可以看到标记三处JS进行了一次Minor GC恢复了一些内存，大量的dom nodes和listeners已经超出新生代内存空间进入老生代内存，Minor GC无法释放。
- 在标记2~3左右，我们切换了db，页面不再渲染这些dom nodes，卡顿现象消失
- 标记4：切换db一段时间后，JS进行了Major GC，nodes, listeners, js heap占用全部大幅降低
- 标记5：再次切换回大列表db，又回到了标记1时的状态

通过性能分析，可以验证，确实是在table列表节点过多时会影响整个页面性能，导致编辑器输入卡顿。

## 解决方案

> 本章简易demo参考自[聊聊前端开发中的长列表](https://zhuanlan.zhihu.com/p/26022258)，感谢@Furybean

### 滚动加载

- 思路：页面只渲染部分部分dom，在用户向下滚动时陆续加载后续dom
- 使用前提：
    - PM接受这种形式的列表，某些场景用户体验不太好。
    - 适用于无限加载的需求，如微博列表
- 简易demo(vue)
    ```html
    <template>
        <div class="lazy-list">
            <div class="lazy-render-list-item" v-for="item in data">{{ item }}</div>
        </div>
    </template>

    <style>
    .lazy-render-list {
        border: 1px solid #666;
    }

    .lazy-render-list-item {
        padding: 5px;
        color: #666;
        height: 30px;
        line-height: 30px;
        box-sizing: border-box;
    }
    </style>

    <script>
    export default {
        name: 'lazy-render-list',

        data() {
        const count = 40;
        const data = [];
        for (let i = 0; i < count; i++) {
            data.push(i);
        }
        return {
            count,
            data
        };
        },

        mounted() {
            window.onscroll = () => {
                const maxScrollTop = Math.max(document.body.scrollHeight, document.documentElement.scrollHeight) - window.innerHeight;
                const currentScrollTop = Math.max(document.documentElement.scrollTop, document.body.scrollTop);
                if (maxScrollTop - currentScrollTop < 20) {
                const count = this.count;
                for (let i = count; i < count + 40; i++) {
                    this.data.push(i);
                }
                this.count = count + 40;
                }
            };
        }
    };
    </script>
    ```
- 缺陷
    - 前端搜索受影响：项目中有目前使用前端搜索，如果使用该方案，需要改为后端搜索。若数据量不大也可以在前端保存全量数据做前端搜索，滚动加载只负责渲染。
    - 用户持续向下滚动：难免列表越来越大，最终达到性能瓶颈，依然会造成输入卡顿。
    - 综上：**滚动加载方案不适用于本项目**

### 可视区渲染

- 思路：仅渲染视图范围内dom，移除不在视图内的dom。如图：
![rds_可视区渲染](https://ws4.sinaimg.cn/large/006tNc79gy1fzgaajkwtag31qi0kae81.gif)
- 使用前提：
    - 每个数据的展现形式的高度最好一致（不然计算困难）
    - 一次需要加载的数据量比较大
    - 滚动条需要挂载在一个固定高度的区域
- 简易demo（vue）
    ```html
    <template>
        <div class="list-view" @scroll="handleScroll($event)">
            <div class="list-view-phantom" :style="{ height: data.length * 30 + 'px' }"></div>
            <div v-el:content class="list-view-content">
            <div class="list-view-item" v-for="item in visibleData">{{ item.value }}</div>
            </div>
        </div>
    </template>
    ​
    <style>
    .list-view {
        height: 400px;
        overflow: auto;
        position: relative;
        border: 1px solid #666;
    }
    ​
    .list-view-phantom {
        position: absolute;
        left: 0;
        top: 0;
        right: 0;
        z-index: -1;
    }
    ​
    .list-view-content {
        left: 0;
        right: 0;
        top: 0;
        position: absolute;
    }
    ​
    .list-view-item {
        padding: 5px;
        color: #666;
        height: 30px;
        line-height: 30px;
        box-sizing: border-box;
    }
    </style>
    ​
    <script>
    export default {
        props: {
        data: {
            type: Array
        },
    ​
        itemHeight: {
            type: Number,
            default: 30
        }
        },
    ​
        ready() {
        this.visibleCount = Math.ceil(this.$el.clientHeight / this.itemHeight);
        this.start = 0;
        this.end = this.start + this.visibleCount;
        this.visibleData = this.data.slice(this.start, this.end);
        },
    ​
        data() {
        return {
            start: 0,
            end: null,
            visibleCount: null,
            visibleData: [],
            scrollTop: 0
        };
        },
    ​
        methods: {
        handleScroll(event) {
            const scrollTop = this.$el.scrollTop;
            const fixedScrollTop = scrollTop - scrollTop % 30;
            this.$els.content.style.webkitTransform = `translate3d(0, ${fixedScrollTop}px, 0)`;
    ​
            this.start = Math.floor(scrollTop / 30);
            this.end = this.start + this.visibleCount;
            this.visibleData = this.data.slice(this.start, this.end);
        }
        }
    };
    </script>
    ```
    了解代码中几个要点，便于理解：
    - 使用一个 phantom 元素来撑起整个这个列表，让列表的滚动条出现。
    - 列表里面使用变量 visibleData(Array 类型) 记录目前需要显示的所有数据。
    - 列表里面使用变量 visibleCount 记录可见区域最多显示多少条数据。
    - 列表里面使用变量 start、end 记录可见区域数据的开始和结束索引。
    - 在滚动的时候，修改真实显示区域的 transform: translate2d(0, y, 0)。
    - const fixedScrollTop = scrollTop - scrollTop % 30; 之所以要减掉scrollTop % {{item.style.height}}是为了保证滚动的顺滑，否则如果直接用scrolltop，会把list-view-content永远固定在顶部。可以尝试把item高度调高，数量调小，更利于理解这里
- 缺陷
    - iOS 上 UIWebView 的 onscroll 事件并不能实时触发: 因为这个原因，你可能会发现无限滚动在移动端很常见，但是可见区域渲染并不常见
- 由于本项目符合可视区渲染的前提条件，且不需要支持移动端，故选用该方案。

### 相关工具

> 简易demo还是存在一些瑕疵，生产环境中建议使用成熟的开源工具  

- [vue-virtual-scroll-list](https://github.com/tangbc/vue-virtual-scroll-list#readme)：A vue component support big data list with high scroll performance.
- [vue-infinite-scroll](https://github.com/ElemeFE/vue-infinite-scroll/)：An infinite scroll directive for vue.js.
- [Clusterize.js](https://github.com/NeXTs/Clusterize.js)：Tiny vanilla JS plugin to display large data sets easily
- [react-virtualized](https://github.com/bvaughn/react-virtualized)：React components for efficiently rendering large lists and tabular data 

## 结果

### 指标对比

![rds_指标对比](https://ws2.sinaimg.cn/large/006tNc79ly1fzxqjfko30j317e0eegmf.jpg)
> 注：高性能是在当前机器上（mac pro）的处理速度，低性能为4x slowdown后的处理速度

### 优化后性能检测对比图

![rds_性能分析记录3](https://ws3.sinaimg.cn/large/006tNc79ly1fzxqlkege3j31hc0mpwgo.jpg)

- 标记1：与之前标记1相同的db大列表渲染时段，可见各个指标占用（js heap，node，listeners）有明显下降
- 标记2：keypress handler处理也不再有红色的性能警告
- 标记3：同样的切换列表后的第一次GC回收过程，指标波动已远小于优化前

### 结论

通过指标对比表格，和优化后的分析图可以看出，使用了可视区渲染优化后，各项指标都有了明显好转。

## 感想

做这次优化时，脑海中有“只渲染视图区域内容，移除视野外dom”的模糊想法，Google一下，果然已有前人实现了这个思路，原来叫“可视区域渲染”，并且还有了不少开源库。赶紧借此机会，深入学习下长列表渲染的几种优化方式。同时，对于chrome performance的使用，我了解的还比较浅，文中有什么错误，欢迎指正。如果你也遇到类似问题，希望这篇文章，能给你带来一些帮助~
