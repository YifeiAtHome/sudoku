<template>
  <div style="margin-bottom: 16px">
    <label for="difficulty">难度：</label>
    <select id="difficulty" v-model="selectedDifficulty" @change="onDifficultyChange">
      <option :value="Difficulty.Easy">Easy</option>
      <option :value="Difficulty.Medium">Medium</option>
      <option :value="Difficulty.Hard">Hard</option>
      <option :value="Difficulty.Extreme">Extreme</option>
    </select>
  </div>
  <main style="display: flex; flex-direction: row">
    <!-- Step 历史列表 -->
    <aside
      style="
        width: 140px;
        border-right: 1px solid #eee;
        padding: 8px;
        max-height: 500px;
        overflow-y: auto;
      "
    >
      <div v-for="(node, idx) in mainChain" :key="idx" style="margin-bottom: 8px">
        <div
          :style="{
            textAlign: 'center',
            fontWeight: node === currentMove ? 'bold' : 'normal',
            color: node === currentMove ? '#1976d2' : '#333',
            background: node === currentMove ? '#e3f2fd' : 'transparent',
            borderRadius: '4px',
            padding: '2px 8px',
            marginBottom: '2px',
            cursor: 'pointer',
            border: node === currentMove ? '1px solid #1976d2' : '1px solid transparent',
            transition: 'background 0.2s, border 0.2s',
          }"
          @click="restoreMove(node)"
        >
          <span style="font-size: 12px; color: #aaa">
            <template v-if="node.parent">
              <span
                v-if="node.children.length > 1 && node?.children?.[0]?.branchInfo"
                :class="{ failed: node.status === 'failed', success: node.status === 'success' }"
              >
                Branch at [{{ node.children[0].branchInfo.row + 1 }},{{
                  node.children[0].branchInfo.col + 1
                }}]
              </span>
              <span v-else-if="node.children.length < 1 && node.status === 'success'">🎉🎉🎉</span>
              <span
                v-else
                :class="{ failed: node.status === 'failed', success: node.status === 'success' }"
              >
                Step
              </span>
            </template>
            <span :class="{ success: node.status === 'success' }" v-else>Start</span>
          </span>
          <!-- 横向分支 -->
          <div v-if="node.children.length > 1" style="display: flex; justify-content: center">
            <div style="display: flex; flex-direction: row; gap: 2px; margin-left: 4px">
              <div
                v-for="(child, cidx) in node.children"
                :key="cidx"
                :class="{
                  failed: child.status === 'failed',
                  success: child.status === 'success',
                }"
                :style="{
                  fontSize: '12px',
                  color:
                    child.status === 'failed'
                      ? '#fa5151'
                      : child.status === 'success'
                        ? '#07c160'
                        : mainChainSet.has(child)
                          ? '#1976d2'
                          : '#888',
                  fontWeight: mainChainSet.has(child) ? 'bold' : 'normal',
                  background: mainChainSet.has(child) ? '#e3f2fd' : 'transparent',
                  borderRadius: '4px',
                  padding: '0 4px',
                  border: mainChainSet.has(child) ? '1px solid #1976d2' : '1px solid transparent',
                  cursor: 'pointer',
                  transition: 'background 0.2s, border 0.2s',
                }"
                @click.stop="restoreMove(child, true)"
              >
                {{ child.branchInfo ? child.branchInfo.value : '?' }}
              </div>
            </div>
          </div>
        </div>
        <!-- 树形竖线（非最后一个节点） -->
        <div v-if="idx < mainChain.length - 1" style="display: flex; justify-content: center">
          <i
            style="border-left: 2px solid #eee; height: 12px; margin: auto; display: inline-block"
          ></i>
        </div>
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
                    <button
                      v-if="
                        activeCell &&
                        rowIndex === activeCell.row &&
                        colIndex === activeCell.col &&
                        userInput[rowIndex][colIndex] === '' &&
                        possibleValues[rowIndex][colIndex]?.size > 1
                      "
                      class="explode-btn"
                      @click="onExplode"
                    >
                      💥
                    </button>
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
import { ref, onMounted, computed, watch, defineEmits } from 'vue'

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

const mainChain = ref<MoveNode[]>([])
const mainChainSet = computed(() => new Set(mainChain.value))

const emit = defineEmits(['success'])

enum Difficulty {
  Easy = 'easy',
  Medium = 'medium',
  Hard = 'hard',
  Extreme = 'extreme',
}
const selectedDifficulty = ref(Difficulty.Medium)

