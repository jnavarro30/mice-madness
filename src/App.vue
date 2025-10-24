<script setup></script>

<template>
  <div class="game-container">
    <h1 style="color: #667eea; margin-bottom: 20px">🐭 Rodents' Revenge 🐭</h1>

    <div class="game-header">
      <div class="stat">
        <div>Score</div>
        <div class="stat-value">{{ score }}</div>
      </div>
      <div class="stat">
        <div>Level</div>
        <div class="stat-value">{{ level }}</div>
      </div>
      <div class="stat">
        <div>Trapped Cats</div>
        <div class="stat-value">{{ trappedCats }} / {{ totalCats }}</div>
      </div>
    </div>

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
        <button class="button" @click="won ? nextLevel() : restartGame()">
          {{ won ? 'Next Level' : 'Try Again' }}
        </button>
      </div>
    </div>

    <div class="instructions">
      <h3>How to Play:</h3>
      <ul>
        <li>Use <strong>Arrow Keys</strong> or <strong>WASD</strong> to move</li>
        <li>Push blocks to trap all the cats</li>
        <li>A cat is trapped when surrounded by blocks/walls</li>
        <li>Don't let the cats catch you!</li>
        <li>Trap all cats to win the level</li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onBeforeUnmount } from 'vue'

const GRID_SIZE = 15
const CELL_SIZE = 32
const CELL_TYPES = {
  EMPTY: 0,
  BLOCK: 1,
  WALL: 2,
}

const grid = ref([])
const mousePos = reactive({ x: 7, y: 7 })
const cats = ref([])
const score = ref(0)
const level = ref(1)
const gameOver = ref(false)
const won = ref(false)
const trappedCats = ref(0)
let gameLoopInterval = null

// 🧮 COMPUTED
const gridStyle = computed(() => ({
  gridTemplateColumns: `repeat(${GRID_SIZE}, ${CELL_SIZE}px)`,
  gridTemplateRows: `repeat(${GRID_SIZE}, ${CELL_SIZE}px)`,
}))

const totalCats = computed(() => cats.value.length)

const flattenedGrid = computed(() => {
  const flattened = []
  for (let y = 0; y < grid.value.length; y++) {
    for (let x = 0; x < grid.value[y].length; x++) {
      flattened.push({
        x,
        y,
        key: `${x}-${y}`,
      })
    }
  }
  return flattened
})

// 🧩 METHODS
function initializeGrid() {
  const newGrid = Array(GRID_SIZE)
    .fill(null)
    .map(() => Array(GRID_SIZE).fill(CELL_TYPES.EMPTY))

  for (let i = 0; i < GRID_SIZE; i++) {
    newGrid[0][i] = CELL_TYPES.WALL
    newGrid[GRID_SIZE - 1][i] = CELL_TYPES.WALL
    newGrid[i][0] = CELL_TYPES.WALL
    newGrid[i][GRID_SIZE - 1] = CELL_TYPES.WALL
  }

  const blockCount = 40 + level.value * 5
  for (let i = 0; i < blockCount; i++) {
    let x, y
    do {
      x = Math.floor(Math.random() * (GRID_SIZE - 2)) + 1
      y = Math.floor(Math.random() * (GRID_SIZE - 2)) + 1
    } while (newGrid[y][x] !== CELL_TYPES.EMPTY || (Math.abs(x - 7) < 2 && Math.abs(y - 7) < 2))
    newGrid[y][x] = CELL_TYPES.BLOCK
  }

  grid.value = newGrid
}

function initializeCats() {
  const newCats = []
  const catCount = 3 + Math.floor(level.value / 2)

  for (let i = 0; i < catCount; i++) {
    let x, y
    do {
      x = Math.floor(Math.random() * (GRID_SIZE - 2)) + 1
      y = Math.floor(Math.random() * (GRID_SIZE - 2)) + 1
    } while (grid.value[y][x] !== CELL_TYPES.EMPTY || (Math.abs(x - 7) < 3 && Math.abs(y - 7) < 3))
    newCats.push({ x, y, trapped: false })
  }

  cats.value = newCats
}

function getCellClass(x, y) {
  const cell = grid.value[y]?.[x]

  if (mousePos.x === x && mousePos.y === y) {
    return 'mouse'
  }

  const cat = cats.value.find((c) => c.x === x && c.y === y)
  if (cat) return cat.trapped ? 'cat trapped' : 'cat'

  if (cell === CELL_TYPES.WALL) return 'wall'
  if (cell === CELL_TYPES.BLOCK) return 'block'
  return 'empty'
}

function getCellContent(x, y) {
  if (mousePos.x === x && mousePos.y === y) return '🐭'
  const cat = cats.value.find((c) => c.x === x && c.y === y)
  if (cat) return cat.trapped ? '🧀' : '🐱'
  return ''
}

function handleKeyDown(e) {
  if (gameOver.value || won.value) return

  const key = e.key.toLowerCase()
  let dx = 0,
    dy = 0

  if (key === 'arrowup' || key === 'w') dy = -1
  else if (key === 'arrowdown' || key === 's') dy = 1
  else if (key === 'arrowleft' || key === 'a') dx = -1
  else if (key === 'arrowright' || key === 'd') dx = 1
  else return

  e.preventDefault()
  moveMouse(dx, dy)
}

