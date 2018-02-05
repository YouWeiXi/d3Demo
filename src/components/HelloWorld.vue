<template>
  <div class="hello">
  <el-row>
    <el-col :span="6" :offset="9">
      <el-input v-model="name" placeholder="请输入内容" @keyup.native="search" @blur.native="search1"></el-input>
    </el-col>
    <el-col :span="1">
      <el-button type="primary" @click="search">查询</el-button>
    </el-col>
  </el-row>
    <el-row>
      <el-col :span="3">
        <el-menu
        @select="handleSelect">
        <el-menu-item index="1">
          <i class="el-icon-circle-plus"></i>
          <span slot="title">增加节点</span>
        </el-menu-item>
        <el-menu-item index="2">
          <i class="el-icon-remove"></i>
          <span slot="title">删除节点</span>
        </el-menu-item>
        <el-menu-item index="3">
          <i class="el-icon-circle-plus"></i>
          <span slot="title">增加连线</span>
        </el-menu-item>
        <el-menu-item index="4">
          <i class="el-icon-remove"></i>
          <span slot="title">删除连线</span>
        </el-menu-item>
        <el-menu-item index="5">
          <i class="el-icon-remove"></i>
          <span slot="title">圆形布局</span>
        </el-menu-item>
        <el-menu-item index="6">
          <i class="el-icon-remove"></i>
          <span slot="title">矩形布局</span>
        </el-menu-item>
        <!-- <el-menu-item index="7">
          <i class="el-icon-remove"></i>
          <span slot="title">树形布局</span>
        </el-menu-item> -->
      </el-menu>
    </el-col>
    <el-col :span="20">
      <div class="graph-area"></div>
    </el-col>
  </el-row>
  <el-dialog
    title="提示"
    :visible.sync="dialogVisible"
    width="30%">
    <span>这是一段信息</span>
    <span slot="footer" class="dialog-footer">
      <el-button @click="dialogVisible = false">取 消</el-button>
      <el-button type="primary" @click="dialogVisible = false">确 定</el-button>
    </span>
  </el-dialog>
</div>
</template>

