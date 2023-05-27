<script setup lang="ts">
import { ref } from 'vue'

enum serverType {
  empty,
  bomb
}

interface square {
  type: serverType
  revealed: boolean
  flagged: boolean
  neighbors: number
}

const board = ref<square[][]>([])
const size = ref(10)
const bombs = ref(10)
const gameover = ref(false)
const won = ref(false)
const timer = ref(60)
const firstMove = ref(true)
const timerLoop = ref()
const modal = ref(false)

const createBoard = () => {
  won.value = false
  timer.value = 60
  gameover.value = false
  firstMove.value = true

  board.value = []
  for (let i = 0; i < size.value; i++) {
    board.value.push([])
    for (let j = 0; j < size.value; j++) {
      board.value[i].push({
        type: serverType.empty,
        revealed: false,
        flagged: false,
        neighbors: 0
      })
    }
  }
  for (let i = 0; i < bombs.value; i++) {
    let x = Math.floor(Math.random() * size.value)
    let y = Math.floor(Math.random() * size.value)
    if (board.value[x][y].type == serverType.bomb) {
      i--
    } else {
      board.value[x][y].type = serverType.bomb
    }
  }

  updateNeighbors()

  console.log(board.value)
}

const updateNeighbors = () => {
  for (let i = 0; i < size.value; i++) {
    for (let j = 0; j < size.value; j++) {
      if (board.value[i][j].type == serverType.empty) {
        let neighbors = 0
        if (i > 0 && j > 0 && board.value[i - 1][j - 1].type == serverType.bomb) {
          neighbors++
        }
        if (i > 0 && board.value[i - 1][j].type == serverType.bomb) {
          neighbors++
        }
        if (i > 0 && j < size.value - 1 && board.value[i - 1][j + 1].type == serverType.bomb) {
          neighbors++
        }
        if (j > 0 && board.value[i][j - 1].type == serverType.bomb) {
          neighbors++
        }
        if (j < size.value - 1 && board.value[i][j + 1].type == serverType.bomb) {
          neighbors++
        }
        if (i < size.value - 1 && j > 0 && board.value[i + 1][j - 1].type == serverType.bomb) {
          neighbors++
        }
        if (i < size.value - 1 && board.value[i + 1][j].type == serverType.bomb) {
          neighbors++
        }
        if (
          i < size.value - 1 &&
          j < size.value - 1 &&
          board.value[i + 1][j + 1].type == serverType.bomb
        ) {
          neighbors++
        }
        board.value[i][j].neighbors = neighbors
      } else {
        board.value[i][j].neighbors = 0
      }
    }
  }
}

let updateBoard: square[][]

const viewSquare = async (x: number, y: number) => {
  updateBoard = JSON.parse(JSON.stringify(board.value))

  if (firstMove.value) {
    firstMove.value = false
    timerLoop.value = setInterval(() => {
      if (timer.value <= 0) {
        clearInterval(timerLoop.value)
        modal.value = true
        return (gameover.value = true)
      }
      timer.value--
    }, 1000)
    if (updateBoard[x][y].type == serverType.bomb) {
      updateBoard[x][y].type = serverType.empty
      let newBomb = {
        x: Math.floor(Math.random() * size.value),
        y: Math.floor(Math.random() * size.value)
      }

      while (updateBoard[newBomb.x][newBomb.y].type == serverType.bomb) {
        newBomb = {
          x: Math.floor(Math.random() * size.value),
          y: Math.floor(Math.random() * size.value)
        }
      }

      updateBoard[newBomb.x][newBomb.y].type = serverType.bomb
      updateNeighbors()
    }
  }
  if (updateBoard[x][y].flagged) {
    return
  }
  if (updateBoard[x][y].type == serverType.bomb) {
    for (let i = 0; i < board.value.length; i++) {
      for (let j = 0; j < board.value[i].length; j++) {
        if (board.value[i][j].type == serverType.bomb) {
          board.value[i][j].revealed = true
        }
      }
    }
    modal.value = true
    clearInterval(timerLoop.value)
    return (gameover.value = true)
  }
  updateBoard[x][y].revealed = true

  await Promise.all([
    recursiveView(x, y - 1),
    recursiveView(x - 1, y - 1),
    recursiveView(x + 1, y - 1),
    recursiveView(x, y + 1),
    recursiveView(x - 1, y + 1),
    recursiveView(x + 1, y + 1)
  ])

  board.value = updateBoard

  for (let i = 0; i < board.value.length; i++) {
    for (let j = 0; j < board.value[i].length; j++) {
      if (board.value[i][j].type == serverType.empty && !board.value[i][j].revealed) return
    }
  }
  won.value = true
  modal.value = true
  clearInterval(timerLoop.value)
}

const recursiveView = async (x: number, y: number) => {
  if (x < 0 || y < 0 || x > size.value || y > size.value || !board.value[x] || !board.value[x][y])
    return
  //   console.log(x, y)
  if (updateBoard[x][y].revealed) return
  if (updateBoard[x][y].type == serverType.bomb) {
    return console.log('is bomb')
  }
  updateBoard[x][y].revealed = true
  if (updateBoard[x][y].neighbors > 0) {
    return console.log('is neighbors')
  }

  // check all neighboring squares
  return Promise.all([
    recursiveView(x, y - 1),
    recursiveView(x - 1, y - 1),
    recursiveView(x + 1, y - 1),
    recursiveView(x - 1, y),
    recursiveView(x + 1, y),
    recursiveView(x, y + 1),
    recursiveView(x - 1, y + 1),
    recursiveView(x + 1, y + 1)
  ])
}

createBoard()
</script>

<template>
  <main class="min-h-screen flex justify-center items-center flex-col">
    <p>{{ timer }} seconds left</p>
    <div class="flex flex-col">
      <div v-for="(row, i) in board" :key="i" class="flex">
        <button
          v-for="(square, j) in row"
          :key="`${i} ${j}`"
          class="w-8 h-8 border-white border bg-slate-500"
          :class="{
            'bg-slate-300': square.revealed,
            'bg-red-500': square.type === serverType.bomb
          }"
          @click="viewSquare(i, j)"
          v-on:contextmenu.prevent="square.flagged = true"
        >
          {{ square.flagged && !square.revealed ? '🚩' : '' }}
          {{ square.neighbors > 0 && square.revealed ? square.neighbors : '' }}
        </button>
      </div>
    </div>
    <!-- {{ timer }} seconds | {{ gameover }} | {{ won }} -->
  </main>
  <div
    v-if="modal"
    class="h-screen absolute top-0 left-0 w-screen bg-black/75 flex justify-center items-center"
  >
    <div class="bg-white shadow-md rounded-2xl px-8 py-6 flex flex-col items-center">
      <h1 class="text-4xl mb-2 font-semibold">{{ gameover ? 'Game Over' : 'You Won!' }}</h1>
      <p class="text-xl">
        {{
          gameover
            ? timer <= 0
              ? 'You ran out of time'
              : 'You click on a mine'
            : 'You cleared the entire board'
        }}
      </p>
      <button
        class="bg-blue-500 text-white px-4 py-2 rounded-md mt-2"
        @click="
          () => {
            modal = false
            createBoard()
          }
        "
      >
        Try Again
      </button>
    </div>
  </div>
</template>
