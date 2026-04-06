<!DOCTYPE html>
<html>
<head>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Snake Mobile Pro</title>
    <style>
        body {
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
            background-color: #1a1a1a;
            font-family: 'Segoe UI', sans-serif;
            overflow: hidden;
            touch-action: none;
        }
        #ui { color: #00ff00; margin-bottom: 5px; font-size: 24px; font-weight: bold; }
        #game-container { position: relative; border: 5px solid #333; border-radius: 10px; }
        canvas { background-color: #000; display: block; border-radius: 5px; }
        
        /* Joystick Controls */
        #controls {
            margin-top: 20px;
            display: grid;
            grid-template-areas: 
                ". up ."
                "left . right"
                ". down .";
            gap: 15px;
        }
        .btn {
            width: 70px;
            height: 70px;
            background: #444;
            border: none;
            border-radius: 50%;
            color: white;
            font-size: 25px;
            box-shadow: 0 4px #222;
        }
        .btn:active { background: #00ff00; box-shadow: 0 0px #222; transform: translateY(4px); }
        #up { grid-area: up; } #left { grid-area: left; } #right { grid-area: right; } #down { grid-area: down; }

        /* Overlays */
        #overlay {
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.8);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            z-index: 10;
            border-radius: 5px;
        }
        button#start-btn {
            padding: 15px 30px;
            font-size: 20px;
            background: #00ff00;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-weight: bold;
        }
    </style>
</head>
<body>

    <div id="ui">SCORE: <span id="score">0</span></div>

    <div id="game-container">
        <canvas id="snakeGame" width="300" height="300"></canvas>
        <div id="overlay">
            <button id="start-btn" onclick="startGame()">START GAME</button>
        </div>
    </div>

    <div id="controls">
        <button class="btn" id="up">↑</button>
        <button class="btn" id="left">←</button>
        <button class="btn" id="right">→</button>
        <button class="btn" id="down">↓</button>
    </div>

    <script>
        const canvas = document.getElementById("snakeGame");
        const ctx = canvas.getContext("2d");
        const scoreElement = document.getElementById("score");
        const overlay = document.getElementById("overlay");
        const startBtn = document.getElementById("start-btn");

        const box = 20;
        let snake, food, d, score, gameInterval;
        
        // --- SOUND ENGINE ---
        const audioCtx = new (window.AudioContext || window.webkitAudioContext)();

        function playSound(freq, type, duration) {
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.type = type;
            osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
            gain.gain.setValueAtTime(0.1, audioCtx.currentTime);
            gain.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime + duration);
            osc.connect(gain);
            gain.connect(audioCtx.destination);
            osc.start();
            osc.stop(audioCtx.currentTime + duration);
        }

        function playEatSound() { playSound(600, 'square', 0.1); }
        function playGameOverSound() { 
            playSound(300, 'sawtooth', 0.3);
            setTimeout(() => playSound(200, 'sawtooth', 0.5), 200);
        }

        // --- GAME LOGIC ---
        function startGame() {
            if (audioCtx.state === 'suspended') audioCtx.resume();
            snake = [{ x: 8 * box, y: 10 * box }];
            spawnFood();
            score = 0;
            d = "RIGHT";
            scoreElement.innerHTML = score;
            overlay.style.display = "none";
            if(gameInterval) clearInterval(gameInterval);
            gameInterval = setInterval(draw, 150);
        }

        function spawnFood() {
            food = {
                x: Math.floor(Math.random() * 14 + 0.5) * box,
                y: Math.floor(Math.random() * 14 + 0.5) * box
            };
        }

        // Joystick Inputs
        document.getElementById("up").onclick = () => { if(d != "DOWN") d = "UP"; };
        document.getElementById("down").onclick = () => { if(d != "UP") d = "DOWN"; };
        document.getElementById("left").onclick = () => { if(d != "RIGHT") d = "LEFT"; };
        document.getElementById("right").onclick = () => { if(d != "LEFT") d = "RIGHT"; };

        function draw() {
            ctx.fillStyle = "black";
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Draw Snake
            for (let i = 0; i < snake.length; i++) {
                ctx.fillStyle = (i == 0) ? "#00ff00" : "#008800";
                ctx.fillRect(snake[i].x, snake[i].y, box-2, box-2);
            }

            // Draw Fruit
            ctx.fillStyle = "#ff0000";
            ctx.beginPath();
            ctx.arc(food.x + box/2, food.y + box/2, box/2 - 2, 0, Math.PI*2);
            ctx.fill();

            let snakeX = snake[0].x;
            let snakeY = snake[0].y;

            if (d == "LEFT") snakeX -= box;
            if (d == "UP") snakeY -= box;
            if (d == "RIGHT") snakeX += box;
            if (d == "DOWN") snakeY += box;

            // Score Logic
            if (snakeX == food.x && snakeY == food.y) {
                score++;
                scoreElement.innerHTML = score;
                playEatSound();
                spawnFood();
            } else {
                snake.pop();
            }

            let newHead = { x: snakeX, y: snakeY };

            // Death Logic
            if (snakeX < 0 || snakeX >= canvas.width || snakeY < 0 || snakeY >= canvas.height || collision(newHead, snake)) {
                clearInterval(gameInterval);
                playGameOverSound();
                startBtn.innerHTML = "RESTART GAME";
                overlay.style.display = "flex";
                return;
            }

            snake.unshift(newHead);
        }

        function collision(head, array) {
            for (let i = 0; i < array.length; i++) {
                if (head.x == array[i].x && head.y == array[i].y) return true;
            }
            return false;
        }
    </script>
</body>
</html>
