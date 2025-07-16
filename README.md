@ -0,0 +1,331 @@
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Snake Game – Realistic UI</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <style>
    /* === Snake Game Styles === */
    * {
      box-sizing: border-box;
    }

    body {
      margin: 0;
      background: linear-gradient(to bottom right, #0d0d0d, #1a1a1a);
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: flex-start;
      color: #00ff99;
      user-select: none;
      height: 100vh;
      padding: 20px 0;
    }

    #scoreDisplay {
      font-size: 28px;
      margin: 25px 0 10px;
      color: #00ff99;
      text-shadow: 1px 1px 3px #000000aa;
      font-weight: 600;
      user-select: none;
    }

    canvas {
      background-color: #111;
      border: 4px solid #00ff99;
      box-shadow:
        0 0 15px #00ff99aa,
        0 0 25px #00ff9977;
      border-radius: 12px;
      display: block;
    }

    .controls {
      margin-top: 20px;
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 12px;
      user-select: none;
    }

    .control-row {
      display: flex;
      gap: 16px;
    }

    .btn {
      width: 60px;
      height: 60px;
      font-size: 24px;
      font-weight: bold;
      color: #00ff99;
      background-color: #222;
      border: 2px solid #00ff99;
      border-radius: 12px;
      box-shadow: 0 2px 10px rgba(0, 255, 153, 0.3);
      cursor: pointer;
      transition: transform 0.1s ease, box-shadow 0.3s ease;
      touch-action: manipulation;
      display: flex;
      align-items: center;
      justify-content: center;
    }

    .btn:hover {
      box-shadow:
        0 0 15px #00ff99,
        inset 0 0 8px #00ff99;
      transform: scale(1.05);
    }

    .btn:active {
      transform: scale(0.95);
      box-shadow:
        0 0 8px #00cc77,
        inset 0 0 5px #00cc77;
    }

    @media (min-width: 600px) {
      .btn {
        width: 80px;
        height: 80px;
        font-size: 28px;
      }
    }

    #startBtn {
      margin-bottom: 15px;
      width: 150px;
      font-size: 20px;
      font-weight: 700;
      background: #004d00;
      border: 3px solid #00ff99;
      border-radius: 15px;
      color: #00ff99;
      cursor: pointer;
      box-shadow:
        0 0 20px #00ff99cc;
      transition: transform 0.15s ease;
      user-select: none;
    }

    #startBtn:hover {
      transform: scale(1.1);
    }
  </style>
