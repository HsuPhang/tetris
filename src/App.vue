<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

// --- 游戏常量 ---
const COLS = 10
const ROWS = 20
const BLOCK_SIZE = 22

const SHAPES = [
  [[1, 1, 1, 1]],
  [[1, 1], [1, 1]],
  [[1, 1, 1], [0, 1, 0]],
  [[1, 1, 1], [1, 0, 0]],
  [[1, 1, 1], [0, 0, 1]],
  [[1, 1, 0], [0, 1, 1]],
  [[0, 1, 1], [1, 1, 0]]
]

const COLORS = ['#000', '#000', '#000', '#000', '#000', '#000', '#000']

// --- 响应式状态 ---
const board = ref([])
const currentPiece = ref(null)
const nextPiece = ref(null)
const score = ref(0)
const maxScore = ref(0)
const level = ref(1)
const startLine = ref(0)
const gameOver = ref(false)
const isPaused = ref(false)
const soundEnabled = ref(true)
let intervalId = null

// --- 核心逻辑 ---
const initBoard = () => {
  board.value = Array.from({ length: ROWS }, () => Array(COLS).fill(null))
}

const createPiece = () => {
  const index = Math.floor(Math.random() * SHAPES.length)
  return {
    shape: SHAPES[index],
    color: COLORS[index],
    x: Math.floor(COLS / 2) - Math.floor(SHAPES[index][0].length / 2),
    y: 0
  }
}

const checkCollision = (piece, offsetX = 0, offsetY = 0) => {
  return piece.shape.some((row, y) => {
    return row.some((value, x) => {
      if (!value) return false
      const newX = piece.x + x + offsetX
      const newY = piece.y + y + offsetY
      return (
          newX < 0 ||
          newX >= COLS ||
          newY >= ROWS ||
          (newY >= 0 && board.value[newY][newX])
      )
    })
  })
}

const rotate = (piece) => {
  const newShape = piece.shape[0].map((_, index) =>
      piece.shape.map(row => row[index]).reverse()
  )
  return { ...piece, shape: newShape }
}

const lockPiece = () => {
  currentPiece.value.shape.forEach((row, y) => {
    row.forEach((value, x) => {
      if (value) {
        board.value[currentPiece.value.y + y][currentPiece.value.x + x] = currentPiece.value.color
      }
    })
  })
  clearLines()
  currentPiece.value = nextPiece.value
  nextPiece.value = createPiece()
  if (checkCollision(currentPiece.value)) {
    gameOver.value = true
    clearInterval(intervalId)
    if (score.value > maxScore.value) maxScore.value = score.value
  }
}

const clearLines = () => {
  let linesCleared = 0
  for (let y = ROWS - 1; y >= 0; y--) {
    if (board.value[y].every(cell => cell !== null)) {
      board.value.splice(y, 1)
      board.value.unshift(Array(COLS).fill(null))
      linesCleared++
      y++
    }
  }
  if (linesCleared > 0) {
    score.value += linesCleared * 100 * linesCleared
    startLine.value += linesCleared
    level.value = Math.floor(startLine.value / 10) + 1
    resetInterval()
  }
}

const resetInterval = () => {
  if (intervalId) clearInterval(intervalId)
  intervalId = setInterval(() => {
    if (!isPaused.value && !gameOver.value) drop()
  }, Math.max(100, 1000 - (level.value - 1) * 100))
}

const drop = () => {
  if (!checkCollision(currentPiece.value, 0, 1)) {
    currentPiece.value.y++
  } else {
    lockPiece()
  }
}

const hardDrop = () => {
  if (gameOver.value || isPaused.value) return
  while (!checkCollision(currentPiece.value, 0, 1)) {
    currentPiece.value.y++
    score.value += 2
  }
  lockPiece()
}

const move = (dir) => {
  if (gameOver.value || isPaused.value) return
  if (!checkCollision(currentPiece.value, dir, 0)) {
    currentPiece.value.x += dir
  }
}

