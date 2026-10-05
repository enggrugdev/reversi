<script setup lang="ts">
import { computed, onUnmounted, ref } from 'vue'

type Player = 'black' | 'white'
type Difficulty = 'easy' | 'medium' | 'hard'
type GameMode = 'two-player' | 'solo'
type Cell = Player | null
type Position = { row: number; col: number }
type Board = Cell[][]
type GameSnapshot = {
  board: Board
  player: Player
  moves: number
  notice: string
}

const boardSize = 8
const columns = ['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H']
const directions = [
  [-1, -1],
  [-1, 0],
  [-1, 1],
  [0, -1],
  [0, 1],
  [1, -1],
  [1, 0],
  [1, 1],
] as const
const positionValues = [
  [100, -20, 10, 5, 5, 10, -20, 100],
  [-20, -40, -5, -5, -5, -5, -40, -20],
  [10, -5, 8, 3, 3, 8, -5, 10],
  [5, -5, 3, 3, 3, 3, -5, 5],
  [5, -5, 3, 3, 3, 3, -5, 5],
  [10, -5, 8, 3, 3, 8, -5, 10],
  [-20, -40, -5, -5, -5, -5, -40, -20],
  [100, -20, 10, 5, 5, 10, -20, 100],
]

function createInitialBoard(): Board {
  const nextBoard = Array.from({ length: boardSize }, () =>
    Array<Cell>(boardSize).fill(null),
  )

  nextBoard[3]![3] = 'white'
  nextBoard[3]![4] = 'black'
  nextBoard[4]![3] = 'black'
  nextBoard[4]![4] = 'white'

  return nextBoard
}

function getFlips(boardState: Board, row: number, col: number, player: Player): Position[] {
  if (boardState[row]?.[col] !== null) return []

  const opponent: Player = player === 'black' ? 'white' : 'black'
  const flips: Position[] = []

  for (const [rowStep, colStep] of directions) {
    const line: Position[] = []
    let nextRow = row + rowStep
    let nextCol = col + colStep

    while (
      nextRow >= 0 &&
      nextRow < boardSize &&
      nextCol >= 0 &&
      nextCol < boardSize &&
      boardState[nextRow]?.[nextCol] === opponent
    ) {
      line.push({ row: nextRow, col: nextCol })
      nextRow += rowStep
      nextCol += colStep
    }

    if (line.length > 0 && boardState[nextRow]?.[nextCol] === player) {
      flips.push(...line)
    }
  }

  return flips
}

function getLegalMoves(boardState: Board, player: Player): Map<string, Position[]> {
  const legalMoves = new Map<string, Position[]>()

  for (let row = 0; row < boardSize; row += 1) {
    for (let col = 0; col < boardSize; col += 1) {
      if (boardState[row]?.[col] !== null) continue

      const flips = getFlips(boardState, row, col, player)
      if (flips.length > 0) legalMoves.set(`${row}-${col}`, flips)
    }
  }

  return legalMoves
}

function applyMove(boardState: Board, position: Position, player: Player, flips: Position[]): Board {
  const nextBoard = boardState.map((line) => line.slice())
  nextBoard[position.row]![position.col] = player
  for (const flip of flips) nextBoard[flip.row]![flip.col] = player
  return nextBoard
}

function otherPlayer(player: Player): Player {
  return player === 'black' ? 'white' : 'black'
}

function evaluateBoard(boardState: Board, player: Player): number {
  const opponent = otherPlayer(player)
  let score = 0
  let playerDiscs = 0
  let opponentDiscs = 0

  for (let row = 0; row < boardSize; row += 1) {
    for (let col = 0; col < boardSize; col += 1) {
      const cell = boardState[row]?.[col]
      if (cell === player) {
        score += positionValues[row]![col]!
        playerDiscs += 1
      } else if (cell === opponent) {
        score -= positionValues[row]![col]!
        opponentDiscs += 1
      }
    }
  }

  const occupied = playerDiscs + opponentDiscs
  const discWeight = occupied > 48 ? 4 : 1
  const mobility =
    getLegalMoves(boardState, player).size - getLegalMoves(boardState, opponent).size

  return score + (playerDiscs - opponentDiscs) * discWeight + mobility * 8
}

