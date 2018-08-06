<template>
  <div :id="id">
  </div>
</template>

<script>
import * as d3 from 'd3'

export default {
  name: 'tree',

  props: {
    id: {
      type: String,
      required: true
    },
    data: {
      type: Object,
      required: true
    },
    duration: {
      type: Number,
      required: false,
      default: 750
    },
    children: {
      type: String,
      required: false,
      default: 'children'
    },
    width: {
      type: Number,
      required: false,
      default: 500
    },
    height: {
      type: Number,
      required: false,
      default: 500
    },
    'custom-class': {
      type: Function,
      required: false,
      default () { return '' }
    }
  },

  data () {
    return {
      canvas: null,
      tree: null,
      root: null,
      count: 0
    }
  },

  watch: {
    data () {
      this.count = 0

      this.root = d3.hierarchy(this.data, d => d[this.children])
      this.root.x0 = this.height / 2
      this.root.y0 = 0

      this.update(this.root)
    }
  },

  mounted () {
    this.canvas = d3.select(`#${this.id}`)
      .append('svg')
      .attr('width', this.width)
      .attr('height', this.height)
      .append('g')
      .attr('transform', 'translate(50, 50)')

    this.tree = d3.tree()
      .size([this.height, this.width])

    this.root = d3.hierarchy(this.data, d => d[this.children])
    this.root.x0 = this.height / 2
    this.root.y0 = 0

    this.update(this.root)
  },

  methods: {
    update (source) {
      const treeData = this.tree(this.root)

      const nodes = treeData.descendants()
      const links = treeData.descendants().slice(1)

      const depth = Math.max(...nodes.map(n => n.depth)) + 1
      nodes.forEach(d => { d.y = d.depth * (this.width / depth) })

      const node = this.canvas.selectAll(`g.node`)
        .data(nodes, d => d.id || (d.id = this.count++))
      const link = this.canvas.selectAll('path.link')
        .data(links, d => d.id)

      this.refreshNodes(source, node)
      this.refreshLinks(source, link)

      // save origin for transition
      nodes.forEach(d => {
        d.x0 = d.x
        d.y0 = d.y
      })
    },

    refreshNodes (source, node) {
      // enter
      const nodeEnter = node.enter()
        .append('g')
        .attr('class', 'node')
        .attr('cursor', 'pointer')
        .attr('transform', d => `translate(${source.y0}, ${source.x0})`)
      nodeEnter.append('circle')
        .attr('r', 0)
        .on('click', d => {
          if (d.children) {
            d._children = d.children
            d.children = null
          } else {
            d.children = d._children
            d._children = null
          }
          this.update(d)
        })
      nodeEnter.append('text')
        .text(d => d.data.name)
        .attr('x', d => d.children || d._children ? -13 : 13)
        .attr('y', 4)
        .attr('text-anchor', d => d.children || d._children ? 'end' : 'start')
        .style('fill-opacity', 0)
        .on('click', d => this.$emit('click', d.data))

      // update
      const nodeUpdate = nodeEnter.merge(node)
      nodeUpdate.transition()
        .duration(this.duration)
        .attr('transform', d => `translate(${d.y}, ${d.x})`)
      nodeUpdate.selectAll('circle')
        .transition()
        .duration(this.duration)
        .attr('class', d => `node-circle${d._children ? '-children' : ''}`)
        .attr('r', 7)
      nodeUpdate.selectAll('text')
        .transition()
        .duration(this.duration)
        .style('fill-opacity', 1)

      // exit
      const nodeExit = node.exit()
        .transition()
        .duration(this.duration)
        .attr('transform', d => `translate(${source.y}, ${source.x})`)
        .remove()
      nodeExit.select('circle')
        .attr('r', 0)
      nodeExit.select('text')
        .style('fill-opacity', 0)
    },

    refreshLinks (source, link) {
      // enter
      const linkEnter = link.enter()
        .insert('path', 'g')
        .attr('class', d => this.customClass(d.data))
        .classed('link', true)
        .attr('fill', 'none')
        .attr('stroke', 'gray')
        .attr('d', () => {
          const origin = {x: source.x0, y: source.y0}
          return this.diagonal(origin, origin)
        })

      // update
      const linkUpdate = linkEnter.merge(link)
      linkUpdate.transition()
        .duration(this.duration)
        .attr('d', d => this.diagonal(d, d.parent))

      // exit
      link.exit()
        .transition()
        .duration(this.duration)
        .attr('d', () => this.diagonal(source, source))
        .remove()
    },

    diagonal (s, d) {
      return `M ${s.y} ${s.x}
            C ${(s.y + d.y) / 2} ${s.x},
              ${(s.y + d.y) / 2} ${d.x},
              ${d.y} ${d.x}`
    }
  }
}
</script>