# flappy-face
<!DOCTYPE html>
<html lang="hu">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Flappy Face - Proporcionális Fejjel</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #70c5ce;
            font-family: 'Arial', sans-serif;
            touch-action: manipulation;
        }
        canvas {
            display: block;
            margin: 0 auto;
            background-color: #70c5ce;
        }
        #ui-layer {
            position: absolute;
            top: 40px;
            left: 0;
            right: 0;
            text-align: center;
            pointer-events: none;
            color: white;
            text-shadow: 3px 3px 0px #000;
            z-index: 10;
        }
        .message-box {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: #ded895;
            border: 4px solid #543847;
            padding: 1.5rem;
            border-radius: 8px;
            color: #543847;
            text-align: center;
            display: block;
            z-index: 20;
            pointer-events: auto;
            box-shadow: 0 10px 0 rgba(0,0,0,0.2);
            width: 90%;
            max-width: 350px;
        }
    </style>
</head>
<body>

    <div id="ui-layer">
        <div class="text-6xl font-black" id="score-display">0</div>
    </div>

    <div id="start-screen" class="message-box">
        <h1 class="text-4xl font-black mb-4 uppercase">Flappy Face</h1>
        
        <div class="mb-6">
            <p class="text-sm font-bold mb-2 uppercase">Válaszd ki a fotót:</p>
            <input type="file" id="image-upload" accept="image/*" class="text-xs block w-full text-slate-500 file:mr-4 file:py-2 file:px-4 file:rounded-full file:border-0 file:text-sm file:font-semibold file:bg-white file:text-[#543847] hover:file:bg-gray-100 cursor-pointer" />
        </div>

        <button id="start-btn" class="bg-white border-4 border-[#543847] text-[#543847] font-black py-2 px-8 rounded hover:bg-gray-100 active:scale-95 transition-all uppercase w-full">Start</button>
        <p class="mt-4 text-xs opacity-70">Szóköz vagy kattintás a repüléshez</p>
    </div>

    <div id="game-over-screen" class="message-box" style="display: none;">
        <h1 class="text-4xl font-black mb-2 text-red-600 uppercase">Vége!</h1>
        <p class="text-2xl font-bold mb-1">Pont: <span id="final-score">0</span></p>
        <p class="text-lg mb-6">Rekord: <span id="final-high-score">0</span></p>
        <button id="back-to-menu-btn" class="bg-white border-4 border-[#543847] text-[#543847] font-black py-2 px-8 rounded hover:bg-gray-100 active:scale-95 transition-all uppercase w-full">Vissza a menübe</button>
    </div>

    <canvas id="gameCanvas"></canvas>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreDisplay = document.getElementById('score-display');
        const startScreen = document.getElementById('start-screen');
        const gameOverScreen = document.getElementById('game-over-screen');
        const startBtn = document.getElementById('start-btn');
        const backMenuBtn = document.getElementById('back-to-menu-btn');
        const imageUpload = document.getElementById('image-upload');

        // Audio Setup
        let audioCtx = null;
        function playSound(type) {
            if (!audioCtx) return;
            try {
                const oscillator = audioCtx.createOscillator();
                const gainNode = audioCtx.createGain();
                oscillator.connect(gainNode);
                gainNode.connect(audioCtx.destination);
                const now = audioCtx.currentTime;
                if (type === 'jump') {
                    oscillator.type = 'triangle';
                    oscillator.frequency.setValueAtTime(250, now);
                    oscillator.frequency.exponentialRampToValueAtTime(400, now + 0.1);
                    gainNode.gain.setValueAtTime(0.1, now);
                    gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.1);
                    oscillator.start(); oscillator.stop(now + 0.1);
                } else if (type === 'score') {
                    oscillator.type = 'square';
                    oscillator.frequency.setValueAtTime(600, now);
                    oscillator.frequency.exponentialRampToValueAtTime(800, now + 0.1);
                    gainNode.gain.setValueAtTime(0.1, now);
                    gainNode.gain.exponentialRampToValueAtTime(0.01, now + 0.2);
                    oscillator.start(); oscillator.stop(now + 0.2);
                } else if (type === 'hit') {
                    oscillator.type = 'sawtooth';
                    oscillator.frequency.setValueAtTime(150, now);
                    oscillator.frequency.exponentialRampToValueAtTime(40, now + 0.3);
                    gainNode.gain.setValueAtTime(0.2, now);
                    gainNode.gain.linearRampToValueAtTime(0.01, now + 0.3);
                    oscillator.start(); oscillator.stop(now + 0.3);
                }
            } catch (e) {}
        }

        // Karakter kép kezelése
        let birdImage = new Image();
        let imageLoaded = false;

        imageUpload.addEventListener('change', function(e) {
            const file = e.target.files[0];
            if (file) {
                const reader = new FileReader();
                reader.onload = function(event) {
                    birdImage.src = event.target.result;
                    birdImage.onload = () => { imageLoaded = true; };
                };
                reader.readAsDataURL(file);
            }
        });

        let gameRunning = false;
        let score = 0;
        let highScore = localStorage.getItem('flappyHighScore') || 0;
        let groundOffset = 0;
        let frameCount = 0;

        const GRAVITY = 0.22;
        const JUMP = -4.8;
        const GROUND_HEIGHT = 80;
        const BASE_PIPE_GAP = 180;
        const MIN_PIPE_GAP = 130;
        const BASE_PIPE_SPEED = 2.5;

        let currentPipeSpeed = BASE_PIPE_SPEED;
        let currentPipeGap = BASE_PIPE_GAP;

        let bird = { x: 50, y: 0, velocity: 0, radius: 25, rotation: 0 }; 
        let pipes = [];

        function resize() {
            canvas.width = Math.min(window.innerWidth, 450);
            canvas.height = window.innerHeight;
            bird.x = canvas.width / 3;
            if (!gameRunning) bird.y = canvas.height / 2;
        }

        window.addEventListener('resize', resize);
        resize();

        function jumpAction() {
            if (!gameRunning) return;
            bird.velocity = JUMP;
            playSound('jump');
        }

        window.addEventListener('keydown', (e) => {
            if (e.code === 'Space') {
                if (gameRunning) jumpAction();
                else if (startScreen.style.display !== 'none') startGame();
            }
        });

        canvas.addEventListener('mousedown', () => { if (gameRunning) jumpAction(); });
        canvas.addEventListener('touchstart', (e) => {
            e.preventDefault();
            if (gameRunning) jumpAction();
        }, { passive: false });

        function startGame() {
            if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            if (audioCtx.state === 'suspended') audioCtx.resume();

            score = 0;
            pipes = [];
            frameCount = 0;
            currentPipeSpeed = BASE_PIPE_SPEED;
            currentPipeGap = BASE_PIPE_GAP;
            bird.y = canvas.height / 2;
            bird.velocity = 0;
            gameRunning = true;
            
            startScreen.style.display = 'none';
            gameOverScreen.style.display = 'none';
            scoreDisplay.innerText = '0';
            
            requestAnimationFrame(gameLoop);
        }

        function showMenu() {
            gameOverScreen.style.display = 'none';
            startScreen.style.display = 'block';
            draw(); // Alaphelyzet kirajzolása
        }

        startBtn.addEventListener('click', (e) => { e.stopPropagation(); startGame(); });
        backMenuBtn.addEventListener('click', (e) => { e.stopPropagation(); showMenu(); });

        function gameOver() {
            if (!gameRunning) return;
            gameRunning = false;
            playSound('hit');
            if (score > highScore) {
                highScore = score;
                localStorage.setItem('flappyHighScore', highScore);
            }
            document.getElementById('final-score').innerText = score;
            document.getElementById('final-high-score').innerText = highScore;
            gameOverScreen.style.display = 'block';
        }

        function createPipe() {
            const minH = 80;
            const maxH = canvas.height - GROUND_HEIGHT - currentPipeGap - minH;
            const topHeight = Math.floor(Math.random() * (maxH - minH + 1)) + minH;
            pipes.push({ x: canvas.width, top: topHeight, width: 60, gap: currentPipeGap, passed: false });
        }

        function drawBird() {
            ctx.save();
            ctx.translate(bird.x, bird.y);
            bird.rotation = Math.min(Math.PI / 4, Math.max(-Math.PI / 6, bird.velocity * 0.15));
            ctx.rotate(bird.rotation);

            if (imageLoaded) {
                ctx.beginPath();
                ctx.arc(0, 0, bird.radius, 0, Math.PI * 2);
                ctx.clip();
                
                // --- ASPECT FILL / COVER LOGIC ---
                // Megkeressük a rövidebb oldalt, hogy ahhoz igazítsuk a nagyítást
                const scale = Math.max(bird.radius * 2 / birdImage.width, bird.radius * 2 / birdImage.height);
                const w = birdImage.width * scale;
                const h = birdImage.height * scale;
                // Kirajzolás középre igazítva
                ctx.drawImage(birdImage, -w / 2, -h / 2, w, h);
                
                ctx.beginPath();
                ctx.arc(0, 0, bird.radius, 0, Math.PI * 2);
                ctx.strokeStyle = '#543847';
                ctx.lineWidth = 3;
                ctx.stroke();
            } else {
                ctx.fillStyle = '#facc15';
                ctx.strokeStyle = '#000';
                ctx.lineWidth = 2;
                ctx.beginPath();
                ctx.arc(0, 0, bird.radius, 0, Math.PI * 2);
                ctx.fill();
                ctx.stroke();
            }
            ctx.restore();
        }

        function drawPipes() {
            pipes.forEach(pipe => {
                ctx.fillStyle = '#73bf2e';
                ctx.strokeStyle = '#543847';
                ctx.lineWidth = 4;
                ctx.fillRect(pipe.x, 0, pipe.width, pipe.top);
                ctx.strokeRect(pipe.x, -10, pipe.width, pipe.top + 10);
                const bY = pipe.top + pipe.gap;
                const bH = canvas.height - bY - GROUND_HEIGHT;
                ctx.fillRect(pipe.x, bY, pipe.width, bH);
                ctx.strokeRect(pipe.x, bY, pipe.width, bH + 10);
                ctx.fillStyle = '#8ce03b';
                ctx.fillRect(pipe.x - 5, pipe.top - 25, pipe.width + 10, 25);
                ctx.strokeRect(pipe.x - 5, pipe.top - 25, pipe.width + 10, 25);
                ctx.fillRect(pipe.x - 5, bY, pipe.width + 10, 25);
                ctx.strokeRect(pipe.x - 5, bY, pipe.width + 10, 25);
            });
        }

        function drawGround() {
            ctx.fillStyle = '#ded895';
            ctx.fillRect(0, canvas.height - GROUND_HEIGHT, canvas.width, GROUND_HEIGHT);
            ctx.strokeStyle = '#543847';
            ctx.lineWidth = 4;
            ctx.strokeRect(-2, canvas.height - GROUND_HEIGHT, canvas.width + 4, GROUND_HEIGHT + 4);
            ctx.beginPath();
            ctx.strokeStyle = '#dfd36b';
            ctx.lineWidth = 15;
            for (let i = 0; i < canvas.width + 80; i += 40) {
                ctx.moveTo(i - (groundOffset % 40), canvas.height - GROUND_HEIGHT + 10);
                ctx.lineTo(i - 20 - (groundOffset % 40), canvas.height);
            }
            ctx.stroke();
        }

        function update() {
            if (!gameRunning) return;
            bird.velocity += GRAVITY;
            bird.y += bird.velocity;
            groundOffset += currentPipeSpeed;

            if (bird.y + bird.radius > canvas.height - GROUND_HEIGHT || bird.y - bird.radius < 0) {
                gameOver();
            }

            const dynamicSpawnRate = Math.max(60, 100 - Math.floor(score * 1.5));
            if (frameCount % dynamicSpawnRate === 0) createPipe();

            for (let i = pipes.length - 1; i >= 0; i--) {
                pipes[i].x -= currentPipeSpeed;
                if (bird.x + bird.radius - 6 > pipes[i].x && bird.x - bird.radius + 6 < pipes[i].x + pipes[i].width) {
                    if (bird.y - bird.radius + 6 < pipes[i].top || bird.y + bird.radius - 6 > pipes[i].top + pipes[i].gap) {
                        gameOver();
                    }
                }
                if (!pipes[i].passed && pipes[i].x + pipes[i].width < bird.x) {
                    score++;
                    scoreDisplay.innerText = score;
                    pipes[i].passed = true;
                    playSound('score');
                    currentPipeSpeed += 0.1; 
                    if (currentPipeGap > MIN_PIPE_GAP) currentPipeGap -= 2;
                }
                if (pipes[i].x + pipes[i].width < -20) pipes.splice(i, 1);
            }
            frameCount++;
        }

        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.fillStyle = '#4ec0ca';
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            drawPipes();
            drawGround();
            drawBird();
        }

        function gameLoop() {
            if (!gameRunning) return;
            update();
            draw();
            requestAnimationFrame(gameLoop);
        }
        draw();
    </script>
</body>
</html>