function searchBestMove(
  boardState: Board,
  player: Player,
  rootPlayer: Player,
  depth: number,
  alpha: number,
  beta: number,
): number {
  const movesForPlayer = getLegalMoves(boardState, player)
  if (movesForPlayer.size === 0) {
    const opponent = otherPlayer(player)
    if (getLegalMoves(boardState, opponent).size === 0) {
      return evaluateBoard(boardState, rootPlayer)
    }
    return searchBestMove(boardState, opponent, rootPlayer, depth, alpha, beta)
  }
  if (depth === 0) return evaluateBoard(boardState, rootPlayer)

  const maximizing = player === rootPlayer
  let bestScore = maximizing ? -Infinity : Infinity
  const orderedMoves = [...movesForPlayer.entries()].sort(([firstKey], [secondKey]) => {
    const [firstRow, firstCol] = firstKey.split('-').map(Number)
    const [secondRow, secondCol] = secondKey.split('-').map(Number)
    const firstValue = positionValues[firstRow!]![firstCol!]!
    const secondValue = positionValues[secondRow!]![secondCol!]!
    return maximizing ? secondValue - firstValue : firstValue - secondValue
  })

  for (const [key, flips] of orderedMoves) {
    const [row, col] = key.split('-').map(Number)
    const nextBoard = applyMove(boardState, { row: row!, col: col! }, player, flips)
    const score = searchBestMove(nextBoard, otherPlayer(player), rootPlayer, depth - 1, alpha, beta)

    if (maximizing) {
      bestScore = Math.max(bestScore, score)
      alpha = Math.max(alpha, bestScore)
    } else {
      bestScore = Math.min(bestScore, score)
      beta = Math.min(beta, bestScore)
    }
    if (beta <= alpha) break
  }

  return bestScore
}

function chooseAiMove(boardState: Board, difficulty: Difficulty): Position {
  const movesForAi = getLegalMoves(boardState, 'white')
  const entries = [...movesForAi.entries()]
  if (entries.length === 0) throw new Error('The AI was asked to move without a legal move.')

  if (difficulty === 'easy') {
    const [key] = entries[Math.floor(Math.random() * entries.length)]!
    const [row, col] = key.split('-').map(Number)
    return { row: row!, col: col! }
  }

  let bestScore = -Infinity
  let bestMoves: Position[] = []
  for (const [key, flips] of entries) {
    const [row, col] = key.split('-').map(Number)
    const nextBoard = applyMove(boardState, { row: row!, col: col! }, 'white', flips)
    const score =
      difficulty === 'medium'
        ? evaluateBoard(nextBoard, 'white')
        : searchBestMove(nextBoard, 'black', 'white', 3, -Infinity, Infinity)

    if (score > bestScore) {
      bestScore = score
      bestMoves = [{ row: row!, col: col! }]
    } else if (score === bestScore) {
      bestMoves.push({ row: row!, col: col! })
    }
  }

  return bestMoves[Math.floor(Math.random() * bestMoves.length)]!
}

const board = ref<Board>(createInitialBoard())
const currentPlayer = ref<Player>('black')
const moves = ref(0)
const notice = ref('')
const history = ref<GameSnapshot[]>([])
const gameMode = ref<GameMode>('two-player')
const difficulty = ref<Difficulty>('medium')
const difficultyName = computed(
  () => difficulty.value[0]!.toUpperCase() + difficulty.value.slice(1),
)
const isAiThinking = ref(false)
let aiTimer: ReturnType<typeof setTimeout> | undefined
const legalMoves = computed(() => getLegalMoves(board.value, currentPlayer.value))
const canHumanMove = computed(
  () => gameMode.value === 'two-player' || currentPlayer.value === 'black',
)
const scores = computed(() => {
  let black = 0
  let white = 0

  for (const row of board.value) {
    for (const cell of row) {
      if (cell === 'black') black += 1
      if (cell === 'white') white += 1
    }
  }

  return { black, white }
})
const gameOver = computed(
  () =>
    getLegalMoves(board.value, 'black').size === 0 &&
    getLegalMoves(board.value, 'white').size === 0,
)
const winner = computed<Player | null>(() => {
  if (!gameOver.value || scores.value.black === scores.value.white) return null
  return scores.value.black > scores.value.white ? 'black' : 'white'
})

