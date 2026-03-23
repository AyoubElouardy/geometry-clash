<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Neon Dash - Juego estilo Geometry Dash</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            user-select: none;
        }

        body {
            background: linear-gradient(145deg, #0a0f1e 0%, #0c1122 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            font-family: 'Courier New', 'VT323', monospace;
            touch-action: manipulation;
        }

        .game-container {
            background: #000000aa;
            border-radius: 32px;
            padding: 20px;
            box-shadow: 0 20px 35px rgba(0, 0, 0, 0.5), inset 0 1px 0 rgba(255,255,255,0.1);
        }

        canvas {
            display: block;
            margin: 0 auto;
            border-radius: 20px;
            box-shadow: 0 0 0 3px #ffcc44, 0 0 0 6px #2a2a3a;
            cursor: pointer;
        }

        .info-panel {
            display: flex;
            justify-content: space-between;
            align-items: baseline;
            margin-top: 18px;
            gap: 20px;
            flex-wrap: wrap;
            justify-content: center;
        }

        .score-box, .best-box {
            background: #11151f;
            padding: 8px 20px;
            border-radius: 60px;
            font-weight: bold;
            font-size: 1.7rem;
            letter-spacing: 2px;
            color: #f3c26b;
            text-shadow: 0 0 5px #ffaa33;
            box-shadow: inset 0 1px 3px #00000055, 0 5px 10px rgba(0,0,0,0.3);
            font-family: monospace;
        }

        .best-box span, .score-box span {
            color: #9bb8ff;
            font-size: 1rem;
            margin-right: 8px;
        }

        button {
            background: #2a2f3f;
            border: none;
            font-family: monospace;
            font-weight: bold;
            font-size: 1.2rem;
            padding: 6px 20px;
            border-radius: 60px;
            color: #ffdd99;
            cursor: pointer;
            transition: 0.1s linear;
            box-shadow: 0 5px 0 #0f111a;
        }
        button:active {
            transform: translateY(2px);
            box-shadow: 0 2px 0 #0f111a;
        }
        .controls {
            display: flex;
            gap: 15px;
            align-items: center;
            background: #00000066;
            padding: 5px 15px;
            border-radius: 60px;
            backdrop-filter: blur(4px);
        }
        .key-hint {
            font-size: 0.8rem;
            background: #1e1f2c;
            padding: 4px 12px;
            border-radius: 30px;
            color: #ffdd99;
        }
        @media (max-width: 780px) {
            .score-box, .best-box { font-size: 1.2rem; padding: 4px 12px; }
            .key-hint { font-size: 0.7rem; }
            button { font-size: 1rem; padding: 4px 16px; }
        }
    </style>
</head>
<body>
<div>
    <div class="game-container">
        <canvas id="gameCanvas" width="1000" height="400"></canvas>
        <div class="info-panel">
            <div class="score-box"><span>⚡ SCORE</span> <span id="scoreValue">0</span></div>
            <div class="best-box"><span>🏆 BEST</span> <span id="bestValue">0</span></div>
            <div class="controls">
                <button id="resetBtn">🔁 REINICIAR</button>
                <div class="key-hint">🖱️ CLICK / ESPACIO / ↑</div>
            </div>
        </div>
    </div>
</div>

<script>
    (function(){
        // ---------- CANVAS ----------
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        // ---------- DIMENSIONES FIJAS ----------
        const W = 1000, H = 400;
        canvas.width = W;
        canvas.height = H;

        // ---------- CONSTANTES DE JUEGO (estilo Geometry Dash) ----------
        const GRAVITY = 0.8;
        const JUMP_POWER = -9.2;
        const GROUND_Y = H - 55;    // piso visual
        const PLAYER_SIZE = 28;
        
        // Obstáculos (cubos / spikes)
        const OBSTACLE_WIDTH = 26;
        const OBSTACLE_HEIGHT = 28;
        const SPIKE_HEIGHT = 24;
        
        // Velocidad de desplazamiento (aumenta con la dificultad)
        let baseSpeed = 5.8;
        let currentSpeed = baseSpeed;
        
        // Distancia entre obstáculos (generación)
        const MIN_GAP = 280;
        const MAX_GAP = 430;
        
        // ---------- VARIABLES DE JUEGO ----------
        let gameRunning = true;
        let score = 0;            // puntuación = distancia / 10 (aproximadamente)
        let bestScore = 0;
        
        // Jugador
        let player = {
            x: 130,              // fijo en X, scrolling de obstáculos
            y: GROUND_Y - PLAYER_SIZE,
            vy: 0,
            width: PLAYER_SIZE,
            height: PLAYER_SIZE,
            grounded: true
        };
        
        // Lista de obstáculos
        let obstacles = [];
        
        // Timer de generación (frames)
        let frameCounter = 0;
        let nextObstacleGap = 90;   // frames hasta generar nuevo obstáculo
        
        // Efecto de destello al morir
        let deathFlash = 0;
        
        // Para ground detection (collision suave)
        function updateGroundCollision() {
            // límite inferior: si está por debajo del suelo, muere
            if(player.y + player.height > GROUND_Y) {
                if(gameRunning) {
                    gameOver();
                }
                player.y = GROUND_Y - player.height;
                player.vy = 0;
                player.grounded = true;
            } else {
                player.grounded = false;
            }
            // límite superior (techo)
            if(player.y < 0) {
                player.y = 0;
                if(player.vy < 0) player.vy = 0;
            }
        }
        
        // Reiniciar juego completo
        function resetGame() {
            gameRunning = true;
            score = 0;
            updateScoreUI();
            currentSpeed = baseSpeed;
            player.y = GROUND_Y - PLAYER_SIZE;
            player.vy = 0;
            player.grounded = true;
            obstacles = [];
            frameCounter = 0;
            nextObstacleGap = 40;  // primer obstáculo rápido
            deathFlash = 0;
        }
        
        // Game Over
        function gameOver() {
            if(!gameRunning) return;
            gameRunning = false;
            deathFlash = 15;   // frames de flash rojo
            
            // Actualizar mejor puntuación
            let currentPoints = Math.floor(score);
            if(currentPoints > bestScore) {
                bestScore = currentPoints;
                localStorage.setItem('neondash_best', bestScore);
                document.getElementById('bestValue').innerText = bestScore;
            }
        }
        
        // Saltar (solo si el juego está activo)
        function jump() {
            if(!gameRunning) return;
            // Sólo puede saltar si está en el suelo (grounded)
            if(player.grounded) {
                player.vy = JUMP_POWER;
                player.grounded = false;
                // pequeño efecto sonoro simulado (beep sutil con web audio opcional? pero no necesario)
            }
        }
        
        // Generar nuevo obstáculo (aleatorio entre spike o bloque)
        function generateObstacle() {
            const type = Math.random() < 0.6 ? 'block' : 'spike';  // 60% bloque, 40% pico
            let obsX = W;
            let obsY;
            let width = OBSTACLE_WIDTH;
            let height;
            if(type === 'block') {
                height = OBSTACLE_HEIGHT;
                obsY = GROUND_Y - height;
            } else { // spike: forma triangular, pero usaremos un rectángulo puntiagudo (dibujo)
                height = SPIKE_HEIGHT;
                obsY = GROUND_Y - height;
            }
            obstacles.push({
                x: obsX,
                y: obsY,
                width: width,
                height: height,
                type: type
            });
        }
        
        // Actualizar lógica (movimiento, colisiones, generación)
        function updateGame() {
            if(!gameRunning) return;
            
            // Aumentar dificultad gradual con la puntuación (velocidad)
            let scoreInt = Math.floor(score);
            let speedBonus = Math.min(3.2, Math.floor(scoreInt / 220) * 0.35);
            currentSpeed = baseSpeed + speedBonus;
            
            // 1. Física del jugador
            player.vy += GRAVITY;
            player.y += player.vy;
            updateGroundCollision();
            if(!gameRunning) return; // si murió por salirse del suelo
            
            // 2. Mover obstáculos y eliminar los que salen de pantalla
            for(let i=0; i<obstacles.length; i++) {
                obstacles[i].x -= currentSpeed;
            }
            obstacles = obstacles.filter(obs => obs.x + obs.width > 0);
            
            // 3. Colisiones (AABB)
            const playerRect = {
                x: player.x,
                y: player.y,
                w: player.width,
                h: player.height
            };
            
            for(let obs of obstacles) {
                const obsRect = {
                    x: obs.x,
                    y: obs.y,
                    w: obs.width,
                    h: obs.height
                };
                if(playerRect.x < obsRect.x + obsRect.w &&
                   playerRect.x + playerRect.w > obsRect.x &&
                   playerRect.y < obsRect.y + obsRect.h &&
                   playerRect.y + playerRect.h > obsRect.y) {
                    gameOver();
                    return;
                }
            }
            
            // 4. Generación de obstáculos con espaciado variable (evitar acumulación)
            if(frameCounter >= nextObstacleGap) {
                generateObstacle();
                // Siguiente intervalo aleatorio (en frames, teniendo en cuenta velocidad)
                let gapFrames = Math.floor( (MIN_GAP + Math.random() * (MAX_GAP - MIN_GAP)) / currentSpeed );
                gapFrames = Math.max(45, Math.min(95, gapFrames));
                nextObstacleGap = gapFrames;
                frameCounter = 0;
            } else {
                frameCounter++;
            }
            
            // 5. Aumentar puntuación (distancia recorrida) + bonus por frame
            // Cada frame sobrevivido da 0.2 puntos, más fluido
            score += 0.2 + (currentSpeed * 0.03);
            updateScoreUI();
        }
        
        // Actualizar UI de puntuación
        function updateScoreUI() {
            let displayScore = Math.floor(score);
            document.getElementById('scoreValue').innerText = displayScore;
            // si supera best local, actualizar visualmente pero no guardar hasta game over
            if(displayScore > bestScore && !gameRunning === false) {
                // solo visual en tiempo real pero el mejor se actualiza al morir
                document.getElementById('bestValue').innerText = displayScore;
            } else {
                // mantener best real
                document.getElementById('bestValue').innerText = bestScore;
            }
        }
        
        // ---------- DIBUJADO ESTILO GEOMETRY DASH (neon, efectos) ----------
        function draw() {
            ctx.clearRect(0, 0, W, H);
            
            // Fondo gradiente dinámico
            let grad = ctx.createLinearGradient(0, 0, 0, H);
            grad.addColorStop(0, '#0c1126');
            grad.addColorStop(0.7, '#1a1f32');
            ctx.fillStyle = grad;
            ctx.fillRect(0, 0, W, H);
            
            // Patrón de rejilla al estilo dash (cuadrícula)
            ctx.strokeStyle = '#2f3b6e';
            ctx.lineWidth = 1;
            for(let i = 0; i < W; i += 40) {
                ctx.beginPath();
                ctx.moveTo(i, 0);
                ctx.lineTo(i, H);
                ctx.stroke();
            }
            for(let i = 0; i < H; i += 40) {
                ctx.beginPath();
                ctx.moveTo(0, i);
                ctx.lineTo(W, i);
                ctx.stroke();
            }
            
            // Dibujar piso (con brillo)
            ctx.fillStyle = '#2b2f45';
            ctx.fillRect(0, GROUND_Y, W, H - GROUND_Y + 5);
            ctx.fillStyle = '#ffc857';
            ctx.fillRect(0, GROUND_Y - 4, W, 4);
            ctx.fillStyle = '#e0a800';
            ctx.fillRect(0, GROUND_Y - 2, W, 2);
            
            // Sombras del suelo
            ctx.fillStyle = '#00000055';
            ctx.fillRect(0, GROUND_Y - 8, W, 8);
            
            // ---------- OBSTÁCULOS (Bloques y picos) ----------
            for(let obs of obstacles) {
                if(obs.type === 'block') {
                    // bloque con brillo neón
                    ctx.shadowBlur = 8;
                    ctx.shadowColor = '#ff3366';
                    ctx.fillStyle = '#e34d6e';
                    ctx.fillRect(obs.x, obs.y, obs.width, obs.height);
                    ctx.fillStyle = '#ff90aa';
                    ctx.fillRect(obs.x+3, obs.y+3, obs.width-6, obs.height-6);
                    ctx.fillStyle = '#b32d4a';
                    ctx.fillRect(obs.x+1, obs.y+1, obs.width-2, 4);
                    ctx.shadowBlur = 0;
                } 
                else { // spike (triángulo invertido)
                    ctx.beginPath();
                    ctx.moveTo(obs.x, obs.y + obs.height);
                    ctx.lineTo(obs.x + obs.width/2, obs.y);
                    ctx.lineTo(obs.x + obs.width, obs.y + obs.height);
                    ctx.closePath();
                    ctx.fillStyle = '#f0a34b';
                    ctx.shadowBlur = 6;
                    ctx.shadowColor = '#ff7700';
                    ctx.fill();
                    ctx.fillStyle = '#ffcc77';
                    ctx.beginPath();
                    ctx.moveTo(obs.x+3, obs.y+obs.height-4);
                    ctx.lineTo(obs.x+obs.width/2, obs.y+6);
                    ctx.lineTo(obs.x+obs.width-3, obs.y+obs.height-4);
                    ctx.fill();
                    ctx.shadowBlur = 0;
                }
            }
            
            // ---------- JUGADOR (cuadrado con carita tipo dash) ----------
            ctx.shadowBlur = 10;
            ctx.shadowColor = '#3ec1ff';
            // cuerpo principal
            ctx.fillStyle = '#3cc7ff';
            ctx.fillRect(player.x, player.y, player.width, player.height);
            ctx.fillStyle = '#ffffff';
            ctx.fillRect(player.x+5, player.y+5, 6, 6);
            ctx.fillRect(player.x+player.width-11, player.y+5, 6, 6);
            // pupilas
            ctx.fillStyle = '#001133';
            ctx.fillRect(player.x+6, player.y+7, 3, 3);
            ctx.fillRect(player.x+player.width-10, player.y+7, 3, 3);
            // sonrisa dinámica según velocidad/salto
            ctx.beginPath();
            let smileY = player.y + 18;
            if(player.vy < -2) smileY = player.y + 14;
            ctx.arc(player.x+player.width/2, smileY, 8, 0.05, Math.PI - 0.05);
            ctx.strokeStyle = '#ffdd88';
            ctx.lineWidth = 2;
            ctx.stroke();
            // efecto de salto (estela)
            if(!player.grounded) {
                ctx.fillStyle = '#ffffff88';
                ctx.beginPath();
                ctx.ellipse(player.x+player.width/2, player.y+player.height-5, 8, 4, 0, 0, Math.PI*2);
                ctx.fill();
            }
            ctx.shadowBlur = 0;
            
            // Efecto de partículas al correr (simple)
            if(gameRunning && (Math.floor(Date.now()/80) % 3 === 0)) {
                ctx.fillStyle = '#ffcc88';
                ctx.beginPath();
                ctx.arc(player.x-6, GROUND_Y-8, 3, 0, Math.PI*2);
                ctx.fill();
                ctx.fillStyle = '#ffaa55';
                ctx.beginPath();
                ctx.arc(player.x-12, GROUND_Y-12, 2, 0, Math.PI*2);
                ctx.fill();
            }
            
            // destello de muerte
            if(deathFlash > 0) {
                ctx.fillStyle = 'rgba(255, 60, 60, 0.7)';
                ctx.fillRect(0, 0, W, H);
                deathFlash--;
            }
            
            // Texto GAME OVER si está muerto
            if(!gameRunning) {
                ctx.font = 'bold 42px "Courier New", monospace';
                ctx.shadowBlur = 0;
                ctx.fillStyle = '#ff2266';
                ctx.shadowColor = '#aa0044';
                ctx.fillText('GAME OVER', W/2-150, H/2-40);
                ctx.font = '22px monospace';
                ctx.fillStyle = '#ffcc99';
                ctx.fillText('presiona REINICIAR o ESPACIO', W/2-175, H/2+35);
            }
            
            // Instrucción flotante
            ctx.font = 'bold 14px monospace';
            ctx.fillStyle = '#aab9ff';
            ctx.fillText('▲ CLICK / ESPACIO para saltar', W-210, 35);
        }
        
        // ----- Bucle principal de animación -----
        let lastTimestamp = 0;
        function gameLoop() {
            updateGame();
            draw();
            requestAnimationFrame(gameLoop);
        }
        
        // ----- Gestión de eventos táctiles y teclado -----
        function handleJump(e) {
            // Evita comportamientos por defecto como scroll con espacio o flecha arriba
            if(e.type === 'keydown') {
                if(e.code === 'Space' || e.code === 'ArrowUp') {
                    e.preventDefault();
                    jump();
                }
                // Tecla R para reiniciar rápido
                if(e.code === 'KeyR') {
                    e.preventDefault();
                    resetGame();
                }
            } else if(e.type === 'click' || e.type === 'touchstart') {
                // Click en canvas o botón? pero evitamos que el reset haga doble acción
                let target = e.target;
                if(target.id !== 'resetBtn' && target.tagName !== 'BUTTON') {
                    e.preventDefault();
                    jump();
                }
            }
        }
        
        // Touch para móvil (eventos táctiles)
        function handleTouch(e) {
            e.preventDefault();
            // Verificar si no tocó el botón reiniciar
            let rect = canvas.getBoundingClientRect();
            let touch = e.touches[0];
            let x = touch.clientX;
            let y = touch.clientY;
            let resetRect = document.getElementById('resetBtn').getBoundingClientRect();
            if(!(x >= resetRect.left && x <= resetRect.right && y >= resetRect.top && y <= resetRect.bottom)) {
                jump();
            }
        }
        
        // ---------- INICIALIZACIÓN Y EVENTOS ----------
        function loadBestFromStorage() {
            let stored = localStorage.getItem('neondash_best');
            if(stored !== null && !isNaN(parseInt(stored))) {
                bestScore = parseInt(stored);
            } else {
                bestScore = 0;
            }
            document.getElementById('bestValue').innerText = bestScore;
        }
        
        // Configurar listeners
        function bindEvents() {
            window.addEventListener('keydown', handleJump);
            canvas.addEventListener('click', (e) => {
                // si el click está en canvas, saltar (pero no si es en botón)
                if(e.target === canvas) jump();
            });
            canvas.addEventListener('touchstart', handleTouch, {passive: false});
            const resetBtn = document.getElementById('resetBtn');
            resetBtn.addEventListener('click', (e) => {
                e.stopPropagation();
                resetGame();
            });
            // Evitar el menú contextual en canvas
            canvas.addEventListener('contextmenu', (e) => e.preventDefault());
        }
        
        // Pequeño efecto de inicio
        function init() {
            loadBestFromStorage();
            resetGame();  // estado inicial listo
            bindEvents();
            gameLoop();
        }
        
        init();
    })();
</script>
</body>
</html>
