<script setup lang="ts">
import { ref, onMounted, watch, nextTick } from 'vue';
import * as d3 from 'd3';
import * as math from 'mathjs';

// State for function input and parameters
const functionExpression = ref('x^2');
const x0 = ref(0);
const xMin = ref(-10);
const xMax = ref(10);
const yMin = ref(-10);
const yMax = ref(10);
const resolution = ref(1000);
const showGrid = ref(true);
const showAxis = ref(true);
const showDerivative = ref(true);
const errorMessage = ref('');

// Reference to SVG element
const svgRef = ref<SVGElement | null>(null);
const svgWidth = ref(800);
const svgHeight = ref(500);
const margin = { top: 20, right: 20, bottom: 50, left: 60 };

// Function to parse and evaluate the expression
const parseFunction = (expr: string, x: number): number => {
  try {
    const scope = { x: x };
    return math.evaluate(expr, scope);
  } catch (e) {
    errorMessage.value = `Error evaluating function: ${e instanceof Error ? e.message : String(e)}`;
    return NaN;
  }
};

// Calculate derivative using mathjs
const calculateDerivative = () => {
  try {
    const f = math.parse(functionExpression.value);
    const derivative = math.derivative(f, 'x');
    return derivative.toString();
  } catch (e) {
    errorMessage.value = `Error calculating derivative: ${e instanceof Error ? e.message : String(e)}`;
    return '';
  }
};

// Evaluate derivative at a specific x value
const evaluateDerivative = (x: number): number => {
  try {
    const derivativeExpr = calculateDerivative();
    if (!derivativeExpr) return NaN;
    const scope = { x: x };
    return math.evaluate(derivativeExpr, scope);
  } catch (e) {
    console.error('Error evaluating derivative:', e);
    return NaN;
  }
};

// Generate function points
const generatePoints = (expr: string) => {
  const points = [];
  const step = (xMax.value - xMin.value) / resolution.value;

  for (let x = xMin.value; x <= xMax.value; x += step) {
    try {
      const y = parseFunction(expr, x);
      if (!isNaN(y) && isFinite(y) && y >= yMin.value && y <= yMax.value) {
        points.push({ x, y });
      }
    } catch (e) {
      console.error('Error calculating point:', e);
    }
  }

  return points;
};

// Calculate tangent line points that span the entire viewport
const calculateTangentPoints = () => {
  const derivativeAtX0 = evaluateDerivative(x0.value);
  const fAtX0 = parseFunction(functionExpression.value, x0.value);

  // Calculate tangent line equation: y = derivativeAtX0 * (x - x0) + fAtX0

  // Find points at the edges of the x domain
  const x1 = xMin.value;
  const y1 = derivativeAtX0 * (x1 - x0.value) + fAtX0;

  const x2 = xMax.value;
  const y2 = derivativeAtX0 * (x2 - x0.value) + fAtX0;

  return [
    { x: x1, y: y1 },
    { x: x2, y: y2 }
  ];
};

// Draw function, derivative, and tangent
const draw = () => {
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

  // Create line generator
  const line = d3.line<{ x: number, y: number }>()
    .x(d => xScale(d.x))
    .y(d => yScale(d.y));

  // Draw original function
  const originalPoints = generatePoints(functionExpression.value);

  if (originalPoints.length > 0) {
    // Create gradient for original function
    const originalGradient = svg.append("defs")
      .append("linearGradient")
      .attr("id", "original-line-gradient")
      .attr("gradientUnits", "userSpaceOnUse")
      .attr("x1", 0)
      .attr("y1", 0)
      .attr("x2", width)
      .attr("y2", 0);

    originalGradient.append("stop")
      .attr("offset", "0%")
      .attr("stop-color", "var(--primary-color)");

    originalGradient.append("stop")
      .attr("offset", "100%")
      .attr("stop-color", "var(--accent-color)");

    svg.append('path')
      .datum(originalPoints)
      .attr('fill', 'none')
      .attr('stroke', 'url(#original-line-gradient)')
      .attr('stroke-width', 3)
      .attr('stroke-linecap', 'round')
      .attr('stroke-linejoin', 'round')
      .attr('d', line)
      .style('filter', 'drop-shadow(0 2px 3px rgba(0, 0, 0, 0.3))');
  }

  // Draw derivative function if enabled
  if (showDerivative.value) {
    const derivativeExpr = calculateDerivative();
    if (derivativeExpr) {
      const derivativePoints = generatePoints(derivativeExpr);

      if (derivativePoints.length > 0) {
        svg.append('path')
          .datum(derivativePoints)
          .attr('fill', 'none')
          .attr('stroke', 'var(--success-color)')
          .attr('stroke-width', 2)
          .attr('stroke-linecap', 'round')
          .attr('stroke-linejoin', 'round')
          .attr('stroke-dasharray', '5,5')
          .attr('d', line);
      }
    }
  }

  // Draw tangent line
  const tangentPoints = calculateTangentPoints();

  if (tangentPoints.every(p => !isNaN(p.y) && isFinite(p.y))) {
    svg.append('path')
      .datum(tangentPoints)
      .attr('fill', 'none')
      .attr('stroke', 'var(--warning-color)')
      .attr('stroke-width', 2)
      .attr('stroke-linecap', 'round')
      .attr('stroke-linejoin', 'round')
      .attr('d', line);

    // Draw tangent point
    const tangentPoint = {
      x: x0.value,
      y: parseFunction(functionExpression.value, x0.value)
    };

    if (!isNaN(tangentPoint.y) && isFinite(tangentPoint.y)) {
      // Add glow effect for tangent point
      const glow = svg.append('filter')
        .attr('id', 'tangent-glow')
        .attr('x', '-50%')
        .attr('y', '-50%')
        .attr('width', '200%')
        .attr('height', '200%');

      glow.append('feGaussianBlur')
        .attr('stdDeviation', '4')
        .attr('result', 'coloredBlur');

      const femerge = glow.append('feMerge');
      femerge.append('feMergeNode')
        .attr('in', 'coloredBlur');
      femerge.append('feMergeNode')
        .attr('in', 'SourceGraphic');

      svg.append('circle')
        .attr('cx', xScale(tangentPoint.x))
        .attr('cy', yScale(tangentPoint.y))
        .attr('r', 8)
        .attr('fill', 'var(--warning-color)')
        .attr('stroke', 'white')
        .attr('stroke-width', 2)
        .style('filter', 'url(#tangent-glow)');
    }
  }
};

