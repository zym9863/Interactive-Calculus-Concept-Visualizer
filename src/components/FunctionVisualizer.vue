<script setup lang="ts">
import { ref, onMounted, watch, nextTick } from 'vue';
import * as d3 from 'd3';
import * as math from 'mathjs';

// State for function input and parameters
const functionExpression = ref('x^2');
const xMin = ref(-10);
const xMax = ref(10);
const yMin = ref(-10);
const yMax = ref(10);
const resolution = ref(1000);
const paramA = ref(1);
const paramB = ref(0);
const paramC = ref(0);
const showGrid = ref(true);
const showAxis = ref(true);
const errorMessage = ref('');

// Reference to SVG element
const svgRef = ref<SVGElement | null>(null);
const svgWidth = ref(800);
const svgHeight = ref(500);
const margin = { top: 20, right: 20, bottom: 50, left: 60 };

// Function to parse and evaluate the expression
const parseFunction = (expr: string, x: number): number => {
  try {
    // Replace parameter placeholders with actual values
    const parsedExpr = expr
      .replace(/a/g, paramA.value.toString())
      .replace(/b/g, paramB.value.toString())
      .replace(/c/g, paramC.value.toString());

    const scope = { x: x };
    return math.evaluate(parsedExpr, scope);
  } catch (e) {
    errorMessage.value = `Error evaluating function: ${e instanceof Error ? e.message : String(e)}`;
    return NaN;
  }
};

// Generate function points
const generatePoints = () => {
  const points = [];
  const step = (xMax.value - xMin.value) / resolution.value;

  for (let x = xMin.value; x <= xMax.value; x += step) {
    try {
      const y = parseFunction(functionExpression.value, x);
      if (!isNaN(y) && isFinite(y) && y >= yMin.value && y <= yMax.value) {
        points.push({ x, y });
      }
    } catch (e) {
      console.error('Error calculating point:', e);
    }
  }

  return points;
};

