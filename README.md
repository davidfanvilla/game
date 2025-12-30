<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>Hand Maze Responsive</title>
    
    <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.9.4/p5.min.js"></script>
    <script src="https://unpkg.com/ml5@1/dist/ml5.min.js"></script>
    
    <style>
        body {
            background-color: #0f172a;
            margin: 0; padding: 0;
            overflow: hidden; 
            touch-action: none;
            -webkit-user-select: none; user-select: none;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            display: flex; justify-content: center; align-items: center;
            height: 100dvh; /* dvh maneja mejor las barras de safari */
        }

        #overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(15, 23, 42, 1);
            display: flex; flex-direction: column;
            justify-content: center; align-items: center;
            z-index: 100; color: white; text-align: center;
            transition: opacity 0.5s;
        }
        
        h1 { margin: 0; font-size: 8vmin; color: #60a5fa; }
        p { font-size: 4vmin; color: #94a3b8; margin: 2vh 0; }
        
        .btn-start {
            margin-top: 3vh; padding: 2vh 6vw;
            font-size: 5vmin; font-weight: bold; color: #fff;
            background: linear-gradient(135deg, #3b82f6, #2563eb);
            border: none; border-radius: 50px; cursor: pointer;
            box-shadow: 0 0 25px rgba(59, 130, 246, 0.5);
        }
        
        #loading-text { margin-top: 2vh; color: #64748b; font-size: 3vmin; font-family: monospace; }
    </style>
</head>
<body>

    <div id="overlay">
        <h1>🎱 Hand Maze</h1>
        <p>Apunta al hoyo verde</p>
        <button id="btn-init" class="btn-start" onclick="initSystem()">JUGAR</button>
        <div id="loading-text">Github Edition</div>
    </div>

    <script>
        let handPose, video, hands = [];
        let systemReady = false;

        // --- JUEGO ---
        let level = 1, maxLevels = 20, state = 'MENU';
        let ball = { x: 0, y: 0, vx: 0, vy: 0, r: 0 };
        let goal = { x: 0, y: 0, r: 0 };
        let holes = [], walls = [];
        let friction = 0.96;
        let gravity = { x: 0, y: 0 };
        let animFrame = 0;

        // --- CONTROL ---
        let handCtrl = { active: false, x: 0, y: 0, smoothX: 0, smoothY: 0 };

        function preload() {
            // flipped: false para control manual del espejo
            handPose = ml5.handPose({ flipped: false }); 
        }

        function setup() {
            // Usamos windowWidth/Height para llenar la pantalla
            createCanvas(windowWidth, windowHeight);
            pixelDensity(1); 
            // Inicializar posiciones relativas al tamaño actual
            resizeGameElements();
        }

        function initSystem() {
            document.getElementById('btn-init').style.display = 'none';
            document.getElementById('loading-text').innerText = "Solicitando cámara...";

            let constraints = {
                audio: false,
                video: {
                    facingMode: "user",
                    width: { ideal: 640 },
                    height: { ideal: 480 }
                }
            };

            video = createCapture(constraints, function() {
                document.getElementById('loading-text').innerText = "Cargando IA...";
            });

            video.size(640, 480);
            video.elt.setAttribute('playsinline', ''); 
            video.hide();

            handPose.detectStart(video, results => { 
                hands = results;
                if(!systemReady) {
                    systemReady = true;
                    document.getElementById('overlay').style.opacity = '0';
                    setTimeout(() => document.getElementById('overlay').style.display = 'none', 500);
                    // Cargar nivel inicial ajustado
                    loadLevel(level);
                }
            });
        }

        // ==========================================
        // SISTEMA RESPONSIVO
        // ==========================================
        function windowResized() {
            resizeCanvas(windowWidth, windowHeight);
            resizeGameElements();
            // Si el juego ya empezó, regeneramos el nivel para que los muros no queden mal
            if(systemReady) {
                // Reiniciamos posición de bola para evitar que quede atrapada en un muro
                resetBallPosition();
                loadLevel(level); 
            }
        }

        function resizeGameElements() {
            // Unidad base: el lado más pequeño de la pantalla
            let minDim = min(windowWidth, windowHeight);
            
            // Tamaños relativos (Responsive)
            ball.r = max(10, minDim * 0.025); // Bola ~2.5% pantalla
            goal.r = max(20, minDim * 0.055); // Meta ~5.5% pantalla
            
            // Si no ha empezado el juego, centramos todo
            if(state === 'MENU') {
                resetBallPosition();
                goal.x = width * 0.9;
                goal.y = height * 0.9;
            }
        }

        function resetBallPosition() {
            ball.x = width * 0.1; 
            ball.y = height * 0.1;
            ball.vx = 0; ball.vy = 0;
        }

        function draw() {
            if (!systemReady) return;
            animFrame += 0.05;

            background(30, 41, 59); 
            drawGrid();

            // INPUT
            processHandInput();
            
            // FÍSICA
            if (state === 'PLAY') {
                updatePhysics();
                checkCollisions();
            }

            // DIBUJAR CAPAS
            drawObstacles();
            drawGoal();
            drawBall();
            drawSign();
            drawHUD();

            // MENSAJES OVERLAY RESPONSIVOS
            if (state === 'MENU') drawMessage("NIVEL " + level, "Haz PINZA 👌 para calibrar");
            else if (state === 'GAMEOVER') drawMessage("¡CAÍSTE!", "Suelta la pinza");
            else if (state === 'WIN') drawMessage("¡CAMPEÓN!", "Juego Completado");

            // CÁMARA PIP ADAPTABLE
            drawCameraFeed();
        }

        // ==========================================
        // CÁMARA PIP RESPONSIVA
        // ==========================================
        function drawCameraFeed() {
            // Tamaño dinámico: 25% del ancho, pero con límites
            let pipW = width * 0.25;
            if (pipW < 100) pipW = 100; // Mínimo en móvil vertical
            if (pipW > 240) pipW = 240; // Máximo en desktop
            
            let pipH = pipW * 0.75; // Aspect ratio 4:3
            
            // Márgenes dinámicos
            let margin = width * 0.02; 
            
            // Posición: Esquina superior derecha
            let x = width - pipW - margin;
            let y = margin; 

            push();
            // Marco
            stroke(255, 50); strokeWeight(2); fill(0);
            rect(x, y, pipW, pipH, 10);
            
            // Zona segura
            translate(x, y);

            // ESPEJO MANUAL
            push();
            translate(pipW, 0);
            scale(-1, 1);
            
            if (handCtrl.active) tint(255, 255); else tint(255, 120);
            image(video, 0, 0, pipW, pipH);
            noTint();

            // MANO (Puntos coinciden por scale -1,1)
            if (hands.length > 0) {
                let h = hands[0];
                let sx = pipW / video.width;
                let sy = pipH / video.height;

                stroke(handCtrl.active ? '#60a5fa' : '#ef4444'); 
                strokeWeight(2);
                drawSkeletonLines(h, sx, sy);

                if (handCtrl.active) {
                    fill(96, 165, 250); noStroke();
                    let cx = ((h.keypoints[4].x + h.keypoints[8].x)/2) * sx;
                    let cy = ((h.keypoints[4].y + h.keypoints[8].y)/2) * sy;
                    circle(cx, cy, 15);
                }
            }
            pop();

            fill(255); noStroke(); textAlign(LEFT, TOP); 
            textSize(10); text("CÁMARA", 5, 5);
            pop();
        }

        // ==========================================
        // INPUT PROCESADO
        // ==========================================
        function processHandInput() {
            let active = false;
            if (hands.length > 0) {
                let h = hands[0];
                let t = h.keypoints[4];
                let i = h.keypoints[8];
                let distPinch = dist(t.x, t.y, i.x, i.y);
                
                // Mapeo Espejo Manual
                let rawX = (t.x + i.x) / 2;
                let rawY = (t.y + i.y) / 2;
                
                // IMPORTANTE: Invertimos X (Video Width - X)
                let mirrorX = video.width - rawX;

                // Escalar a pantalla actual (Responsivo)
                let targetX = map(mirrorX, 0, video.width, 0, width);
                let targetY = map(rawY, 0, video.height, 0, height);

                if (distPinch < 50) {
                    active = true;
                    if (state === 'MENU') {
                        state = 'PLAY';
                        handCtrl.smoothX = targetX;
                        handCtrl.smoothY = targetY;
                    }
                } else {
                    if (state === 'GAMEOVER') resetBallPosition();
                }

                if (active) {
                    // LERP para móviles
                    handCtrl.smoothX = lerp(handCtrl.smoothX, targetX, 0.2);
                    handCtrl.smoothY = lerp(handCtrl.smoothY, targetY, 0.2);
                }
            }
            handCtrl.active = active;
        }

        // ==========================================
        // GENERADOR DE NIVELES RESPONSIVO
        // ==========================================
        function loadLevel(n) {
            holes = []; walls = [];
            resetBallPosition();
            
            // Meta siempre al 90% de la pantalla actual
            goal.x = width * 0.9; 
            goal.y = height * 0.9;
            
            let count = n * 1.5;
            let safeDist = min(width,height) * 0.20;

            // Agujeros
            if (n > 1) {
                for(let i=0; i<count; i++) {
                    let hr = ball.r * 1.4; // Radio relativo a la bola
                    let hx, hy, tries=0;
                    do {
                        // Posición aleatoria dentro de márgenes seguros
                        hx = random(width*0.1, width*0.9); 
                        hy = random(height*0.1, height*0.9);
                        tries++; if(tries>100) break;
                    } while(dist(hx,hy,ball.x,ball.y)<safeDist || dist(hx,hy,goal.x,goal.y)<safeDist);
                    holes.push({x:hx, y:hy, r:hr});
                }
            }
            
            // Paredes
            if (n > 2) {
                let wallCount = isMobilePortrait() ? count/2 : count; // Menos paredes en vertical
                for(let i=0; i<wallCount; i++) {
                    let w = random(width*0.05, width*0.2); // Ancho relativo
                    let h = random(height*0.05, height*0.2); // Alto relativo
                    walls.push({
                        x: random(width*0.1, width*0.8), 
                        y: random(height*0.1, height*0.8), 
                        w:w, h:h
                    });
                }
            }
        }

        // Helper para detectar orientación vertical
        function isMobilePortrait() {
            return height > width;
        }

        function updatePhysics() {
            if (handCtrl.active) {
                let cx = width / 2;
                let cy = height / 2;
                let tiltX = (handCtrl.smoothX - cx) / cx;
                let tiltY = (handCtrl.smoothY - cy) / cy;
                
                // Sensibilidad un poco más alta en móviles verticales
                let sensitivity = isMobilePortrait() ? 2.0 : 1.5;
                
                gravity.x = tiltX * sensitivity; 
                gravity.y = tiltY * sensitivity;
            } else {
                gravity.x = lerp(gravity.x, 0, 0.1); 
                gravity.y = lerp(gravity.y, 0, 0.1);
            }
            ball.vx += gravity.x; ball.vy += gravity.y;
            ball.vx *= friction; ball.vy *= friction;
            ball.x += ball.vx; ball.y += ball.vy;

            // Colisiones Bordes
            if(ball.x<ball.r){ball.x=ball.r; ball.vx*=-0.5;}
            if(ball.x>width-ball.r){ball.x=width-ball.r; ball.vx*=-0.5;}
            if(ball.y<ball.r){ball.y=ball.r; ball.vy*=-0.5;}
            if(ball.y>height-ball.r){ball.y=height-ball.r; ball.vy*=-0.5;}
        }

        function checkCollisions() {
            if(dist(ball.x, ball.y, goal.x, goal.y) < goal.r) {
                level++; 
                if(level > maxLevels) {
                    state='WIN'; 
                } else {
                    loadLevel(level);
                }
            }
            for(let h of holes) if(dist(ball.x, ball.y, h.x, h.y) < h.r) state='GAMEOVER';
            for(let w of walls) {
                if (ball.x > w.x - ball.r && ball.x < w.x + w.w + ball.r &&
                    ball.y > w.y - ball.r && ball.y < w.y + w.h + ball.r) {
                    ball.vx *= -0.8; ball.vy *= -0.8;
                }
            }
        }

        // ==========================================
        // DIBUJADO RESPONSIVO
        // ==========================================
        function drawGoal() {
            if (goal.x === 0) { goal.x = width*0.9; goal.y = height*0.9; }
            let wave = sin(animFrame * 3) * 5;
            noFill(); stroke(96, 165, 250, 100); strokeWeight(2);
            circle(goal.x, goal.y, (goal.r * 2) + wave + 5);
            fill(59, 130, 246); noStroke(); circle(goal.x, goal.y, goal.r * 2);
            fill(30, 64, 175); circle(goal.x, goal.y, goal.r * 1.5);
            fill(255, 255, 255, 100);
            let arrowY = map(sin(animFrame * 5), -1, 1, -5, 5);
            triangle(goal.x - 5, goal.y - 5 + arrowY, goal.x + 5, goal.y - 5 + arrowY, goal.x, goal.y + 5 + arrowY);
        }

        function drawSign() {
            let offset = goal.r + 25;
            let signY = goal.y - offset + (sin(animFrame * 4) * 3);
            
            // Tamaño de letrero relativo
            let bw = max(100, width * 0.15);
            let bh = max(25, height * 0.04);
            let txtSize = max(10, min(width, height) * 0.02);

            push(); translate(goal.x, signY);
            rectMode(CENTER); fill(255); stroke(59, 130, 246); strokeWeight(3);
            rect(0, 0, bw, bh, 8);
            noStroke(); fill(255); triangle(-10, bh/2, 10, bh/2, 0, bh/2 + 10);
            fill(30, 58, 138); textAlign(CENTER, CENTER); textSize(txtSize); textStyle(BOLD);
            text("¡DEPOSÍTALA AQUÍ!", 0, 0);
            pop();
        }

        function drawBall() {
            fill(240); noStroke(); circle(ball.x, ball.y, ball.r*2);
            fill(200); circle(ball.x-2, ball.y+2, ball.r*2);
            fill(255); circle(ball.x-3, ball.y-3, ball.r*0.8);
        }

        function drawObstacles() {
            fill(15); noStroke(); for(let h of holes) circle(h.x, h.y, h.r*2);
            fill(100, 116, 139); stroke(255,255,255,20); strokeWeight(1);
            for(let w of walls) rect(w.x, w.y, w.w, w.h, 4);
        }

        function drawGrid() {
            stroke(255, 10);
            let spacing = min(width, height) / 10;
            for(let i=0; i<width; i+=spacing) line(i,0,i,height);
            for(let i=0; i<height; i+=spacing) line(0,i,width,i);
        }

        function drawHUD() {
            fill(255); noStroke(); 
            // Texto adaptable: 3% del tamaño mínimo de pantalla
            let hudSize = min(width, height) * 0.05; 
            textSize(hudSize); textAlign(LEFT, TOP); 
            text(`NIVEL ${level}`, width*0.03, height*0.03); 
        }

        function drawMessage(t, s) {
            fill(0, 0, 0, 200); rect(0, 0, width, height);
            fill(255); textAlign(CENTER, CENTER);
            // Textos gigantes relativos a pantalla (viewport units)
            textSize(min(width, height) * 0.15); text(t, width/2, height/2 - 20);
            textSize(min(width, height) * 0.05); fill(200); text(s, width/2, height/2 + 60);
        }

        function drawSkeletonLines(h, sx, sy) {
            let p=[[0,1],[1,2],[2,3],[3,4],[0,5],[5,6],[6,7],[7,8],[5,9],[9,13],[13,17],[0,17],[13,14],[14,15],[15,16],[17,18],[18,19],[19,20]];
            for(let i of p) line(h.keypoints[i[0]].x*sx, h.keypoints[i[0]].y*sy, h.keypoints[i[1]].x*sx, h.keypoints[i[1]].y*sy);
        }
    </script>
</body>
</html>
