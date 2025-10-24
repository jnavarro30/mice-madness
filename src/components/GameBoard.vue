<template>
  <div class="game-board">
    <div class="grid" :style="gridStyle">
      <div
        v-for="cell in flattenedGrid"
        :key="cell.key"
        :class="getCellClass(cell.x, cell.y)"
        class="cell"
      >
        {{ getCellContent(cell.x, cell.y) }}
      </div>
    </div>

    <div v-if="gameOver || won" class="game-over-overlay">
      <h2>{{ won ? '🎉 You Win! 🎉' : '💀 Game Over 💀' }}</h2>
      <p>Final Score: {{ score }}</p>
      <button class="button" @click="won ? emit('next') : emit('restart')">
        {{ won ? 'Next Level' : 'Try Again' }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({
  grid: { type: Array, required: true },
  cats: { type: Array, required: true },
  mousePos: { type: Object, required: true },
  mouse2Pos: { type: Object, required: false },
  twoPlayer: { type: Boolean, required: false, default: false },
  score: { type: Number, required: true },
  level: { type: Number, required: true },
  trappedCats: { type: Number, required: true },
  totalCats: { type: Number, required: true },
  gameOver: { type: Boolean, required: true },
  won: { type: Boolean, required: true },
  gridSize: { type: Number, required: true },
  cellSize: { type: Number, required: true },
})

const emit = defineEmits(['restart', 'next'])

const gridStyle = computed(() => ({
  gridTemplateColumns: `repeat(${props.gridSize}, ${props.cellSize}px)`,
  gridTemplateRows: `repeat(${props.gridSize}, ${props.cellSize}px)`,
}))

const flattenedGrid = computed(() => {
  const flattened = []
  for (let y = 0; y < props.grid.length; y++) {
    for (let x = 0; x < props.grid[y].length; x++) {
      flattened.push({ x, y, key: `${x}-${y}` })
    }
  }
  return flattened
})

function getCellClass(x, y) {
  const cell = props.grid[y]?.[x]

  if (props.mousePos.x === x && props.mousePos.y === y) {
    return 'mouse'
  }
  if (props.twoPlayer && props.mouse2Pos && props.mouse2Pos.x === x && props.mouse2Pos.y === y) {
    return 'mouse mouse2'
  }

  const cat = props.cats.find((c) => c.x === x && c.y === y)
  if (cat) return cat.trapped ? 'cat trapped' : 'cat'

  if (cell === 2) return 'wall'
  if (cell === 1) return 'block'
  return 'empty'
}

function getCellContent(x, y) {
  if (props.mousePos.x === x && props.mousePos.y === y) return '🐭'
  if (props.twoPlayer && props.mouse2Pos && props.mouse2Pos.x === x && props.mouse2Pos.y === y)
    return '🐭'
  const cat = props.cats.find((c) => c.x === x && c.y === y)
  if (cat) return cat.trapped ? '🧀' : '🐱'
  return ''
}
</script>

<style scoped>
.game-board {
  display: inline-block;
  border: 3px solid #333;
  background: #e0e0e0;
  position: relative;
  margin: 0 auto;
}

.grid {
  display: grid;
  gap: 1px;
  background: #999;
}

.cell {
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  transition: all 0.1s;
}

.empty {
  background: #f5f5f5;
}

.block {
  background: #8b4513;
  border: 2px solid #654321;
}

.wall {
  background: #333;
  border: 2px solid #000;
}

.mouse {
  background: #90ee90;
  border-radius: 50%;
  position: relative;
}
.mouse2 {
  background: #6ee7b7;
}

.cat {
  background: #ff6b6b;
  border-radius: 50%;
  position: relative;
}

.trapped {
  background: #ffd93d !important;
  animation: pulse 0.5s ease-in-out infinite;
}

@keyframes pulse {
  0%,
  100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.1);
  }
}

.game-over-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  color: white;
  font-size: 32px;
  font-weight: bold;
}

.game-over-overlay h2 {
  margin-bottom: 20px;
  font-size: 48px;
}

.button {
  background: #667eea;
  color: white;
  border: none;
  padding: 12px 24px;
  font-size: 18px;
  border-radius: 8px;
  cursor: pointer;
  margin-top: 20px;
  transition: background 0.3s;
}

.button:hover {
  background: #5568d3;
}
</style>