// Draw function
const drawFunction = () => {
  if (!svgRef.value) return;

  // Clear error message
  errorMessage.value = '';

  // Clear previous SVG contents
  d3.select(svgRef.value).selectAll('*').remove();

  const width = svgWidth.value - margin.left - margin.right;
  const height = svgHeight.value - margin.top - margin.bottom;

  // Create scales
  const xScale = d3.scaleLinear()
    .domain([xMin.value, xMax.value])
    .range([0, width]);

  const yScale = d3.scaleLinear()
    .domain([yMin.value, yMax.value])
    .range([height, 0]);

  // Create SVG element
  const svg = d3.select(svgRef.value)
    .attr('width', svgWidth.value)
    .attr('height', svgHeight.value)
    .append('g')
    .attr('transform', `translate(${margin.left}, ${margin.top})`);

  // Add background for the graph
  svg.append('rect')
    .attr('width', width)
    .attr('height', height)
    .attr('fill', 'var(--bg-medium)')
    .attr('rx', 8)
    .attr('ry', 8);

  // Draw grid if enabled
  if (showGrid.value) {
    // Vertical grid lines
    svg.selectAll('grid-vertical')
      .data(xScale.ticks())
      .enter()
      .append('line')
      .attr('x1', d => xScale(d))
      .attr('y1', 0)
      .attr('x2', d => xScale(d))
      .attr('y2', height)
      .attr('stroke', 'var(--chart-grid)')
      .attr('stroke-width', 0.5)
      .attr('stroke-dasharray', '3,3');

    // Horizontal grid lines
    svg.selectAll('grid-horizontal')
      .data(yScale.ticks())
      .enter()
      .append('line')
      .attr('x1', 0)
      .attr('y1', d => yScale(d))
      .attr('x2', width)
      .attr('y2', d => yScale(d))
      .attr('stroke', 'var(--chart-grid)')
      .attr('stroke-width', 0.5)
      .attr('stroke-dasharray', '3,3');
  }

  // Draw axes if enabled
  if (showAxis.value) {
    // X-axis
    svg.append('g')
      .attr('transform', `translate(0, ${yScale(0)})`)
      .call(d3.axisBottom(xScale))
      .attr('color', 'var(--chart-axis)')
      .attr('font-size', '12px')
      .attr('font-weight', '500')
      .selectAll('line')
      .attr('stroke', 'var(--chart-axis)');

    // Y-axis
    svg.append('g')
      .attr('transform', `translate(${xScale(0)}, 0)`)
      .call(d3.axisLeft(yScale))
      .attr('color', 'var(--chart-axis)')
      .attr('font-size', '12px')
      .attr('font-weight', '500')
      .selectAll('line')
      .attr('stroke', 'var(--chart-axis)');

    // X-axis label
    svg.append('text')
      .attr('transform', `translate(${width / 2}, ${height + 40})`)
      .style('text-anchor', 'middle')
      .text('x')
      .attr('fill', 'var(--text-secondary)')
      .attr('font-size', '14px')
      .attr('font-weight', '600');

    // Y-axis label
    svg.append('text')
      .attr('transform', 'rotate(-90)')
      .attr('y', -40)
      .attr('x', -(height / 2))
      .style('text-anchor', 'middle')
      .text('y')
      .attr('fill', 'var(--text-secondary)')
      .attr('font-size', '14px')
      .attr('font-weight', '600');
  }

  // Generate points
  const points = generatePoints();

  if (points.length === 0) {
    errorMessage.value = 'No valid points to display with current settings.';
    return;
  }

  // Create line generator
  const line = d3.line<{ x: number, y: number }>()
    .x(d => xScale(d.x))
    .y(d => yScale(d.y));

  // Create gradient for function line
  const gradient = svg.append("defs")
    .append("linearGradient")
    .attr("id", "line-gradient")
    .attr("gradientUnits", "userSpaceOnUse")
    .attr("x1", 0)
    .attr("y1", 0)
    .attr("x2", width)
    .attr("y2", 0);

  gradient.append("stop")
    .attr("offset", "0%")
    .attr("stop-color", "var(--primary-color)");

  gradient.append("stop")
    .attr("offset", "50%")
    .attr("stop-color", "var(--chart-line)");

  gradient.append("stop")
    .attr("offset", "100%")
    .attr("stop-color", "var(--accent-color)");

  // Draw function line with animation
  const path = svg.append('path')
    .datum(points)
    .attr('fill', 'none')
    .attr('stroke', 'url(#line-gradient)')
    .attr('stroke-width', 3)
    .attr('stroke-linecap', 'round')
    .attr('stroke-linejoin', 'round')
    .attr('d', line)
    .style('filter', 'drop-shadow(0 2px 3px rgba(0, 0, 0, 0.3))');

  // Add animation effect
  const pathLength = path.node().getTotalLength();
  path.attr('stroke-dasharray', pathLength)
    .attr('stroke-dashoffset', pathLength)
    .transition()
    .duration(1000)
    .ease(d3.easeLinear)
    .attr('stroke-dashoffset', 0);

  // Interactive point tracking with improved styling
  const tooltip = d3.select('body')
    .append('div')
    .style('position', 'absolute')
    .style('background-color', 'var(--bg-light)')
    .style('color', 'var(--text-primary)')
    .style('padding', '10px 14px')
    .style('border-radius', 'var(--radius-md)')
    .style('font-size', '14px')
    .style('font-weight', '500')
    .style('pointer-events', 'none')
    .style('opacity', 0)
    .style('box-shadow', 'var(--shadow-md)')
    .style('border', '1px solid var(--border-color)')
    .style('z-index', 1000);

  // Add glow effect for tracking point
  const glow = svg.append('filter')
    .attr('id', 'glow')
    .attr('x', '-50%')
    .attr('y', '-50%')
    .attr('width', '200%')
    .attr('height', '200%');

  glow.append('feGaussianBlur')
    .attr('stdDeviation', '3')
    .attr('result', 'coloredBlur');

  const femerge = glow.append('feMerge');
  femerge.append('feMergeNode')
    .attr('in', 'coloredBlur');
  femerge.append('feMergeNode')
    .attr('in', 'SourceGraphic');

  // Tracking circle with glow effect
  const trackingCircle = svg.append('circle')
    .attr('r', 6)
    .attr('fill', 'var(--accent-light)')
    .attr('stroke', 'white')
    .attr('stroke-width', 2)
    .style('filter', 'url(#glow)')
    .style('opacity', 0);

  // Tracking line with improved styling
  const trackingLine = svg.append('line')
    .attr('stroke', 'var(--accent-light)')
    .attr('stroke-width', 1.5)
    .attr('stroke-dasharray', '4,4')
    .style('opacity', 0);

  // Create overlay for mouse interaction
  const overlay = svg.append('rect')
    .attr('width', width)
    .attr('height', height)
    .attr('fill', 'none')
    .attr('pointer-events', 'all')
    .on('mousemove', function(event) {
      const [mouseX] = d3.pointer(event);
      const xValue = xScale.invert(mouseX);
      const yValue = parseFunction(functionExpression.value, xValue);

      if (!isNaN(yValue) && isFinite(yValue) && yValue >= yMin.value && yValue <= yMax.value) {
        // Smooth animation for tracking elements
        trackingCircle
          .transition()
          .duration(50)
          .attr('cx', xScale(xValue))
          .attr('cy', yScale(yValue))
          .style('opacity', 1);

        trackingLine
          .transition()
          .duration(50)
          .attr('x1', xScale(xValue))
          .attr('y1', yScale(yValue))
          .attr('x2', xScale(xValue))
          .attr('y2', height)
          .style('opacity', 0.7);

        // Add horizontal tracking line
        const horizontalLine = svg.selectAll('.horizontal-track')
          .data([0])
          .join('line')
          .attr('class', 'horizontal-track')
          .attr('x1', 0)
          .attr('y1', yScale(yValue))
          .attr('x2', xScale(xValue))
          .attr('y2', yScale(yValue))
          .attr('stroke', 'var(--accent-light)')
          .attr('stroke-width', 1)
          .attr('stroke-dasharray', '4,4')
          .style('opacity', 0.5);

        // Enhanced tooltip with formatted values
        tooltip
          .transition()
          .duration(100)
          .style('opacity', 1)
          .style('left', `${event.pageX + 15}px`)
          .style('top', `${event.pageY - 40}px`)
          .html(`
            <div style="font-weight: 600; margin-bottom: 4px; color: var(--primary-color);">函数值</div>
            <div style="display: flex; justify-content: space-between; gap: 10px;">
              <span>x:</span>
              <span style="font-family: monospace; font-weight: 600;">${xValue.toFixed(3)}</span>
            </div>
            <div style="display: flex; justify-content: space-between; gap: 10px;">
              <span>f(x):</span>
              <span style="font-family: monospace; font-weight: 600; color: var(--accent-color);">${yValue.toFixed(3)}</span>
            </div>
          `);
      } else {
        // Hide tracking elements when out of range
        trackingCircle.transition().duration(100).style('opacity', 0);
        trackingLine.transition().duration(100).style('opacity', 0);
        svg.selectAll('.horizontal-track').transition().duration(100).style('opacity', 0);
        tooltip.transition().duration(100).style('opacity', 0);
      }
    })
    .on('mouseout', function() {
      // Hide all tracking elements on mouseout
      trackingCircle.transition().duration(200).style('opacity', 0);
      trackingLine.transition().duration(200).style('opacity', 0);
      svg.selectAll('.horizontal-track').transition().duration(200).style('opacity', 0);
      tooltip.transition().duration(200).style('opacity', 0);
    });
};