const puzzles = {
  [Difficulty.Easy]: {
    mission: '030490010740018000196700024000501762003027059000040300078900000429000073000370098',
    solution: '832496517745218936196753824984531762613827459257649381378962145429185673561374298',
  },
  [Difficulty.Medium]: {
    mission: '920510876480000300560328041005089060010000700036140000000031507008900000050000019',
    solution: '923514876481697325567328941245789163819263754736145298692431587178956432354872619',
  },
  [Difficulty.Hard]: {
    mission: '000019256001700009000054000002000910000000407100040683085000102030006000209003700',
    solution: '473819256851762349926354871342678915568931427197245683685497132734126598219583764',
  },
  [Difficulty.Extreme]: {
    mission: '090002070000003006001960080400000010000006000030750008000200000070890005003000700',
    solution: '396582174548173296721964583465328917987416352132759468659237841274891635813645729',
  },
}

function loadPuzzleByDifficulty() {
  const board = puzzles[selectedDifficulty.value].mission
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
  initBoards(board)
}

function onDifficultyChange() {
  loadPuzzleByDifficulty()
}

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
      failCurrentAndBacktrack()
      return
    }
    if (!userBoard.value[r][c]) {
      if (possibleValues.value[r][c].size === 0) {
        // 无解
        console.error('无解', r, c)
        errorCells.value[centerRow][centerCol] = true
        errorCells.value[r][c] = true
        failCurrentAndBacktrack()
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
  const n = parseInt(val)
  setNumber(row, col, n)
}

function setNumber(row: number, col: number, val: number) {
  userBoard.value[row][col] = val
  asyncCheck(row, col, val)
}

let blurTimeout: ReturnType<typeof setTimeout> | null = null
function onFocus(row: number, col: number) {
  if (blurTimeout) {
    clearTimeout(blurTimeout)
  }
  activeCell.value = { row, col }
}

function onBlur() {
  if (blurTimeout) {
    clearTimeout(blurTimeout)
  }
  blurTimeout = setTimeout(() => {
    activeCell.value = null
  }, 200)
}

function onFillUnique(row: number, col: number) {
  if (hasError.value) {
    return
  }
  // 取唯一解
  const val = Array.from(possibleValues.value[row][col])[0]
  userInput.value[row][col] = String(val)
  setNumber(row, col, val)
}

type MoveNode = {
  snapshot: {
    userBoard: number[][]
    userInput: string[][]
    errorCells: boolean[][]
  }
  parent: MoveNode | null
  children: MoveNode[]
  branchInfo?: { row: number; col: number; value: number }
  childIndex?: number
  status?: 'failed' | 'exploring' | 'unexplored' | 'success'
}
const moveRoot = ref<MoveNode | null>(null)
const currentMove = ref<MoveNode | null>(null)
const moveIndex = computed(() => {
  let idx = 0
  let node = moveRoot.value
  while (node && node !== currentMove.value) {
    node = node.children[0]
    idx++
  }
  return node ? idx : -1
})
function pushMoveSnapshot() {
  console.log('pushMoveSnapshot before', currentMove.value)

  const snapshot = {
    userBoard: userBoard.value.map((row) => [...row]),
    userInput: userInput.value.map((row) => [...row]),
    errorCells: errorCells.value.map((row) => [...row]),
  }
  const newNode: MoveNode = {
    snapshot,
    parent: currentMove.value,
    children: [],
  }
  if (currentMove.value) {
    currentMove.value.children = [newNode]
  } else {
    moveRoot.value = newNode
  }
  currentMove.value = newNode
  updateMainChain()
  console.log('pushMoveSnapshot', currentMove.value)

  if (isBoardSuccess()) {
    successAndBacktrack(currentMove.value)
    emit('success')
  }
}

function restoreMove(_node: MoveNode, goDeep = false) {
  // 找到第一个非失败的分叉/叶子节点
  let node = _node
  if (goDeep) {
    while (node.status !== 'failed' && node.children.length === 1) {
      node = node.children[0]
    }
  }

  userBoard.value = node.snapshot.userBoard.map((row) => [...row])
  userInput.value = node.snapshot.userInput.map((row) => [...row])
  errorCells.value = node.snapshot.errorCells.map((row) => [...row])
  updatePossibleValues()
  currentMove.value = node
  updateMainChain()

  if (node.status === 'failed' || node.status === 'success') {
    return
  }

  // 更新分支状态
  if (node.parent) {
    node.parent.children.forEach((child) => {
      if (child === node) {
        child.status = 'exploring'
      } else if (child.status !== 'failed') {
        child.status = 'unexplored'
      }
    })
  }

  // 自动 fail 检查
  let hasImpossible = false
  for (let r = 0; r < 9; r++) {
    for (let c = 0; c < 9; c++) {
      if (node.snapshot.userBoard[r][c] === 0 && possibleValues.value[r][c]?.size === 0) {
        hasImpossible = true
        break
      }
    }
    if (hasImpossible) break
  }
  if (hasImpossible) {
    node.status = 'failed'
    failCurrentAndBacktrack()
  }
}

function jumpToStep(idx: number) {
  let node = moveRoot.value
  let i = 0
  while (node && i < idx) {
    node = node.children[0]
    i++
  }
  if (node) restoreMove(node)
}

function undo() {
  if (currentMove.value && currentMove.value.parent) {
    restoreMove(currentMove.value.parent)
  }
}

function redo() {
  if (currentMove.value && currentMove.value.children.length > 0) {
    restoreMove(currentMove.value.children[0])
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
  moveRoot.value = null
  currentMove.value = null
  pushMoveSnapshot()
}

function updateMainChain() {
  let chain: MoveNode[] = []
  let node = currentMove.value
  // 先回溯到根节点
  while (node) {
    chain.push(node)
    node = node.parent
  }

  // 从根节点向下，按分支规则走主链
  node = chain[0]
  chain = chain.slice(1).reverse()

  while (node) {
    chain.push(node)
    if (node.children.length > 1) {
      // 找第一个未失败的分支
      const next: MoveNode | undefined = node.children.find((child) => child.status !== 'failed')
      if (next) {
        node = next
      } else {
        break // 全失败，停止
      }
    } else if (node.children.length === 1) {
      node = node.children[0]
    } else {
      break
    }
  }
  mainChain.value = chain
}

function failCurrentAndBacktrack() {
  console.log('failCurrentAndBacktrack', currentMove.value)
  pushMoveSnapshot()

  let node = currentMove.value
  if (!node) return
  node.status = 'failed'

  while (node?.parent) {
    const parent: MoveNode = node.parent
    const allFailed = parent.children.every((child) => child.status === 'failed')
    if (allFailed) {
      parent.status = 'failed'
      node = parent
    } else {
      // const next = parent.children.find((child) => child.status !== 'failed')
      // if (next) {
      //   restoreMove(next)
      // }
      return
    }
  }
  // 如果回溯到根节点且全部 fail，可以提示"无解"
}

function onExplode() {
  if (!activeCell.value || !currentMove.value) return
  console.log('onExplode', activeCell.value, currentMove.value)
  const { row, col } = activeCell.value
  const pv = possibleValues.value[row][col]
  if (!pv || pv.size <= 1) return

  // 清空原有分支
  currentMove.value.children = []

  let idx = 0
  for (const v of pv) {
    // 复制当前快照
    const newUserBoard = currentMove.value.snapshot.userBoard.map((row) => [...row])
    const newUserInput = currentMove.value.snapshot.userInput.map((row) => [...row])
    const newErrorCells = currentMove.value.snapshot.errorCells.map((row) => [...row])
    newUserBoard[row][col] = v
    newUserInput[row][col] = String(v)
    const newNode: MoveNode = {
      snapshot: {
        userBoard: newUserBoard,
        userInput: newUserInput,
        errorCells: newErrorCells,
      },
      parent: currentMove.value,
      children: [],
      branchInfo: { row, col, value: v },
      childIndex: idx,
      status: 'unexplored',
    }
    currentMove.value.children.push(newNode)
    idx++
  }
  if (currentMove.value.children.length > 0) {
    restoreMove(currentMove.value.children[0])
  }
}

function successAndBacktrack(node: MoveNode | null) {
  // 自下而上标记 success
  while (node) {
    node.status = 'success'
    if (node.parent) {
      node = node.parent
    } else {
      break
    }
  }
}

function isBoardSuccess() {
  return !hasError.value && userBoard.value.every((row) => row.every((cell) => cell !== 0))
}

onMounted(() => {
  loading.value = true
  loadPuzzleByDifficulty()
  loading.value = false
})
</script>

<style lang="less" scoped>
.sudoku-board {
  border-collapse: collapse;
  max-width: 434px;
  min-width: 434px;
  font-size: 20px;
  user-select: none;
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
  color: #fa5151 !important;
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
    --gap: 1px;
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
.explode-btn {
  position: absolute;
  top: 45%;
  left: 30%;
  width: 40%;
  height: 40%;
  cursor: pointer;
  background: #fbdebb;
  border-radius: 4px;
  border: 1px solid #f9d59b;
  font-size: 12px;
  color: #ccc;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 0 1px 0 0;
  &:hover {
    background: #f9d59b;
  }
}
.failed {
  color: #fa5151;
  text-decoration: line-through;
}
.success {
  color: #07c160;
  font-weight: bold;
  &:after {
    content: '✅';
  }
}
</style>
