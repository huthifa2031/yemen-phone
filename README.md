<!DOCTYPE html>
<html lang="ar">
<head>
<meta charset="UTF-8" />
<title>لعبة متاهة الشبكية مع الميزات المتقدمة</title>
<style>
  body {
    font-family: monospace;
    text-align: center;
    background: #222;
    color: white;
    margin: 20px;
  }
  #maze {
    margin: 20px auto;
    display: grid;
    gap: 2px;
    background: #333;
    padding: 5px;
    border-radius: 10px;
  }
  .cell {
    border-radius: 5px;
    user-select: none;
    width: 30px;
    height: 30px;
  }
  .wall {
    background-color: black;
  }
  .path {
    background-color: #eee;
  }
  .player {
    background-color: #4caf50;
    display: flex;
    justify-content: center;
    align-items: center;
    font-weight: bold;
    font-size: 20px;
    color: white;
  }
  .goal {
    background-color: #ff5722;
  }
  .grim {
    background-color: #b71c1c;
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 20px;
  }
  #status {
    margin-top: 20px;
    font-size: 20px;
  }
  button {
    margin: 10px 5px;
    padding: 10px 20px;
    font-size: 16px;
    border-radius: 8px;
    border: none;
    background-color: #4caf50;
    color: white;
    cursor: pointer;
    transition: background-color 0.3s ease;
  }
  button:hover {
    background-color: #388e3c;
  }
  #timer {
    font-size: 24px;
    margin-top: 10px;
  }
  select {
    font-size: 16px;
    padding: 5px;
    margin-left: 10px;
  }

  /* أزرار التحكم اللمسية */
  #touch-controls {
    position: fixed;
    bottom: 20px;
    left: 50%;
    transform: translateX(-50%);
    z-index: 1000;
    display: flex;
    gap: 10px;
  }
  #touch-controls button {
    width: 60px;
    height: 60px;
    font-size: 24px;
    border-radius: 10px;
    background-color: #4caf50;
    color: white;
    border: none;
    user-select: none;
  }
  #touch-controls button:active {
    background-color: #388e3c;
  }
</style>
</head>
<body>

<h1>لعبة متاهة الشبكية المتقدمة</h1>
<p>استخدم الأسهم لتحريك اللاعب (🧍). هدفك الوصول إلى النقطة الحمراء (🏁) والهروب من رجل الموت (💀).</p>

<label for="difficulty">اختر مستوى الصعوبة: </label>
<select id="difficulty">
  <option value="easy">سهل (10×10)</option>
  <option value="medium">متوسط (15×15)</option>
  <option value="hard">صعب (25×25)</option>
</select>

<div id="timer">الوقت: 0 ثانية</div>

<div id="maze"></div>

<div id="status"></div>

<!-- أزرار تحكم للمس في الجوال -->
<div id="touch-controls">
  <button id="btn-up">▲</button>
  <button id="btn-left">◀</button>
  <button id="btn-down">▼</button>
  <button id="btn-right">▶</button>
</div>

<button id="restartBtn">إعادة تشغيل</button>

<!-- أصوات -->
<audio id="moveSound" src="https://actions.google.com/sounds/v1/cartoon/wood_plank_flicks.ogg"></audio>
<audio id="winSound" src="https://actions.google.com/sounds/v1/cartoon/clang_and_wobble.ogg"></audio>
<audio id="loseSound" src="https://actions.google.com/sounds/v1/cartoon/cartoon_boing.ogg"></audio>

