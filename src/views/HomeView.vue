<template>
  <main style="display: flex; flex-direction: row">
    <!-- Step 历史列表 -->
    <aside
      style="
        width: 120px;
        border-right: 1px solid #eee;
        padding: 8px 0;
        max-height: 315px;
        overflow-y: auto;
      "
    >
      <div
        v-for="(step, idx) in moveHistoryList"
        :key="idx"
        :style="{
          padding: '6px 12px',
          cursor: 'pointer',
          background: idx === moveIndex ? '#e3f2fd' : 'transparent',
          fontWeight: idx === moveIndex ? 'bold' : 'normal',
        }"
        @click="jumpToStep(idx)"
      >
        Step {{ idx }}
      </div>
    </aside>
    <!-- 主体 -->
    <div style="flex: 1; padding-left: 24px">
      <div v-if="loading">加载中...</div>
      <div v-else>
        <table class="sudoku-board">
          <tbody>
            <tr v-for="(row, rowIndex) in userBoard" :key="rowIndex">
              <td
                v-for="(cell, colIndex) in row"
                :key="colIndex"
                :class="{
                  'highlight-checking': highlightCells[rowIndex]?.[colIndex] > 0,
                  error: errorCells[rowIndex][colIndex],
                  'highlight-active':
                    activeCell && isInSameGroup(rowIndex, colIndex, activeCell.row, activeCell.col),
                  'highlight-active-center':
                    activeCell && rowIndex === activeCell.row && colIndex === activeCell.col,
                }"
                :style="{
                  '--highlight-level': highlightCells[rowIndex]?.[colIndex] * 0.3 + 0.1,
                }"
              >
                <template v-if="originBoard[rowIndex][colIndex] === 0">
                  <template
                    v-if="
                      possibleValues[rowIndex]?.[colIndex]?.size === 1 &&
                      !userBoard[rowIndex][colIndex]
                    "
                  >
                    <button class="unique-btn" @click="onFillUnique(rowIndex, colIndex)">
                      {{ Array.from(possibleValues[rowIndex][colIndex])[0] }}
                    </button>
                  </template>

                  <template
                    v-else-if="
                      possibleValues[rowIndex]?.[colIndex]?.size === 0 &&
                      !userBoard[rowIndex][colIndex]
                    "
                  >
                    <span>🚫</span>
                  </template>
                  <template v-else>
                    <div class="possible-values" v-if="!userInput[rowIndex][colIndex]">
                      <span
                        v-for="v in Array.from(possibleValues[rowIndex]?.[colIndex] ?? [])"
                        :key="v"
                        class="pv"
                        >{{ v }}</span
                      >
                    </div>
                    <input
                      type="text"
                      maxlength="1"
                      v-model="userInput[rowIndex][colIndex]"
                      @input="onInput(rowIndex, colIndex)"
                      @focus="onFocus(rowIndex, colIndex)"
                      @blur="onBlur"
                      :readonly="userInput[rowIndex][colIndex] !== ''"
                      :disabled="userInput[rowIndex][colIndex] !== ''"
                      :class="{ error: errorCells[rowIndex][colIndex] }"
                    />
                  </template>
                </template>
                <span v-else>{{ originBoard[rowIndex][colIndex] }}</span>
              </td>
            </tr>
          </tbody>
        </table>
        <div style="margin: 16px 0; display: flex; gap: 8px">
          <div style="display: flex; align-items: center; gap: 4px">
            <input type="checkbox" id="combo" v-model="combo" />
            <label for="combo">Combo</label>
          </div>
          <button v-if="hasError" @click="restoreMoveSnapshot">Reset</button>
          <button v-if="waitingForNext" @click="onNext">Next</button>
        </div>
      </div>
    </div>
  </main>
</template>

<script setup lang="ts">
import { ref, onMounted, computed, watch } from 'vue'