const rotatePiece = () => {
  if (gameOver.value || isPaused.value) return
  const rotated = rotate(currentPiece.value)
  if (!checkCollision(rotated)) {
    currentPiece.value = rotated
  }
}

const startGame = () => {
  initBoard()
  score.value = 0
  startLine.value = 0
  level.value = 1
  gameOver.value = false
  isPaused.value = false
  currentPiece.value = createPiece()
  nextPiece.value = createPiece()
  resetInterval()
}

const togglePause = () => {
  if (!gameOver.value) isPaused.value = !isPaused.value
}

const toggleSound = () => {
  soundEnabled.value = !soundEnabled.value
}

// --- 绘图逻辑 ---
let ctx = null
let nextCtx = null

const drawPiece = (targetCtx, piece, offsetX = 0, offsetY = 0, blockSize = BLOCK_SIZE) => {
  piece.shape.forEach((row, y) => {
    row.forEach((value, x) => {
      if (value) {
        targetCtx.fillStyle = '#000'
        targetCtx.fillRect((piece.x + x + offsetX) * blockSize + 1, (piece.y + y + offsetY) * blockSize + 1, blockSize - 2, blockSize - 2)
        targetCtx.strokeStyle = '#333'
        targetCtx.strokeRect((piece.x + x + offsetX) * blockSize + 1, (piece.y + y + offsetY) * blockSize + 1, blockSize - 2, blockSize - 2)
      }
    })
  })
}

const draw = () => {
  if (!ctx || !nextCtx) return
  // 清除画布
  ctx.fillStyle = '#b8c49c'; ctx.fillRect(0, 0, COLS * BLOCK_SIZE, ROWS * BLOCK_SIZE)
  nextCtx.fillStyle = '#b8c49c'; nextCtx.fillRect(0, 0, 100, 100)

  // 绘制网格
  ctx.strokeStyle = '#9ca87c'; ctx.lineWidth = 0.5
  for (let i = 0; i <= ROWS; i++) { ctx.beginPath(); ctx.moveTo(0, i * BLOCK_SIZE); ctx.lineTo(COLS * BLOCK_SIZE, i * BLOCK_SIZE); ctx.stroke() }
  for (let i = 0; i <= COLS; i++) { ctx.beginPath(); ctx.moveTo(i * BLOCK_SIZE, 0); ctx.lineTo(i * BLOCK_SIZE, ROWS * BLOCK_SIZE); ctx.stroke() }

  // 绘制落下的块
  board.value.forEach((row, y) => {
    row.forEach((color, x) => {
      if (color) {
        ctx.fillStyle = '#000'; ctx.fillRect(x * BLOCK_SIZE + 1, y * BLOCK_SIZE + 1, BLOCK_SIZE - 2, BLOCK_SIZE - 2)
      }
    })
  })

  if (currentPiece.value) drawPiece(ctx, currentPiece.value)
  if (nextPiece.value) {
    const ox = Math.floor((4 - nextPiece.value.shape[0].length) / 2)
    const oy = Math.floor((4 - nextPiece.value.shape.length) / 2)
    drawPiece(nextCtx, { ...nextPiece.value, x: ox, y: oy }, 0, 0, 22)
  }
}

const handleKeydown = (e) => {
  if (gameOver.value) { if (['Enter', 'r', 'R'].includes(e.key)) startGame(); return }
  if (isPaused.value) { if (['p', 'P'].includes(e.key)) togglePause(); return }
  switch (e.key) {
    case 'ArrowLeft': move(-1); break
    case 'ArrowRight': move(1); break
    case 'ArrowDown': drop(); score.value += 1; break
    case 'ArrowUp': rotatePiece(); break
    case ' ': hardDrop(); break
    case 'p': case 'P': togglePause(); break
    case 's': case 'S': toggleSound(); break
    case 'r': case 'R': startGame(); break
  }
}

onMounted(() => {
  ctx = document.getElementById('board').getContext('2d')
  nextCtx = document.getElementById('next').getContext('2d')
  window.addEventListener('keydown', handleKeydown)
  const gameLoop = () => { draw(); requestAnimationFrame(gameLoop) }
  gameLoop()
  startGame()
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  if (intervalId) clearInterval(intervalId)
})
</script>

