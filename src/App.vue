<template>
  <div class="game-container">
    <GameHeader :score="score" :level="level" :trapped-cats="trappedCats" :total-cats="totalCats" />

    <StartMenu
      v-if="showMenu"
      @start-1p="startSinglePlayer"
      @start-2p="startTwoPlayers"
      @how-to="openHowTo"
      @settings="openSettings"
    />

    <GameBoard
      v-else
      :grid="grid"
      :cats="cats"
      :mouse-pos="mousePos"
      :mouse2-pos="mouse2Pos"
      :two-player="twoPlayer"
      :score="score"
      :level="level"
      :trapped-cats="trappedCats"
      :total-cats="totalCats"
      :game-over="gameOver"
      :won="won"
      :grid-size="GRID_SIZE"
      :cell-size="CELL_SIZE"
      :top-dog-active="topDogActive"
      @restart="restartGame"
      @next="nextLevel"
    />
  </div>
</template>

<script>
export default {
  computed: {},
}
</script>

<script setup>
import { ref, reactive, computed, onMounted, onBeforeUnmount } from 'vue'
import GameBoard from './components/GameBoard.vue'
import GameHeader from './components/GameHeader.vue'
import StartMenu from './components/StartMenu.vue'

const GRID_SIZE = 20
const CELL_SIZE = 32
const CELL_TYPES = {
  EMPTY: 0,
  BLOCK: 1,
  WALL: 2,
}

const grid = ref([])
const mousePos = reactive({ x: Math.floor(GRID_SIZE / 2), y: Math.floor(GRID_SIZE / 2) })
const mouse2Pos = reactive({ x: 12, y: 12 })
const twoPlayer = ref(false)
const cats = ref([])
const score = ref(0)
const level = ref(1)
const gameOver = ref(false)
const won = ref(false)
const trappedCats = ref(0)
let gameLoopInterval = null
const showMenu = ref(true)

// TopDog power-up state (replace omni color logic)
const topDogActive = ref(false)
let topDogTimeout = null

// computed
const totalCats = computed(() => cats.value.length)

// methods
const initializeGrid = () => {
  const newGrid = Array(GRID_SIZE)
    .fill(null)
    .map(() => Array(GRID_SIZE).fill(CELL_TYPES.EMPTY))

  for (let i = 0; i < GRID_SIZE; i++) {
    newGrid[0][i] = CELL_TYPES.WALL
    newGrid[GRID_SIZE - 1][i] = CELL_TYPES.WALL
    newGrid[i][0] = CELL_TYPES.WALL
    newGrid[i][GRID_SIZE - 1] = CELL_TYPES.WALL
  }

  const blockCount = 100 - level.value * 5
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
  // Place cats at the four corners inside the walls
  const innerMax = GRID_SIZE - 2
  const corners = [
    { x: 1, y: 1 }, // top-left
    { x: innerMax, y: 1 }, // top-right
    { x: 1, y: innerMax }, // bottom-left
    { x: innerMax, y: innerMax }, // bottom-right
  ]

  const newCats = corners
    // Ensure corner cells are empty (not blocks); if a block is there, clear it
    .map((pos) => {
      if (grid.value[pos.y][pos.x] !== CELL_TYPES.EMPTY) {
        grid.value[pos.y][pos.x] = CELL_TYPES.EMPTY
      }
      return { x: pos.x, y: pos.y, trapped: false }
    })

  cats.value = newCats
}

function handleKeyDown(e) {
  if (gameOver.value || won.value) return

  const key = e.key.toLowerCase()

  // Player 1: Arrow keys
  if (key === 'arrowup') return moveMouseBy(mousePos, 0, -1)
  if (key === 'arrowdown') return moveMouseBy(mousePos, 0, 1)
  if (key === 'arrowleft') return moveMouseBy(mousePos, -1, 0)
  if (key === 'arrowright') return moveMouseBy(mousePos, 1, 0)

  // Player 2: WASD when enabled
  if (twoPlayer.value) {
    if (key === 'w') return moveMouseBy(mouse2Pos, 0, -1)
    if (key === 's') return moveMouseBy(mouse2Pos, 0, 1)
    if (key === 'a') return moveMouseBy(mouse2Pos, -1, 0)
    if (key === 'd') return moveMouseBy(mouse2Pos, 1, 0)
  }
}