const loading = ref(true)
const originBoard = ref<number[][]>([]) // 初始题面
const userBoard = ref<number[][]>([]) // 用户填写
const userInput = ref<string[][]>([]) // 用于v-model绑定input
const errorCells = ref<boolean[][]>([]) // 标记错误格子
const possibleValues = ref<Set<number>[][]>([]) // 可能的值
const highlightCells = ref<number[][]>([]) // 由 HIGHLIGHT[][] 改为 number[][]，表示高亮层数
const activeCell = ref<{ row: number; col: number } | null>(null)
const waitingForNext = ref(false)
const nextResolvers: (() => void)[] = []
const combo = ref(true)

function updatePossibleValues() {
  const newPossible: Set<number>[][] = Array.from({ length: 9 }, (_, row) =>
    Array.from({ length: 9 }, (_, col) => {
      if (userBoard.value[row][col] !== 0) {
        // 已有数字，不需要可能值
        return new Set<number>()
      }
      // 1-9都可能，逐步排除
      const used = new Set<number>()
      // 行
      for (let c = 0; c < 9; c++) {
        if (userBoard.value[row][c] !== 0) used.add(userBoard.value[row][c])
      }
      // 列
      for (let r = 0; r < 9; r++) {
        if (userBoard.value[r][col] !== 0) used.add(userBoard.value[r][col])
      }
      // 宫
      const startRow = Math.floor(row / 3) * 3
      const startCol = Math.floor(col / 3) * 3
      for (let r = startRow; r < startRow + 3; r++) {
        for (let c = startCol; c < startCol + 3; c++) {
          if (userBoard.value[r][c] !== 0) used.add(userBoard.value[r][c])
        }
      }
      // 1-9中未被used的就是可能值
      return new Set(Array.from({ length: 9 }, (_, i) => i + 1).filter((n) => !used.has(n)))
    }),
  )
  possibleValues.value = newPossible
}

const isInSameGroup = (r1: number, c1: number, r2: number, c2: number) => {
  return (
    r1 === r2 ||
    c1 === c2 ||
    (Math.floor(r1 / 3) === Math.floor(r2 / 3) && Math.floor(c1 / 3) === Math.floor(c2 / 3))
  )
}

function check(row: number, col: number) {
  console.log('check', row, col)
  const val = userBoard.value[row][col]
  if (val === 0) {
    errorCells.value[row][col] = false
    return
  }
  // 检查行、列、宫
  let valid = true
  // 行
  for (let c = 0; c < 9; c++) {
    if (c !== col && userBoard.value[row][c] === val) valid = false
  }
  // 列
  for (let r = 0; r < 9; r++) {
    if (r !== row && userBoard.value[r][col] === val) valid = false
  }
  // 宫
  const startRow = Math.floor(row / 3) * 3
  const startCol = Math.floor(col / 3) * 3
  for (let r = startRow; r < startRow + 3; r++) {
    for (let c = startCol; c < startCol + 3; c++) {
      if ((r !== row || c !== col) && userBoard.value[r][c] === val) valid = false
    }
  }
  errorCells.value[row][col] = !valid
  updatePossibleValues()
}

const resetError = () => {
  errorCells.value = Array.from({ length: 9 }, () => Array(9).fill(false))
}

function waitForNext() {
  waitingForNext.value = true
  return new Promise<void>((resolve) => {
    nextResolvers.push(() => {
      resolve()
      // 只有队列空了才隐藏按钮
      if (nextResolvers.length === 0) {
        waitingForNext.value = false
      }
    })
  })
}

function onNext() {
  const resolver = nextResolvers.shift()
  if (resolver) {
    resolver()
  }
}

