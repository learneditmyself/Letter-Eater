# Letter-Eater
Eat Letters
<!DOCTYPE html>
<html>
<head>
  <style>
    canvas { border: 1px solid black; }
    body { margin: 0; display: flex; justify-content: center; align-items: center; touch-action: none; }
    @font-face {
      font-family: 'OpenDyslexic';
      src: url('https://cdn.jsdelivr.net/npm/open-dyslexic@1.0.1/OpenDyslexic-Regular.ttf') format('truetype');
      font-weight: normal;
      font-style: normal;
    }
  </style>
</head>
<body>
  <canvas id="gameCanvas" width="800" height="400" tabindex="0"></canvas>
  <!-- Audio placeholders - replace src with your hosted files -->
  <audio id="collectSound" src="https://example.com/collect.mp3"></audio>
  <audio id="bonusSound" src="https://example.com/bonus.mp3"></audio>
  <audio id="beastSound" src="https://example.com/beast.mp3"></audio>
  <script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    canvas.focus();

    // Game objects
    const player = {
      x: 100,
      y: 350,
      radius: 20,
      originalRadius: 20,
      speed: 5,
      dy: 0,
      gravity: 0.5,
      jumpPower: -12,
      color: 'blue',
      powerUpLetters: 0,
      beastMode: false,
      beastModeTimer: 0,
      beastModeDuration: 5000 // 5 seconds in milliseconds
    };
    const letters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'.split('');
    let availableLetters = [...letters];
    let collectedLetters = new Set();
    let currentLetter = null;
    let bonuses = [];
    let score = 0;
    let flashColor = null;
    let flashTimer = 0;
    let mouthAngle = 0.2;
    let mouthSpeed = 0.05;
    let lastStarSpawn = 0;

    // Input handling
    let leftPressed = false;
    let rightPressed = false;
    let spacePressed = false;
    let isTouching = false;
    let touchX = null;

    // Sounds
    const collectSound = document.getElementById('collectSound');
    const bonusSound = document.getElementById('bonusSound');
    const beastSound = document.getElementById('beastSound');

    // Keyboard controls
    canvas.addEventListener('keydown', (e) => {
      e.preventDefault();
      if (e.key === 'ArrowLeft') leftPressed = true;
      if (e.key === 'ArrowRight') rightPressed = true;
      if (e.key === ' ' && !spacePressed) {
        player.dy = player.jumpPower;
        spacePressed = true;
      }
    });
    canvas.addEventListener('keyup', (e) => {
      e.preventDefault();
      if (e.key === 'ArrowLeft') leftPressed = false;
      if (e.key === 'ArrowRight') rightPressed = false;
      if (e.key === ' ') spacePressed = false;
    });

    // Touch controls
    canvas.addEventListener('touchstart', (e) => {
      e.preventDefault();
      isTouching = true;
      touchX = e.touches[0].clientX - canvas.offsetLeft;
      player.dy = player.jumpPower;
    });
    canvas.addEventListener('touchmove', (e) => {
      e.preventDefault();
      touchX = e.touches[0].clientX - canvas.offsetLeft;
    });
    canvas.addEventListener('touchend', (e) => {
      e.preventDefault();
      isTouching = false;
      touchX = null;
    });

    canvas.addEventListener('click', () => canvas.focus());

    // Spawn letter
    function spawnLetter() {
      if (availableLetters.length === 0) {
        availableLetters = [...letters];
      }
      const letterIndex = Math.floor(Math.random() * availableLetters.length);
      const upper = availableLetters[letterIndex];
      const lower = upper.toLowerCase();
      currentLetter = {
        text: `${upper}${lower}`,
        x: canvas.width,
        y: 186,
        speed: player.beastMode ? 4 : 2
      };
      availableLetters.splice(letterIndex, 1);
    }

    // Spawn bonus (yellow star or red beast star)
    function spawnBonus(timestamp) {
      const letterCount = collectedLetters.size;
      if (bonuses.length < 2 && letterCount > lastStarSpawn) {
        if (letterCount === 5 || (letterCount >= 10 && (letterCount - 5) % 10 === 0)) {
          const type = Math.random() < 0.3 ? 'beast' : 'star';
          bonuses.push({ x: canvas.width, y: Math.random() * 300, type });
          lastStarSpawn = letterCount;
        }
      }
    }

    // Handle power-up and beast mode effects
    function updatePowerUp(timestamp) {
      if (player.beastMode && timestamp - player.beastModeTimer > player.beastModeDuration) {
        player.beastMode = false;
        player.speed = 5;
        player.radius = player.powerUpLetters > 0 ? player.originalRadius * 2 : player.originalRadius;
        player.color = player.powerUpLetters > 0 ? 'orange' : 'blue';
      }
      if (player.powerUpLetters >= 5 && collectedLetters.size === lastStarSpawn + 5) {
        player.radius = player.originalRadius;
        player.color = 'blue';
        player.powerUpLetters = 0;
      } else if (player.powerUpLetters >= 10 && collectedLetters.size === lastStarSpawn + 10) {
        player.radius = player.originalRadius;
        player.color = 'blue';
        player.powerUpLetters = 0;
      }
    }

    // Game loop
    function gameLoop(timestamp = 0) {
      // Theme background
      ctx.fillStyle = player.beastMode ? '#FF4500' : (flashColor || '#87CEEB');
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = '#90EE90';
      ctx.fillRect(0, 300, canvas.width, 100);
      ctx.fillStyle = '#FFD700';
      ctx.beginPath();
      ctx.arc(700, 50, 30, 0, Math.PI * 2);
      ctx.fill();

      // Flash timing
      if (flashColor && timestamp - flashTimer > 200) {
        flashColor = null;
      } else if (flashColor === 'red' && timestamp - flashTimer > 100) {
        flashColor = 'yellow';
      }

      // Player physics
      player.y += player.dy;
      player.dy += player.gravity;
      if (player.y + player.radius > 350) {
        player.y = 350 - player.radius;
        player.dy = 0;
      }

      // Move player
      const currentSpeed = player.beastMode ? player.speed * 1.5 : player.speed;
      if (leftPressed && player.x > player.radius) player.x -= currentSpeed;
      if (rightPressed && player.x < canvas.width - player.radius) player.x += currentSpeed;
      if (isTouching && touchX !== null) {
        if (touchX < player.x && player.x > player.radius) player.x -= currentSpeed;
        else if (touchX > player.x && player.x < canvas.width - player.radius) player.x += currentSpeed;
      }

      // Animate and draw player
      mouthAngle += mouthSpeed;
      if (mouthAngle > 0.4 || mouthAngle < 0.2) mouthSpeed = -mouthSpeed;
      ctx.fillStyle = flashColor ? 'cyan' : player.color;
      ctx.beginPath();
      ctx.arc(player.x, player.y, player.radius, mouthAngle, Math.PI * 2 - mouthAngle);
      ctx.lineTo(player.x, player.y);
      ctx.fill();

      // Move and draw letter
      if (currentLetter) {
        currentLetter.x -= currentLetter.speed;
        ctx.fillStyle = 'white';
        ctx.font = '80px OpenDyslexic';
        ctx.fillText(currentLetter.text, currentLetter.x, currentLetter.y);

        // Collision
        if (
          player.x + player.radius > currentLetter.x &&
          player.x - player.radius < currentLetter.x + 80 &&
          player.y - player.radius < currentLetter.y &&
          player.y + player.radius > currentLetter.y - 40
        ) {
          score += player.beastMode ? 3 : 1;
          collectedLetters.add(currentLetter.text[0]);
          if (player.powerUpLetters > 0) player.powerUpLetters++;
          collectSound.play();
          currentLetter = null;
          flashColor = 'red';
          flashTimer = timestamp;
          spawnLetter();
          updatePowerUp(timestamp);
        }
        if (currentLetter.x < -80) currentLetter = null;
      }

      // Bonus stars and beast mode
      bonuses.forEach((b, i) => {
        b.x -= player.beastMode ? 4 : 2;
        ctx.fillStyle = b.type === 'beast' ? 'red' : 'yellow';
        ctx.fillText('★', b.x, b.y);
        if (Math.abs(b.x - player.x) < 30 && Math.abs(b.y - player.y) < 30) {
          if (b.type === 'beast') {
            player.beastMode = true;
            player.beastModeTimer = timestamp;
            player.speed = 8;
            player.radius = player.originalRadius * 2.5;
            player.color = 'red';
            beastSound.play();
          } else {
            score += 5;
            bonusSound.play();
            const letterCount = collectedLetters.size;
            if (letterCount === 5) {
              player.color = 'purple';
              player.radius = player.originalRadius * 1.5;
              player.powerUpLetters = 1;
            } else if (letterCount >= 10) {
              player.color = 'orange';
              player.radius = player.originalRadius * 2;
              player.powerUpLetters = 1;
            }
          }
          bonuses.splice(i, 1);
        }
      });

      // Score and progress
      ctx.fillStyle = 'white';
      ctx.font = '20px OpenDyslexic';
      ctx.fillText(`Score: ${score}`, 10, 30);
      ctx.fillText(`Letters: ${collectedLetters.size}/26`, 10, 50);
      if (player.beastMode) {
        const timeLeft = Math.ceil((player.beastModeDuration - (timestamp - player.beastModeTimer)) / 1000);
        ctx.fillText(`Beast Mode: ${timeLeft}s`, 10, 70);
      }
      if (collectedLetters.size === 26) {
        ctx.fillStyle = 'green';
        ctx.fillText('You Win! Tap to Restart', canvas.width/2 - 150, canvas.height/2);
        canvas.addEventListener('click', () => {
          collectedLetters.clear();
          score = 0;
          player.radius = player.originalRadius;
          player.color = 'blue';
          player.powerUpLetters = 0;
          player.beastMode = false;
          bonuses = [];
          lastStarSpawn = 0;
          spawnLetter();
        }, { once: true });
      }

      // Spawn new items
      if (!currentLetter && Math.random() < 0.03) spawnLetter();
      spawnBonus(timestamp);

      requestAnimationFrame(gameLoop);
    }

    spawnLetter();
    gameLoop();
  </script>
</body>
</html>