function moveMouseBy(playerPos, dx, dy) {
  const newX = playerPos.x + dx
  const newY = playerPos.y + dy

  if (newX < 0 || newX >= GRID_SIZE || newY < 0 || newY >= GRID_SIZE) return

  const targetCell = grid.value[newY][newX]
  // Prevent both players from occupying the same cell
  if (twoPlayer.value) {
    const other = playerPos === mousePos ? mouse2Pos : mousePos
    if (other.x === newX && other.y === newY) return
  }

  if (targetCell === CELL_TYPES.WALL) return

  // If moving onto a trapped cat (cheese), activate TopDog and remove the cheese
  const cheeseAtTarget = cats.value.find((c) => c.x === newX && c.y === newY && c.trapped)
  if (playerPos === mousePos && cheeseAtTarget) {
    activateTopDog()
    // remove the cheese from the board
    const idx = cats.value.findIndex((c) => c.x === newX && c.y === newY && c.trapped)
    if (idx !== -1) cats.value.splice(idx, 1)
  }

  if (targetCell === CELL_TYPES.BLOCK) {
    // Find contiguous chain of blocks in the push direction
    let chainEndX = newX
    let chainEndY = newY
    while (true) {
      const nextX = chainEndX + dx
      const nextY = chainEndY + dy
      if (nextX < 0 || nextX >= GRID_SIZE || nextY < 0 || nextY >= GRID_SIZE) {
        // Out of bounds: cannot push
        return
      }
      const nextCell = grid.value[nextY][nextX]
      if (nextCell === CELL_TYPES.BLOCK) {
        // Continue extending the chain
        chainEndX = nextX
        chainEndY = nextY
        continue
      }
      // First non-block encountered
      // Only allow push if this cell is empty and not occupied by a cat
      if (
        nextCell === CELL_TYPES.EMPTY &&
        !cats.value.some((c) => c.x === nextX && c.y === nextY)
      ) {
        // Shift the entire chain forward by one cell, from the end backward
        let curX = chainEndX
        let curY = chainEndY
        while (!(curX === newX && curY === newY)) {
          const prevX = curX - dx
          const prevY = curY - dy
          grid.value[curY][curX] = grid.value[prevY][prevX]
          curX = prevX
          curY = prevY
        }
        // Move the first block into the adjacent cell to the chain end
        grid.value[nextY][nextX] = CELL_TYPES.BLOCK
        // Vacate the original adjacent cell where the mouse will step
        grid.value[newY][newX] = CELL_TYPES.EMPTY

        // Move player into the space vacated by the first block
        playerPos.x = newX
        playerPos.y = newY
        checkAllTrapped()
      }
      // If the next cell is not empty (wall or cat or out of bounds), cannot push
      return
    }
  }

  // Handle collision with active cat
  const activeCatIndex = cats.value.findIndex((c) => c.x === newX && c.y === newY && !c.trapped)
  if (activeCatIndex !== -1) {
    if (playerPos === mousePos && topDogActive.value) {
      // Eat the cat during TopDog power-up
      cats.value.splice(activeCatIndex, 1)
      score.value += 200
    } else {
      gameOver.value = true
      return
    }
  }

  playerPos.x = newX
  playerPos.y = newY
  checkAllTrapped()
}

function activateTopDog() {
  if (topDogTimeout) clearTimeout(topDogTimeout)
  topDogActive.value = true
  topDogTimeout = setTimeout(() => {
    topDogActive.value = false
    topDogTimeout = null
  }, 5000)
}