const checkTask = ref<string[]>([])
async function asyncCheck(centerRow: number, centerCol: number, val: number) {
  // 检查给 [centerRow, centerCol] 的格子赋值 val 是否冲突
  console.log('asyncCheck', centerRow, centerCol, val)
  const key = `${centerRow}-${centerCol}-${val}`
  if (checkTask.value.includes(key)) {
    return
  }
  checkTask.value.push(key)

  const visited = Array.from({ length: 9 }, () => Array(9).fill(false))
  const thisCheckCells: [number, number][] = [] // 记录本次检查过的格子

  function getCellsAtDistance(n: number) {
    const cells: [number, number][] = []
    for (let r = 0; r < 9; r++) {
      for (let c = 0; c < 9; c++) {
        if (Math.abs(r - centerRow) + Math.abs(c - centerCol) !== n || visited[r][c]) {
          continue
        }
        if (!isInSameGroup(r, c, centerRow, centerCol)) {
          continue
        }
        cells.push([r, c])
      }
    }
    return cells
  }

  const checkCell = async (r: number, c: number) => {
    visited[r][c] = true
    highlightCells.value[r][c] = (highlightCells.value[r][c] ?? 0) + 1 // 高亮层数+1
    setTimeout(() => {
      highlightCells.value[r][c] = Math.max(0, (highlightCells.value[r][c] ?? 1) - 1)
    }, 1000)

    thisCheckCells.push([r, c])
    if (originBoard.value[r][c] === 0) {
      // 计算possibleValues
      const used = new Set<number>()
      for (let cc = 0; cc < 9; cc++) {
        if (userBoard.value[r][cc] !== 0) used.add(userBoard.value[r][cc])
      }
      for (let rr = 0; rr < 9; rr++) {
        if (userBoard.value[rr][c] !== 0) used.add(userBoard.value[rr][c])
      }
      const startRow = Math.floor(r / 3) * 3
      const startCol = Math.floor(c / 3) * 3
      for (let rr = startRow; rr < startRow + 3; rr++) {
        for (let cc = startCol; cc < startCol + 3; cc++) {
          if (userBoard.value[rr][cc] !== 0) used.add(userBoard.value[rr][cc])
        }
      }
      possibleValues.value[r][c] = new Set(
        Array.from({ length: 9 }, (_, i) => i + 1).filter((num) => !used.has(num)),
      )
    }
    // 检查这个格子是否冲突
    const pointerVal = userBoard.value[r][c] || originBoard.value[r][c]
    if ((r !== centerRow || c !== centerCol) && pointerVal === val) {
      // 冲突
      console.error('冲突', r, c, pointerVal, val)
      errorCells.value[centerRow][centerCol] = true
      errorCells.value[r][c] = true
      return
    }
    if (!userBoard.value[r][c]) {
      if (possibleValues.value[r][c].size === 0) {
        // 无解
        console.error('无解', r, c)
        errorCells.value[centerRow][centerCol] = true
        errorCells.value[r][c] = true
        return
      }
      if (possibleValues.value[r][c].size === 1) {
        // 唯一解
        const onlyVal = Array.from(possibleValues.value[r][c])[0]
        console.log('唯一解', r, c, onlyVal)

        if (combo.value) {
          userInput.value[r][c] = String(onlyVal)
          userBoard.value[r][c] = onlyVal
          asyncCheck(r, c, onlyVal)
        }
      }
    }
  }

  for (let n = 0; n < 9; n++) {
    const cells = getCellsAtDistance(n)
    if (cells.length === 0 && n > 8) return
    for (const [r, c] of cells) {
      checkCell(r, c)
    }
    await new Promise((resolve) => setTimeout(resolve, 150))
    if (hasError.value) {
      break
    }
    // await waitForNext()
  }
  // 检查结束后，把本次高亮的格子层数-1
  // for (const [r, c] of thisCheckCells) {
  //   highlightCells.value[r][c] = Math.max(0, (highlightCells.value[r][c] ?? 1) - 1)
  // }

  checkTask.value = checkTask.value.filter((k) => k !== key)
}

watch(checkTask, (newVal) => {
  if (newVal.length === 0 && !hasError.value) {
    pushMoveSnapshot()
  }
})

