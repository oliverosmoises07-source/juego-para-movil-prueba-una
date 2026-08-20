# juego-para-movil-prueba-una
juego para movil hecho por seriousms
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Super Plataformas - Móvil y PC</title>
    <style>
        body {
            background-color: #0b0b0b;
            color: white;
            font-family: 'Courier New', Courier, monospace;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
            overflow: hidden;
            touch-action: none;
        }
        canvas {
            border: 4px solid #333;
            box-shadow: 0px 0px 30px rgba(0, 150, 255, 0.3);
            border-radius: 4px;
            max-width: 100%;
            height: auto;
        }
        .info {
            margin-bottom: 5px;
            font-size: 16px;
            font-weight: bold;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.7);
            text-align: center;
        }
        #audioPrompt {
            position: absolute;
            top: 10px;
            background: rgba(0,0,0,0.85);
            padding: 8px 16px;
            border: 2px solid #2ecc71;
            border-radius: 8px;
            cursor: pointer;
            font-size: 14px;
            font-weight: bold;
            z-index: 10;
        }
        /* Controles táctiles para celular */
        .touch-controls {
            display: flex;
            justify-content: space-between;
            width: 800px;
            max-width: 100%;
            padding: 10px 20px;
            box-sizing: border-box;
            margin-top: 10px;
        }
        .control-group {
            display: flex;
            gap: 15px;
        }
        .t-btn {
            width: 70px;
            height: 70px;
            background: rgba(255, 255, 255, 0.2);
            border: 2px solid rgba(255, 255, 255, 0.5);
            border-radius: 50%;
            font-size: 24px;
            color: white;
            display: flex;
            align-items: center;
            justify-content: center;
            user-select: none;
            outline: none;
            -webkit-tap-highlight-color: transparent;
        }
        .t-btn:active {
            background: rgba(255, 255, 255, 0.5);
        }
    </style>