<script>
  import * as d3 from 'd3';
  import { mapState } from 'vuex';
  import nodeIcon from "../assets/node.png";
  export default {
    name: 'HelloWorld',
    data () {
      return {
        width: 1150,
        height: 700,
        init: null,
        name: '',
        dialogVisible: false
      }
    },
    mounted() {
      this.graphInit();
    },
    computed: {
      ...mapState({
        json: state => state.json
      })
    },
    methods: {
      search() {
        let id = this.name;
        let nodes, links;
        
        if (id) {
          links = this.json.links.filter((item)=> {
            return item.source.id.toLowerCase().includes(id.toLowerCase()) ||
                   item.target.id.toLowerCase().includes(id.toLowerCase())
          })
          let linksNames = [];
          links.forEach(item => {
            linksNames.push(item.target.id.toLowerCase(), item.source.id.toLowerCase());
          })
          linksNames = Array.from(new Set(linksNames));

          nodes = this.json.nodes.filter((item)=> {
            return linksNames.includes(item.id.toLowerCase())
          })
          
        }
        if (links && nodes) {
          this.graphInit({
            nodes: nodes,
            links: links
          })
        } else {
          this.graphInit()
        }
      },
      handleSelect(key, keyPath) {
        // 清楚增加节点事件和鼠标样式
        function unbindAddNode() {
          d3.select('.graph-area').on('click.add-node-ev', null).style('cursor', 'default');
        }
        // 清楚增加连线的绑定事件
        function unbindAddLink(node) {
          node.on("mousedown.add-link", null);
          node.on("mouseup.add-link", null);
          d3.select(".graph-area").on("mousedown.add-link", null);
          d3.select(".graph-area").on("mousemove.add-link", null);
          d3.select(".graph-area").on("mouseup.add-link", null);
        }
        unbindAddNode();
        unbindAddLink(this.node(this.json, this.init));

        switch(key) {
          case '1': this.addNode()
          break;
          case '2': this.deleteNode(this.json, this.update)
          break;
          case '3': this.addLink(this.json, this.update)
          break;
          case '4': this.deleteLink(this.json, this.update)
          break;
          case '5': this.circleLayout(this.json)
          break;
          case '6': this.rectLayout(this.json)
          break;
          // case '7': this.treeLayout(this.json)
          // break;
          default: console.log('default');
        }
      },
      addNode() {
        let that = this;
        d3.select('.graph-area').style('cursor', 'crosshair');
        d3.select('.graph-area').on('click.add-node-ev', function() {
          let { translate, scale } = that.getTranslateAndScale();
          let [x, y] = d3.mouse(this);
          let newNodeId = +new Date();
          let newNode = {id: newNodeId, group: [], isNew: true};
          that.json.nodes.push(newNode);
          that.update(that.json);
          let [tx, ty] = [x / scale - +translate[0] / scale, y / scale - +translate[1] / scale];

          d3.select("g#node-" + newNodeId).attr("transform", `translate(${tx},${ty})`)
                .datum(Object.assign(d3.select("g#node-" + newNodeId).data()[0], { "x": tx, "y": ty }));
        })
      },
      deleteNode(json, update, selNode) {
        console.log('deleteNode');
        let needDelLinks = new Set();
        selNode = selNode || d3.selectAll('.node.active').data();
        console.log(selNode)
        if (selNode.length == 1) {
          json.nodes.forEach(function(d, i) {
            if (d['id'] == selNode[0].id) {
              json.nodes.splice(i, i)
            }
          })
          for (let linkIdx = 0; linkIdx < json.links.length; linkIdx++) {
            if (json.links[linkIdx].source.id == selNode[0].id || json.links[linkIdx].target.id == selNode[0].id) {
              json.links.splice(linkIdx, 1);
              linkIdx--
            }
          }
          this.update(this.json);
        } else {
          this.$message({
            message: '请选择一个节点',
            type: 'warning'
          });
        }
      },
      addLink() {
        let that = this;
        let json = this.json;
        let vis = this.init;
        let node = this.node(json, vis);
        let mousedownNode = null,
            mouseupNode = null;
        let dragLine = vis.append('line').attr('class', 'drag-line');
        node.on('mousedown.add-link', function(d) {
          mousedownNode = d;
          dragLine.attr('class', 'drag-line')
                  .lower()
                  .attr('x1', d['x'])
                  .attr('y1', d['y'])
                  .attr('x2', d['x'])
                  .attr('y2', d['y']);
          d3.event.stopPropagation();
        })
        .on('mouseup.add-link', function(d) {
          dragLine.attr('class', 'drag-line-hidden')
          if (mousedownNode) {
            mouseupNode = d;
            if (mouseupNode == mousedownNode) {
              mousedownNode = null;
              mouseupNode = null;
              return;
            }
            let link = { source: mousedownNode, target: mouseupNode};
            let hasLink = json.links.findIndex(function(linkItem) {
              return linkItem.source.id == mousedownNode.id && linkItem.target.id == mouseupNode.id
            }) > -1;
            let hasOppositeLink = json.links.findIndex(function(linkItem) { return linkItem.source.id == mouseupNode.id && linkItem.target.id == mousedownNode.id }) > -1;
            if (!hasLink) {
              // 如果是反向边 则合并为一条双向边
              if (hasOppositeLink) {
                json.links.forEach(function(linkItem) {
                  linkItem.source.id == mouseupNode.id && linkItem.target.id == mousedownNode.id && (linkItem["isTwoWay"] = true);
                });
              } else {
                json.links.push(link);
              }
              that.update(json);
            } else {
              that.$message({
                message: '已经有连线了，不能重复添加',
                type: 'warning'
              });
            }
          }
        });
        d3.select('.graph-area')
          .on('mousedown.add-link', function() {
            if (!mousedownNode) return;
          })
          .on('mousemove.add-link', function() {
            if (!mousedownNode) return;
            let { translate, scale } = that.getTranslateAndScale();
            let [ x, y ] = d3.mouse(this);
            dragLine.attr('x1', mousedownNode['x'])
                    .attr('y1', mousedownNode['y'])
                    .attr('x2', `${x / scale- +translate[0] / scale}`)
                    .attr('y2', `${y / scale- +translate[1] / scale}`)
            d3.event.preventDefault();
          })
          .on('mouseup.add-link', function() {
            if (mousedownNode) {
              dragLine.attr('class', 'drag-line-hidden')
            }
            mousedownNode = null;
            mouseupNode = null;
          });
      },
      deleteLink(json, update, selLink) {
        selLink = selLink || d3.selectAll('.link.active').data();
        if (selLink.length == 1) {
          json.links.forEach(function(d, i) {
            if (d['source']['id'] == selLink[0]['source']['id'] && d['target']['id'] == selLink[0]["target"]["id"]) {
              json.links.splice(i, 1);
            }
          })
          update(json);
        } else {
          this.$message({
            message: '请选择一条连线！',
            type: 'warning'
          });
        }
      },
      circleLayout(json) {
        if (json.nodes.length) {
          let centerPoint = [this.width / 2, this.height / 2];
          let radian = 360 / json.nodes.length * Math.PI / 180;
          let radius = json['nodes'].length * 10;
          let endCordinates = {};
          json.nodes.forEach(function(nodeItem, nodeIdx) {
            endCordinates['x' + nodeIdx] = radius * Math.cos(radian * nodeIdx) + centerPoint[0];
            endCordinates['y' + nodeIdx] = radius * Math.sin(radian * nodeIdx) + centerPoint[1];
          })
          this.initAnimate(endCordinates);
        }
      },
      rectLayout(json) {
        let that = this;
        if (json.nodes.length) {
          let endCordinates = {};
          let sqr = Math.floor(Math.sqrt(json.nodes.length));
          let count4row = 0;
          let row = 0;
          d3.selectAll('.node').each(function(d, i) {
            endCordinates['x' + i] = that.width * 1 / 5 + count4row * (that.width * 4 / 5 / (sqr));
                endCordinates['y' + i] = that.height * 1 / 6 + row * (that.height * 5 / 6 / (sqr + 2));
                if (count4row < sqr - 1) {
                    count4row++;
                } else {
                    count4row = 0;
                    row++;
                }
          });
          this.initAnimate(endCordinates);
        }
      },
      // treeLayout(json) {
      //   let that = this;
      //   let root = d3.hierarchy(json);
      //   let treeLayout = d3.tree();
      //   treeLayout.size([400, 200]);
      //   treeLayout(root);

      // },
      initAnimate(endCordinates) {
        let circleLayoutTimer;
        d3.selectAll('.node').each(function(nodeItem, nodeIdx) {
          d3.select(this).transition().duration(700).ease(d3.easeCircleInOut).attr("transform", `translate(${endCordinates['x' + nodeIdx]},${endCordinates['y' + nodeIdx]})`)
        })
        d3.selectAll('.link').each(function(nodeItem, nodeIdx) {
          d3.select(this).transition().duration(700).ease(d3.easeCircleInOut)
            .attr('x1', endCordinates['x' + nodeItem['source']['index']])
            .attr('y1', endCordinates['y' + nodeItem["source"]["index"]])
            .attr('x2', endCordinates['x' + nodeItem["target"]["index"]])
            .attr('y2', endCordinates['y' + nodeItem["target"]["index"]]);
        });
        circleLayoutTimer && clearTimeout(circleLayoutTimer);
        circleLayoutTimer = setTimeout(function() {
          d3.selectAll('.node').each(function(nodeItem, nodeIdx) {
            nodeItem.x = endCordinates['x' + nodeIdx];
            nodeItem.y = endCordinates['y' + nodeIdx];
            nodeItem.fx = nodeItem.x;
            nodeItem.fy = nodeItem.y;
          });
        }, 350);
      },
      graphInit(json)   {
        json = json || this.json;
        let width = this.width;
        let height = this.height;
        let force = this.force();
        force.alphaTarget(.1);
        force.restart();
        this.update(json);
        this.autoZoom(json);
      },
      update(json) {
        let vis = this.vis();
        this.init = vis;
        let force = this.force();

        force.nodes(json.nodes);
        force.force('link').links(json.links);
        let link = this.link(json, vis);
        let node = this.node(json, vis);
        let linetext = this.linetext(json, vis);
        // let bindEvent = this.bindEvent(json, update, vis, force, node, link);
        this.bindLinkAndNodeEvent(json, this.update, vis, node, link);
        force.on('tick', () => {
          this.tick(link, node, linetext)
        })
      },
      vis() {
        d3.select('.graph-area').select('*').remove();
        let zoom = d3.zoom()
        .scaleExtent([0.01, 10])
        .on('zoom', () => {
         vis.attr('transform', () => (d3.event.transform))
       });
        let vis = d3.select('.graph-area')
        .append('svg')
        .attr('id', 'J_SvgView')
        .attr('width', this.width)
        .attr('height', this.height)
        .call(zoom)
        .on('dblclick.zoom', null)
        .append('g')
        .attr('class', 'all')
        .attr('data-width', this.width)
        .attr('data-height', this.height);

        return vis;
      },
      force() {
       return d3.forceSimulation([])
                .force('link', d3.forceLink([]).id((d) => (d.id)).distance(100))
                .force('change', d3.forceManyBody().strength(-2500))
                .force('center', d3.forceCenter(this.width / 2, this.height / 2))
                .force('x', d3.forceX())
                .force('y', d3.forceY())
                .force('collide', d3.forceCollide().strength(0.2).iterations(5));
     },
     link(json, vis) {
      let arrow = vis.append('defs')
      .selectAll('marker');
      arrow.data(['start-arrow']).enter().append('marker')
      .attr('id', d => d)
      .attr('class', 'arrow')
      .attr('viewBox', '0 -5 10 10')
      .attr('refX', -7)
      .attr('refY', 0)
      .attr('markerWidth', 10)
      .attr('markerHeight', 16)
      .attr('orient', 'auto')
      .append('path')
      .attr('d', 'M0,0L10,5L10,-5')
      .attr('fill', '#666');

      arrow.data(["end-arrow"]).enter().append("svg:marker")
      .attr("id", d=>d)
      .attr('class', 'arrow')
      .attr("viewBox", "0 -5 10 10")
      .attr("refX", 17)
      .attr("refY", 0)
      .attr("markerWidth", 10)
      .attr("markerHeight", 16)
      .attr("markerUnits", "userSpaceOnUse")
      .attr("orient", "auto")
      .append("svg:path")
      .attr("d", "M0,-5L10,0L0,5")
      .attr('fill', '#666');

      let link = vis.selectAll('line.link');
      link = link.data(json.links, (d) => {
        `${d["source"]["id"]}_${d["target"]["id"]}`
      });
      link.exit().remove();
      link = link.enter()
      .append('line')
      .lower()
      .attr("class", "link")
      .merge(link)
      .attr("id",(d) => {
        `link-${d["source"]["index"]}_${d["target"]["index"]}`
      })
      .attr("marker-start",d=>d.source.index===d.target.index ? false :(d["isTwoWay"] ? "url(#start-arrow)" : "") )
      .attr("marker-end",d=>d.source.index===d.target.index ? false :"url(#end-arrow)")
      .attr("stroke", "#5bc0de")
      .attr("stroke-width",0.7)
      .attr("fill","none");
      return link;
    },
    node(json, vis) {
      let colors = d3.schemeCategory10,
      groupMap = {};
      let groups = [...new Set(json.nodes.reduce((prev, cur) => (prev.concat(cur.group)), []))]
      groups.forEach((groupItem, groupIdx) => { groupMap[groupItem] = colors[groupIdx] });
      let node = vis.selectAll('g.node');
      node = node.data(json.nodes, d => d.id)
      node.exit().remove();
      node = node.enter()
      .append('g')
      .attr('class', d => (d['isNew'] ? 'node new-node' : 'node'))
      .attr('id', d => ('node-' + d['id']))
      .merge(node);
      node.selectAll('.node-bg').remove();
      node.selectAll('.node-icon').remove();
      node.selectAll(".node-name").remove();
      node.selectAll(".node-group").remove();

      node.append('svg:circle')
      .attr("class","node-bg")
      .attr("r", "12px")
      .attr("fill", "#5bc0de");
      node.append("svg:image")
      .attr("class", "node-icon")
      .attr("xlink:href", nodeIcon)
      .attr("x", "-10px")
      .attr("y", "-10px")
      .attr("width", "20px")
      .attr("height", "20px");
      node.append("svg:text")
      .attr("class", "node-name")
      .attr("dy", "25px")
      .attr("text-anchor", "middle")
      .attr("fill", "#5bc0de")
      .text(d => (d["id"]));
      node.append("svg:text")
      .attr("class", "node-group")
      .attr("dy", d => (d["id"] ? "40px" : "25px"))
      .attr("text-anchor", "middle")
      .attr("font-size", "12px")
      .attr("fill", "#286090")
      .html(d => d["group"].length ? ("【" + d["group"].reduce((prev, cur) => (`${prev}<tspan fill=${groupMap[cur]}>${cur}</tspan>` + " "), "") + "】").replace(/\s+】/, "】") : "");
      return node;
    },
    linetext(json, vis) {
      let linetext = vis.selectAll('.linetext')
      linetext.data(json.links);
      linetext.exit().remove();
      linetext.enter()
      .append('text')
      .attr('class', 'linetext')
      .merge(linetext)
      .attr('startOffset', '50%')
      .attr('text-anchor', 'middle')
      .attr('xlink:href', (d) => (d.source.index === d.target.index ? false : `#link-${d.source.index}_${d.target.index}`))
      .text(d => d['relation'])
      .attr('fill', '#5bc0de')
      return linetext;
    },
    autoZoom(json) {
      zoomTimeout && clearTimeout(zoomTimeout);
      let zoomTimeout = setTimeout(() => {
        let yMin = +Infinity,
        yMax = -Infinity,
        defaultScale = 1;
        json.nodes.forEach(d => {
          if (d['y'] < yMin) yMin = d['y'];
          if (d['y'] > yMax) yMax = d['y'];
        });
        let graphOffsetHeight = +yMax - +yMin,
        graphClientHeight = this.height;
        let scale = 1 / (graphOffsetHeight / graphClientHeight);
        zoomInterval && clearInterval(zoomInterval);
        let zoomInterval = setInterval(function () {
          if (defaultScale > scale) {
            defaultScale -= 0.05;
            d3.zoom().on("zoom", () => {
              d3.select('.all').attr("transform", () => (d3.event.transform))
            }).scaleTo(d3.select("#J_SvgView"), defaultScale)
          } else {
            clearInterval(zoomInterval);
          }
        })
      }, 700);
    },
    tick(link, node, linetext) {
      link.attr('x1', (d) => { return d.source.x; })
      .attr('y1', (d) => { return d.source.y; })
      .attr('x2', (d) => { return d.target.x; })
      .attr('y2', (d) => { return d.target.y; })
      linetext.attr('x', d => (d.source.x + d.target.x) / 2)
      .attr('y', d => (d.source.x + d.target.y) / 2);
      node.attr('transform', d => ('translate(' + d.x + ',' + d.y + ')'))
    },
    bindEvent(json, update, vis, force, node, link) {
      console.log('update')
      console.log(update)
      // 侧边栏按钮切换

    },
    bindLinkAndNodeEvent(json, update, vis, node, link) {
      let that = this;
      
      
      node.on('contextmenu', function(d) {
        that.isShowContextMenu(d, node);
        
        d3.select('.edit').on('click', function() {
          that.dialogVisible = true;
          that.isShowContextMenu(null, node);
        })
        d3.event.preventDefault();
        d3.event.stopPropagation();
      })
      .on('mouseover', function(d) {
        node.mouseoutTimeout && clearTimeout(node.mouseoutTimeout);
        node.mouseoutTimeout = null;
      })
      .on('mouseout', function(d) {
        node.mouseoutTimeout && clearTimeout(node.mouseoutTimeout);
        node.mouseoutTimeout = setTimeout(() => { that.isShowContextMenu(null, node) }, 100)
      })
      .on('click', function() {
        let thisNode = ' ' +  this.getAttribute('class') + ' ';
        if (thisNode.indexOf(' active ') != -1) {
          d3.select(this).classed('active', false)
        } else {
          d3.select(this).classed('active', true)
        }
      })
      link.on('click', function() {
        let thisLink = ' ' +  this.getAttribute('class') + ' ';
        if (thisLink.indexOf(' active ') != -1) {
          d3.select(this).classed('active', false)
        } else {
          d3.select(this).classed('active', true)
        }
      })
    },
    isShowContextMenu(obj, node) {
      d3.select('.graph-contextmenu').remove();
      let contextmenu = d3.select("body").append("div").attr("class", "graph-contextmenu");
      if (obj) {
        let html = `<div class="contextmenu">
                      <h3 class="edit">编辑</h3>
                    </div>`;
        contextmenu.html(html)
                   .style('left', `${d3.event.pageX}px`)
                   .style('top', `${d3.event.pageY}px`)
                   .style('display', 'block');
      } else {
        contextmenu.style('display', 'none')
      }
      contextmenu.on('mouseover', function() {
        node.mouseoutTimeout && clearTimeout(node.mouseoutTimeout);
      })
      .on('mouseout', function() {
        node.mouseoutTimeout && clearTimeout(node.mouseoutTimeout);
        node.mouseoutTimeout = setTimeout(() => {
          contextmenu.style('display', 'none')
        }, 100)
      })
    },
    editNode(d) {
      console.log(d);
    },
    getTranslateAndScale() {
      let transform = document.getElementsByClassName('all')[0].getAttribute('transform');
      let matchArr = transform && /translate/.test(transform) && /scale/.test(transform) && transform.match(/translate\(([^\)]+)\)\s?scale\(([^\)]+)/);
      let translate = matchArr && matchArr[1].split(",") || [0, 0];
      let scale = matchArr && matchArr[2] || 1;
      return {translate, scale};
    }
  }
}
</script>

