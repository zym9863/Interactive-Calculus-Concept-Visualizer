<script setup lang="ts">
import FunctionVisualizer from './components/FunctionVisualizer.vue'
import LimitDemonstrator from './components/LimitDemonstrator.vue'
import { ref } from 'vue'

const activeTab = ref('function')
</script>

<template>
  <div class="container">
    <header>
      <h1>Interactive Calculus Concept Visualizer</h1>
      <div class="tabs">
        <button
          :class="{ active: activeTab === 'function' }"
          @click="activeTab = 'function'">
          函数图像与参数探索
        </button>
        <button
          :class="{ active: activeTab === 'limit' }"
          @click="activeTab = 'limit'">
          极限与连续性演示
        </button>
      </div>
    </header>

    <main>
      <FunctionVisualizer v-if="activeTab === 'function'" />
      <LimitDemonstrator v-else-if="activeTab === 'limit'" />
    </main>

    <footer>
      <p>Interactive Calculus Concept Visualizer &copy; 2025</p>
    </footer>
  </div>
</template>

<style>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
  text-align: center;
}

header {
  margin-bottom: 2.5rem;
  position: relative;
}

header::after {
  content: '';
  position: absolute;
  bottom: -1rem;
  left: 50%;
  transform: translateX(-50%);
  width: 100px;
  height: 3px;
  background: linear-gradient(90deg, var(--primary-color), var(--accent-color));
  border-radius: 3px;
}

.tabs {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin: 1.5rem 0 2.5rem;
  position: relative;
  z-index: 1;
}

.tabs button {
  position: relative;
  overflow: hidden;
  padding: 0.8em 1.5em;
  font-weight: 600;
  letter-spacing: 0.5px;
  transition: all var(--transition-normal);
}

.tabs button::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(135deg, var(--primary-color), var(--primary-light));
  opacity: 0;
  z-index: -1;
  transition: opacity var(--transition-normal);
}

.tabs button:hover::before {
  opacity: 0.1;
}

.tabs button.active {
  border-color: var(--primary-color);
  background-color: var(--bg-light);
  transform: translateY(-3px);
  box-shadow: var(--shadow-md);
}

.tabs button.active::before {
  opacity: 0.15;
}

main {
  min-height: 500px;
  position: relative;
  z-index: 0;
}

footer {
  margin-top: 3rem;
  padding-top: 1.5rem;
  font-size: 0.85rem;
  color: var(--text-tertiary);
  position: relative;
}

footer::before {
  content: '';
  position: absolute;
  top: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 80px;
  height: 2px;
  background: linear-gradient(90deg, var(--primary-color), var(--accent-color));
  opacity: 0.3;
  border-radius: 2px;
}

@media (max-width: 768px) {
  .container {
    padding: 1.5rem 1rem;
  }

  .tabs {
    flex-direction: column;
    align-items: center;
    gap: 0.75rem;
  }

  .tabs button {
    width: 100%;
    max-width: 300px;
  }
}
</style>