<script>
  // المتغيرات العامة
  let mazeMap = [];
  let mazeSize = 10;
  let playerPos = {row: 1, col: 1};
  let goalPos = {row: 0, col: 0};
  let grimPos = {row: 0, col: 0};
  let gameInterval = null;
  let grimInterval = null;
  let timerInterval = null;
  let seconds = 0;
  let gameRunning = false;

  const mazeDiv = document.getElementById('maze');
  const status = document.getElementById('status');
  const restartBtn = document.getElementById('restartBtn');
  const timerDisplay = document.getElementById('timer');
  const difficultySelect = document.getElementById('difficulty');

  const moveSound = document.getElementById('moveSound');
  const winSound = document.getElementById('winSound');
  const loseSound = document.getElementById('loseSound');

  // --- توليد متاهة عشوائية (خوارزمية DFS) ---
  function generateMaze(size) {
    // ماتريكس 2D مع كل الخلايا جدران في البداية
    const maze = Array(size).fill(0).map(() => Array(size).fill(1));

    // تحرك DFS (عمق البحث) لإنشاء المسارات
    const directions = [
      [0, 2], [0, -2], [2, 0], [-2, 0]
    ];

    function shuffle(array) {
      for(let i = array.length -1; i > 0; i--) {
        let j = Math.floor(Math.random() * (i+1));
        [array[i], array[j]] = [array[j], array[i]];
      }
      return array;
    }

    function carvePath(r, c) {
      maze[r][c] = 0;
      let dirs = shuffle(directions.slice());
      for (const [dr, dc] of dirs) {
        let nr = r + dr;
        let nc = c + dc;
        if (nr > 0 && nr < size-1 && nc > 0 && nc < size-1 && maze[nr][nc] === 1) {
          maze[r + dr/2][c + dc/2] = 0; // ازالة الجدار بين الخلايا
          carvePath(nr, nc);
        }
      }
    }

    carvePath(1, 1);
    return maze;
  }

  // ضبط حجم الشبكة حسب الصعوبة
  function setupDifficulty(level) {
    if (level === 'easy') mazeSize = 10;
    else if (level === 'medium') mazeSize = 15;
    else if (level === 'hard') mazeSize = 25;
    mazeDiv.style.gridTemplateColumns = `repeat(${mazeSize}, 30px)`;
    mazeDiv.style.gridTemplateRows = `repeat(${mazeSize}, 30px)`;
  }

  // إنشاء المتاهة ومواقع اللاعب، الهدف، رجل الموت
  function initGame() {
    const level = difficultySelect.value;
    setupDifficulty(level);

    mazeMap = generateMaze(mazeSize);
    playerPos = {row: 1, col: 1};
    goalPos = {row: mazeSize - 2, col: mazeSize - 2};
    grimPos = {row: mazeSize - 2, col: 1};

    seconds = 0;
    timerDisplay.textContent = `الوقت: 0 ثانية`;
    status.textContent = '';
    gameRunning = true;

    createMaze();
    window.addEventListener('keydown', keyHandler);

    if (timerInterval) clearInterval(timerInterval);
    timerInterval = setInterval(() => {
      seconds++;
      timerDisplay.textContent = `الوقت: ${seconds} ثانية`;
    }, 1000);

    if (grimInterval) clearInterval(grimInterval);
    grimInterval = setInterval(moveGrim, 1000);
  }

  // رسم المتاهة
  function createMaze() {
    mazeDiv.innerHTML = '';
    for (let r = 0; r < mazeSize; r++) {
      for (let c = 0; c < mazeSize; c++) {
        const cell = document.createElement('div');
        cell.classList.add('cell');
        if (mazeMap[r][c] === 1) cell.classList.add('wall');
        else cell.classList.add('path');

        if (r === playerPos.row && c === playerPos.col) {
          cell.classList.add('player');
          cell.textContent = '🧍';
        }
        else if (r === goalPos.row && c === goalPos.col) {
          cell.classList.add('goal');
          cell.textContent = '🏁';
        }
        else if (r === grimPos.row && c === grimPos.col) {
          cell.classList.add('grim');
          cell.textContent = '💀';
        }

        mazeDiv.appendChild(cell);
      }
    }
  }

  // تحريك اللاعب
  function movePlayer(direction) {
    if (!gameRunning) return;
    let newRow = playerPos.row;
    let newCol = playerPos.col;

    if (direction === 'up') newRow--;
    else if (direction === 'down') newRow++;
    else if (direction === 'left') newCol--;
    else if (direction === 'right') newCol++;

    if (
      newRow >= 0 && newRow < mazeSize &&
      newCol >= 0 && newCol < mazeSize &&
      mazeMap[newRow][newCol] === 0
    ) {
      playerPos.row = newRow;
      playerPos.col = newCol;
      moveSound.play();
      createMaze();
      checkGameState();
    }
  }

  // تحريك رجل الموت باتجاه اللاعب (أقصر مسار)
  function moveGrim() {
    if (!gameRunning) return;

    // خوارزمية BFS لإيجاد أقصر مسار من grimPos إلى playerPos
    let queue = [];
    let visited = Array(mazeSize).fill(0).map(() => Array(mazeSize).fill(false));
    let parent = Array(mazeSize).fill(0).map(() => Array(mazeSize).fill(null));

    queue.push(grimPos);
    visited[grimPos.row][grimPos.col] = true;

    while(queue.length > 0) {
      const current = queue.shift();
      if(current.row === playerPos.row && current.col === playerPos.col) {
        break; // وجدنا اللاعب
      }
      const directions = [
        {r: current.row-1, c: current.col},
        {r: current.row+1, c: current.col},
        {r: current.row, c: current.col-1},
        {r: current.row, c: current.col+1}
      ];

      for (const d of directions) {
        if(d.r >= 0 && d.r < mazeSize && d.c >= 0 && d.c < mazeSize &&
          !visited[d.r][d.c] && mazeMap[d.r][d.c] === 0) {
          queue.push({row: d.r, col: d.c});
          visited[d.r][d.c] = true;
          parent[d.r][d.c] = current;
        }
      }
    }

    // استرجاع الخطوة التالية لرجل الموت نحو اللاعب
    let step = playerPos;
    let prev = parent[step.row][step.col];
    if (!prev) return; // لا يوجد مسار

    while(parent[prev.row][prev.col] !== null) {
      step = prev;
      prev = parent[step.row][step.col];
    }

    grimPos = step;
    createMaze();
    checkGameState();
  }

  // تحقق حالة الفوز أو الخسارة
  function checkGameState() {
    if(playerPos.row === goalPos.row && playerPos.col === goalPos.col) {
      gameRunning = false;
      clearInterval(timerInterval);
      clearInterval(grimInterval);
      status.textContent = `مبروك! فزت باللعبة في ${seconds} ثانية.`;
      winSound.play();
    } else if(playerPos.row === grimPos.row && playerPos.col === grimPos.col) {
      gameRunning = false;
      clearInterval(timerInterval);
      clearInterval(grimInterval);
      status.textContent = `خسرت! رجل الموت أمسك بك بعد ${seconds} ثانية.`;
      loseSound.play();
    }
  }

  // تحكم لوحة المفاتيح
  function keyHandler(e) {
    if(!gameRunning) return;
    switch(e.key) {
      case 'ArrowUp': movePlayer('up'); break;
      case 'ArrowDown': movePlayer('down'); break;
      case 'ArrowLeft': movePlayer('left'); break;
      case 'ArrowRight': movePlayer('right'); break;
    }
  }

  // إعادة تشغيل اللعبة
  restartBtn.addEventListener('click', () => {
    clearInterval(timerInterval);
    clearInterval(grimInterval);
    seconds = 0;
    initGame();
  });

  // تغيير مستوى الصعوبة
  difficultySelect.addEventListener('change', () => {
    clearInterval(timerInterval);
    clearInterval(grimInterval);
    seconds = 0;
    initGame();
  });

  // **أزرار التحكم اللمسية للجوال**
  document.getElementById('btn-up').addEventListener('touchstart', (e) => {
    e.preventDefault();
    movePlayer('up');
  });
  document.getElementById('btn-left').addEventListener('touchstart', (e) => {
    e.preventDefault();
    movePlayer('left');
  });
  document.getElementById('btn-down').addEventListener('touchstart', (e) => {
    e.preventDefault();
    movePlayer('down');
  });
  document.getElementById('btn-right').addEventListener('touchstart', (e) => {
    e.preventDefault();
    movePlayer('right');
  });

  // بدء اللعبة لأول مرة عند تحميل الصفحة
  window.onload = () => {
    initGame();
  }
</script>

</body>
</html>