</head>
<body>
  <button id="startBtn">▶️ Start Game</button>
  <div id="scoreDisplay" style="display:none;">Score: <span id="score">0</span></div>
  <canvas id="gameCanvas" width="400" height="400" style="display:none;"></canvas>

  <div class="controls" style="display:none;">
    <div class="control-row">
      <button class="btn" onclick="setDirection('UP')">⬆️</button>
    </div>
    <div class="control-row">
      <button class="btn" onclick="setDirection('LEFT')">⬅️</button>
      <button class="btn" onclick="setDirection('DOWN')">⬇️</button>
      <button class="btn" onclick="setDirection('RIGHT')">➡️</button>
    </div>
  </div>

  <!-- Sound Effects -->
  <audio id="eatSound" src="https://actions.google.com/sounds/v1/cartoon/pop.mp3" preload="auto"></audio>
  <audio id="gameOverSound" src="https://actions.google.com/sounds/v1/cartoon/clang_and_wobble.mp3" preload="auto"></audio>
  <!-- Background Music -->
  <audio id="bgMusic" src="https://cdn.pixabay.com/download/audio/2022/03/15/audio_6f6b07e279.mp3?filename=electronic-chill-ambient-11110.mp3" preload="auto" loop></audio>

  <script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const box = 20;

    let score = 0;
    let direction = 'RIGHT';
    let changingDirection = false; // to avoid simultaneous key press issues

    const eatSound = document.getElementById('eatSound');
    const gameOverSound = document.getElementById('gameOverSound');
    const bgMusic = document.getElementById('bgMusic');

    const startBtn = document.getElementById('startBtn');
    const scoreDisplay = document.getElementById('scoreDisplay');
    const controlsDiv = document.querySelector('.controls');

    let gameInterval = null;

    // Unlock audio on first user interaction (to avoid autoplay blocking)
    function unlockAudio() {
      eatSound.play().catch(() => {});
      gameOverSound.play().catch(() => {});
      bgMusic.play().catch(() => {});
      eatSound.pause();
      gameOverSound.pause();
      bgMusic.pause();
      document.body.removeEventListener('click', unlockAudio);
    }
    document.body.addEventListener('click', unlockAudio);

    let snake, food;

    function randomFood() {
      let position;
      do {
        position = {
          x: Math.floor(Math.random() * 20) * box,
          y: Math.floor(Math.random() * 20) * box
        };
      } while (collision(position, snake));
      return position;
    }

    function setDirection(dir) {
      if (changingDirection) return; // prevent changing direction multiple times in one frame
      if (dir === 'LEFT' && direction !== 'RIGHT') direction = 'LEFT';
      else if (dir === 'RIGHT' && direction !== 'LEFT') direction = 'RIGHT';
      else if (dir === 'UP' && direction !== 'DOWN') direction = 'UP';
      else if (dir === 'DOWN' && direction !== 'UP') direction = 'DOWN';
      changingDirection = true;
    }

    document.addEventListener('keydown', e => {
      const key = e.key.toLowerCase();
      if (key === 'a') setDirection('LEFT');
      else if (key === 'w') setDirection('UP');
      else if (key === 'd') setDirection('RIGHT');
      else if (key === 's') setDirection('DOWN');
    });

    function collision(head, body) {
      return body.some(segment => head.x === segment.x && head.y === segment.y);
    }

    // Add roundRect support if missing
    if (!CanvasRenderingContext2D.prototype.roundRect) {
      CanvasRenderingContext2D.prototype.roundRect = function (x, y, w, h, r) {
        r = Math.min(r, w / 2, h / 2);
        this.beginPath();
        this.moveTo(x + r, y);
        this.arcTo(x + w, y, x + w, y + h, r);
        this.arcTo(x + w, y + h, x, y + h, r);
        this.arcTo(x, y + h, x, y, r);
        this.arcTo(x, y, x + w, y, r);
        this.closePath();
        return this;
      };
    }

    function drawSnakePart(x, y, isHead = false) {
      const gradient = ctx.createLinearGradient(x, y, x + box, y + box);
      gradient.addColorStop(0, isHead ? '#00FFB3' : '#00cc66');
      gradient.addColorStop(1, isHead ? '#00cc99' : '#006633');

      ctx.fillStyle = gradient;
      ctx.beginPath();
      ctx.roundRect(x, y, box, box, 6);
      ctx.fill();

      // Eyes for the snake head
      if (isHead) {
        ctx.fillStyle = '#000';
        const eyeSize = 3;
        ctx.beginPath();
        ctx.arc(x + 6, y + 6, eyeSize, 0, Math.PI * 2);
        ctx.arc(x + box - 6, y + 6, eyeSize, 0, Math.PI * 2);
        ctx.fill();
      }
    }

    function draw() {
      changingDirection = false; // reset direction change flag

      ctx.clearRect(0, 0, canvas.width, canvas.height);

      // Draw snake
      for (let i = 0; i < snake.length; i++) {
        drawSnakePart(snake[i].x, snake[i].y, i === 0);
      }

      // Draw food
      ctx.fillStyle = '#ff4d4d';
      ctx.beginPath();
      ctx.arc(food.x + box / 2, food.y + box / 2, box / 2 - 2, 0, Math.PI * 2);
      ctx.fill();

      let head = { x: snake[0].x, y: snake[0].y };
      if (direction === 'LEFT') head.x -= box;
      else if (direction === 'UP') head.y -= box;
      else if (direction === 'RIGHT') head.x += box;
      else if (direction === 'DOWN') head.y += box;

      if (
        head.x < 0 || head.y < 0 ||
        head.x >= canvas.width || head.y >= canvas.height ||
        collision(head, snake)
      ) {
        clearInterval(gameInterval);
        gameOverSound.play();
        alert("💀 Game Over! Score: " + score);
        resetGame();
        return;
      }

      if (head.x === food.x && head.y === food.y) {
        eatSound.play();
        score++;
        document.getElementById("score").innerText = score;
        food = randomFood();
      } else {
        snake.pop();
      }

      snake.unshift(head);
    }

    function resetGame() {
      // Reset variables and hide game UI
      clearInterval(gameInterval);
      score = 0;
      direction = 'RIGHT';
      changingDirection = false;
      snake = [{ x: 5 * box, y: 5 * box }];
      food = randomFood();

      scoreDisplay.style.display = 'none';
      canvas.style.display = 'none';
      controlsDiv.style.display = 'none';
      startBtn.style.display = 'inline-block';

      bgMusic.pause();
      bgMusic.currentTime = 0;
    }

    function startGame() {
      score = 0;
      direction = 'RIGHT';
      changingDirection = false;
      snake = [{ x: 5 * box, y: 5 * box }];
      food = randomFood();

      scoreDisplay.style.display = 'block';
      canvas.style.display = 'block';
      controlsDiv.style.display = 'flex';
      startBtn.style.display = 'none';

      bgMusic.play();

      gameInterval = setInterval(draw, 150);
    }

    startBtn.addEventListener('click', startGame);

    // Initialize on page load
    resetGame();
  </script>
</body>
</html>
