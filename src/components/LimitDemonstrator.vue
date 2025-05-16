<script setup lang="ts">
import { ref, onMounted, watch, nextTick } from 'vue';
import * as d3 from 'd3';
import * as math from 'mathjs';

// State for function input and parameters
const functionExpression = ref('1/x');
const pointOfInterest = ref(0); // The x₀ value to examine
const epsilon = ref(0.1); // Distance from x₀
const animationSpeed = ref(50); // Control animation speed
const xMin = ref(-10);
const xMax = ref(10);
const yMin = ref(-10);
const yMax = ref(10);
const showAnimation = ref(false);
const isAnimating = ref(false);
const errorMessage = ref('');

// Animation values
const leftApproachX = ref(null as number | null);
const rightApproachX = ref(null as number | null);
const leftLimit = ref('未定义');
const rightLimit = ref('未定义');
const functionValue = ref('未定义');
const isContinuous = ref(false);

// SVG reference and dimensions
const svgRef = ref<SVGElement | null>(null);
const svgWidth = ref(800);
const svgHeight = ref(500);
const margin = { top: 20, right: 20, bottom: 50, left: 60 };

// Parse and evaluate the expression
const parseFunction = (expr: string, x: number): number => {
  try {
    const scope = { x: x };
    return math.evaluate(expr, scope);
  } catch (e) {
    errorMessage.value = `Error evaluating function: ${e instanceof Error ? e.message : String(e)}`;
    return NaN;
  }
};

// Check if a point is a valid numerical value
const isValidPoint = (y: number): boolean => {
  return !isNaN(y) && isFinite(y);
};

// Generate function points
const generatePoints = (skipPointOfInterest = false): { x: number, y: number }[] => {
  const points = [];
  const step = (xMax.value - xMin.value) / 1000;

  for (let x = xMin.value; x <= xMax.value; x += step) {
    // Skip the point of interest if requested to create a "hole" in the graph
    if (skipPointOfInterest && Math.abs(x - pointOfInterest.value) < step / 2) {
      continue;
    }

    try {
      const y = parseFunction(functionExpression.value, x);
      if (isValidPoint(y) && y >= yMin.value && y <= yMax.value) {
        points.push({ x, y });
      }
    } catch (e) {
      console.error('Error calculating point:', e);
    }
  }

  return points;
};

// Calculate limits
const calculateLimits = () => {
  // Calculate left limit (approaching from left)
  let leftLimitValue;
  try {
    const x = pointOfInterest.value - 1e-10;
    leftLimitValue = parseFunction(functionExpression.value, x);
    leftLimit.value = isValidPoint(leftLimitValue) ? leftLimitValue.toFixed(6) : '未定义';
  } catch (e) {
    leftLimit.value = '未定义';
  }

  // Calculate right limit (approaching from right)
  let rightLimitValue;
  try {
    const x = pointOfInterest.value + 1e-10;
    rightLimitValue = parseFunction(functionExpression.value, x);
    rightLimit.value = isValidPoint(rightLimitValue) ? rightLimitValue.toFixed(6) : '未定义';
  } catch (e) {
    rightLimit.value = '未定义';
  }

  // Calculate function value at the point of interest
  let actualValue;
  try {
    actualValue = parseFunction(functionExpression.value, pointOfInterest.value);
    functionValue.value = isValidPoint(actualValue) ? actualValue.toFixed(6) : '未定义';
  } catch (e) {
    functionValue.value = '未定义';
  }

  // Check continuity
  if (
    leftLimit.value !== '未定义' &&
    rightLimit.value !== '未定义' &&
    functionValue.value !== '未定义' &&
    leftLimit.value === rightLimit.value &&
    leftLimit.value === functionValue.value
  ) {
    isContinuous.value = true;
  } else {
    isContinuous.value = false;
  }
};