<style>
  /*.graph-area{
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    height: auto;
  }

  .fn-toolbar {
    position: absolute;
    left: 27px;
    top: 100px;
    z-index: 19871005;
  }*/

  .graph-area {
    overflow: hidden;
  }

  .graph-area .node .node-bg {
    display: none;
  }

  .graph-area .node.active .node-bg {
    display: block;
  }
  .graph-area .drag-line{
    stroke-width:1px;
    stroke:#5bc0de;
    cursor: crosshair;  

  }
  .graph-area .drag-line-hidden{
    stroke-width:0; 
    cursor: crosshair;  
  }

  .graph-area .link:hover{
    stroke:#86b7f9;
    stroke-width:10px;
  }

  .graph-area .link.active{
    stroke:#1c7dfa; 
    stroke-width:5px;
  }

  .graph-contextmenu {
    font-family: "microsoft yahei", "simsun";
    font-size: 12px;
    width: 170px;
    height: auto;
    z-index: 2;
    position: absolute;
    background: #fff;
    border-radius: 5px;
    box-shadow: 0 5px 10px rgba(0, 0, 0, .2);
  }

  .graph-contextmenu .title {
    color: #fff;
    padding: 5px;
    font-size: 14px;
    background-color: #337ab7;
    border-radius: 5px 5px 0 0;
  }

  .graph-contextmenu .context-menu-item {
    width: 100%;
    border-collapse: collapse;
    border: 1px solid #337ab7;
  }

  .graph-contextmenu .context-menu-item td {
    padding: 3px 5px;
    color: #666;
    vertical-align: middle;
  }

  .graph-contextmenu .context-menu-item tr:nth-of-type(odd) {
    background: #f9f9f9;
  }

  .tooltip .context-menu-item td.td-label {
    color: #333;
    width: 42px;
  }

  .graph-contextmenu .context-menu-item td a {
    color: #666
  }

  .graph-contextmenu .menu-item-btn {
    cursor: pointer;
  }

  .graph-contextmenu .menu-item-btn:hover {
    color: #337ab7
  }
</style>