// Watch for changes to redraw the function
watch(
  [
    functionExpression,
    xMin, xMax, yMin, yMax,
    paramA, paramB, paramC,
    showGrid, showAxis, resolution
  ],
  () => {
    nextTick(drawFunction);
  },
  { deep: true }
);

// Initial drawing
onMounted(() => {
  drawFunction();

  // Handle window resize
  const handleResize = () => {
    if (svgRef.value && svgRef.value.parentElement) {
      const parentWidth = svgRef.value.parentElement.clientWidth;
      svgWidth.value = Math.min(800, parentWidth - 40);
      nextTick(drawFunction);
    }
  };

  window.addEventListener('resize', handleResize);
  handleResize();
});

// Function examples
const functionExamples = [
  { name: '二次函数', expr: 'a*x^2 + b*x + c' },
  { name: '正弦函数', expr: 'a*sin(b*x + c)' },
  { name: '指数函数', expr: 'a*e^(b*x) + c' },
  { name: '对数函数', expr: 'a*log(b*x) + c' },
  { name: '分数函数', expr: 'a/(b*x + c)' },
];

// Reset view
const resetView = () => {
  xMin.value = -10;
  xMax.value = 10;
  yMin.value = -10;
  yMax.value = 10;
};

// Apply function example
const applyExample = (expr: string) => {
  functionExpression.value = expr;
};
</script>