function moveMouse(dx, dy) {
  const newX = mousePos.x + dx
  const newY = mousePos.y + dy

  if (newX < 0 || newX >= GRID_SIZE || newY < 0 || newY >= GRID_SIZE) return

  const targetCell = grid.value[newY][newX]

  if (targetCell === CELL_TYPES.WALL) return

  if (targetCell === CELL_TYPES.BLOCK) {
    const pushX = newX + dx
    const pushY = newY + dy

    if (
      pushX >= 0 &&
      pushX < GRID_SIZE &&
      pushY >= 0 &&
      pushY < GRID_SIZE &&
      grid.value[pushY][pushX] === CELL_TYPES.EMPTY &&
      !cats.value.some((c) => c.x === pushX && c.y === pushY)
    ) {
      grid.value[pushY][pushX] = CELL_TYPES.BLOCK
      grid.value[newY][newX] = CELL_TYPES.EMPTY
      mousePos.x = newX
      mousePos.y = newY
      checkAllTrapped()
    }
    return
  }

  if (cats.value.some((c) => c.x === newX && c.y === newY && !c.trapped)) {
    gameOver.value = true
    return
  }

  mousePos.x = newX
  mousePos.y = newY
}

function moveCats() {
  if (gameOver.value || won.value) return

  cats.value.forEach((cat) => {
    if (cat.trapped) return

    const dx = Math.sign(mousePos.x - cat.x)
    const dy = Math.sign(mousePos.y - cat.y)
    const moves = []

    if (dx !== 0) moves.push({ x: cat.x + dx, y: cat.y })
    if (dy !== 0) moves.push({ x: cat.x, y: cat.y + dy })
    if (dx !== 0 && dy !== 0) moves.push({ x: cat.x + dx, y: cat.y + dy })

    moves.sort(() => Math.random() - 0.5)

    for (const move of moves) {
      if (isValidCatMove(move.x, move.y)) {
        cat.x = move.x
        cat.y = move.y
        break
      }
    }

    if (cat.x === mousePos.x && cat.y === mousePos.y) {
      gameOver.value = true
    }
  })

  checkAllTrapped()
}

function isValidCatMove(x, y) {
  if (x < 0 || x >= GRID_SIZE || y < 0 || y >= GRID_SIZE) return false
  const cell = grid.value[y][x]
  if (cell !== CELL_TYPES.EMPTY) return false
  if (cats.value.some((c) => c.x === x && c.y === y)) return false
  return true
}

function checkAllTrapped() {
  let newlyTrapped = 0

  cats.value.forEach((cat) => {
    if (!cat.trapped && isTrapped(cat.x, cat.y)) {
      cat.trapped = true
      newlyTrapped++
      score.value += 100
    }
  })

  trappedCats.value = cats.value.filter((c) => c.trapped).length

  if (trappedCats.value === cats.value.length && cats.value.length > 0) {
    won.value = true
    score.value += 500 * level.value
  }
}

function isTrapped(x, y) {
  const directions = [
    { dx: 0, dy: -1 },
    { dx: 0, dy: 1 },
    { dx: -1, dy: 0 },
    { dx: 1, dy: 0 },
  ]

  for (const dir of directions) {
    const newX = x + dir.dx
    const newY = y + dir.dy

    if (newX < 0 || newX >= GRID_SIZE || newY < 0 || newY >= GRID_SIZE) continue

    const cell = grid.value[newY][newX]
    if (cell === CELL_TYPES.EMPTY) return false
  }

  return true
}

function startNewGame() {
  mousePos.x = 7
  mousePos.y = 7
  gameOver.value = false
  won.value = false
  trappedCats.value = 0

  initializeGrid()
  initializeCats()

  if (gameLoopInterval) clearInterval(gameLoopInterval)

  gameLoopInterval = setInterval(() => moveCats(), 500)
}

function restartGame() {
  level.value = 1
  score.value = 0
  startNewGame()
}

function nextLevel() {
  level.value++
  startNewGame()
}

// 🎮 LIFECYCLE
onMounted(() => {
  startNewGame()
  window.addEventListener('keydown', handleKeyDown)
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeyDown)
  if (gameLoopInterval) clearInterval(gameLoopInterval)
})
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Arial', sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  padding: 20px;
}

#app {
  text-align: center;
}

.game-container {
  background: white;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.3);
}

.game-header {
  display: flex;
  justify-content: space-around;
  margin-bottom: 20px;
  gap: 20px;
}

.stat {
  background: #f0f0f0;
  padding: 10px 20px;
  border-radius: 8px;
  font-weight: bold;
}

.stat-value {
  color: #667eea;
  font-size: 24px;
}

.game-board {
  display: inline-block;
  border: 3px solid #333;
  background: #e0e0e0;
  position: relative;
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

.instructions {
  margin-top: 20px;
  text-align: left;
  background: #f9f9f9;
  padding: 15px;
  border-radius: 8px;
  max-width: 500px;
  margin-left: auto;
  margin-right: auto;
}

.instructions h3 {
  margin-bottom: 10px;
  color: #667eea;
}

.instructions ul {
  margin-left: 20px;
}

.instructions li {
  margin: 5px 0;
}
</style>
