<template>
  <div>
    <h3>踩地雷</h3>
    <div class="game-info">
      <div>地雷總數：{{ minesCount }}</div>
      <p class="annotationText">點擊右鍵可猜測地雷，再次點擊右鍵取消猜測</p>
    </div>
    <div class="board">
      <div
        v-for="(row, rowIndex) in board"
        :key="rowIndex"
        class="row"
      >
        <div
          v-for="(cell, colIndex) in row"
          :key="colIndex"
          class="cell"
          :class="{ revealed: cell.revealed, mine: cell.mine, guess: cell.guess }"
          @click="revealCell(rowIndex, colIndex)"
          @contextmenu.prevent="toggleGuess(rowIndex, colIndex)"
        >
          <!-- 條件渲染 -->
          <template v-if="cell.revealed && cell.adjacentMines !== 0 && !cell.mine">
            {{ cell.adjacentMines }}
          </template>
        </div>
      </div>
    </div>
    <button @click="restart">重新開始</button>
  </div>
</template>

<script setup>
import { ref } from 'vue';

const rows = 8;
const cols = 8;
const maxMinesCount = 10; // 最多地雷數量
let minesCount = ref(maxMinesCount);
const board = ref([]);

const initializeBoard = () => { // 遊戲初始化
  for (let i = 0; i < rows; i++) {
    board.value.push([]);
    for (let j = 0; j < cols; j++) {
      board.value[i].push({
        revealed: false,
        mine: false,
        adjacentMines: 0,
        guess: false // 新增猜測地雷的屬性
      });
    }
  }
  placeMines();
  calculateAdjacentMines();
};

const placeMines = () => { // 隨機放置地雷的功能
  let minesPlaced = 0;
  while (minesPlaced < maxMinesCount) {
    const row = Math.floor(Math.random() * rows);
    const col = Math.floor(Math.random() * cols);
    if (!board.value[row][col].mine) {
      board.value[row][col].mine = true;
      minesPlaced++;
    }
  }
};

const calculateAdjacentMines = () => { // 計算每個非地雷格子周圍的地雷數量
  for (let i = 0; i < rows; i++) {
    for (let j = 0; j < cols; j++) {
      if (!board.value[i][j].mine) {
        let count = 0;
        for (let di = -1; di <= 1; di++) {
          for (let dj = -1; dj <= 1; dj++) {
            const ni = i + di;
            const nj = j + dj;
            if (
              ni >= 0 &&
              ni < rows &&
              nj >= 0 &&
              nj < cols &&
              board.value[ni][nj].mine
            ) {
              count++;
            }
          }
        }
        board.value[i][j].adjacentMines = count;
      }
    }
  }
};

const revealAdjacentZeroes = (row, col) => { // 檢查每個相鄰格子，符合條件揭示
  const directions = [
    [-1, -1], [-1, 0], [-1, 1],
    [0, -1],           [0, 1],
    [1, -1],  [1, 0],  [1, 1]
  ];
  for (const [dx, dy] of directions) {
    const newRow = row + dx;
    const newCol = col + dy;
    if (
      newRow >= 0 && newRow < rows &&
      newCol >= 0 && newCol < cols &&
      !board.value[newRow][newCol].revealed &&
      !board.value[newRow][newCol].guess // 檢查相鄰格子是否被註記為藍色格子
    ) {
      if (board.value[newRow][newCol].adjacentMines === 0) {
        revealCell(newRow, newCol);
      } else if (!board.value[newRow][newCol].mine) {
        board.value[newRow][newCol].revealed = true;
      }
    }
  }
};

const revealCell = (row, col) => { // 揭示單元格的行為
  const cell = board.value[row][col];
  if (!cell.revealed && !cell.guess) {
    cell.revealed = true;
    if (cell.adjacentMines === 0 && !cell.mine) {
      revealAdjacentZeroes(row, col);
    }
    if (cell.mine) {
      cell.revealed = true;
      showAllCells(); // 在彈跳視窗之前顯示所有格子
      setTimeout(() => {
        alert('Game Over!');
        restart();
      }, 100);
    }

    // 检查除了地雷以外的所有单元格是否已被揭示
    let allNonMineCellsRevealed = true;
    for (let i = 0; i < rows; i++) {
      for (let j = 0; j < cols; j++) {
        const nonMineCell = board.value[i][j];
        if (!nonMineCell.mine && !nonMineCell.revealed) {
          allNonMineCellsRevealed = false;
          break;
        }
      }
      if (!allNonMineCellsRevealed) {
        break;
      }
    }
    if (allNonMineCellsRevealed && !cell.mine) {
      cell.revealed = true;
      showAllCells(); // 在彈跳視窗之前顯示所有格子
      setTimeout(() => {
        alert('You Win!');
        restart();
      }, 100);
    }
  }
};

const showAllCells = () => { // 遊戲結束時顯示所有單元格的狀態
  for (let i = 0; i < rows; i++) {
    for (let j = 0; j < cols; j++) {
      const cell = board.value[i][j];
      if (!cell.revealed) {
        cell.revealed = true;
        cell.guess = false;
        minesCount.value = maxMinesCount; // 重置地雷總數
        if (cell.mine) {
          if (cell.guess) {
            // 如果之前猜測為地雷的藍色格子，取消猜測並設置為灰色
            cell.revealed = false; // 不顯示顏色
          } else {
            // 如果是地雷格子，顯示為紅色
            cell.mine = true;
          }
        } else {
          // 如果是數值格子，設置為#ddd色
          cell.revealed = true;
        }
      }
    }
  }
};

const toggleGuess = (row, col) => { // 點擊一個格子的後續反應
  const cell = board.value[row][col];
  if (!cell.revealed) {
    if (!cell.guess && minesCount.value > 0) {
      cell.guess = true;
      minesCount.value--;
    } else if (cell.guess) {
      cell.guess = false;
      minesCount.value++;
    }
  }
};

const restart = () => { // 重新開始遊戲的功能
  minesCount.value = maxMinesCount; // 重置地雷總數
  for (let i = 0; i < rows; i++) {
    for (let j = 0; j < cols; j++) {
      const cell = board.value[i][j];
      cell.revealed = false;
      cell.mine = false;
      cell.adjacentMines = 0;
      cell.guess = false;
    }
  }
  placeMines();
  calculateAdjacentMines();
};

initializeBoard();
</script>

<style scoped>
h3 {
  font-size: 28px;
}

.annotationText {
  margin-top: 5px;
  font-size: 12px;
}

.board {
  display: flex;
  flex-direction: column;
  margin-bottom: 10px;
}

.row {
  display: flex;
}

.cell {
  width: 30px;
  height: 30px;
  border: 1px solid #ccc;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
}

.cell.revealed {
  background-color: #ddd;
}

.cell.mine {
  background-color: transparent;
  border: 1px solid #ccc;
  color: transparent;
}

.cell.mine.revealed {
  background-color: red;
  color: white;
}

.cell.guess {
  background-color: blue;
}

.cell.guess.revealed {
  background-color: transparent;
}
</style>