<template>
  <div class="function-visualizer">
    <div class="control-panel">
      <div class="input-group">
        <label for="function-input">函数表达式:</label>
        <input
          id="function-input"
          v-model="functionExpression"
          placeholder="输入数学表达式，例如：x^2"
        />
        <div class="examples">
          <span>示例:</span>
          <button
            v-for="example in functionExamples"
            :key="example.name"
            @click="applyExample(example.expr)"
            class="example-btn"
          >
            {{ example.name }}
          </button>
        </div>
      </div>

      <div class="parameters">
        <h3>参数调整</h3>
        <div class="param-controls">
          <div class="param-slider">
            <label for="param-a">参数 a: {{ paramA }}</label>
            <input
              type="range"
              id="param-a"
              v-model.number="paramA"
              min="-10"
              max="10"
              step="0.1"
            />
          </div>

          <div class="param-slider">
            <label for="param-b">参数 b: {{ paramB }}</label>
            <input
              type="range"
              id="param-b"
              v-model.number="paramB"
              min="-10"
              max="10"
              step="0.1"
            />
          </div>

          <div class="param-slider">
            <label for="param-c">参数 c: {{ paramC }}</label>
            <input
              type="range"
              id="param-c"
              v-model.number="paramC"
              min="-10"
              max="10"
              step="0.1"
            />
          </div>
        </div>
      </div>

      <div class="view-controls">
        <h3>视图设置</h3>
        <div class="range-inputs">
          <div class="range-group">
            <label for="x-min">X 最小值:</label>
            <input type="number" id="x-min" v-model.number="xMin" />
          </div>

          <div class="range-group">
            <label for="x-max">X 最大值:</label>
            <input type="number" id="x-max" v-model.number="xMax" />
          </div>

          <div class="range-group">
            <label for="y-min">Y 最小值:</label>
            <input type="number" id="y-min" v-model.number="yMin" />
          </div>

          <div class="range-group">
            <label for="y-max">Y 最大值:</label>
            <input type="number" id="y-max" v-model.number="yMax" />
          </div>

          <button @click="resetView" class="reset-btn">重置视图</button>
        </div>

        <div class="display-options">
          <label>
            <input type="checkbox" v-model="showGrid" />
            显示网格
          </label>

          <label>
            <input type="checkbox" v-model="showAxis" />
            显示坐标轴
          </label>
        </div>
      </div>
    </div>

    <div class="graph-container">
      <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
      <svg ref="svgRef" class="function-graph"></svg>
    </div>
  </div>
</template>

<style scoped>
.function-visualizer {
  display: grid;
  grid-template-columns: 320px 1fr;
  gap: 24px;
  text-align: left;
}

.control-panel {
  background-color: var(--bg-light);
  border-radius: var(--radius-lg);
  padding: 24px;
  display: flex;
  flex-direction: column;
  gap: 24px;
  box-shadow: var(--shadow-md);
  position: relative;
  overflow: hidden;
}

.control-panel::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 4px;
  background: linear-gradient(90deg, var(--primary-color), var(--accent-color));
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.input-group label {
  font-weight: 500;
  color: var(--text-secondary);
  font-size: 0.9rem;
  margin-bottom: -4px;
}

.input-group input {
  padding: 10px 12px;
  border-radius: var(--radius-sm);
  border: 1px solid var(--border-color);
  background-color: var(--bg-medium);
  color: var(--text-primary);
  font-size: 0.95rem;
  transition: all var(--transition-fast);
  box-shadow: inset 0 1px 2px rgba(0, 0, 0, 0.1);
}

.input-group input:focus {
  border-color: var(--primary-light);
  box-shadow: 0 0 0 2px rgba(67, 97, 238, 0.2), inset 0 1px 2px rgba(0, 0, 0, 0.1);
}

.examples {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  align-items: center;
  margin-top: 8px;
}

.examples span {
  font-size: 0.85rem;
  color: var(--text-secondary);
  margin-right: 4px;
}

.example-btn {
  font-size: 0.8rem;
  padding: 5px 10px;
  background-color: var(--bg-medium);
  border-radius: var(--radius-sm);
  border: 1px solid var(--border-color);
  color: var(--text-secondary);
  transition: all var(--transition-fast);
}