function restoreMoveSnapshot() {
  if (moveIndex.value < 0) return
  restoreMove(currentMove.value!)
}
const hasError = computed(() => {
  return errorCells.value.some((row) => row.some((e) => e))
})
const hasUniqueButton = computed(() => {
  for (let r = 0; r < 9; r++) {
    for (let c = 0; c < 9; c++) {
      if (
        originBoard.value[r][c] === 0 &&
        !userBoard.value[r][c] &&
        possibleValues.value[r][c]?.size === 1
      ) {
        return true
      }
    }
  }
  return false
})
function onInput(row: number, col: number) {
  console.log('onInput', row, col, userInput.value[row][col])
  const val = userInput.value[row][col]
  if (!/^[1-9]$/.test(val)) {
    userInput.value[row][col] = ''
    userBoard.value[row][col] = 0
    resetError()
    check(row, col)
    updatePossibleValues()
    return
  }
  activeCell.value = null
  userBoard.value[row][col] = parseInt(val)
  asyncCheck(row, col, parseInt(val))
}

function onFocus(row: number, col: number) {
  activeCell.value = { row, col }
}

function onBlur() {
  activeCell.value = null
}

function onFillUnique(row: number, col: number) {
  if (hasError.value) {
    return
  }
  // 取唯一解
  const val = Array.from(possibleValues.value[row][col])[0]
  userInput.value[row][col] = String(val)
  userBoard.value[row][col] = val
  asyncCheck(row, col, val)
}

type MoveNode = {
  snapshot: {
    userBoard: number[][]
    userInput: string[][]
    errorCells: boolean[][]
  }
  prev: MoveNode | null
  next: MoveNode | null
}
const moveHead = ref<MoveNode | null>(null)
const currentMove = ref<MoveNode | null>(null)
const moveHistoryList = computed(() => {
  const list: MoveNode[] = []
  let node = moveHead.value
  while (node) {
    list.push(node)
    node = node.next
  }
  return list
})
const moveIndex = computed(() => {
  let idx = 0
  let node = moveHead.value
  while (node && node !== currentMove.value) {
    node = node.next
    idx++
  }
  return node ? idx : -1
})
function pushMoveSnapshot() {
  const snapshot = {
    userBoard: userBoard.value.map((row) => [...row]),
    userInput: userInput.value.map((row) => [...row]),
    errorCells: errorCells.value.map((row) => [...row]),
  }
  const newNode: MoveNode = {
    snapshot,
    prev: currentMove.value,
    next: null,
  }
  if (currentMove.value) {
    currentMove.value.next = newNode
  } else {
    moveHead.value = newNode
  }
  currentMove.value = newNode
}

function restoreMove(node: MoveNode) {
  userBoard.value = node.snapshot.userBoard.map((row) => [...row])
  userInput.value = node.snapshot.userInput.map((row) => [...row])
  errorCells.value = node.snapshot.errorCells.map((row) => [...row])
  updatePossibleValues()
  currentMove.value = node
}

function jumpToStep(idx: number) {
  let node = moveHead.value
  let i = 0
  while (node && i < idx) {
    node = node.next
    i++
  }
  if (node) restoreMove(node)
}

function undo() {
  if (currentMove.value && currentMove.value.prev) {
    restoreMove(currentMove.value.prev)
  }
}

function redo() {
  if (currentMove.value && currentMove.value.next) {
    restoreMove(currentMove.value.next)
  }
}

function initBoards(puzzle: number[][]) {
  originBoard.value = puzzle.map((row) => [...row])
  userBoard.value = puzzle.map((row) => row.map((cell) => cell))
  userInput.value = puzzle.map((row) => row.map((cell) => (cell === 0 ? '' : String(cell))))
  errorCells.value = puzzle.map((row) => row.map(() => false))
  highlightCells.value = Array.from({ length: 9 }, () => Array(9).fill(0))
  updatePossibleValues()
  // 重置链表头和当前指针
  moveHead.value = null
  currentMove.value = null
  pushMoveSnapshot()
}

