<script setup>
import { events } from '../data/events.js'
import { onMounted, ref } from 'vue'
import * as d3 from 'd3'

const timelineContainer = ref(null)

onMounted(() => {
  // Clear previous SVG if any
  if (timelineContainer.value) timelineContainer.value.innerHTML = '';

  // Timeline dimensions
  const margin = { top: 60, right: 40, bottom: 60, left: 80 }
  const width = window.innerWidth - margin.left - margin.right
  const height = window.innerHeight - margin.top - margin.bottom

  // Prepare data
  const minYear = d3.min(events, d => d.start)
  const maxYear = d3.max(events, d => d.end)

  // X scale: years (BC, negative)
  // Make the time scale bigger by increasing the pixel range
  const x = d3.scaleLinear()
    .domain([minYear, maxYear])
    .range([0, width * 2]) // double the width for a bigger time scale

  // Y scale: stack events to avoid overlap
  // Simple greedy stacking: sort by start, assign next available row
  const sorted = [...events].sort((a, b) => a.start - b.start)
  const rows = []
  sorted.forEach(event => {
    let row = 0
    while (rows[row] && rows[row] > event.start) row++
    event._row = row
    rows[row] = event.end
  })
  const rowHeight = 100 // Increased from 70 to 100 for more vertical spacing
  const nRows = d3.max(sorted, d => d._row) + 1

  // SVG
  const svg = d3.select(timelineContainer.value)
    .append('svg')
    .attr('width', width * 2 + margin.left + margin.right) // match the new scale
    .attr('cursor', 'pointer')
    .attr('height', nRows * rowHeight + margin.top + margin.bottom)
    .style('background', '#f8f6f1')

  // Axis (BC labels) - timeline ruler (bottom)
  const axis = d3.axisBottom(x)
    .tickFormat(d => `${Math.abs(d)} BC`)
    .ticks(Math.ceil((maxYear - minYear) / 100))
  svg.append('g')
    .attr('class', 'timeline-ruler bottom')
    .attr('transform', `translate(${margin.left},${nRows * rowHeight + margin.top + 10})`)
    .call(axis)
    .selectAll('text')
    .style('font-family', 'Segoe UI, Arial, sans-serif')
    .style('font-size', '1.1em')
    .style('fill', '#888');

  // Add ruler line (bottom)
  svg.append('line')
    .attr('x1', margin.left)
    .attr('x2', margin.left + width)
    .attr('y1', nRows * rowHeight + margin.top + 10)
    .attr('y2', nRows * rowHeight + margin.top + 10)
    .attr('stroke', '#bbb')
    .attr('stroke-width', 2)

  // Axis (BC labels) - timeline ruler (top)
  svg.append('g')
    .attr('class', 'timeline-ruler top')
    .attr('transform', `translate(${margin.left},${margin.top - 30})`)
    .call(axis)
    .selectAll('text')
    .style('font-family', 'Segoe UI, Arial, sans-serif')
    .style('font-size', '1.1em')
    .style('fill', '#888');

  // Add ruler line (top)
  svg.append('line')
    .attr('x1', margin.left)
    .attr('x2', margin.left + width)
    .attr('y1', margin.top - 30)
    .attr('y2', margin.top - 30)
    .attr('stroke', '#bbb')
    .attr('stroke-width', 2)

  // Color scale
  const color = d3.scaleOrdinal()
    .domain(sorted.map(d => d.title))
    .range([
      '#e57373', '#64b5f6', '#81c784', '#ffd54f', '#ba68c8', '#4db6ac', '#ffb74d', '#a1887f', '#90a4ae', '#f06292'
    ])

  // Draw events as rounded rects
  const g = svg.append('g').attr('transform', `translate(${margin.left},${margin.top})`)
  g.selectAll('rect')
    .data(sorted)
    .enter()
    .append('rect')
    .attr('x', d => x(d.start))
    .attr('y', d => d._row * rowHeight)
    .attr('width', d => Math.max(30, x(d.end) - x(d.start)))
    .attr('height', rowHeight - 18)
    .attr('rx', 14)
    .attr('fill', d => color(d.title))
    .attr('stroke', d => d3.color(color(d.title)).darker(0.7))
    .attr('stroke-width', 2)
    .on('mouseover', function (event, d) {
      d3.select(this).attr('fill', d3.color(color(d.title)).brighter(0.7));
      // Show vertical lines at start and end on both rulers
      svg.append('line')
        .attr('class', 'hover-vline start')
        .attr('x1', x(d.start) + margin.left)
        .attr('x2', x(d.start) + margin.left)
        .attr('y1', margin.top - 30)
        .attr('y2', nRows * rowHeight + margin.top + 10)
        .attr('stroke', color(d.title))
        .attr('stroke-width', 3)
        .attr('stroke-dasharray', '6,4');
      svg.append('line')
        .attr('class', 'hover-vline end')
        .attr('x1', x(d.end) + margin.left)
        .attr('x2', x(d.end) + margin.left)
        .attr('y1', margin.top - 30)
        .attr('y2', nRows * rowHeight + margin.top + 10)
        .attr('stroke', color(d.title))
        .attr('stroke-width', 3)
        .attr('stroke-dasharray', '6,4');
    })
    .on('mouseout', function (event, d) {
      d3.select(this).attr('fill', color(d.title));
      svg.selectAll('.hover-vline').remove();
    })

  // Add event labels
  g.selectAll('text')
    .data(sorted)
    .enter()
    .append('text')
    .attr('x', d => x(d.start) + 8)
    .attr('y', d => d._row * rowHeight + 24)
    .attr('fill', '#222')
    .attr('font-family', 'Segoe UI, Arial, sans-serif')
    .attr('font-size', '1.1em')
    .attr('font-weight', 'bold')
    .text(d => d.title)

  // Add year range and age
  g.selectAll('desc')
    .data(sorted)
    .enter()
    .append('text')
    .attr('x', d => x(d.start) + 8)
    .attr('y', d => d._row * rowHeight + 44)
    .attr('fill', '#444')
    .attr('font-family', 'Segoe UI, Arial, sans-serif')
    .attr('font-size', '0.95em')
    .text(d => {
      const age = Math.abs(d.end - d.start);
      return `${Math.abs(d.start)} BC - ${Math.abs(d.end)} BC  •  Age: ${age}`;
    })
  g.selectAll('desc2')
    .data(sorted)
    .enter()
    .append('text')
    .attr('x', d => x(d.start) + 8)
    .attr('y', d => d._row * rowHeight + 62)
    .attr('fill', '#666')
    .attr('font-family', 'Segoe UI, Arial, sans-serif')
    .attr('font-size', '0.92em')
    .text(d => d.description)
})
</script>

<template>
  <div class="timeline-horizontal">
    <div ref="timelineContainer" class="d3-timeline-container"></div>
  </div>
</template>

<style scoped>
.timeline-horizontal {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 100vh;
  z-index: 1;
  background: #f8f6f1;
  box-shadow: 0 0 32px #e0e0e0;
  border-radius: 0;
  overflow: auto;
}
.d3-timeline-container {
  width: 100vw;
  height: 100vh;
  overflow: auto;
}
</style>