function playerName(player: Player): string {
  return player === 'black' ? 'Black' : 'White'
}

function cellLabel(row: number, col: number): string {
  const piece = board.value[row]?.[col]
  const location = `${columns[col]}${row + 1}`
  if (piece) return `${location}, ${playerName(piece)} disc`
  if (canHumanMove.value && legalMoves.value.has(`${row}-${col}`)) return `${location}, legal move`
  return `${location}, empty`
}

function performMove(row: number, col: number) {
  const flips = legalMoves.value.get(`${row}-${col}`)
  if (!flips || gameOver.value) return

  history.value.push({
    board: board.value.map((line) => line.slice()),
    player: currentPlayer.value,
    moves: moves.value,
    notice: notice.value,
  })

  const player = currentPlayer.value
  const nextPlayer = otherPlayer(player)
  const nextBoard = applyMove(board.value, { row, col }, player, flips)
  board.value = nextBoard
  moves.value += 1
  notice.value = ''

  if (getLegalMoves(nextBoard, nextPlayer).size > 0) {
    currentPlayer.value = nextPlayer
  } else if (getLegalMoves(nextBoard, player).size > 0) {
    notice.value = `${playerName(nextPlayer)} has no legal moves. ${playerName(player)} goes again.`
  }

  scheduleAiMove()
}

function playMove(row: number, col: number) {
  if (!canHumanMove.value || isAiThinking.value) return
  performMove(row, col)
}

function cancelAiMove() {
  if (aiTimer !== undefined) clearTimeout(aiTimer)
  aiTimer = undefined
  isAiThinking.value = false
}

function scheduleAiMove() {
  cancelAiMove()
  if (gameMode.value !== 'solo' || currentPlayer.value !== 'white' || gameOver.value) return
  if (legalMoves.value.size === 0) return

  isAiThinking.value = true
  aiTimer = setTimeout(() => {
    aiTimer = undefined
    if (gameMode.value !== 'solo' || currentPlayer.value !== 'white' || gameOver.value) {
      isAiThinking.value = false
      return
    }
    const move = chooseAiMove(board.value, difficulty.value)
    isAiThinking.value = false
    performMove(move.row, move.col)
  }, 350)
}

function handleModeChange() {
  cancelAiMove()
  scheduleAiMove()
}

function undoMove() {
  let previous = history.value.pop()
  if (!previous) return

  cancelAiMove()
  if (gameMode.value === 'solo' && previous.player === 'white') {
    while (history.value.length > 0 && previous.player === 'white') {
      const earlier = history.value.pop()!
      previous = earlier
      if (earlier.player === 'black') break
    }
  }

  board.value = previous.board
  currentPlayer.value = previous.player
  moves.value = previous.moves
  notice.value = previous.notice
}

function newGame() {
  cancelAiMove()
  board.value = createInitialBoard()
  currentPlayer.value = 'black'
  moves.value = 0
  notice.value = ''
  history.value = []
}

onUnmounted(cancelAiMove)
</script>