<template>
  <div class="game-container">
    <div class="game-wrapper">
      <div class="game-header">
        <div class="header-decoration">••••</div>
        <div class="header-title">Good Old Tetris</div>
        <div class="header-decoration">••••</div>
      </div>

      <div class="game-screen">
        <div class="screen-border">
          <div class="screen-content">
            <canvas id="board" :width="COLS * BLOCK_SIZE" :height="ROWS * BLOCK_SIZE" class="game-board"></canvas>
            <div class="info-panel">
              <div class="info-row"><span class="info-label">Max</span><span class="info-value">{{ maxScore.toString().padStart(6, '0') }}</span></div>
              <div class="info-row"><span class="info-label">Start Line</span><span class="info-value">{{ startLine.toString().padStart(6, '0') }}</span></div>
              <div class="info-row"><span class="info-label">Level</span><span class="info-value">{{ level.toString().padStart(2, '0') }}</span></div>
              <div class="info-row"><span class="info-label">Next</span><canvas id="next" width="88" height="88" class="next-canvas"></canvas></div>
            </div>
            <div class="overlay" v-if="gameOver"><div class="text red">GAME OVER</div><div class="hint">Press R to restart</div></div>
            <div class="overlay" v-if="isPaused && !gameOver"><div class="text yellow">PAUSED</div><div class="hint">Press P to continue</div></div>
          </div>
        </div>
      </div>

      <div class="game-controls">
        <div class="control-group-left">
          <div class="functional-buttons">
            <button class="btn-small" @click="togglePause">
              <div class="circle green"></div><span class="label">Pause(P)</span>
            </button>
            <button class="btn-small" @click="toggleSound">
              <div class="circle green"></div><span class="label">Sound(S)</span>
            </button>
            <button class="btn-small" @click="startGame">
              <div class="circle red"></div><span class="label">Reset(R)</span>
            </button>
          </div>

          <button class="btn-big-drop" @click="hardDrop">
            <div class="circle-blue drop-size"></div>
            <span class="label">Drop (SPACE)</span>
          </button>
        </div>

        <div class="control-group-right">
          <div class="dpad">
            <div class="dpad-arrows">
              <div class="a-up"></div><div class="a-down"></div>
              <div class="a-left"></div><div class="a-right"></div>
            </div>
            <button class="dpad-btn pos-up" @click="rotatePiece">
              <div class="circle-blue mid-size"></div><span class="label">Rotation</span>
            </button>
            <button class="dpad-btn pos-left" @click="move(-1)">
              <div class="circle-blue mid-size"></div><span class="label">Left</span>
            </button>
            <button class="dpad-btn pos-right" @click="move(1)">
              <div class="circle-blue mid-size"></div><span class="label">Right</span>
            </button>
            <button class="dpad-btn pos-down" @click="drop">
              <div class="circle-blue mid-size"></div><span class="label">Down</span>
            </button>
          </div>
        </div>
      </div>

      <div class="side-decoration left"></div><div class="side-decoration right"></div>
    </div>
  </div>
</template>

<style scoped>
/* 容器布局 */
.game-container {
  display: flex; justify-content: center; align-items: center;
  height: 100vh; width: 100vw; background: #87ceeb;
}