// Draw function and limit visualization
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

  // Draw grid
  const drawGrid = () => {
    // Add background for the graph
    svg.append('rect')
      .attr('width', width)
      .attr('height', height)
      .attr('fill', 'var(--bg-medium)')
      .attr('rx', 8)
      .attr('ry', 8);

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
  };

  drawGrid();

  // Generate points with a hole at the point of interest
  const points = generatePoints(false);

  if (points.length === 0) {
    errorMessage.value = 'No valid points to display with current settings.';
    return;
  }

  // Create line generator
  const line = d3.line<{ x: number, y: number }>()
    .defined(d => isValidPoint(d.y))
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

  // Highlight the point of interest
  const pointOfInterestX = pointOfInterest.value;
  let pointOfInterestY;

  try {
    pointOfInterestY = parseFunction(functionExpression.value, pointOfInterestX);
  } catch (e) {
    pointOfInterestY = NaN;
  }

  // Add glow effect for points
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

  // Draw vertical line at x = x₀ with improved styling
  svg.append('line')
    .attr('x1', xScale(pointOfInterestX))
    .attr('y1', 0)
    .attr('x2', xScale(pointOfInterestX))
    .attr('y2', height)
    .attr('stroke', 'var(--text-tertiary)')
    .attr('stroke-width', 1.5)
    .attr('stroke-dasharray', '6,4');

  // Add label for point of interest
  svg.append('text')
    .attr('x', xScale(pointOfInterestX))
    .attr('y', 20)
    .attr('text-anchor', 'middle')
    .attr('fill', 'var(--text-secondary)')
    .attr('font-size', '12px')
    .attr('font-weight', '600')
    .text(`x₀ = ${pointOfInterestX}`);

  // Mark the point of interest with enhanced styling
  if (isValidPoint(pointOfInterestY) && pointOfInterestY >= yMin.value && pointOfInterestY <= yMax.value) {
    // Add highlight circle
    svg.append('circle')
      .attr('cx', xScale(pointOfInterestX))
      .attr('cy', yScale(pointOfInterestY))
      .attr('r', 8)
      .attr('fill', isContinuous.value ? 'var(--success-color)' : 'var(--error-color)')
      .attr('stroke', 'white')
      .attr('stroke-width', 2)
      .style('filter', 'url(#glow)');

    // Add label for function value
    svg.append('text')
      .attr('x', xScale(pointOfInterestX) + 15)
      .attr('y', yScale(pointOfInterestY) - 10)
      .attr('fill', isContinuous.value ? 'var(--success-color)' : 'var(--error-color)')
      .attr('font-size', '12px')
      .attr('font-weight', '600')
      .text(`f(${pointOfInterestX}) = ${pointOfInterestY.toFixed(2)}`);
  }

  // Create circles for animation with enhanced styling
  const leftCircle = svg.append('circle')
    .attr('r', 6)
    .attr('fill', 'var(--warning-color)')
    .attr('stroke', 'white')
    .attr('stroke-width', 1.5)
    .style('filter', 'url(#glow)')
    .style('opacity', showAnimation.value ? 1 : 0);

  const rightCircle = svg.append('circle')
    .attr('r', 6)
    .attr('fill', 'var(--info-color)')
    .attr('stroke', 'white')
    .attr('stroke-width', 1.5)
    .style('filter', 'url(#glow)')
    .style('opacity', showAnimation.value ? 1 : 0);

  const updateCircles = () => {
    if (leftApproachX.value !== null) {
      try {
        const y = parseFunction(functionExpression.value, leftApproachX.value);
        if (isValidPoint(y)) {
          leftCircle
            .attr('cx', xScale(leftApproachX.value))
            .attr('cy', yScale(y));
        }
      } catch (e) {
        // Error computing function value
      }
    }

    if (rightApproachX.value !== null) {
      try {
        const y = parseFunction(functionExpression.value, rightApproachX.value);
        if (isValidPoint(y)) {
          rightCircle
            .attr('cx', xScale(rightApproachX.value))
            .attr('cy', yScale(y));
        }
      } catch (e) {
        // Error computing function value
      }
    }
  };

  // Initial update
  updateCircles();

  // Start animation if enabled
  if (showAnimation.value && !isAnimating.value) {
    startAnimation(updateCircles);
  }
};

// Animation function
const startAnimation = (updateFn: () => void) => {
  isAnimating.value = true;

  // Set initial positions
  leftApproachX.value = pointOfInterest.value - epsilon.value;
  rightApproachX.value = pointOfInterest.value + epsilon.value;

  const animate = () => {
    if (!showAnimation.value) {
      isAnimating.value = false;
      return;
    }

    // Move circles closer to the point of interest
    if (leftApproachX.value !== null && rightApproachX.value !== null) {
      const newLeftX = leftApproachX.value + (pointOfInterest.value - leftApproachX.value) * 0.05;
      const newRightX = rightApproachX.value - (rightApproachX.value - pointOfInterest.value) * 0.05;

      if (Math.abs(pointOfInterest.value - newLeftX) < 0.0001) {
        leftApproachX.value = pointOfInterest.value - epsilon.value;
      } else {
        leftApproachX.value = newLeftX;
      }

      if (Math.abs(newRightX - pointOfInterest.value) < 0.0001) {
        rightApproachX.value = pointOfInterest.value + epsilon.value;
      } else {
        rightApproachX.value = newRightX;
      }

      // Update circle positions
      updateFn();

      // Continue animation
      setTimeout(animate, 1000 / animationSpeed.value);
    }
  };

  animate();
};