function moveCats() {
  if (gameOver.value || won.value) return

  cats.value.forEach((cat) => {
    if (cat.trapped) return

    const dx = Math.sign(mousePos.x - cat.x)
    const dy = Math.sign(mousePos.y - cat.y)

    // Primary preferred moves: axis-aligned toward the mouse, then diagonal
    const primaryMoves = []
    if (dx !== 0) primaryMoves.push({ x: cat.x + dx, y: cat.y })
    if (dy !== 0) primaryMoves.push({ x: cat.x, y: cat.y + dy })
    if (dx !== 0 && dy !== 0) primaryMoves.push({ x: cat.x + dx, y: cat.y + dy })

    let moved = false
    for (const move of primaryMoves) {
      if (canCatEnter(move.x, move.y)) {
        // If moving onto cheese (trapped cat), revert it to active
        reviveCheeseAt(move.x, move.y)
        cat.x = move.x
        cat.y = move.y
        moved = true
        break
      }
    }

    // If stuck (no primary move), try lateral wiggle: left/right relative to desired horizontal direction
    if (!moved) {
      const lateralMoves = []
      // If trying to move horizontally, wiggle up/down
      if (dx !== 0) {
        lateralMoves.push({ x: cat.x, y: cat.y + 1 })
        lateralMoves.push({ x: cat.x, y: cat.y - 1 })
      }
      // If trying to move vertically, wiggle left/right
      if (dy !== 0) {
        lateralMoves.push({ x: cat.x + 1, y: cat.y })
        lateralMoves.push({ x: cat.x - 1, y: cat.y })
      }

      // Shuffle to avoid bias
      lateralMoves.sort(() => Math.random() - 0.5)
      for (const move of lateralMoves) {
        if (canCatEnter(move.x, move.y)) {
          reviveCheeseAt(move.x, move.y)
          cat.x = move.x
          cat.y = move.y
          moved = true
          break
        }
      }
    }

    // Fallback: try any adjacent orthogonal move to unstick
    if (!moved) {
      const anyMoves = [
        { x: cat.x + 1, y: cat.y },
        { x: cat.x - 1, y: cat.y },
        { x: cat.x, y: cat.y + 1 },
        { x: cat.x, y: cat.y - 1 },
      ]
      anyMoves.sort(() => Math.random() - 0.5)
      for (const move of anyMoves) {
        if (canCatEnter(move.x, move.y)) {
          reviveCheeseAt(move.x, move.y)
          cat.x = move.x
          cat.y = move.y
          moved = true
          break
        }
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

// Cats can enter empty cells or cells occupied by a trapped cat (cheese)
function canCatEnter(x, y) {
  if (x < 0 || x >= GRID_SIZE || y < 0 || y >= GRID_SIZE) return false
  // A cat can always move onto the mouse position (this would trigger game over)
  if (x === mousePos.x && y === mousePos.y) return true
  const cell = grid.value[y][x]
  if (cell !== CELL_TYPES.EMPTY) return false
  // Blocked if an active cat occupies it
  if (cats.value.some((c) => c.x === x && c.y === y && !c.trapped)) return false
  // Allowed if occupied by a trapped cat (cheese)
  return true
}

function reviveCheeseAt(x, y) {
  const cheese = cats.value.find((c) => c.x === x && c.y === y && c.trapped)
  if (cheese) {
    cheese.trapped = false
    // Adjust trappedCats/score because cheese reverted
    trappedCats.value = cats.value.filter((c) => c.trapped).length
  }
}

function checkAllTrapped() {
  let changed = true
  let newlyTrappedCount = 0

  while (changed) {
    changed = false

    cats.value.forEach((cat, idx) => {
      if (cat.trapped) return

      const directions = [
        { dx: 0, dy: -1 },
        { dx: 0, dy: 1 },
        { dx: -1, dy: 0 },
        { dx: 1, dy: 0 },
      ]

      let hasEscape = false
      for (const dir of directions) {
        const nx = cat.x + dir.dx
        const ny = cat.y + dir.dy
        // If move is within board and the cell is empty and not occupied by a non-trapped cat, it's an escape
        if (
          nx >= 0 &&
          nx < GRID_SIZE &&
          ny >= 0 &&
          ny < GRID_SIZE &&
          grid.value[ny][nx] === CELL_TYPES.EMPTY &&
          // occupied by another cat that is NOT trapped is considered an escape; trapped cats block
          !cats.value.some((c) => c.x === nx && c.y === ny && !c.trapped)
        ) {
          hasEscape = true
          break
        }
      }

      if (!hasEscape) {
        cat.trapped = true
        newlyTrappedCount++
        score.value += 100
        changed = true
      }
    })
  }

  trappedCats.value = cats.value.filter((c) => c.trapped).length

  if (trappedCats.value === cats.value.length && cats.value.length > 0) {
    won.value = true
    score.value += 500 * level.value
  }
}

function isTrapped(x, y) {
  // A cat is trapped if it has no valid orthogonal moves.
  const directions = [
    { dx: 0, dy: -1 },
    { dx: 0, dy: 1 },
    { dx: -1, dy: 0 },
    { dx: 1, dy: 0 },
  ]

  for (const dir of directions) {
    const nx = x + dir.dx
    const ny = y + dir.dy
    if (isValidCatMove(nx, ny)) {
      return false
    }
  }
  return true
}

function startNewGame() {
  // Center the primary mouse at the grid center
  mousePos.x = Math.floor(GRID_SIZE / 2)
  mousePos.y = Math.floor(GRID_SIZE / 2)
  gameOver.value = false
  won.value = false
  trappedCats.value = 0

  initializeGrid()
  initializeCats()

  if (gameLoopInterval) clearInterval(gameLoopInterval)

  gameLoopInterval = setInterval(() => moveCats(), 500)
}

function startSinglePlayer() {
  showMenu.value = false
  twoPlayer.value = false
  level.value = 1
  score.value = 0
  startNewGame()
}

function startTwoPlayers() {
  showMenu.value = false
  twoPlayer.value = true
  level.value = 1
  score.value = 0
  // Position players apart
  mousePos.x = 5
  mousePos.y = 5
  mouse2Pos.x = GRID_SIZE - 6
  mouse2Pos.y = GRID_SIZE - 6
  startNewGame()
}

function openHowTo() {
  alert(
    'How to Play: Use arrow keys or WASD to move the mouse. Push blocks to trap cats on all four sides.',
  )
}

function openSettings() {
  alert('Settings coming soon.')
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
  // Do not auto-start game; show menu first
  window.addEventListener('keydown', handleKeyDown)
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeyDown)
  if (gameLoopInterval) clearInterval(gameLoopInterval)
  if (topDogTimeout) clearTimeout(topDogTimeout)
})
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html,
body,
#app {
  border: 5px solid red;
  min-height: 100vh;
  height: 100%;
}

body {
  font-family: 'Arial', sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  padding: 20px;
  height: 100%;
}

#app {
  text-align: center;
  border: 5px solid red;
}

/* no CSS vars needed for TopDog */

.game-container {
  display: flex;
  flex-direction: column;
  background: black;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.3);
  height: 100%;
  border: 5px solid red;
}

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
  color: white;
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
  background: #90ee90; /* base color */
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