onMounted(async () => {
  loading.value = true
  // 拉取题面
  // const res = await fetch('https://sudoku.com/api/v2/level/easy')
  // const data = await res.json()
  const easy = {
    mission: '030490010740018000196700024000501762003027059000040300078900000429000073000370098',
    solution: '832496517745218936196753824984531762613827459257649381378962145429185673561374298',
  }
  const extereme = {
    mission: '090002070000003006001960080400000010000006000030750008000200000070890005003000700',
    solution: '396582174548173296721964583465328917987416352132759468659237841274891635813645729',
  }
  const board = extereme.mission
    .split('')
    .map((item) => parseInt(item))
    .reduce((acc, item, index) => {
      const row = Math.floor(index / 9)
      const col = index % 9
      if (!acc[row]) {
        acc[row] = []
      }
      acc[row][col] = item
      return acc
    }, [] as number[][])
  // 题面在 data.board
  // 题面是9x9的数组，0表示空
  initBoards(board)
  loading.value = false
})
</script>

<style lang="less" scoped>
.sudoku-board {
  border-collapse: collapse;
  max-width: 434px;
  min-width: 434px;
  font-size: 20px;
}
.sudoku-board td {
  border: 1px solid #aaa;
  width: 48px;
  height: 48px;
  text-align: center;
  padding: 0;
  position: relative;
  z-index: 0;
}
.sudoku-board input {
  width: 100%;
  height: 100%;
  text-align: center;
  border: none;
  outline: none;
  font-size: 18px;
  background: transparent;
  color: #1976d2;
  font-weight: bold;
  font-size: 24px;
  font-family: 'Courier New', Courier, monospace;
  caret-color: transparent;
}
.sudoku-board .error {
  background: #ffcccc !important;
  color: #f00 !important;
}
.sudoku-board span {
  font-weight: bold;
  color: #333;
}
.possible-values {
  font-size: 11px;
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  align-items: center;
  min-height: 14px;
  margin-top: 6px;
  padding: 0 4px;
  line-height: 1.4;
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  z-index: -1;

  span {
    color: #ccc;
  }
}
.pv {
  margin: 0 1px;
}
.sudoku-board td.highlight-active {
  background: #f0f0f0;
}
.sudoku-board td.highlight-active-center {
  &:after {
    content: '';
    position: absolute;
    --gap: 4px;
    top: var(--gap);
    left: var(--gap);
    bottom: var(--gap);
    right: var(--gap);
    border: 1px solid #ccc;
    z-index: -2;
    background: white;
  }
}
.sudoku-board td {
  position: relative;
  &:before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: #90caf9;
    opacity: 0;
    z-index: -1;
    transition: opacity 0.5s ease-in-out;
  }
}
.sudoku-board td.highlight-checking {
  &:before {
    opacity: var(--highlight-level);
  }
}
// 九宫格粗边框
.sudoku-board tr:nth-child(1) td,
.sudoku-board tr:nth-child(4) td,
.sudoku-board tr:nth-child(7) td {
  border-top: 2px solid #333;
}
.sudoku-board td:nth-child(1),
.sudoku-board td:nth-child(4),
.sudoku-board td:nth-child(7) {
  border-left: 2px solid #333;
}
.sudoku-board tr td:last-child {
  border-right: 2px solid #333;
}
.sudoku-board tr:last-child td {
  border-bottom: 2px solid #333;
}
.unique-btn {
  background: #e3f2fd;
  border: 1px solid #90caf9;
  color: #1976d2;
  cursor: pointer;
  padding: 0;
  margin: 0;
  font-weight: bold;
  border-radius: 4px;
  transition: background 0.2s;
  width: 70%;
  height: 70%;
  font-size: 18px;
  &:hover {
    background: #bbdefb;
  }
}
</style>