// Reset limits calculation
const resetLimits = () => {
  calculateLimits();
  drawFunction();
};

// Watch for changes to redraw
watch(
  [
    functionExpression,
    pointOfInterest,
    xMin, xMax, yMin, yMax,
    showAnimation
  ],
  () => {
    calculateLimits();
    nextTick(drawFunction);
  },
  { deep: true }
);

// Toggle animation
watch(showAnimation, (newVal) => {
  if (newVal && !isAnimating.value) {
    nextTick(() => {
      const updateCircles = () => {
        if (svgRef.value && leftApproachX.value !== null && rightApproachX.value !== null) {
          const svg = d3.select(svgRef.value);

          const width = svgWidth.value - margin.left - margin.right;
          const height = svgHeight.value - margin.top - margin.bottom;

          const xScale = d3.scaleLinear()
            .domain([xMin.value, xMax.value])
            .range([0, width]);

          const yScale = d3.scaleLinear()
            .domain([yMin.value, yMax.value])
            .range([height, 0]);

          try {
            const leftY = parseFunction(functionExpression.value, leftApproachX.value);
            if (isValidPoint(leftY)) {
              svg.select('circle:nth-of-type(2)')
                .attr('cx', xScale(leftApproachX.value))
                .attr('cy', yScale(leftY));
            }
          } catch (e) {
            // Error computing function value
          }

          try {
            const rightY = parseFunction(functionExpression.value, rightApproachX.value);
            if (isValidPoint(rightY)) {
              svg.select('circle:nth-of-type(3)')
                .attr('cx', xScale(rightApproachX.value))
                .attr('cy', yScale(rightY));
            }
          } catch (e) {
            // Error computing function value
          }
        }
      };

      startAnimation(updateCircles);
    });
  }
});