<template>
  <main class="page-shell">
    <header class="topbar">
      <a class="brand" href="#" aria-label="Othello home">
        <span class="brand-mark" aria-hidden="true"><span></span><span></span></span>
        <span>boardroom<span class="brand-period">.</span></span>
      </a>
      <div class="topbar-note"><span class="online-dot"></span> A little friendly rivalry</div>
      <button class="text-button" type="button" @click="newGame">
        <svg viewBox="0 0 20 20" aria-hidden="true">
          <path d="M16.5 9a6.5 6.5 0 0 0-11.3-3L3.5 8M3.5 8V4.5M3.5 8H7m-3.5 3a6.5 6.5 0 0 0 11.3 3l1.7-2m0 0v3.5m0-3.5H13" />
        </svg>
        New game
      </button>
    </header>

    <section class="game-layout" aria-label="Othello game">
      <div class="game-main">
        <div class="intro">
          <div class="eyebrow"><span class="eyebrow-line"></span> THE CLASSIC STRATEGY GAME</div>
          <h1>Othello<span class="title-period">.</span></h1>
          <p class="intro-copy">A minute to learn. A lifetime to master.</p>
        </div>

        <section class="game-card" aria-label="Game board">
          <div class="game-card-top">
            <div class="turn-indicator" :class="`turn-${currentPlayer}`">
              <span v-if="gameOver" class="turn-disc turn-disc-tie">=</span>
              <span v-else class="turn-disc"></span>
              <div>
                <span class="turn-caption">{{ gameOver ? 'GAME OVER' : 'NOW PLAYING' }}</span>
                <strong>
                  {{ gameOver
                    ? (winner ? `${playerName(winner)} wins` : 'It’s a tie')
                    : isAiThinking
                      ? 'AI is thinking…'
                      : gameMode === 'solo' && currentPlayer === 'white'
                        ? 'AI’s turn'
                        : `${playerName(currentPlayer)}’s turn` }}
                </strong>
              </div>
            </div>
            <div class="move-count"><span>MOVE</span><strong>{{ String(moves).padStart(2, '0') }}</strong></div>
          </div>

          <p class="game-notice" aria-live="polite">
            {{ gameOver
              ? winner
                ? `${playerName(winner)} takes the board, ${scores[winner]} to ${scores[winner === 'black' ? 'white' : 'black']}.`
                : `A perfectly even match — ${scores.black} discs each.`
              : isAiThinking
                ? 'The AI is considering its move.'
                : notice || 'Place a disc to capture your opponent’s pieces.' }}
          </p>

          <div class="board-wrap">
            <div class="row-coordinates" aria-hidden="true">
              <span v-for="row in boardSize" :key="`row-${row}`">{{ row }}</span>
            </div>
            <div class="board-area">
              <div class="board" role="group" aria-label="Othello board, 8 by 8">
                <div
                  v-for="(row, rowIndex) in board"
                  :key="`row-${rowIndex}`"
                  class="board-row"
                >
                  <button
                    v-for="(cell, colIndex) in row"
                    :key="`${rowIndex}-${colIndex}`"
                    class="square"
                    :class="{
                      'square-legal': legalMoves.has(`${rowIndex}-${colIndex}`) && canHumanMove && !gameOver,
                      'square-filled': cell !== null,
                    }"
                    type="button"
                    :disabled="!legalMoves.has(`${rowIndex}-${colIndex}`) || !canHumanMove || gameOver"
                    :aria-label="cellLabel(rowIndex, colIndex)"
                    :aria-disabled="!legalMoves.has(`${rowIndex}-${colIndex}`) || !canHumanMove || gameOver"
                    @click="playMove(rowIndex, colIndex)"
                  >
                    <span v-if="cell" class="disc" :class="`disc-${cell}`"></span>
                    <span v-else-if="legalMoves.has(`${rowIndex}-${colIndex}`) && canHumanMove && !gameOver" class="move-hint"></span>
                  </button>
                </div>
              </div>
              <div class="column-coordinates" aria-hidden="true">
                <span v-for="column in columns" :key="column">{{ column }}</span>
              </div>
            </div>
          </div>

          <div class="game-card-bottom">
            <span><i class="legend-dot"></i> Legal moves</span>
            <button class="undo-button" type="button" :disabled="history.length === 0" @click="undoMove">
              <svg viewBox="0 0 20 20" aria-hidden="true">
                <path d="M8 6 4.5 9.5 8 13M5 9.5h6a4 4 0 0 1 0 8h-1" />
              </svg>
              Undo move
            </button>
          </div>
        </section>
        <p class="fine-print">{{ gameMode === 'solo' ? 'You vs. the AI · Choose your challenge' : 'Two players · One board · No take-backs (well, one)' }}</p>
      </div>

      <aside class="sidebar" aria-label="Game information">
        <section class="mode-card" aria-label="Game settings">
          <span class="eyebrow">MAKE IT YOURS</span>
          <label class="control-label" for="game-mode">Game mode</label>
          <select id="game-mode" v-model="gameMode" @change="handleModeChange">
            <option value="two-player">Two players</option>
            <option value="solo">Solo vs. AI</option>
          </select>
          <template v-if="gameMode === 'solo'">
            <label class="control-label difficulty-label" for="ai-difficulty">AI difficulty</label>
            <select id="ai-difficulty" v-model="difficulty">
              <option value="easy">Easy · Random moves</option>
              <option value="medium">Medium · Positional strategy</option>
              <option value="hard">Hard · Looks ahead</option>
            </select>
          </template>
        </section>
        <section class="score-card">
          <div class="section-heading">
            <div>
              <span class="eyebrow">THE SCORE</span>
              <h2>On the board</h2>
            </div>
            <span class="score-total">{{ scores.black + scores.white }}<span> / 64</span></span>
          </div>
          <div class="score-row" :class="{ 'score-active': currentPlayer === 'black' && !gameOver }">
            <div class="player-ident">
              <span class="score-disc score-disc-black"></span>
              <div><strong>Black</strong><span>{{ currentPlayer === 'black' && !gameOver ? 'Playing now' : gameMode === 'solo' ? 'You' : 'Player one' }}</span></div>
            </div>
            <strong class="score-number">{{ scores.black }}</strong>
          </div>
          <div class="score-row" :class="{ 'score-active': currentPlayer === 'white' && !gameOver }">
            <div class="player-ident">
              <span class="score-disc score-disc-white"></span>
              <div><strong>White</strong><span>{{ currentPlayer === 'white' && !gameOver ? (isAiThinking ? 'AI thinking…' : 'Playing now') : gameMode === 'solo' ? `AI · ${difficultyName}` : 'Player two' }}</span></div>
            </div>
            <strong class="score-number">{{ scores.white }}</strong>
          </div>
          <div class="score-meter" aria-label="Score distribution">
            <span class="meter-black" :style="{ width: `${(scores.black / (scores.black + scores.white)) * 100}%` }"></span>
            <span class="meter-white"></span>
          </div>
          <div class="meter-labels"><span>BLACK</span><span>WHITE</span></div>
        </section>

        <section class="how-card">
          <div class="how-heading">
            <span class="how-icon" aria-hidden="true">
              <svg viewBox="0 0 24 24"><path d="M12 20.2a8.2 8.2 0 1 0 0-16.4 8.2 8.2 0 0 0 0 16.4Zm0-11.8v4.2m0 3h.01" /></svg>
            </span>
            <h2>How to play</h2>
          </div>
          <p>Surround your opponent’s discs in a straight line to flip them to your color.</p>
          <ol class="steps">
            <li><span>01</span> Black always moves first.</li>
            <li><span>02</span> Every move must capture at least one disc.</li>
            <li><span>03</span> Most discs wins when no moves remain.</li>
          </ol>
          <div class="tip"><span aria-hidden="true">✳</span> Corners are powerful. Keep an eye on them.</div>
        </section>
      </aside>
    </section>

    <footer class="footer">
      <span>GOOD GAMES ARE BETTER TOGETHER.</span>
      <span>BUILT FOR YOUR NEXT REMATCH <i>↗</i></span>
    </footer>
  </main>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&family=Manrope:wght@400;500;600;700;800&display=swap');