.example-btn:hover {
  background-color: var(--primary-color);
  color: white;
  border-color: var(--primary-color);
  transform: translateY(-2px);
}

.parameters, .view-controls {
  border-top: 1px solid var(--border-color);
  padding-top: 20px;
  position: relative;
}

h3 {
  margin-top: 0;
  margin-bottom: 16px;
  font-size: 1.1rem;
  font-weight: 600;
  color: var(--text-primary);
  display: flex;
  align-items: center;
}

h3::before {
  content: '';
  display: inline-block;
  width: 12px;
  height: 12px;
  background: linear-gradient(135deg, var(--primary-color), var(--primary-light));
  border-radius: 50%;
  margin-right: 8px;
}

.param-controls, .range-inputs {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.param-slider {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.param-slider label {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.9rem;
  color: var(--text-secondary);
}

.param-slider label span {
  font-weight: 500;
  color: var(--primary-light);
  background-color: var(--bg-medium);
  padding: 2px 8px;
  border-radius: var(--radius-sm);
  font-size: 0.85rem;
}

.range-group {
  display: flex;
  align-items: center;
  gap: 10px;
}

.range-group label {
  font-size: 0.9rem;
  color: var(--text-secondary);
  min-width: 30px;
}

.range-group input {
  width: 70px;
  padding: 6px 8px;
  border-radius: var(--radius-sm);
  border: 1px solid var(--border-color);
  background-color: var(--bg-medium);
  color: var(--text-primary);
  font-size: 0.9rem;
  text-align: center;
}

.reset-btn {
  align-self: flex-start;
  margin-top: 12px;
  background-color: var(--bg-medium);
  color: var(--text-secondary);
  border: 1px solid var(--border-color);
  padding: 8px 16px;
  font-size: 0.9rem;
  border-radius: var(--radius-sm);
  transition: all var(--transition-fast);
}

.reset-btn:hover {
  background-color: var(--primary-color);
  color: white;
  border-color: var(--primary-color);
}

.display-options {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-top: 16px;
}

.display-options label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.9rem;
  color: var(--text-secondary);
  cursor: pointer;
}

.display-options input[type="checkbox"] {
  appearance: none;
  -webkit-appearance: none;
  width: 18px;
  height: 18px;
  border: 1px solid var(--border-color);
  border-radius: var(--radius-sm);
  background-color: var(--bg-medium);
  display: grid;
  place-content: center;
  cursor: pointer;
}

.display-options input[type="checkbox"]::before {
  content: "";
  width: 10px;
  height: 10px;
  transform: scale(0);
  transition: transform var(--transition-fast);
  box-shadow: inset 1em 1em var(--primary-color);
  transform-origin: center;
  clip-path: polygon(14% 44%, 0 65%, 50% 100%, 100% 16%, 80% 0%, 43% 62%);
}

.display-options input[type="checkbox"]:checked::before {
  transform: scale(1);
}

.graph-container {
  background-color: var(--bg-light);
  border-radius: var(--radius-lg);
  padding: 24px;
  overflow: hidden;
  min-height: 500px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  box-shadow: var(--shadow-md);
  position: relative;
}

.function-graph {
  width: 100%;
  max-width: 800px;
  border-radius: var(--radius-md);
  overflow: hidden;
}

.error-message {
  color: var(--error-color);
  margin-bottom: 16px;
  padding: 12px;
  border-radius: var(--radius-md);
  background-color: rgba(255, 82, 82, 0.1);
  border: 1px solid var(--error-color);
  width: 100%;
  text-align: center;
  font-size: 0.9rem;
  display: flex;
  align-items: center;
  justify-content: center;
}

.error-message::before {
  content: "⚠️";
  margin-right: 8px;
  font-size: 1.2rem;
}

@media (max-width: 900px) {
  .function-visualizer {
    grid-template-columns: 1fr;
    gap: 20px;
  }

  .control-panel {
    padding: 20px;
  }
}

@media (max-width: 600px) {
  .range-inputs {
    gap: 12px;
  }

  .range-group {
    flex-direction: column;
    align-items: flex-start;
    gap: 6px;
  }

  .range-group input {
    width: 100%;
  }
}
</style>