.game-wrapper {
  position: relative; padding: 25px 35px 40px; border-radius: 30px;
  background: linear-gradient(180deg, #f4d53c 0%, #e6c000 100%);
  border: 4px solid #b89500; box-shadow: 0 15px 35px rgba(0,0,0,0.4);
  transform: scale(0.9);
  transform-origin: center center;
}

/* 屏幕样式 */
.header-title { font-family: 'Courier New', monospace; font-weight: bold; font-size: 24px; color: #000; }
.screen-border { background: #333; border-radius: 12px; padding: 10px; box-shadow: inset 0 2px 8px #000; }
.screen-content { background: #b8c49c; border-radius: 5px; padding: 10px; display: flex; gap: 15px; position: relative; }
.game-board { border: 2px solid #5a5a5a; }
.info-panel { width: 100px; display: flex; flex-direction: column; gap: 12px; }
.info-label { font-family: 'Courier New', monospace; font-size: 14px; color: #5a5a5a; }
.info-value { font-family: 'Courier New', monospace; font-size: 12px; font-weight: bold; color: #000; }
.next-canvas { border: 1px solid #5a5a5a; background: #b8c49c; }

/* 浮层 */
.overlay {
  position: absolute; top: 10px; left: 10px; right: 10px; bottom: 10px;
  background: rgba(0,0,0,0.85); display: flex; flex-direction: column;
  justify-content: center; align-items: center; border-radius: 5px; z-index: 10;
}
.overlay .text { font-family: 'Courier New', monospace; font-size: 24px; font-weight: bold; margin-bottom: 10px; }
.text.red { color: #ff4444; }
.text.yellow { color: #ffff00; }
.overlay .hint { color: #ccc; font-size: 12px; }

/* 控制台核心样式 */
.game-controls {
  display: flex; justify-content: space-between; align-items: flex-start;
  margin-top: 30px; width: 440px;
}

.control-group-left { display: flex; flex-direction: column; align-items: center; gap: 25px; }
.functional-buttons { display: flex; gap: 15px; }

/* 圆形按钮基础 */
button { background: none; border: none; cursor: pointer; padding: 0; outline: none; }
.circle { width: 34px; height: 34px; border-radius: 50%; border: 1px solid rgba(0,0,0,0.2); }
.circle.green { background: radial-gradient(circle at 30% 30%, #5af25a, #008000); box-shadow: 0 4px #005000, 0 6px 10px rgba(0,0,0,0.3); }
.circle.red { background: radial-gradient(circle at 30% 30%, #ff5a5a, #8b0000); box-shadow: 0 4px #500000, 0 6px 10px rgba(0,0,0,0.3); }
.circle:active { transform: translateY(2px); box-shadow: 0 2px #000; }

.circle-blue {
  border-radius: 50%; border: 1px solid rgba(0,0,0,0.3);
  background: radial-gradient(circle at 35% 35%, #7a96ff, #3b55d9);
  box-shadow: 0 6px #1a1a4a, 0 10px 15px rgba(0,0,0,0.3);
}
.circle-blue:active { transform: translateY(3px); box-shadow: 0 3px #1a1a4a; }

.drop-size { width: 110px; height: 110px; }
.mid-size { width: 68px; height: 68px; }

.label { font-family: 'Courier New', monospace; font-size: 11px; font-weight: bold; color: #000; margin-top: 5px; display: block; text-align: center;}

/* D-Pad 布局 */
.dpad { position: relative; width: 200px; height: 200px; }
.dpad-btn { position: absolute; display: flex; flex-direction: column; align-items: center; }
.pos-up { top: 0; left: 50%; transform: translateX(-50%); }
.pos-down { bottom: 0; left: 50%; transform: translateX(-50%); }
.pos-left { left: 0; top: 50%; transform: translateY(-50%); }
.pos-right { right: 0; top: 50%; transform: translateY(-50%); }

/* 中心箭头 */
.dpad-arrows { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 40px; height: 40px; }
.a-up, .a-down, .a-left, .a-right { position: absolute; width: 0; height: 0; border: 7px solid transparent; }
.a-up { border-bottom-color: #000; top: 0; left: 50%; transform: translateX(-50%); }
.a-down { border-top-color: #000; bottom: 0; left: 50%; transform: translateX(-50%); }
.a-left { border-right-color: #000; left: 0; top: 50%; transform: translateY(-50%); }
.a-right { border-left-color: #000; right: 0; top: 50%; transform: translateY(-50%); }

/* 装饰 */
.side-decoration { position: absolute; top: 50%; color: #b89500; font-size: 10px; line-height: 1.2; }
.side-decoration.left { left: 10px; }
.side-decoration.right { right: 10px; }
.side-decoration::after { content: "●●\A●●\A●●"; white-space: pre; }
</style>