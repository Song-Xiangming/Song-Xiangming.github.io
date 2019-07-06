---
title: 拓扑图实现
comments: true
date: 2019-07-05 15:04:37
tags:
---

> 最近业务要实现一个拓扑图，之前接触数据可视化工具较少，尝试了多种相关实现工具，在此记录并总结。阅读本文可以了解：
1. 常见拓扑图实现工具优劣势对比
2. 通过一个例子，学习实现常见拓扑图的方式和小技巧

## 需求

**展示效果：**
![blade_dbTopo_设计图1](http://ww1.sinaimg.cn/large/006tNc79ly1g4ozzh48d5j31tg0togq1.jpg)

**hover 高亮链路效果：**
![blade_dbTopo_设计图2](http://ww2.sinaimg.cn/large/006tNc79ly1g4p010cos6j31jk0gs75r.jpg)

**要点**
- 布局：节点组x轴分为四部分（可能不均分），y轴垂直居中，存在无连线的孤立节点
- 交互需求：鼠标hover时要高亮整条链路

## 技术选型

### 拓扑图开发工具尝试

> 尝试顺序由上至下

#### [Echarts](https://www.echartsjs.com/index.html)

- 尝试理由：
    - 中文文档，上手容易
    - [关系图 - 节点图](https://echarts.baidu.com/examples/editor.html?c=graph-simple) 接近需求,封装功能全面，hover效果可以低成本实现
    - 可由开发者维护节点位置，方便处理孤立节点
- 放弃理由：
    - 文字label不能扩大节点体积，导致连线会盖住文字，如图：
        ![blade_dbTopo_echarts](http://ww2.sinaimg.cn/large/006tNc79ly1g4p23ta3dej308u02a743.jpg)
        - 尝试解决方案1：文字放下方，本需求不适用（太丑）
        - 尝试解决方案2：[text-to-svg](https://www.npmjs.com/package/text-to-svg) ，让文字成为symbol的一部分
            - 每个动态的text转svg，性能不好
            - symbolSize的限制：会导致无论多长的文字都缩放到同样宽度

#### [HighCharts](https://www.highcharts.com/)

- 尝试理由：
    - svg版的echarts（也可以说：echarts是canvas版的highcharts），希望用灵活的svg解决echarts节点热区的问题
- 放弃理由：
    - 甚至没有合适的关系图
- 备注：
    - HighCharts和Echarts 本质上是一类东西，跟 d3.js 等工具的维度不同。它们自带的图表类型能满足你最好，满足不了的话就只能自己造轮子了

#### [JTopo](http://www.jtopo.com/index.html)

- 尝试理由：
    - 看名字就知道，专门用来绘制拓扑图，覆盖类型很广
    - 可由开发者维护节点位置，方便处理孤立节点
    - 上手容易，文档极其友好
- 放弃理由：
    - 尝试绘制后发现，依然有echarts的问题，文字label不能扩大节点体积。根本问题还是节点的定制不够自由
    <div style="width: 200px;">![blade_dbTopo_JTopo](http://ww1.sinaimg.cn/large/006tNc79ly1g4p2q47ustj307o042mxh.jpg)</div>
- 备注：
    - 国人开发，轻量级的图表库，很推荐大家了解下

#### [Darge-d3](https://github.com/dagrejs/dagre-d3)

- 尝试理由：
    - 基于d3的有向无环图绘制工具，相比d3较容易上手，不需要自己造轮子了
    - 给定节点间关系，可自动化生成拓扑图，无需维护节点坐标
- 放弃理由：
    - 由于自动生成坐标，孤立的节点会和拓扑图并列（孤立节点被认为是另一张拓扑图），难以控制。
    - 如果要用它强行实现，要前端补充孤立节点的关系（仅用于绘制），再隐藏掉连线
- 备注：
    - 自动化布局的功能很强大，对于没有孤立节点的需求，可以尝试。[看个Demo](https://dagrejs.github.io/project/dagre-d3/latest/demo/sentence-tokenization.html)

#### [G6](https://www.yuque.com/antv/g6/quick-start)

- 尝试理由：
    - 团队内有相关实现“故障大盘链路图”，很接近本需求
    - 插件化，官方没有的功能可以自己写插件实现
- 放弃理由：
    - 和故障大盘开发者交流下，了解到有下面两个问题，故而没有尝试：
        - g6文档很少比d3还难学（指2.0的文档）
        - 孤立节点不好处理（当时主观的猜测g6的实现方式类似darge-d3，不能控制点的位置，实际并不是，可以自己布局）
    - 应该检讨自己，听说有些坑就不去尝试，错过了一个好工具。最终使用JointJS实现了需求，现在回头再看g6文档，发现g6的3.0文档比2.0改善了不少，而且是中文文档，核心概念和Joint相似；插件化很强大，缺啥功能自己补~
- 备注：
    - G6的插件化可以做很多事，比如
        - 借助dagre-d3来布局，还能利用插件把原本自动生成的折线改为直线等
        - 实现vueComponent插件，可以写vue组件节点，实现更复杂的样式和功能。[用Joint实现html element](https://resources.jointjs.com/tutorial/html-elements) 就比较繁琐

#### [JointJS](https://resources.jointjs.com/)

- 尝试理由：
    - 基于svg，高度自定义的element和link终于能满足需求了
    - 能够自己维护节点坐标，解决孤立节点的问题
    - 文档全面，社区资料不算少
- 放弃理由：
    - 这次没有放弃，用它实现了需求
- 备注：
    - 不要被它貌似收费的官网迷惑，Rappid的JointJS的收费升级版，而JointJS本身是开源免费的
    - JointJS内置的交互工具也很强大，处理强交互的需求很合适，看看它的 [Demo](https://resources.jointjs.com/demos)

### JointJS FAQ

- **为啥不再试试其他的库，比如 d3.js ？**
    - 时间紧，d3上手难度较高，原计划作为最后的防线，但在尝试到joint.js时发现可以满足需求，就没有选择d3及其他的库；
    - 但是d3确实是可以实现的，[看个随便搜的Demo](https://segmentfault.com/a/1190000015435525)
- **API看不懂，找不到：**JointJS文档虽全，但是API目录结构和搜索的体验不太好。建议学习方式如下：
    - 结合 jointjs-api 看 demo，如果在 api 中搜不到demo中的实现方法，去看下源码。如 org 图中new joint.shapes.org.Member，看似官方api，实际是自定义元素（custom elements）
    - 搜不到想要的功能，google一下，JointJS的社区讨论相对较多
- **npm包ts类型不全：**
    - 因为部分类型依赖 backbone，需要手动添加。surprise！解决方案：[Incompatible TypeScript definitions in jointjs 2.0.1 #797](https://github.com/clientIO/joint/issues/797)
- **想了解下JointJS核心概念**
    - **paper和graph**
        - paper即画布，图形将在paper上绘制
        - graph即图形数据，一个graph可与多个paper绑定，对graph的修改会即时反映到paper上
    - **cellView和cell**
        - cellView: 视图元素，是paper的基本元素，用来处理UI交互，cellView可以通过.model属性获取它的cell
        - cell: 图形元素，是graph的基本元素，用来存储图形元素数据，graph其实就是cell的集合
    - **link和element**
        - cell有两种类型，link是连线，element是节点，他们的视图元素对应为linkView和elementView
    - **source和target**
        - 即连线的起点和终点

## 实现要点

> 注：本文示例代码均为typescript

### element & link 样式实现

- [Cell.define](https://resources.jointjs.com/docs/jointjs/v2.2/joint.html#dia.Cell.define)：JointJS支持高度自定义的cell，element和link都是svg绘制的cell视图元素，大部分样式用svg attr结合api即可实现

- **如何实现动态的element宽度？**
    
    - 由于svg的宽度不能像div那样根据内容自动撑开，这里我的解决方案
        ```typescript
        // 根据element内部String.length * 单个字符px + padding(留白和图片的宽度和)
        size: { width: name.length * 7.2 + 20, height: 20 }
        ```
    - 你可能会问不同字符宽度不一样怎么办？
        - 使用等宽字体可解决，如下图：不等宽字体 (左)，等宽字体（右）
        <div style="width: 280px;">![blade_dbTopo_等宽字体](http://ww1.sinaimg.cn/large/006tNc79ly1g4pz88a423j30gi04674f.jpg)</div>

### 节点布局

- **实现节点组垂直居中思路：**
    - 计算图整体高度totalHeight: gap * (nodeNum - 1)
    - 定位中心点，topNode, bottomNode：考虑nodeNum为奇数偶数节点两种情况
        - 奇数
            - 中心点/topNode/bottomNode：totalHeight / 2
        - 偶数
            - 中心点：totalHeight / 2
            - topNode: totalHeight / 2 - gap / 2
            - bottomNode: totalHeight / 2 + gap / 2
    - 循环列表，依次更新topNode, bottomNode
- **按上述思路计算Y坐标，上代码**
    ```typescript
    // 为满足绘制顶部分类label的需求，添加了offset偏移
    getNodeCoordY (maxHeight: number, num: number, gap: number, offset: number = 80) {
        if (num < 1) return [];
        if (gap * (num - 1) > maxHeight - offset) {
            console.error('Node height exceeds maximum height~');
            return [];
        }
        let result = [];
        let top = 0;
        let bottom = 0;
        // 未分配数量
        let leftNum = num;

        // 定位中心点
        if (num % 2 === 1) {
            // (maxHeight - offset) / 2 + offset
            top = bottom = (maxHeight + offset) / 2;
            result.push(top);
            leftNum -= 1;
        } else {
            // (maxHeight - offset) / 2 - gap / 2 + offset;
            top = (maxHeight + offset - gap) / 2;
            bottom = (maxHeight + offset + gap) / 2;
            result.push(top);
            result.push(bottom);
            leftNum -= 2;
        }

        if (leftNum < 1) return result;
        // 定位剩余点
        let toggle = true;
        for (let i = 0; i < leftNum; i++) {
            if (toggle) {
            top = top - gap;
            result.push(top);
            } else {
            bottom = bottom + gap;
            result.push(bottom);
            }
            toggle = !toggle;
        }
        return result.sort((a, b) => a - b);
    }
    ```
- **X坐标默认按num均分宽度，支持调整分配比例**
    ```typescript
    getNodeCoordX (maxWidth: number, num: number, proportion?: number[], offset: number = 20) {
        if (maxWidth <= offset) return [];
        let result = [];
        const gapNum = proportion ? proportion.reduce((pre, curr) => pre + curr) : num;
        const gap = Math.floor((maxWidth - offset) / gapNum);
        // 定位第一个点
        let start = offset;
        for (let i = 0; i < num; i++) {
            result.push(start);
            start += proportion ? gap * proportion[i] : gap;
        }
        return result;
    }
    ```

### 实现 hover 高亮链路

- **思路**
    - 初始化highlightNodes: node[]，highlightLinks: link[]，两个数组，用于存储需要高亮的节点和链路
    - link和element（即节点，项目中命名为node）本质都是svg元素，监听其mouseover/mouseout事件，借助svg可以灵活绑定class的特点，可通过统一 添加/删除class 实现 高亮/取消高亮 的效果。
    - cell:mouseover时：以event.target为root节点
        - 向前递归：寻找sourceNode和sourceLink
            - 若root是link，寻找该link的sourceNode存入highlightNodes
            - 若root是node，寻找以该node为target的link存入highlightLinks
            - 递归过程会循环出现link -> node -> link，直到源头node为止
        - 向后递归：寻找targetNode和targetLink
            - 原理同上，递归过程会循环出现link -> node -> link，直到终点node为止
        - 收集到需要高亮的node和link，添加高亮；
    - cell:mouseout：移除highlightNodes，highlightLinks中node和link的高亮，并清空两个数组
    - **注：对于`<g>`元素cell:mouseover/mouseout会在鼠标经过其每个子元素时反复触发**，可以通过svg原生的 [pointer-events](https://developer.mozilla.org/zh-CN/docs/Web/CSS/pointer-events)属性，标识需要触发事件的元素
- **关键代码**（这里借助了部分JointJS api，但思路是相通的，可以同样应用到d3，g6中）
    ```typescript
    // 给所有cell绑定事件
    this.paper.on('cell:mouseover', this.handleCellMouseOver);
    this.paper.on('cell:mouseout', this.handleCellMouseOut);
    handleCellMouseOver(cellView) {
        this.addHighlight(cellView);
    }
    handleCellMouseOut() {
        this.removeHighlight();
    }
    // 向前递归，收集source
    findSourceCell(cell) {
        if (cell.isLink()) {
            if (!this.highlightLinks.includes(cell)) this.highlightLinks.push(cell);
            const sourceNode = cell.getSourceElement();
            if (!sourceNode) return;
            this.findSourceCell(sourceNode);
        } else {
            if (!this.highlightNodes.includes(cell)) this.highlightNodes.push(cell);
            const links = this.graph.getConnectedLinks(cell, {
                inbound: true
            });
            if (!Array.isArray(links) || links.length < 0) return;
            links.forEach(link => this.findSourceCell(link));
        }
    }
    // 向后递归，收集target
    findTargetCell(cell) {
        if (cell.isLink()) {
            if (!this.highlightLinks.includes(cell)) this.highlightLinks.push(cell);
            const sourceNode = cell.getTargetElement();
            if (!sourceNode) return;
            this.findTargetCell(sourceNode);
        } else {
            if (!this.highlightNodes.includes(cell)) this.highlightNodes.push(cell);
            const links = this.graph.getConnectedLinks(cell, {
                outbound: true
            });
            if (!Array.isArray(links) || links.length < 0) return;
            links.forEach(link => this.findTargetCell(link));
        }
    }
    // 添加高亮
    addHighlight(rootCellView) {
        this.highlightNodes = [];
        this.highlightLinks = [];
        const root = rootCellView.model;
        // 收集高亮Cell
        this.findSourceCell(root);
        this.findTargetCell(root);
        // 先降低全部Cell的透明度
        this.allCells.forEach(cell => {
            const cellView = this.paper.findViewByModel(cell);
            cellView.highlight(null, {
                highlighter: {
                    name: 'addClass',
                    options: {
                        className: 'less-opacity'
                    }
                }
            });
        });
        // 恢复需要高亮Cell的透明度
        [...this.highlightNodes, ...this.highlightLinks].forEach(cell => {
            const cellView = this.paper.findViewByModel(cell);
            // 自定义的Options需要在unhighlight方法中声明，不然无法正确移除
            cellView.unhighlight(null, {
                highlighter: {
                    name: 'addClass',
                    options: {
                        className: 'less-opacity'
                    }
                }
            });
        });
    }
    // 移除高亮
    removeHighlight() {
        // 恢复全部Cell的透明度
        this.allCells.forEach(cell => {
            const cellView = this.paper.findViewByModel(cell);
            // 自定义的Options需要在unhighlight方法中声明，不然无法正确移除
            cellView.unhighlight(null, {
                highlighter: {
                    name: 'addClass',
                    options: {
                        className: 'less-opacity'
                    }
                }
            });
        });
        // 清空高亮Cell列表
        this.highlightNodes = [];
        this.highlightLinks = [];
    }
    ```
- **实现效果**
   ![blade_dbTopo_hover高亮实现效果](http://ww2.sinaimg.cn/large/006tNc79ly1g4q0bu1vlxj311w0f83zl.jpg)

## 总结

- 按目前的调研来看，本次需求用JointJS，G6，D3都能实现，相比之下
    - JointJS的优势是内置有灵活的交互工具，封装的基础API全面
    - G6有中文文档更好理解，插件化提供了更多可能。思想和JointJS类似，理解一个就能较快上手另一个。
    - D3自由度最高但是实现起来就更复杂，上手难度较高，我又一次避（cuo）开（guo）了d3，也许总有一天会有复杂的需求让我不得不学习它。
- 广泛调研，不要畏难，勇于尝试，可能就会发现更优质的工具和实现思路。总结出共通的思想后，可能只用svg也写得出来了。