:root {
  font-family: 'DM Sans', sans-serif;
  color: #252a24;
  background: #f5f5ef;
  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  font-optical-sizing: auto;
  font-weight: 400;
}

* { box-sizing: border-box; }
body { min-width: 320px; min-height: 100vh; margin: 0; }
button { font: inherit; }
button:focus-visible, a:focus-visible { outline: 3px solid #e1a940; outline-offset: 3px; }

.page-shell { width: min(1120px, calc(100% - 64px)); margin: 0 auto; }
.topbar { height: 78px; display: flex; align-items: center; justify-content: space-between; border-bottom: 1px solid #e7e8df; }
.brand { display: inline-flex; align-items: center; gap: 10px; color: #26382b; text-decoration: none; font: 800 18px 'Manrope', sans-serif; letter-spacing: -0.8px; }
.brand-period, .title-period { color: #dd9f3d; }
.brand-mark { width: 27px; height: 27px; border-radius: 8px; background: #254d36; display: flex; align-items: center; justify-content: center; gap: 2px; transform: rotate(-5deg); }
.brand-mark span { width: 7px; height: 7px; border-radius: 50%; background: #fff; }
.brand-mark span:last-child { background: #9bc285; }
.topbar-note { color: #777e72; font-size: 12px; display: flex; align-items: center; gap: 8px; }
.online-dot { width: 7px; height: 7px; border-radius: 50%; background: #80a66a; box-shadow: 0 0 0 3px #e8eee3; }
.text-button { display: flex; align-items: center; gap: 8px; background: transparent; border: 1px solid #dfe2d7; border-radius: 7px; color: #3e4b3c; padding: 9px 12px; cursor: pointer; font-size: 12px; font-weight: 600; transition: border-color .2s, background .2s; }
.text-button:hover { background: #fff; border-color: #aab8a4; }
.text-button svg, .undo-button svg { width: 16px; height: 16px; fill: none; stroke: currentColor; stroke-width: 1.5; stroke-linecap: round; stroke-linejoin: round; }
.game-layout { display: grid; grid-template-columns: minmax(0, 650px) minmax(270px, 1fr); gap: 48px; align-items: start; padding: 38px 0 48px; }
.intro { margin: 2px 0 22px; }
.eyebrow { color: #898e7f; font-size: 9px; font-weight: 700; letter-spacing: 1.45px; }
.intro .eyebrow { display: flex; align-items: center; gap: 9px; }
.eyebrow-line { width: 19px; height: 1px; background: #d7a349; }
h1, h2, p { margin: 0; }
h1 { margin-top: 8px; font: 800 45px/.98 'Manrope', sans-serif; letter-spacing: -2.8px; color: #26382b; }
.intro-copy { color: #7a8075; font-size: 13px; margin-top: 8px; }
.game-card { background: #fff; border: 1px solid #e8e9e1; border-radius: 12px; padding: 20px 22px 0; box-shadow: 0 14px 32px #31402b0a; }
.game-card-top { display: flex; align-items: center; justify-content: space-between; }
.turn-indicator { display: flex; align-items: center; gap: 11px; }
.turn-disc, .score-disc { display: block; flex: 0 0 auto; border-radius: 50%; }
.turn-disc { width: 29px; height: 29px; background: radial-gradient(circle at 35% 28%, #657066, #18201b 70%); box-shadow: 0 2px 4px #15221628; }
.turn-white .turn-disc { background: radial-gradient(circle at 35% 28%, #fff, #d9dcd2 78%); border: 1px solid #d1d5ca; }
.turn-disc-tie { display: grid; place-items: center; background: #edf0e8 !important; color: #527448; box-shadow: none; border: 0 !important; font-size: 16px; font-weight: 700; }
.turn-caption { display: block; color: #969b8e; font-size: 8px; font-weight: 700; letter-spacing: 1.2px; margin-bottom: 3px; }
.turn-indicator strong { font: 700 14px 'Manrope', sans-serif; color: #2d392d; }
.move-count { padding-left: 17px; border-left: 1px solid #e8e9e1; text-align: right; }
.move-count span { display: block; color: #a1a599; font-size: 8px; font-weight: 700; letter-spacing: 1px; margin-bottom: 1px; }
.move-count strong { color: #475445; font: 700 15px 'Manrope', sans-serif; font-variant-numeric: tabular-nums; }
.game-notice { min-height: 17px; margin: 12px 0 14px; color: #878d80; font-size: 11px; }
.board-wrap { width: min(100%, 544px); margin: 0 auto; display: grid; grid-template-columns: 17px minmax(0, 1fr); gap: 6px; }
.row-coordinates { display: grid; grid-template-rows: repeat(8, 1fr); align-items: center; justify-items: center; color: #a8ad9f; font-size: 9px; font-weight: 600; }
.board-area { min-width: 0; }
.board { display: grid; grid-template-rows: repeat(8, minmax(0, 1fr)); aspect-ratio: 1; overflow: hidden; border: 5px solid #214e37; border-radius: 6px; background: #286144; box-shadow: 0 5px 12px #193a2d1c; }
.board-row { display: grid; grid-template-columns: repeat(8, minmax(0, 1fr)); min-height: 0; }
.square { position: relative; display: grid; place-items: center; min-width: 0; min-height: 0; padding: 0; border: 0; border-right: 1px solid #ffffff24; border-bottom: 1px solid #ffffff24; background: #34754c; cursor: default; }
.board-row:nth-child(odd) .square:nth-child(odd), .board-row:nth-child(even) .square:nth-child(even) { background: #306e47; }
.square:disabled { opacity: 1; }
.square-legal { cursor: pointer; }
.square-legal:hover { background: #43865a !important; }
.square-legal:focus-visible { z-index: 2; outline: 3px solid #f1c66e; outline-offset: -3px; }
.disc { width: 76%; height: 76%; border-radius: 50%; animation: place-disc .2s ease-out both; }
.disc-black { background: radial-gradient(circle at 34% 27%, #636a61 0, #333a33 35%, #171d18 78%); box-shadow: 0 2px 3px #11281873, inset 0 1px 1px #ffffff45; }
.disc-white { background: radial-gradient(circle at 34% 25%, #fff 0, #f6f5ed 54%, #d9ddce 100%); box-shadow: 0 2px 3px #11281840, inset 0 1px 1px #ffffff; }
.move-hint { width: 17%; height: 17%; min-width: 5px; min-height: 5px; border-radius: 50%; background: #d6e5ce; opacity: .82; transition: transform .16s, opacity .16s; }
.square-legal:hover .move-hint { transform: scale(1.55); opacity: 1; }
.column-coordinates { display: grid; grid-template-columns: repeat(8, 1fr); text-align: center; padding-top: 7px; color: #a8ad9f; font-size: 9px; font-weight: 600; }
.game-card-bottom { display: flex; align-items: center; justify-content: space-between; height: 49px; margin-top: 11px; border-top: 1px solid #eff0eb; color: #858b7d; font-size: 10px; }
.game-card-bottom > span { display: flex; align-items: center; gap: 7px; }
.legend-dot { width: 7px; height: 7px; border-radius: 50%; background: #8eaa7c; }
.undo-button { display: flex; align-items: center; gap: 6px; padding: 5px 0 5px 8px; color: #65705f; border: 0; background: transparent; cursor: pointer; font-size: 10px; font-weight: 600; }
.undo-button:disabled { color: #bdc1b7; cursor: default; }
.undo-button:not(:disabled):hover { color: #315a3c; }
.undo-button svg { width: 15px; height: 15px; }
.fine-print { margin-top: 13px; color: #a2a698; font-size: 10px; text-align: center; letter-spacing: .2px; }
.sidebar { display: grid; gap: 16px; padding-top: 98px; }
.mode-card, .score-card, .how-card { background: #fff; border: 1px solid #e8e9e1; border-radius: 10px; padding: 20px; }
.mode-card { display: grid; gap: 8px; }
.control-label { color: #525d50; font-size: 10px; font-weight: 700; }
.difficulty-label { margin-top: 5px; }
.mode-card select { width: 100%; min-height: 36px; padding: 0 10px; border: 1px solid #dfe2d7; border-radius: 6px; background: #fff; color: #3e4b3c; font: inherit; font-size: 11px; }
.mode-card select:focus-visible { outline: 3px solid #e1a940; outline-offset: 2px; }
.section-heading { display: flex; align-items: center; justify-content: space-between; margin-bottom: 15px; }
.section-heading h2, .how-heading h2 { margin-top: 4px; color: #2c392d; font: 700 16px 'Manrope', sans-serif; letter-spacing: -.45px; }
.score-total { color: #394638; font: 700 17px 'Manrope', sans-serif; }
.score-total span { color: #a4a99c; font-size: 10px; font-weight: 500; }
.score-row { display: flex; align-items: center; justify-content: space-between; min-height: 53px; padding: 0 10px; border: 1px solid transparent; border-radius: 7px; transition: background .2s, border-color .2s; }
.score-row + .score-row { margin-top: 5px; }
.score-active { background: #f4f7f0; border-color: #e7eddf; }
.player-ident { display: flex; align-items: center; gap: 10px; }
.score-disc { width: 24px; height: 24px; }
.score-disc-black { background: radial-gradient(circle at 34% 27%, #636a61, #171d18 78%); box-shadow: inset 0 1px 1px #ffffff45; }
.score-disc-white { background: radial-gradient(circle at 34% 25%, #fff, #d9ddce 100%); border: 1px solid #d9ddd2; }
.player-ident strong { display: block; color: #3a4438; font: 700 12px 'Manrope', sans-serif; }
.player-ident div > span { display: block; margin-top: 2px; color: #9ca192; font-size: 9px; }
.score-number { color: #344232; font: 700 20px 'Manrope', sans-serif; font-variant-numeric: tabular-nums; }
.score-meter { display: flex; overflow: hidden; height: 6px; margin-top: 18px; border-radius: 8px; background: #eef0e9; }
.meter-black { min-width: 0; border-radius: 8px; background: #354536; transition: width .3s; }
.meter-white { flex: 1; background: #e3e5dd; }
.meter-labels { display: flex; justify-content: space-between; margin-top: 7px; color: #a1a596; font-size: 8px; font-weight: 700; letter-spacing: .8px; }
.how-card { padding-bottom: 16px; }
.how-heading { display: flex; align-items: center; gap: 9px; }
.how-icon { display: grid; place-items: center; width: 27px; height: 27px; border-radius: 7px; background: #edf2e9; color: #57764b; }
.how-icon svg { width: 17px; height: 17px; fill: none; stroke: currentColor; stroke-width: 1.4; stroke-linecap: round; }
.how-heading h2 { margin: 0; }
.how-card > p { margin-top: 12px; color: #777f72; font-size: 11px; line-height: 1.65; }
.steps { display: grid; gap: 11px; list-style: none; padding: 15px 0 0; margin: 14px 0 0; border-top: 1px solid #eff0eb; }
.steps li { display: flex; align-items: center; gap: 9px; color: #525d50; font-size: 10px; }
.steps li span { color: #a3ab98; font: 700 9px 'Manrope', sans-serif; }
.tip { display: flex; gap: 7px; margin-top: 16px; padding: 10px; border-radius: 6px; background: #f7f5ed; color: #7d745b; font-size: 9px; line-height: 1.4; }
.tip span { color: #d1a03f; font-size: 12px; line-height: 10px; }
.footer { display: flex; justify-content: space-between; border-top: 1px solid #e7e8df; padding: 17px 0 20px; color: #a2a698; font-size: 8px; font-weight: 700; letter-spacing: 1px; }
.footer i { color: #c49439; font-size: 12px; font-style: normal; margin-left: 3px; }
@keyframes place-disc { from { transform: scale(.55); opacity: .5; } to { transform: scale(1); opacity: 1; } }

@media (max-width: 900px) {
  .page-shell { width: min(700px, calc(100% - 40px)); }
  .game-layout { grid-template-columns: minmax(0, 1fr); gap: 25px; padding-top: 30px; }
  .sidebar { grid-template-columns: repeat(2, minmax(0, 1fr)); padding-top: 0; }
  .board-wrap { width: min(100%, 570px); }
}

@media (max-width: 560px) {
  .page-shell { width: calc(100% - 28px); }
  .topbar { height: 64px; }
  .topbar-note { display: none; }
  .game-layout { padding-top: 25px; gap: 20px; }
  h1 { font-size: 39px; }
  .intro { margin-bottom: 18px; }
  .game-card { padding: 15px 12px 0; border-radius: 9px; }
  .game-notice { font-size: 10px; margin: 11px 0 12px; }
  .board-wrap { grid-template-columns: 13px minmax(0, 1fr); gap: 4px; }
  .board { border-width: 4px; border-radius: 4px; }
  .row-coordinates, .column-coordinates { font-size: 8px; }
  .game-card-bottom { margin-top: 7px; }
  .sidebar { grid-template-columns: 1fr; gap: 12px; }
  .score-card, .how-card { padding: 17px; }
  .footer { gap: 14px; font-size: 7px; letter-spacing: .65px; }
}

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { scroll-behavior: auto !important; animation-duration: .01ms !important; animation-iteration-count: 1 !important; transition-duration: .01ms !important; }
}
</style>