// Initial setup
onMounted(() => {
  calculateLimits();
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
  { name: '分段函数', expr: 'x < 0 ? -x : x' },
  { name: '不连续函数', expr: '1/x' },
  { name: '跳跃函数', expr: 'floor(x)' },
  { name: '可去间断点', expr: 'x == 0 ? 0 : sin(x)/x' },
  { name: '无定义点', expr: 'tan(x)' },
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
};
</script>

<template>
  <div class="limit-demonstrator">
    <div class="control-panel">
      <div class="input-group">
        <label for="function-input">函数表达式:</label>
        <input
          id="function-input"
          v-model="functionExpression"
          placeholder="输入数学表达式，例如：1/x"
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

      <div class="limit-controls">
        <h3>极限探索</h3>
        <div class="limit-inputs">
          <div class="input-group">
            <label for="point-of-interest">关注点 x₀:</label>
            <input
              type="number"
              id="point-of-interest"
              v-model.number="pointOfInterest"
              step="0.1"
            />
          </div>

          <div class="input-group">
            <label for="epsilon">ε (接近度):</label>
            <input
              type="range"
              id="epsilon"
              v-model.number="epsilon"
              min="0.01"
              max="5"
              step="0.01"
            />
            <span>{{ epsilon.toFixed(2) }}</span>
          </div>

          <div class="animation-controls">
            <label>
              <input type="checkbox" v-model="showAnimation" />
              显示动画演示
            </label>

            <div class="speed-control" v-if="showAnimation">
              <label for="animation-speed">动画速度:</label>
              <input
                type="range"
                id="animation-speed"
                v-model.number="animationSpeed"
                min="10"
                max="100"
                step="5"
              />
              <span>{{ animationSpeed }}</span>
            </div>
          </div>

          <button @click="resetLimits" class="limit-btn">计算极限</button>
        </div>
      </div>

      <div class="results">
        <h3>极限结果</h3>
        <div class="result-info">
          <div class="result-row">
            <span class="label">左极限 (x → {{ pointOfInterest }}⁻):</span>
            <span class="value left-value">{{ leftLimit }}</span>
          </div>

          <div class="result-row">
            <span class="label">右极限 (x → {{ pointOfInterest }}⁺):</span>
            <span class="value right-value">{{ rightLimit }}</span>
          </div>

          <div class="result-row">
            <span class="label">函数值 f({{ pointOfInterest }}):</span>
            <span class="value">{{ functionValue }}</span>
          </div>

          <div class="continuity">
            <span class="label">连续性:</span>
            <span
              class="value"
              :class="{ 'continuous': isContinuous, 'discontinuous': !isContinuous }"
            >
              {{ isContinuous ? '函数在此点连续' : '函数在此点不连续' }}
            </span>
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
      </div>
    </div>

    <div class="graph-container">
      <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
      <svg ref="svgRef" class="limit-graph"></svg>
      <div class="animation-legend" v-if="showAnimation">
        <div class="legend-item">
          <div class="color-box left-color"></div>
          <span>左侧逼近 (从左至右)</span>
        </div>
        <div class="legend-item">
          <div class="color-box right-color"></div>
          <span>右侧逼近 (从右至左)</span>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.limit-demonstrator {
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

.limit-controls, .view-controls, .results {
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

.limit-inputs, .range-inputs {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.animation-controls {
  display: flex;
  flex-direction: column;
  gap: 10px;
  background-color: var(--bg-medium);
  padding: 12px;
  border-radius: var(--radius-md);
  margin-top: 4px;
}

.animation-controls label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.9rem;
  color: var(--text-secondary);
  cursor: pointer;
}

.animation-controls input[type="checkbox"] {
  appearance: none;
  -webkit-appearance: none;
  width: 18px;
  height: 18px;
  border: 1px solid var(--border-color);
  border-radius: var(--radius-sm);
  background-color: var(--bg-dark);
  display: grid;
  place-content: center;
  cursor: pointer;
}

.animation-controls input[type="checkbox"]::before {
  content: "";
  width: 10px;
  height: 10px;
  transform: scale(0);
  transition: transform var(--transition-fast);
  box-shadow: inset 1em 1em var(--primary-color);
  transform-origin: center;
  clip-path: polygon(14% 44%, 0 65%, 50% 100%, 100% 16%, 80% 0%, 43% 62%);
}

.animation-controls input[type="checkbox"]:checked::before {
  transform: scale(1);
}

.speed-control {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-left: 24px;
  margin-top: 6px;
}

.speed-control label {
  font-size: 0.85rem;
  color: var(--text-secondary);
  min-width: 90px;
}

.speed-control input {
  flex-grow: 1;
}

.speed-control span {
  font-weight: 500;
  color: var(--primary-light);
  background-color: var(--bg-dark);
  padding: 2px 8px;
  border-radius: var(--radius-sm);
  font-size: 0.85rem;
  min-width: 30px;
  text-align: center;
}

.limit-btn, .reset-btn {
  align-self: flex-start;
  margin-top: 16px;
  background: linear-gradient(135deg, var(--primary-color), var(--primary-light));
  color: white;
  border: none;
  padding: 10px 20px;
  font-size: 0.95rem;
  font-weight: 500;
  border-radius: var(--radius-sm);
  transition: all var(--transition-fast);
  box-shadow: var(--shadow-sm);
}

.limit-btn:hover, .reset-btn:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-md);
  filter: brightness(1.1);
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

.results {
  background-color: var(--bg-medium);
  border-radius: var(--radius-md);
  padding: 16px;
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.1);
}

.result-info {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.result-row {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.label {
  font-size: 0.9rem;
  color: var(--text-secondary);
}

.value {
  font-weight: 600;
  font-family: 'Consolas', 'Monaco', monospace;
  font-size: 1.1rem;
  padding: 4px 8px;
  background-color: var(--bg-dark);
  border-radius: var(--radius-sm);
  display: inline-block;
}

.left-value {
  color: var(--warning-color);
}

.right-value {
  color: var(--info-color);
}

.continuity {
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px dashed var(--border-color);
}

.continuous {
  color: var(--success-color);
  display: flex;
  align-items: center;
  gap: 6px;
}

.continuous::before {
  content: '✓';
  font-weight: bold;
}

.discontinuous {
  color: var(--error-color);
  display: flex;
  align-items: center;
  gap: 6px;
}

.discontinuous::before {
  content: '✗';
  font-weight: bold;
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
  position: relative;
  box-shadow: var(--shadow-md);
}

.limit-graph {
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

.animation-legend {
  margin-top: 16px;
  display: flex;
  gap: 20px;
  background-color: var(--bg-medium);
  padding: 12px 16px;
  border-radius: var(--radius-md);
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.9rem;
  color: var(--text-secondary);
}

.color-box {
  width: 14px;
  height: 14px;
  border-radius: 3px;
  box-shadow: var(--shadow-sm);
}

.left-color {
  background-color: var(--warning-color);
}

.right-color {
  background-color: var(--info-color);
}

@media (max-width: 900px) {
  .limit-demonstrator {
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

  .animation-legend {
    flex-direction: column;
    gap: 10px;
  }
}
</style>