// Watch for changes to redraw
watch(
  [
    functionExpression,
    x0,
    xMin, xMax, yMin, yMax,
    showGrid, showAxis, showDerivative, resolution
  ],
  () => {
    nextTick(draw);
  },
  { deep: true }
);

// Initial drawing
onMounted(() => {
  draw();

  // Handle window resize
  const handleResize = () => {
    if (svgRef.value && svgRef.value.parentElement) {
      const parentWidth = svgRef.value.parentElement.clientWidth;
      svgWidth.value = Math.min(800, parentWidth - 40);
      nextTick(draw);
    }
  };

  window.addEventListener('resize', handleResize);
  handleResize();
});

// Function examples
const functionExamples = [
  { name: '二次函数', expr: 'x^2' },
  { name: '正弦函数', expr: 'sin(x)' },
  { name: '三次函数', expr: 'x^3' },
  { name: '指数函数', expr: 'e^x' },
  { name: '平方根函数', expr: 'sqrt(x + 5)' },
];

// Apply function example
const applyExample = (expr: string) => {
  functionExpression.value = expr;
};

// Reset view
const resetView = () => {
  xMin.value = -10;
  xMax.value = 10;
  yMin.value = -10;
  yMax.value = 10;
  x0.value = 0;
};
</script>

<template>
  <div class="derivative-visualizer">
    <div class="control-panel">
      <div class="input-group">
        <label for="function-input">函数表达式 f(x):</label>
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
        <h3>切点选择</h3>
        <div class="param-slider">
          <label for="x0-slider">切点 x₀: {{ x0.toFixed(2) }}</label>
          <input
            type="range"
            id="x0-slider"
            v-model.number="x0"
            :min="xMin"
            :max="xMax"
            step="0.1"
          />
        </div>
      </div>

      <div class="display-options">
        <h3>显示选项</h3>
        <label>
          <input type="checkbox" v-model="showGrid" />
          显示网格
        </label>
        <label>
          <input type="checkbox" v-model="showAxis" />
          显示坐标轴
        </label>
        <label>
          <input type="checkbox" v-model="showDerivative" />
          显示导函数 f'(x)
        </label>
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
      </div>

      <div class="info-panel">
        <h3>数值信息</h3>
        <div class="info-item">
          <span class="info-label">x₀:</span>
          <span class="info-value">{{ x0.toFixed(3) }}</span>
        </div>
        <div class="info-item">
          <span class="info-label">f(x₀):</span>
          <span class="info-value">{{ parseFunction(functionExpression, x0).toFixed(3) }}</span>
        </div>
        <div class="info-item">
          <span class="info-label">f'(x₀) (斜率):</span>
          <span class="info-value">{{ evaluateDerivative(x0).toFixed(3) }}</span>
        </div>
        <div class="info-item" v-if="calculateDerivative()">
          <span class="info-label">f'(x):</span>
          <span class="info-value">{{ calculateDerivative() }}</span>
        </div>
      </div>
    </div>

    <div class="graph-container">
      <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
      <svg ref="svgRef" class="derivative-graph"></svg>
    </div>
  </div>
</template>

<style scoped>
.derivative-visualizer {
  display: grid;
  grid-template-columns: 340px 1fr;
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
  background: linear-gradient(90deg, var(--primary-color), var(--warning-color));
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

.parameters, .view-controls, .display-options, .info-panel {
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
  background: linear-gradient(135deg, var(--primary-color), var(--warning-color));
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

.info-panel {
  background-color: var(--bg-medium);
  border-radius: var(--radius-md);
  padding: 16px;
}

.info-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 0;
  border-bottom: 1px solid var(--border-color);
}

.info-item:last-child {
  border-bottom: none;
}

.info-label {
  font-size: 0.9rem;
  color: var(--text-secondary);
  font-weight: 500;
}

.info-value {
  font-size: 0.9rem;
  color: var(--primary-light);
  font-weight: 600;
  font-family: monospace;
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

.derivative-graph {
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
  .derivative-visualizer {
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