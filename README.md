# Knowledge_graph
一个基于d3.js的知识图谱展示vue组件
## 版本号
1. d3.js  ^5.16.0
2. vue  ^2.6.11

## 功能
1.初始化图谱节点   
3.支持单击与双击事件   
4.支持根据节点标签进行节点大小的调整   
5.支持图谱下载   
6.可根据需要添加节点信息展示组件   

## 使用
1. 在父组件中将节点与边的数据源组合完成之后直接调用本组件中的renderGraph方法进行图谱的渲染
### 节点的数据结构 node

```
{
    id:[String],
    name:[String],
    properties:{name:[String]},
    type:[String]
}
```
### 边的数据结构 link
```
{
    id:[String],
    source:[String(nodeID)],
    target:[String(nodeID)],
    properties:{name:[String]},
    name:[String]
}
```
### 数据源的数据结构
```
{
    nodes: [node1, node2,...],
    links: [link1, link2,...]
}
```

2. 节点单击自定义事件为clickNode,双击自定义事件为dbclickNode,触发自定义事件时，子组件会给父组件传递被点击节点的数据（数据结构同上）。
3. 调整节点大小功能需要由父组件调用本组件中modifyNodeSize方法，传入当前选中的标签和大小等级，共支持[0-4]5个等级
4. 下载时只会把视口中的图谱进行保存，所以如果要下载完整图谱需要要适当调整图谱大小和缩放比例。
5. 本组件中预留一个slot插槽，可以根据需要添加新的组件

