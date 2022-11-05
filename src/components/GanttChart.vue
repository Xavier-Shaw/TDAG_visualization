<template>
  <div>
  </div>
</template>

<script>

import * as d3 from "d3";

export default {
  name: "GanttChart",

  data() {
    const width = 1000;
    let height = 600;
    let dynamic_canvas_height = 200
    const padding = 60;
    const bar_height = 15;
    const text_height = 12;
    let X_MAX = 40;

    let root_tasks = [
      {
        name: 'Map-3',
        begin_time: 0,
        duration: 3,
        workstation: 1,
        children: ['Map-1', 'Map-8']
      },
      {
        name: 'Map-4',
        begin_time: 0,
        duration: 3,
        workstation: 2,
        children: ['Map-6', 'Map-18']
      },
      {
        name: 'Map-5',
        begin_time: 0,
        duration: 3,
        workstation: 3,
        children: ['Map-16', 'Map-14']
      },
      {
        name: 'Map-10',
        begin_time: 0,
        duration: 11,
        workstation: 4,
        children: ['Map-12']
      }
    ];

    let tasks = [
      {
        name: 'Map-1',
        begin_time: 20,
        duration: 6,
        workstation: 5,
        children: []
      },
      {
        name: 'Map-8',
        begin_time: 27,
        duration: 9,
        workstation: 6,
        children: []
      },
      {
        name: 'Map-6',
        begin_time: 23,
        duration: 10,
        workstation: 7,
        children: []
      },
      {
        name: 'Map-18',
        begin_time: 31,
        duration: 8,
        workstation: 8,
        children: []
      },
      {
        name: 'Map-16',
        begin_time: 16,
        duration: 6,
        workstation: 9,
        children: []
      },
      {
        name: 'Map-14',
        begin_time: 12,
        duration: 7,
        workstation: 10,
        children: []
      },
      {
        name: 'Map-12',
        begin_time: 8,
        duration: 8,
        workstation: 11,
        children: []
      },
    ];

    let placed_vertexes = []

    let xScale = d3.scaleLinear()
        .domain([0, X_MAX])
        .range([padding, width - padding])

    return {
      tasks, root_tasks, width, height, dynamic_canvas_height , padding, bar_height, text_height, placed_vertexes, X_MAX, xScale
    }

  },

  methods: {
    draw(vertex, svg) {
      svg.append("rect")
          .attr("x", this.xScale(vertex.begin_time))
          .attr("y", vertex.y)
          .attr("width", this.xScale(vertex.begin_time + vertex.duration) - this.xScale(vertex.begin_time))
          .attr("height", this.bar_height)
          .style("fill", '#b2daaf')
          .append("title")
          .text('Name: ' + vertex.name + '\rBegin time: ' + vertex.begin_time + '\rDuration: ' + vertex.duration + '\rWorkstation: ' + vertex.workstation)

      svg.append("text")
          .attr("x", this.xScale(vertex.begin_time) + 3)
          .attr("y", vertex.y - 1)
          .text(vertex.name)
          .style("font-size", 12)
    },

    check_overlap(vertex) {
      // let _this = this
      let overlap = false
      this.placed_vertexes.forEach((placed_vertex) => {
        let x_overlap = (placed_vertex.begin_time <= vertex.begin_time && vertex.begin_time <= (placed_vertex.begin_time + placed_vertex.duration))
            || (vertex.begin_time <= placed_vertex.begin_time && placed_vertex.begin_time <= (vertex.begin_time + vertex.duration))
        let y_overlap = (placed_vertex.y === vertex.y)

        if (x_overlap && y_overlap) {
          overlap = true
        }
      })

      return overlap
    },

    get_vertex(vertex_name) {
      let v = null
      this.tasks.forEach((vertex) => {
        if (vertex.name === vertex_name) {
          v = vertex
        }
      })

      return v
    },


    process(vertex) {
      let _this = this

      vertex.children.forEach((child_name) => {
        let child = _this.get_vertex(child_name)
        _this.process(child)
      })

      let place_success = this.place(vertex)
      if (!place_success) {
        this.dynamic_canvas_height += this.bar_height
      }

      this.placed_vertexes.push(vertex)
    },


    place(vertex) {
      vertex.y = this.padding

      while (vertex.y + this.bar_height < this.dynamic_canvas_height - this.padding) {
        let overlap = this.check_overlap(vertex)

        if (overlap) {
          vertex.y = vertex.y + this.bar_height + this.text_height
        }
        else {
          break
        }
      }

      return vertex.y < (this.dynamic_canvas_height - this.padding);
    },


    original_layout() {

      let _this = this
      this.root_tasks.forEach((root) => {
        _this.process(root)
      })

      let svg = d3.select("div")
          .append("svg")
          .attr("width", this.width)
          .attr("height", this.dynamic_canvas_height)

      this.root_tasks.forEach((t) => {
        _this.draw(t, svg)
      })

      this.tasks.forEach((t) => {
        _this.draw(t, svg)
      })

      // 加入坐标轴
      const xAxis = d3.axisTop(this.xScale);

      svg.append("g")
          .attr("transform", "translate(0," + (this.padding - 12) + ")")
          .call(xAxis);

      svg.append("text")
          .attr("x", this.xScale(this.X_MAX / 2) - 10)
          .attr("y", 25)
          .text("Time")
    },


    judge_crossing(x1, y1, x2, y2, x3, y3, x4, y4) {
      let y_axis_overlap = (Math.min(y1, y2) < Math.min(y3, y4) < Math.max(y1, y2)) || (Math.min(y3, y4) < Math.min(y1, y2) < Math.max(y3, y4))

      let x_axis_overlap = (Math.min(x1, x2) < Math.min(x3, x4) < Math.max(x1, x2)) || (Math.min(x3, x4) < Math.min(x1, x2) < Math.max(x3, x4))

      let axis_overlap = y_axis_overlap && x_axis_overlap

      // (AC x CD) * (BC x CD)
      let vector_product = ((x1 - x3) * (y3 - y4) - (x3 - x4) * (y1 - y3)) * ((x2 - x3) * (y3 - y4) - (x3 - x4) * (y2 - y3))

      return axis_overlap && (vector_product < 0)
    },

    cal_distance(x1, y1, x2, y2) {
      return Math.sqrt(Math.pow((y2 - y1), 2) + Math.pow((x2 - x1), 2))
    },

    // 简单的样例
    sample() {

      let svg = d3.select("div")
          .append("svg")
          .attr("width", this.width)
          .attr("height", this.height);

      let Y_MAX = 14;

      let XScale = d3.scaleLinear()
          .domain([0, this.X_MAX])
          .range([this.padding, this.width - this.padding])

      let YScale = d3.scaleLinear()
          .domain([0, Y_MAX])
          .range([this.height - this.padding, this.padding])

      svg.selectAll("svg")
          .data(this.root_tasks)
          .enter()
          .append("rect")
          .attr("x", (d) => XScale(d.begin_time))
          .attr("y", (d) => YScale(d.workstation) - this.bar_height / 2)
          .attr("width", (d) => XScale(d.begin_time + d.duration) - XScale(d.begin_time))
          .attr("height", this.bar_height)
          .style("fill", '#b2daaf')
          .append("title")
          .text((d) => 'Name: ' + d.name + '\rBegin time: ' + d.begin_time + '\rDuration: ' + d.duration + '\rWorkstation: ' + d.workstation)

      svg.selectAll("svg")
          .data(this.tasks)
          .enter()
          .append("rect")
          .attr("x", (d) => XScale(d.begin_time))
          .attr("y", (d) => YScale(d.workstation) - this.bar_height / 2)
          .attr("width", (d) => XScale(d.begin_time + d.duration) - XScale(d.begin_time))
          .attr("height", this.bar_height)
          .style("fill", '#95c5ef')
          .append("title")
          .text((d) => 'Name: ' + d.name + '\rBegin time: ' + d.begin_time + '\rDuration: ' + d.duration + '\rWorkstation: ' + d.workstation)

      svg.selectAll("svg")
          .data(this.root_tasks)
          .enter()
          .append("text")
          .attr("x", (d) => XScale(d.begin_time) + 3)
          .attr("y", (d) => YScale(d.workstation) - this.bar_height / 2 - 3)
          .text((d) => d.name)
          .style("font-size", 12)

      svg.selectAll("svg")
          .data(this.tasks)
          .enter()
          .append("text")
          .attr("x", (d) => XScale(d.begin_time) + 3)
          .attr("y", (d) => YScale(d.workstation) - this.bar_height / 2 - 3)
          .text((d) => d.name)
          .style("font-size", 12)

      let lines = []
      // 计算 edge crossings
      let edge_crossing_cnt = 0
      // 计算 logic relation -- 父子间的距离
      let total_distance = 0
      let pairs = 0

      let link = d3.linkHorizontal()
      this.root_tasks.forEach((root) => {
        let end_pos_x = XScale(root.begin_time + root.duration)
        let end_pos_y = YScale(root.workstation)

        root.children.forEach((child_name) => {
          this.tasks.forEach((child) => {
            if (child.name === child_name) {
              let start_pos_x = XScale(child.begin_time)
              let start_pos_y = YScale(child.workstation)

              total_distance += this.cal_distance(end_pos_x, end_pos_y, start_pos_x, start_pos_y)
              pairs += 1

              let link_data = {
                source: [end_pos_x, end_pos_y],
                target: [start_pos_x, start_pos_y]
              }
              lines.push(link_data)
              svg
                  .append("path")
                  .attr("d", link(link_data))
                  .attr("stroke", "black")
                  .attr("fill", "none")
            }
          })
        })
      })

      for (let i = 0; i < lines.length; i++) {
        let line_1 = lines.at(i)
        let x1 = line_1.source[0]
        let y1 = line_1.source[1]
        let x2 = line_1.target[0]
        let y2 = line_1.target[1]

        for (let j = i + 1; j < lines.length; j++) {
          let line_2 = lines.at(j)
          let x3 = line_2.source[0]
          let y3 = line_2.source[1]
          let x4 = line_2.target[0]
          let y4 = line_2.target[1]

          let cross = this.judge_crossing(x1, y1, x2, y2, x3, y3, x4, y4)

          if (cross) {
            edge_crossing_cnt += 1
          }
        }
      }

      // 加入edge count 结果
      svg.append("text")
          .attr("x", this.width - 5 * this.padding)
          .attr("y", this.padding)
          .text("Edge Crossing: " + edge_crossing_cnt)

      // 加入logic relation 结果
      svg.append("text")
          .attr("x", this.width - 5 * this.padding)
          .attr("y", this.padding * 2)
          .text("Average Distance: " + Math.round(total_distance / pairs))

      // 加入坐标轴
      const xAxis = d3.axisBottom(XScale);
      const yAxis = d3.axisLeft(YScale);

      svg.append("g")
          .attr("transform", "translate(0," + (this.height - this.padding) + ")")
          .call(xAxis);

      svg.append("g")
          .attr("transform", "translate(" + this.padding + ", " + 0 + ")")
          .call(yAxis);

      svg.append("text")
          .attr("x", XScale(this.X_MAX / 2) - 10)
          .attr("y", this.height - 6)
          .text("Time")

      svg.append("text")
          .attr("x", 8)
          .attr("y", YScale(Y_MAX / 2))
          .attr("transform", "rotate(-90," + 20 + "," + (YScale(Y_MAX / 2)) + ")")
          .text("Workstation")
    }
  },

  mounted() {
    this.original_layout()
    this.sample()
  }

}
</script>

<style scoped>

</style>