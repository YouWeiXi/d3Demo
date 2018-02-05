<template>
  <div class="hello">
    <el-row :gutter="20">
      <el-col :span="10" :offset="2">
        <div class="demo"></div>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import * as d3 from 'd3'
import { mapState } from 'vuex'
export default {
  name: 'HelloWorld',
  data () {
    return {
      width: 1560,
      height: 800
    }
  },
  mounted() {
    this.d3Render();
  },
  computed: {
    ...mapState({
      nodes_data: state => state.nodes_data,
      links_data: state => state.links_data
    })
  },
  methods: {
    d3Render()   {
      let nodes =  this.nodes_data;
      let links = this.links_data;
      let width = this.width;
      let height = this.height;
      let svg = d3.select('.demo')
                .append('svg')
                .attr('width', this.width)
                .attr('height', this.height);
      let simulation = d3.forceSimulation()
                         .nodes(nodes);
      let node = svg.append('g')
                    .selectAll('circle')
                    .data(nodes)
                    .enter()
                    .append('circle')
                    .attr('r', 28)
                    .attr('class', 'node')
                    .style('fill', (d) => {
                      let color;
                      let link = links[d.index];
                      if (d.name == link.source.name && link.rela == '上') {
                        color="#F6E8E9";
                      }else{
                        color="#F9EBF9";
                      }
                      return color;
                    })
                    .style('stroke', (d) => {
                      let color;
                      let link = links[d.index];
                      if (d.name == link.source.name && link.rela == '上') {
                        color="#B43232";
                      }else{
                        color="#A254A2";
                      }
                      return color;
                    })
                    .on('click', (d) => {
                      line.style('stroke-width', (line) => {
                        if (line.source.name == d.name || line.target.name == d.name) {
                          return 4;
                        } else {
                          return 0.5
                        }
                      })
                    });
      let mousedownNode = null;
      let mouseupNode = null;
      let dragLine = svg.append("svg:g")
                        .append('line')
                        .attr('class', 'drag-line');
      let that = this;
      node.on('mousedown', d => {
        mousedownNode = d;
        console.log('mousedown')
        dragLine.attr('class', 'drag-line')
                .lower()
                .attr('x1', d['x'])
                .attr("y1", d["y"])
                .attr("x2", d["x"])
                .attr("y2", d["y"]);
        d3.event.stopPropagation();
      })
      .on('mouseup', d => {
        console.log('mouseup')
        dragLine.attr('class', 'drag-line-hidden')
        if (mousedownNode) {
          mouseupNode = d;
          if (mouseupNode == mousedownNode) {
            mousedownNode = null;
            mouseupNode = null;
            return;
          }
          let link = { source: mousedownNode, target: mouseupNode };
          that.links_data.push(link);
        }
      })

      // d3.select('svg')

      //   .on()

      let marker = svg.append('marker')
                      .attr("id", "resolved")
                      // .attr('id', d => d)
                      .attr('class', 'arrow')
                      .attr('viewBox', '0 -5 10 10')
                      .attr('refX', 17)
                      .attr('refY', 0)
                      .attr("markerWidth", 10)
                      .attr("markerHeight", 16)
                      .attr("markerUnits", "userSpaceOnUse")
                      .attr("orient", "auto")
                      .append("svg:path")
                      .attr("d", "M0,-5L10,0L0,5")
                      .attr('fill', '#666');

      let line = svg.append('g')
                    .selectAll('line')
                    .data(links)
                    // .exit().remove()
                    .enter()
                    // .merge(links)
                    .append('line')
                    // .attr('d', (d) => {
                    //   console.log(d.source)
                    //   // return 'M '+d.source.x+' '+d.source.y+' L '+ d.target.x +' '+d.target.y
                    // })
                    // .attr({
                    //   'd': (d) => {
                    //     console.log(d)
                    //     return 'M '+d.source.x+' '+d.source.y+' L '+ d.target.x +' '+d.target.y
                    //   },
                    //   'class': 'edgepath',
                    //   'id': (d, i) => {
                    //     return 'edgepath' + i;
                    //   }
                    // });
                    .style('stroke', (d) => {
                      let lineColor;
                      if (d.rela == '上' || d.rela == '中') {
                        lineColor = "#A254A2";
                      } else {
                        lineColor = "#B43232";
                      }
                      return lineColor;
                    })
                    .attr("marker-end", "url(#resolved)" );

      simulation.force('charge_force', d3.forceManyBody())
                .force('center_force', d3.forceCenter(width / 2, height / 2));
      let link_force = d3.forceLink(links)
                         .id(function(d) {
                           return d.name;
                         })
                         .distance(180)
      // console.log(line)
      // line.attr('d', (d) => {
      //   console.log(d.source)
      //   // return 'M '+d.source.x+' '+d.source.y+' L '+ d.target.x +' '+d.target.y
      // })
      
      simulation.force('links', link_force);
      simulation.on('tick', tickActions);
      function tickActions() {
        node.attr("cx", function(d) { return d.x = Math.max(5, Math.min(width - 5, d.x)); })
            .attr("cy", function(d) { return d.y = Math.max(5, Math.min(height - 5, d.y)); });
        line.attr('x1', function(d) { return d.source.x; })
            .attr('y1', function(d) { return d.source.y; })
            .attr('x2', function(d) { return d.target.x; })
            .attr('y2', function(d) { return d.target.y; })
      }
    }
  }
}
</script>

<style scoped>
  circle {
    fill: #ccc;
    stroke: #333;
    stroke-width: 1.5px;
  }
  .edgepath {
    stroke: red;
  }
</style>
