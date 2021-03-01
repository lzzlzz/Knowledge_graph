<template>
  <div class="container">
    <div class="canvas-container">
      <slot></slot>
      <i class="el-icon-download" @click="downLoadGraph"></i>
    </div>
  </div>
</template>

<script>
import * as d3 from "d3";
import { saveSvgAsPng } from "save-svg-as-png";
export default {
  name: "cvs",
  data() {
    return {
      //用于更新画布
      nodes: [],
      links: [],
      nodeText: [],
      linksName: [],
      simulation: null,
      width: 600,
      height: 250,
      //用于判断是更新还是初始化
      initialized: false,
      nodeSizeList: new Map(), //类型与半径的对应关系
      nodeTextLengthList: new Map(), //类型与节点文本个数的对应关系
      scale: d3.scaleOrdinal(d3.schemeCategory10) //用于随机展示节点颜色
    };
  },
  methods: {
    renderGraph(data) {
      if (this.initialized) {
        this.updategraph(data);
      } else {
        this.initGraph(data);
        this.initialized = true;
      }
    },
    initGraph(data) {
      console.log("初始化画布...", data);
      let self = this;
      //将json转化为对象
      const links = data.links; //.map(d => Object.create(d));
      const nodes = data.nodes; //.map(d => Object.create(d));
      // console.log("nodes", nodes);
      //初始化力导向
      self.simulation = d3
        .forceSimulation(nodes)
        .force(
          "link",
          d3
            .forceLink(links)
            .id(d => d.id)
            .distance(100) //连线的长度
        )
        .force(
          "charge",
          d3
            .forceManyBody()
            .strength(-1000)
            .distanceMin(10)
            .distanceMax(800)
        ) //以电荷的方式 开始的力
        //设置碰撞半径 就是两个节点最近的距离
        .force("collision", d3.forceCollide(1).strength(0.1))
        .force("center", d3.forceCenter(self.width / 2, self.height / 2)); //力导向图的中心位置
      //创造容器
      //const svg = d3.create("svg").attr("viewBox", [0, 0, width, height]);
      const svg = d3
        .select(".canvas-container")
        .append("svg")
        .attr("viewBox", [0, 0, self.width, self.height]);

      //设置一个背景层，对这个背景层缩放效果好
      const bg = svg.append("g");

      //为了解决因方向改变而导致的箭头位置问题设置两个箭头样式
      //思路就是双箭头，分别指向source节点和target节点，然后根据方向的不同控制不同箭头的显示
      svg
        .append("marker") //让连线和箭头关联
        .attr("id", "positiveMarker")
        .attr("orient", "auto")
        .attr("stroke-width", 2)
        .attr("markerUnits", "strokeWidth")
        .attr("markerUnits", "userSpaceOnUse")
        .attr("viewBox", "0 -5 10 10") //箭头所在的合适范围
        //refX 标记位置在路径开始处的点
        .attr("refX", 11) //箭头在方向上的偏移 让箭头露出
        .attr("refY", 0) //离左右连线偏移
        .attr("markerWidth", 12)
        .attr("markerHeight", 12)
        .append("path") //增加路径类型标签 注意不是属性
        .attr("d", "M 0 -5 L 10 0 L 0 5") //设置路径
        .attr("fill", "#999") //填充和不可见度
        .attr("stroke-opacity", 0.6);

      svg
        .append("marker") //让连线和箭头关联
        .attr("id", "negativeMarker")
        .attr("orient", "auto")
        .attr("stroke-width", 2)
        .attr("markerUnits", "strokeWidth")
        .attr("markerUnits", "userSpaceOnUse")
        .attr("viewBox", "0 -5 10 10") //箭头所在的合适范围
        .attr("refX", 0) //箭头在方向上的偏移 让箭头露出
        .attr("refY", 0) //离左右连线偏移
        .attr("markerWidth", 12)
        .attr("markerHeight", 12)
        .append("path") //增加路径类型标签 注意不是属性
        .attr("d", "M 0 0 L 10 5 L 10 -5") //设置路径
        .attr("fill", "#999") //填充和不可见度
        .attr("stroke-opacity", 0.6);

      //绑定缩放事件
      svg
        .call(
          d3
            .zoom()
            .scaleExtent([0.1, 2]) //增加缩放范围
            .on("zoom", function() {
              bg.attr("transform", d3.event.transform);
            })
        )
        .on("dblclick.zoom", () => {}); //取消双击缩放

      // console.log(marker);

      //构建连接线样式
      self.links = bg
        .append("g")
        .selectAll("path") //用path 比line更灵活
        .data(links, function(d) {
          if (typeof d.source === "object") {
            return d.source.id + "_" + d.name + "_" + d.target.id;
          } else {
            return d.source + "_" + d.name + "_" + d.target;
          }
        })
        .join("path")
        .attr("class", "link")
        .attr("id", function(d) {
          if (typeof d.source === "object") {
            return d.source.id + "_" + d.name + "_" + d.target.id;
          } else {
            return d.source + "_" + d.name + "_" + d.target;
          }
        }); //作为连线的id,让连线上的文字可以找到相应连线

      //构建连接线的文本层
      self.linksName = bg
        .append("g")
        .selectAll("text")
        .data(links)
        .join("text")
        .attr("class", "linksName")
        .append("textPath")
        .attr("xlink:href", function(d) {
          //因为更新后d.source会变为一个对象，所以判断是否是对象然后取对象的id作为目标链接
          if (typeof d.source === "object") {
            return "#" + d.source.id + "_" + d.name + "_" + d.target.id;
          } else {
            return "#" + d.source + "_" + d.name + "_" + d.target;
          }
        })
        .attr("startOffset", "50%") //属性名写错导致不会生效
        .text(function(d) {
          return d.name;
        });
      //构建节点样式
      self.nodes = bg
        .append("g")
        .selectAll("circle")
        .data(nodes)
        .join("circle")
        .attr("class", "node") //设置在前面，样式就会应用于所有节点
        .attr("r", 32.5)
        .attr("fill", self.color())
        .on("dblclick", self.queryProp) //双击事件获得属性
        .on("click", self.queryTest) //单击事件获得信息面板的关系
        .call(self.drag(self.simulation));
      //增加节点的名字
      self.nodes.append("title").text(d => d.name);

      //节点文本层
      self.nodeText = bg
        .append("g")
        .selectAll("text") //选择所有的text元素
        .data(nodes) //绑定源数据
        .join("text") //连接起来
        .text(d => {
          let name = d.name;
          let len = this.nodeTextLengthList.has(d.type)
            ? this.nodeTextLengthList.get(d.type)
            : 6;
          if (name.length > len) return name.slice(0, len) + "...";
          return d.name;
        })
        .attr("dy", 2)
        .style("pointer-events", "none")
        .attr("class", "node-text");

      //将层与仿真关联起来
      self.simulation.on("tick", () => {
        //设置连接线的跟随
        //根据源节点与目标节点的相对关系设置 路径方向
        self.links
          .attr("d", function(d) {
            //这个是为了解决文字方向的问题
            if (d.source.x < d.target.x) {
              return (
                "M " +
                self.getCoord(d.target, d.source) +
                " L " +
                self.getCoord(d.source, d.target)
              );
            } else {
              return (
                "M " +
                self.getCoord(d.source, d.target) +
                " L " +
                self.getCoord(d.target, d.source)
              );
            }
          })
          //根据方向不同选择显示不同的marker
          //这是由于解决文字方向而引起的附加问题
          //对于尾端，如果是源节点指向目的节点的时候要在尾部加箭头
          .attr("marker-end", function(d) {
            if (d.source.x < d.target.x) {
              return "url(#positiveMarker)";
            } else {
              return null;
            }
          })
          .attr("marker-start", function(d) {
            if (d.source.x > d.target.x) {
              //如果源节点比目的节点大的时候，就认为是目的节点是源节点
              return "url(#negativeMarker)";
            } else {
              return null;
            }
          });

        self.nodes.attr("cx", d => d.x).attr("cy", d => d.y);
        //这个x,y是由什么决定的 是由 text的属性决定的 svg中text标签的属性
        self.nodeText.attr("x", d => d.x).attr("y", d => d.y);
      });
    },
    //给出源节点和目标节点，返回目标节点的边上距离源节点最近的点的坐标
    getCoord(source, target) {
      //根据目标节点的类型，计算路径的长度，让路径指向目标节点的边，而不是圆心

      let r = 30;
      if (this.nodeSizeList.has(target.type)) {
        r = this.nodeSizeList.get(target.type);
      }
      // console.log(r);
      let scale =
        r /
        Math.sqrt(
          Math.pow(target.y - source.y, 2) + Math.pow(target.x - source.x, 2)
        );

      let targetX = target.x - scale * (target.x - source.x);
      let targetY = target.y - scale * (target.y - source.y);
      return targetX + " " + targetY;
    },
    color() {
      return d => this.scale(d.type);
    },
    drag(simulation) {
      function dragstarted(d) {
        if (!d.active) simulation.alphaTarget(0.3).restart();
        //不知道为什么网页上的例子变成了这种写法 event.subject.x; 可能是版本不同，写法不i一样了
        d.fx = d.x;
        d.fy = d.y;
      }

      function dragged(d) {
        d.fx = d3.event.x; //d3事件获得鼠标拖动目标的位置然后付给目标
        d.fy = d3.event.y;
      }

      function dragended(d) {
        if (!d.active) simulation.alphaTarget(0);
        d.fx = null;
        d.fy = null;
      }

      return d3
        .drag()
        .on("start", dragstarted)
        .on("drag", dragged)
        .on("end", dragended);
    },
    queryTest(d) {
      //直接获得点击节点及其数据
      this.$emit("clickNode", d);
    },
    //查询属性
    queryProp(d) {
      this.$emit("dbclickNode", d);
    },
    updategraph(data) {
      console.log("更新图谱数据...", data);
      let self = this;
      //解决重复点击线条残留问题
      const links = data.links;
      const nodes = data.nodes;

      //更新连线
      self.links = self.links
        .data(links, function(d) {
          if (typeof d.source === "object") {
            return d.source.id + "_" + d.name + "_" + d.target.id;
          } else {
            return d.source + "_" + d.name + "_" + d.target;
          }
        })
        .join("path") //设置对象
        .merge(self.links)
        .attr("class", "link")
        .attr("id", function(d) {
          //为了解决再次点击之后 source变为节点而导致的获取不到id的情况
          if (typeof d.source === "object") {
            return d.source.id + "_" + d.name + "_" + d.target.id;
          } else {
            return d.source + "_" + d.name + "_" + d.target;
          }
        });
      //更新连线上的文字
      //构建连接线的文本层
      self.linksName = self.linksName
        .data(links, function(d) {
          if (typeof d.source === "object") {
            return d.source.id + "_" + d.name + "_" + d.target.id;
          } else {
            return d.source + "_" + d.name + "_" + d.target;
          }
        })
        .join("text")
        .attr("class", "linksName")
        .append("textPath")
        .attr(
          "xlink:href",
          function(d) {
            if (typeof d.source === "object") {
              return "#" + d.source.id + "_" + d.name + "_" + d.target.id;
            } else {
              return "#" + d.source + "_" + d.name + "_" + d.target;
            }
          }
          // d => "#" + d.source + "_" + d.name + "_" + d.target
        )
        .attr("startOffset", "50%")
        .merge(self.linksName)
        .text(function(d) {
          return d.name;
        });

      //更新节点
      self.nodes = self.nodes
        .data(nodes)
        .join("circle") //会自动处理增加减少的节点
        .merge(self.nodes)
        .on("dblclick", self.queryProp) //双击事件获得属性
        .on("click", self.queryTest) //单击事件获得信息面板的关系
        .call(self.drag(self.simulation))
        .attr("class", "node") //设置在前面，样式就会应用于所有节点
        .attr("r", function(d) {
          if (self.nodeSizeList.has(d.type)) {
            return self.nodeSizeList.get(d.type);
          }
          return 30;
        })
        .attr("fill", self.color());
      //增加节点的名字
      self.nodes.append("title").text(d => d.name);

      //字没跟着走 ,因为文字层没更新
      //节点文本层
      self.nodeText = self.nodeText
        .data(nodes) //绑定源数据
        .join("text") //连接起来
        .merge(self.nodeText)
        .text(d => {
          let name = d.name;
          let len = this.nodeTextLengthList.has(d.type)
            ? this.nodeTextLengthList.get(d.type)
            : 6;
          if (name.length > len) return name.slice(0, len) + "...";
          return d.name;
        })
        .attr("dy", 2)
        .style("pointer-events", "none")
        .attr("class", "node-text");

      //更新力
      self.simulation.nodes(nodes);
      self.simulation
        .force("link")
        // .id(d => d.index)
        .links(links);
      self.simulation.alpha(1).restart();
    },
    modifyNodeSize(targetLabel, sizeLevel) {
      //更新节点的大小
      const sizesList = [5, 10, 25, 32.5, 40]; //存放级别对应的半径大小
      const lengthList = [0, 1, 3, 4, 7]; //存放级别对应的节点文本长度
      this.nodeSizeList.set(targetLabel, sizesList[sizeLevel]);
      this.nodeTextLengthList.set(targetLabel, lengthList[sizeLevel]);
      // console.log(this.nodeSizeList);

      this.nodes = this.nodes.attr("r", d => {
        if (this.nodeSizeList.has(d.type)) {
          return this.nodeSizeList.get(d.type);
        }
        return 32.5;
      });
      //根据节点的类型确定节点中要展示几个文字
      this.nodeText = this.nodeText.text(d => {
        let name = d.name;
        let len = this.nodeTextLengthList.has(d.type)
          ? this.nodeTextLengthList.get(d.type)
          : 6;
        if (name.length > len) return name.slice(0, len) + "...";
        return d.name;
      });
      this.simulation.alpha(1).restart();
    },
    downLoadGraph() {
      // console.log("下载图谱")
      if (
        !d3
          .select(".canvas-container")
          .select("svg")
          .node()
      ) {
        this.$message({
          showClose: true,
          message: "没有要下载的图谱"
        });
        return;
      }
      let date = new Date();
      let pictureName = "图谱_" + date.toLocaleString() + ".png";
      saveSvgAsPng(
        d3
          .select(".canvas-container")
          .select("svg")
          .node(),
        pictureName,
        {
          scale: 10
        }
      );
    }
  }
};
</script>

<style>
.container {
  display: flex;
}

.canvas-container {
  background: #ffffff;
  border: 1px solid #ffffff;
  border-radius: 4px;
  width: 800px;
  height: 70vh;
  flex: 1 1 auto;
  position: relative;
}

.node {
  stroke: #cfebfe;
  stroke-width: 1;
  cursor: pointer;
}
.node:hover {
  stroke-width: 5px;
}
.node-text {
  fill: #ffffff;
  text-anchor: middle;
  font-size: 10px;
  user-select: none;
}
.link {
  stroke: #999;
  stroke-opacity: 0.6;
  stroke-width: 2.25;
}
.linksName {
  text-anchor: middle;
  fill: #4a5a6a;
  font-size: 9px;
  /* font-weight: bold; */
}
.el-icon-download {
  position: absolute;
  bottom: 10px;
  right: 20px;
  cursor: pointer;
  font: 3em sans-serif;
  color: rgba(192, 192, 192, 0.5);
}
</style>