</head>
<body>

    <div id="audioPrompt" onclick="initAudio()">🔊 Activar Sonido y Música</div>
    <div class="info">¡Usa las flechas o los botones táctiles para moverte y saltar!</div>
    <div id="stats" class="info">Monedas: 0/10 | Vidas: 3</div>
    
    <canvas id="gameCanvas" width="800" height="400"></canvas>

    <!-- Botones en pantalla para móviles -->
    <div class="touch-controls">
        <div class="control-group">
            <div class="t-btn" id="btnLeft">←</div>
            <div class="t-btn" id="btnRight">→</div>
        </div>
        <div class="control-group">
            <div class="t-btn" id="btnJump">↑</div>
        </div>
    </div>

    <script>
        const canvas = document.getElementById("gameCanvas");
        const ctx = canvas.getContext("2d");

        let score = 0;
        let lives = 3;
        let gameState = "playing"; 
        let globalTime = 0;

        // --- AUDIO ---
        let audioCtx = null;
        let musicInterval = null;

        function initAudio() {
            if (!audioCtx) {
                audioCtx = new (window.AudioContext || window.webkitAudioContext)();
                document.getElementById('audioPrompt').style.display = 'none';
                startBackgroundMusic();
            }
        }

        function playSound(type) {
            if (!audioCtx) return;
            let osc = audioCtx.createOscillator();
            let gain = audioCtx.createGain();
            osc.connect(gain);
            gain.connect(audioCtx.destination);
            let now = audioCtx.currentTime;

            if (type === 'jump') {
                osc.type = 'square';
                osc.frequency.setValueAtTime(150, now);
                osc.frequency.exponentialRampToValueAtTime(450, now + 0.15);
                gain.gain.setValueAtTime(0.1, now);
                gain.gain.linearRampToValueAtTime(0.01, now + 0.15);
                osc.start(now);
                osc.stop(now + 0.15);
            } else if (type === 'stomp') {
                osc.type = 'sawtooth';
                osc.frequency.setValueAtTime(120, now);
                osc.frequency.linearRampToValueAtTime(40, now + 0.2);
                gain.gain.setValueAtTime(0.2, now);
                gain.gain.linearRampToValueAtTime(0.01, now + 0.2);
                osc.start(now);
                osc.stop(now + 0.2);
            } else if (type === 'coin') {
                osc.type = 'sine';
                osc.frequency.setValueAtTime(987.77, now);
                osc.frequency.setValueAtTime(1318.51, now + 0.08);
                gain.gain.setValueAtTime(0.15, now);
                gain.gain.linearRampToValueAtTime(0.01, now + 0.25);
                osc.start(now);
                osc.stop(now + 0.25);
            }
        }

        function startBackgroundMusic() {
            if (musicInterval) clearInterval(musicInterval);
            const notes = [261.63, 329.63, 392.00, 523.25, 392.00, 329.63];
            let index = 0;
            musicInterval = setInterval(() => {
                if (!audioCtx || gameState !== "playing") return;
                let osc = audioCtx.createOscillator();
                let gain = audioCtx.createGain();
                osc.connect(gain);
                gain.connect(audioCtx.destination);
                let now = audioCtx.currentTime;
                osc.type = 'triangle';
                osc.frequency.setValueAtTime(notes[index], now);
                gain.gain.setValueAtTime(0.04, now);
                gain.gain.linearRampToValueAtTime(0.001, now + 0.2);
                osc.start(now);
                osc.stop(now + 0.2);
                index = (index + 1) % notes.length;
            }, 250);
        }

        // --- JUGADOR Y CONTROLES ---
        let player = {
            x: 50, y: 200, width: 32, height: 40,
            vx: 0, vy: 0, speed: 4.8, jumpPower: -12,
            grounded: false, facing: "right", animFrame: 0
        };

        const gravity = 0.55;
        let keys = {};
        let particles = [];

        // Teclado PC
        window.addEventListener("keydown", e => keys[e.code] = true);
        window.addEventListener("keyup", e => keys[e.code] = false);

        // Controles táctiles móviles
        function setupTouchButton(id, code) {
            const btn = document.getElementById(id);
            btn.addEventListener("touchstart", (e) => { e.preventDefault(); keys[code] = true; initAudio(); });
            btn.addEventListener("touchend", (e) => { e.preventDefault(); keys[code] = false; });
            btn.addEventListener("mousedown", () => { keys[code] = true; initAudio(); });
            btn.addEventListener("mouseup", () => { keys[code] = false; });
        }
        setupTouchButton("btnLeft", "ArrowLeft");
        setupTouchButton("btnRight", "ArrowRight");
        setupTouchButton("btnJump", "ArrowUp");

        // --- NIVEL ---
        let platforms = [
            { x: 0, y: 350, width: 600, height: 50, type: "ground" },
            { x: 250, y: 260, width: 120, height: 24, type: "brick" },
            { x: 420, y: 190, width: 120, height: 24, type: "brick" },
            { x: 750, y: 350, width: 700, height: 50, type: "ground" },
            { x: 620, y: 240, width: 90, height: 24, type: "brick" },
            { x: 800, y: 200, width: 130, height: 24, type: "brick" },
            { x: 1000, y: 140, width: 140, height: 24, type: "brick" },
            { x: 1550, y: 350, width: 800, height: 50, type: "ground" },
            { x: 1250, y: 220, width: 120, height: 24, type: "brick" },
            { x: 1420, y: 170, width: 100, height: 24, type: "brick" },
            { x: 1700, y: 250, width: 130, height: 24, type: "brick" }
        ];

        let coins = [
            { x: 290, y: 215, width: 18, height: 18, collected: false },
            { x: 470, y: 145, width: 18, height: 18, collected: false },
            { x: 650, y: 195, width: 18, height: 18, collected: false },
            { x: 850, y: 155, width: 18, height: 18, collected: false },
            { x: 1050, y: 95, width: 18, height: 18, collected: false },
            { x: 1120, y: 310, width: 18, height: 18, collected: false },
            { x: 1300, y: 175, width: 18, height: 18, collected: false },
            { x: 1460, y: 125, width: 18, height: 18, collected: false },
            { x: 1750, y: 205, width: 18, height: 18, collected: false },
            { x: 1900, y: 310, width: 18, height: 18, collected: false }
        ];

        let enemies = [
            { x: 350, y: 323, width: 30, height: 27, vx: 1.3, minX: 100, maxX: 550, alive: true, squishedTimer: 0 },
            { x: 850, y: 323, width: 30, height: 27, vx: 1.6, minX: 780, maxX: 1350, alive: true, squishedTimer: 0 },
            { x: 1020, y: 113, width: 30, height: 27, vx: 1.0, minX: 1000, maxX: 1120, alive: true, squishedTimer: 0 },
            { x: 1650, y: 323, width: 30, height: 27, vx: 1.8, minX: 1600, maxX: 2100, alive: true, squishedTimer: 0 }
        ];

        let flag = { x: 2200, y: 110, width: 16, height: 240 };
        let cameraX = 0;

        function addParticle(x, y, color) {
            for (let i = 0; i < 6; i++) {
                particles.push({
                    x: x, y: y,
                    vx: (Math.random() - 0.5) * 5, vy: (Math.random() - 0.5) * 5,
                    size: Math.random() * 5 + 2, color: color, life: 25
                });
            }
        }

        function resetPlayer() {
            player.x = 50; player.y = 200; player.vx = 0; player.vy = 0;
        }

        function update() {
            globalTime++;
            if (gameState !== "playing") {
                if (keys["KeyR"] || keys["Space"]) restartGame();
                return;
            }

            if (keys["ArrowLeft"] || keys["KeyA"]) {
                player.vx = -player.speed;
                player.facing = "left";
                player.animFrame += 0.2;
            } else if (keys["ArrowRight"] || keys["KeyD"]) {
                player.vx = player.speed;
                player.facing = "right";
                player.animFrame += 0.2;
            } else {
                player.vx = 0;
                player.animFrame = 0;
            }

            if ((keys["ArrowUp"] || keys["KeyW"] || keys["Space"]) && player.grounded) {
                player.vy = player.jumpPower;
                player.grounded = false;
                playSound('jump');
                addParticle(player.x + 16, player.y + 40, "#c88c00");
            }

            player.vy += gravity;
            player.x += player.vx;
            player.y += player.vy;

            player.grounded = false;
            platforms.forEach(p => {
                if (
                    player.x < p.x + p.width && player.x + player.width > p.x &&
                    player.y + player.height >= p.y && player.y + player.height - player.vy <= p.y + 8 &&
                    player.vy >= 0
                ) {
                    player.y = p.y - player.height;
                    player.vy = 0;
                    player.grounded = true;
                }
            });

            coins.forEach(c => {
                if (!c.collected &&
                    player.x < c.x + c.width && player.x + player.width > c.x &&
                    player.y < c.y + c.height && player.y + player.height > c.y) {
                    c.collected = true;
                    score++;
                    playSound('coin');
                    addParticle(c.x + 9, c.y + 9, "#ffd700");
                }
            });

            enemies.forEach(e => {
                if (!e.alive) {
                    if (e.squishedTimer > 0) e.squishedTimer--;
                    return;
                }
                e.x += e.vx;
                if (e.x < e.minX || e.x > e.maxX) e.vx *= -1;

                if (
                    player.x < e.x + e.width && player.x + player.width > e.x &&
                    player.y < e.y + e.height && player.y + player.height > e.y
                ) {
                    if (player.vy > 0 && player.y + player.height - player.vy <= e.y + 12) {
                        e.alive = false;
                        e.squishedTimer = 30;
                        player.vy = -8;
                        score += 2;
                        playSound('stomp');
                        addParticle(e.x + 15, e.y + 15, "#8b4513");
                    } else {
                        lives--;
                        if (lives <= 0) gameState = "gameover";
                        else resetPlayer();
                    }
                }
            });

            if (player.y > canvas.height) {
                lives--;
                if (lives <= 0) gameState = "gameover";
                else resetPlayer();
            }

            if (player.x + player.width >= flag.x) gameState = "win";

            particles.forEach((p, index) => {
                p.x += p.vx; p.y += p.vy; p.life--;
                if (p.life <= 0) particles.splice(index, 1);
            });

            cameraX = player.x - 220;
            if (cameraX < 0) cameraX = 0;

            document.getElementById("stats").innerText = `Monedas: ${score}/10 | Vidas: ${lives}`;
        }

        function draw() {
            let bgGrad = ctx.createLinearGradient(0, 0, 0, canvas.height);
            bgGrad.addColorStop(0, "#1a252f");
            bgGrad.addColorStop(0.5, "#2980b9");
            bgGrad.addColorStop(1, "#a2d9ce");
            ctx.fillStyle = bgGrad;
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            ctx.fillStyle = "rgba(255, 255, 255, 0.12)";
            let mountainOffset = cameraX * 0.3;
            ctx.beginPath();
            ctx.moveTo(-mountainOffset, 350);
            ctx.lineTo(200 - mountainOffset, 180);
            ctx.lineTo(500 - mountainOffset, 350);
            ctx.lineTo(800 - mountainOffset, 150);
            ctx.lineTo(1200 - mountainOffset, 350);
            ctx.lineTo(1600 - mountainOffset, 170);
            ctx.lineTo(2100 - mountainOffset, 350);
            ctx.lineTo(2500 - mountainOffset, 200);
            ctx.lineTo(2800 - mountainOffset, 350);
            ctx.fill();

            ctx.save();
            ctx.translate(-cameraX, 0);

            platforms.forEach(p => {
                if (p.type === "ground") {
                    ctx.fillStyle = "#1e8449";
                    ctx.fillRect(p.x, p.y, p.width, 12);
                    let groundGrad = ctx.createLinearGradient(0, p.y + 12, 0, p.y + p.height);
                    groundGrad.addColorStop(0, "#78281f");
                    groundGrad.addColorStop(1, "#4a235a");
                    ctx.fillStyle = groundGrad;
                    ctx.fillRect(p.x, p.y + 12, p.width, p.height - 12);
                } else {
                    ctx.fillStyle = "#b03a2e";
                    ctx.fillRect(p.x, p.y, p.width, p.height);
                    ctx.fillStyle = "#e74c3c";
                    ctx.fillRect(p.x, p.y, p.width, 4);
                    ctx.fillStyle = "#641e16";
                    ctx.fillRect(p.x, p.y + p.height - 4, p.width, 4);
                }
            });

            coins.forEach(c => {
                if (!c.collected) {
                    let scaleX = Math.abs(Math.sin(globalTime * 0.12)) * 0.7 + 0.3;
                    ctx.save();
                    ctx.translate(c.x + c.width / 2, c.y + c.height / 2);
                    ctx.scale(scaleX, 1);
                    ctx.fillStyle = "#f1c40f";
                    ctx.beginPath();
                    ctx.arc(0, 0, c.width / 2, 0, Math.PI * 2);
                    ctx.fill();
                    ctx.strokeStyle = "#9a7d0a";
                    ctx.lineWidth = 1.5;
                    ctx.stroke();
                    ctx.restore();
                }
            });

            enemies.forEach(e => {
                if (e.alive) {
                    let legOffset = Math.sin(globalTime * 0.3) * 3;
                    ctx.fillStyle = "#a0522d";
                    ctx.beginPath();
                    ctx.arc(e.x + e.width / 2, e.y + 12, 14, Math.PI, 0, false);
                    ctx.fill();
                    ctx.fillRect(e.x + 1, e.y + 12, e.width - 2, 12);

                    ctx.fillStyle = "#fff";
                    ctx.fillRect(e.x + 5, e.y + 8, 6, 8);
                    ctx.fillRect(e.x + 17, e.y + 8, 6, 8);
                    ctx.fillStyle = "#000";
                    ctx.fillRect(e.x + 8, e.y + 10, 2, 4);
                    ctx.fillRect(e.x + 20, e.y + 10, 2, 4);

                    ctx.fillStyle = "#111";
                    ctx.fillRect(e.x + 2, e.y + 22 + legOffset, 10, 5);
                    ctx.fillRect(e.x + 18, e.y + 22 - legOffset, 10, 5);
                } else if (e.squishedTimer > 0) {
                    ctx.fillStyle = "#6e2c00";
                    ctx.fillRect(e.x, e.y + 18, e.width, 9);
                }
            });

            ctx.fillStyle = "#bdc3c7";
            ctx.fillRect(flag.x + 6, flag.y, 4, flag.height);
            ctx.fillStyle = "#f39c12";
            ctx.beginPath();
            ctx.arc(flag.x + 8, flag.y, 6, 0, Math.PI * 2);
            ctx.fill();

            let wave = Math.sin(globalTime * 0.2) * 5;
            ctx.fillStyle = "#27ae60";
            ctx.beginPath();
            ctx.moveTo(flag.x + 10, flag.y + 10);
            ctx.lineTo(flag.x + 60, flag.y + 25 + wave);
            ctx.lineTo(flag.x + 10, flag.y + 50);
            ctx.fill();

            let px = player.x;
            let py = player.y;

            ctx.fillStyle = "#3e2723";
            ctx.fillRect(px + 3, py + 33, 10, 7);
            ctx.fillRect(px + 19, py + 33, 10, 7);
            ctx.fillStyle = "#2980b9";
            ctx.fillRect(px + 6, py + 22, 20, 12);
            ctx.fillStyle = "#e74c3c";
            ctx.fillRect(px + 5, py + 12, 22, 11);
            ctx.fillStyle = "#f1c40f";
            ctx.fillRect(px + 8, py + 24, 3, 3);
            ctx.fillRect(px + 21, py + 24, 3, 3);
            ctx.fillStyle = "#ffeaa7";
            ctx.fillRect(px + 7, py + 4, 18, 12);
            ctx.fillStyle = "#e74c3c";
            ctx.fillRect(px + 5, py, 20, 6);
            if (player.facing === "right") ctx.fillRect(px + 21, py + 3, 7, 4);
            else ctx.fillRect(px + 4, py + 3, 7, 4);
            ctx.fillStyle = "#2c3e50";
            ctx.fillRect(px + (player.facing === "right" ? 12 : 8), py + 11, 10, 4);

            particles.forEach(p => {
                ctx.fillStyle = p.color;
                ctx.fillRect(p.x, p.y, p.size, p.size);
            });

            ctx.restore();

            if (gameState === "win" || gameState === "gameover") {
                ctx.fillStyle = "rgba(0, 0, 0, 0.85)";
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                ctx.fillStyle = gameState === "win" ? "#2ecc71" : "#e74c3c";
                ctx.font = "bold 30px 'Courier New'";
                ctx.textAlign = "center";
                ctx.fillText(gameState === "win" ? "¡VICTORIA!" : "¡GAME OVER!", canvas.width / 2, canvas.height / 2 - 20);
                ctx.font = "16px 'Courier New'";
                ctx.fillStyle = "#fff";
                ctx.fillText("Toca o presiona 'R' para reiniciar", canvas.width / 2, canvas.height / 2 + 20);
                ctx.textAlign = "left";
            }
        }

        function restartGame() {
            score = 0; lives = 3; gameState = "playing";
            coins.forEach(c => c.collected = false);
            enemies.forEach(e => e.alive = true);
            resetPlayer();
        }

        function loop() {
            update();
            draw();
            requestAnimationFrame(loop);
        }

        loop();
    </script>
</body>
</html>
