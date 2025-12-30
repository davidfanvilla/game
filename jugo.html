<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Hand Maze: Ultra Precision</title>
    
    <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.9.4/p5.min.js"></script>
    <script src="https://unpkg.com/ml5@1/dist/ml5.min.js"></script>
    
    <style>
        body {
            background-color: #0f172a;
            margin: 0; padding: 0;
            overflow: hidden; touch-action: none;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            display: flex; justify-content: center; align-items: center;
            height: 100vh;
        }

        #overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(15, 23, 42, 1);
            display: flex; flex-direction: column;
            justify-content: center; align-items: center;
            z-index: 100; color: white; text-align: center;
            transition: opacity 0.5s;
        }
        
        .btn-start {
            margin-top: 30px; padding: 15px 40px;
            font-size: 1.2rem; font-weight: bold; color: #fff;
            background: linear-gradient(135deg, #3b82f6, #2563eb);
            border: none; border-radius: 50px; cursor: pointer;
            box-shadow: 0 0 25px rgba(59, 130, 246, 0.5);
            animation: pulse 1.5s infinite;
        }
        
        #loading-text { margin-top: 20px; color: #94a3b8; font-size: 0.9rem; font-family: monospace; }
        
        @keyframes pulse { 0% { transform: scale(1); } 50% { transform: scale(1.05); } 100% { transform: scale(1); } }
    </style>
</head>
<body>

    <div id="overlay">
        <h1 style="color: #60a5fa; font-size: 2.5rem; margin:0;">🎱 Hand Maze Pro</h1>
        <p>Precisión Móvil Activada</p>
        <button id="btn-init" class="btn-start" onclick="initSystem()">INICIAR SISTEMA</button>
        <div id="loading-text"></div>
    </div>

    <script>
        let handPose, video, hands = [];
        let systemReady = false;

        // --- JUEGO ---
        let level = 1;
        let maxLevels = 20;
        let state = 'MENU';
        let ball = { x: 50, y: 50, vx: 0, vy: 0, r: 15 };
        let goal = { x: 300, y: 300, r: 30 }; 
        let holes = [], walls = [];
        
        // Físicas ajustadas para móviles
        let friction = 0.96; 
        let gravity = { x: 0, y: 0 };
        let animFrame = 0;

        // --- CONTROL SUAVIZADO (LA CLAVE DE LA PRECISIÓN) ---
        // 'raw': Donde la cámara dice que está tu mano (tiembla)
        // 'smooth': Donde está realmente la mesa (se desliza suave)
        let handCtrl = { 
            active: false, 
            rawX: 0, rawY: 0, 
            smoothX: 0, smoothY: 0 
        };

        function preload() {
            // flipped: false para control manual del espejo
            handPose = ml5.handPose({ flipped: false }); 
        }

        function setup() {
            createCanvas(windowWidth, windowHeight);
            // IMPORTANTE: pixelDensity(1) hace que el iPhone no se caliente y vaya fluido
            pixelDensity(1); 
            resizeGameElements();
        }

        function initSystem() {
            document.getElementById('btn-init').style.display = 'none';
            document.getElementById('loading-text').innerText = "Accediendo a cámara de alto rendimiento...";

            // Configuración optimizada para fluidez en iOS
            let constraints = {
                audio: false,
                video: {
                    facingMode: "user",
                    // Pedimos resolución media para que el procesador vaya sobrado
                    // y el tracking sea más rápido (menos lag)
                    width: { ideal: 640 }, 
                    height: { ideal: 480 },
                    frameRate: { ideal: 30 } 
                }
            };

            video = createCapture(constraints, function() {
                document.getElementById('loading-text').innerText = "Iniciando IA...";
            });

            video.size(640, 480);
            video.elt.setAttribute('playsinline', ''); // Obligatorio iOS
            video.hide();

            handPose.detectStart(video, results => { 
                hands = results;
                if(!systemReady) {
                    systemReady = true;
                    document.getElementById('overlay').style.opacity = '0';
                    setTimeout(() => document.getElementById('overlay').style.display = 'none', 500);
                    loadLevel(level);
                }
            });
        }

        function windowResized() {
            resizeCanvas(windowWidth, windowHeight);
            resizeGameElements();
            if(systemReady) loadLevel(level);
        }

        function resizeGameElements() {
            let minDim = min(windowWidth, windowHeight);
            ball.r = max(12, minDim * 0.025); 
            goal.r = max(25, minDim * 0.055);  
        }

        function draw() {
            if (!systemReady) return;
            animFrame += 0.05;

            background(30, 41, 59); 
            drawGrid();

            // 1. INPUT CON SUAVIZADO
            processHandInput();
            
            if (state === 'PLAY') {
                updatePhysics();
                checkCollisions();
            }

            drawObstacles();
            drawGoal();
            drawBall();
            drawSign();
            drawHUD();

            if (state === 'MENU') drawMessage("NIVEL " + level, "Haz PINZA 👌 para calibrar");
            else if (state === 'GAMEOVER') drawMessage("¡CAÍSTE!", "Suelta la pinza");
            else if (state === 'WIN') drawMessage("¡CAMPEÓN!", "Juego Completado");

            drawCameraFeed();
        }

        // ==========================================
        // INPUT DE ALTA PRECISIÓN (LERP)
        // ==========================================
        function processHandInput() {
            // Por defecto asumimos que no hay mano, pero NO reseteamos smoothX/Y a 0
            // para evitar saltos bruscos si la cámara pierde la mano un milisegundo.
            let currentlyActive = false;

            if (hands.length > 0) {
                let h = hands[0];
                let t = h.keypoints[4];
                let i = h.keypoints[8];
                let distPinch = dist(t.x, t.y, i.x, i.y);
                
                // --- ESPEJO Y COORDENADAS RAW ---
                // Invertimos X para efecto espejo
                let rawX = video.width - ((t.x + i.x) / 2);
                let rawY = (t.y + i.y) / 2;
                
                // Asignamos el objetivo (raw)
                handCtrl.rawX = rawX;
                handCtrl.rawY = rawY;

                // Umbral de pinza (45px es buen balance móvil)
                if (distPinch < 45) {
                    currentlyActive = true;
                    if (state === 'MENU') {
                        state = 'PLAY';
                        // Al iniciar, igualamos suave y raw para que no salte
                        handCtrl.smoothX = rawX;
                        handCtrl.smoothY = rawY;
                    }
                } else {
                    if (state === 'GAMEOVER') resetLevel();
                }
            }

            handCtrl.active = currentlyActive;

            // --- ALGORITMO DE SUAVIZADO (LERP) ---
            if (handCtrl.active) {
                // Lerp (Linear Interpolation): Mueve el valor actual hacia el objetivo un % cada frame.
                // 0.1 = Muy suave (lento), 0.9 = Muy rápido (tiembla).
                // 0.15 es el punto dulce para iPad/iPhone: elimina el temblor pero responde rápido.
                let smoothingFactor = 0.15; 
                
                handCtrl.smoothX = lerp(handCtrl.smoothX, handCtrl.rawX, smoothingFactor);
                handCtrl.smoothY = lerp(handCtrl.smoothY, handCtrl.rawY, smoothingFactor);
            }
        }

        // ==========================================
        // FÍSICA BASADA EN INPUT SUAVE
        // ==========================================
        function updatePhysics() {
            if (handCtrl.active) {
                let cx = video.width / 2;
                let cy = video.height / 2;
                
                // Usamos smoothX/Y en vez de rawX/Y para la gravedad
                let tiltX = (handCtrl.smoothX - cx) / cx;
                let tiltY = (handCtrl.smoothY - cy) / cy;
                
                // Sensibilidad ajustada para móviles (más alta para mover menos la mano)
                let sensitivity = 1.6; 
                
                gravity.x = tiltX * sensitivity; 
                gravity.y = tiltY * sensitivity;
            } else {
                gravity.x = lerp(gravity.x, 0, 0.1); 
                gravity.y = lerp(gravity.y, 0, 0.1);
            }

            ball.vx += gravity.x; ball.vy += gravity.y;
            ball.vx *= friction; ball.vy *= friction;
            ball.x += ball.vx; ball.y += ball.vy;

            // Rebotes bordes
            if(ball.x<ball.r){ball.x=ball.r; ball.vx*=-0.5;}
            if(ball.x>width-ball.r){ball.x=width-ball.r; ball.vx*=-0.5;}
            if(ball.y<ball.r){ball.y=ball.r; ball.vy*=-0.5;}
            if(ball.y>height-ball.r){ball.y=height-ball.r; ball.vy*=-0.5;}
        }

        // ==========================================
        // CÁMARA ESPEJO (ARRIBA DERECHA)
        // ==========================================
        function drawCameraFeed() {
            let pipW = width * 0.25;
            if (pipW < 120) pipW = 120; // Mínimo visible en móvil
            if (pipW > 200) pipW = 200;
            let pipH = pipW * 0.75; 
            
            let padding = 15;
            let x = width - pipW - padding;
            let y = padding; 

            push();
            // Fondo y Marco
            fill(0); stroke(255, 50); strokeWeight(2);
            rect(x, y, pipW, pipH, 8);
            
            // Zona segura (Mask)
            translate(x, y);

            // --- ESPEJO ---
            push();
            translate(pipW, 0);
            scale(-1, 1);
            
            if (handCtrl.active) tint(255, 255); else tint(255, 120);
            image(video, 0, 0, pipW, pipH);
            noTint();

            if (hands.length > 0) {
                let h = hands[0];
                let sx = pipW / video.width;
                let sy = pipH / video.height;

                stroke(handCtrl.active ? '#60a5fa' : '#ef4444'); 
                strokeWeight(2);
                drawSkeletonLines(h, sx, sy);

                // Dibujamos el punto SUAVIZADO (Azul) y el RAW (Rojo transparente) para debug visual
                if (handCtrl.active) {
                    // Raw (donde la cámara cree que estás)
                    /*
                    noStroke(); fill(255, 0, 0, 100);
                    let rx = handCtrl.rawX * sx; 
                    // Nota: Como estamos dentro del scale(-1,1), la X ya está invertida visualmente
                    // pero rawX ya viene invertida lógicamente. Para dibujar sobre el video espejo:
                    // Necesitamos des-invertir para pintar en el canvas local del pip.
                    // Es complejo, así que usamos directo los keypoints para el esqueleto.
                    */
                   
                    // Punto de control efectivo (Centro de la pinza)
                    fill(96, 165, 250); noStroke();
                    // Calculamos posición visual basada en smoothX/Y
                    // smoothX está en coords de video invertidas (640-0).
                    // Para dibujar en espejo (que invierte visualmente), usamos la coord directa
                    // pero ojo: scale(-1, 1) voltea el sistema de coordenadas.
                    
                    // Simplificación: Dibujamos sobre la mano real
                    let cx = ((h.keypoints[4].x + h.keypoints[8].x)/2) * sx;
                    let cy = ((h.keypoints[4].y + h.keypoints[8].y)/2) * sy;
                    
                    circle(cx, cy, 15);
                    
                    // Anillo de "estabilidad"
                    noFill(); stroke(255); strokeWeight(1);
                    circle(cx, cy, 20);
                }
            }
            pop();

            fill(255); noStroke(); textAlign(LEFT, TOP); textSize(10);
            text("CÁMARA", 5, 5);
            pop();
        }

        // ==========================================
        // ELEMENTOS VISUALES & NIVELES
        // ==========================================
        function loadLevel(n) {
            holes = []; walls = [];
            ball.vx = 0; ball.vy = 0;
            ball.x = width * 0.1; ball.y = height * 0.1;
            goal.x = width * 0.9; goal.y = height * 0.9;
            
            let count = n * 1.5;
            let safeDist = min(width,height) * 0.20;

            if (n > 1) {
                for(let i=0; i<count; i++) {
                    let hr = ball.r * 1.4;
                    let hx, hy, tries=0;
                    do {
                        hx = random(50, width-50); hy = random(50, height-50);
                        tries++; if(tries>100) break;
                    } while(dist(hx,hy,ball.x,ball.y)<safeDist || dist(hx,hy,goal.x,goal.y)<safeDist);
                    holes.push({x:hx, y:hy, r:hr});
                }
            }
            if (n > 2) {
                for(let i=0; i<count/1.5; i++) {
                    let w = random(40, 120); let h = random(40, 120);
                    walls.push({x:random(50, width-50), y:random(50, height-50), w:w, h:h});
                }
            }
        }

        function checkCollisions() {
            if(dist(ball.x, ball.y, goal.x, goal.y) < goal.r) {
                level++; if(level > maxLevels) state='WIN'; else loadLevel(level);
            }
            for(let h of holes) if(dist(ball.x, ball.y, h.x, h.y) < h.r) state='GAMEOVER';
            for(let w of walls) {
                if (ball.x > w.x - ball.r && ball.x < w.x + w.w + ball.r &&
                    ball.y > w.y - ball.r && ball.y < w.y + w.h + ball.r) {
                    ball.vx *= -0.8; ball.vy *= -0.8;
                }
            }
        }
        
        function resetLevel() { ball.x=width*0.1; ball.y=height*0.1; ball.vx=0; ball.vy=0; state='MENU'; }
        
        function drawGoal() {
            if (goal.x === 0) { goal.x = width*0.9; goal.y = height*0.9; }
            let wave = sin(animFrame * 3) * 5;
            noFill(); stroke(96, 165, 250, 100); strokeWeight(2);
            circle(goal.x, goal.y, (goal.r * 2) + wave + 10);
            fill(59, 130, 246); noStroke(); circle(goal.x, goal.y, goal.r * 2);
            fill(30, 64, 175); circle(goal.x, goal.y, goal.r * 1.5);
            fill(255, 255, 255, 100);
            let arrowY = map(sin(animFrame * 5), -1, 1, -5, 5);
            triangle(goal.x - 5, goal.y - 5 + arrowY, goal.x + 5, goal.y - 5 + arrowY, goal.x, goal.y + 5 + arrowY);
        }
        function drawSign() {
            let signX = goal.x;
            let signY = goal.y - goal.r - 40 + (sin(animFrame * 4) * 5);
            push(); translate(signX, signY);
            rectMode(CENTER); fill(255); stroke(59, 130, 246); strokeWeight(3);
            rect(0, 0, 160, 36, 8);
            noStroke(); fill(255); triangle(-10, 18, 10, 18, 0, 28);
            fill(30, 58, 138); textAlign(CENTER, CENTER); textSize(12); textStyle(BOLD);
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
        function drawGrid() { stroke(255,10); for(let i=0;i<width;i+=50)line(i,0,i,height); for(let i=0;i<height;i+=50)line(0,i,width,i); }
        function drawHUD() { fill(255); noStroke(); textSize(20); textAlign(LEFT, TOP); text(`NIVEL ${level}`, 20, 20); }
        function drawMessage(t,s) { fill(0,0,0,200); rect(0,0,width,height); fill(255); textAlign(CENTER); textSize(min(width,height)*0.1); text(t,width/2,height/2-20); textSize(16); fill(200); text(s,width/2,height/2+30); }
        function drawSkeletonLines(h,sx,sy) {
            let p=[[0,1],[1,2],[2,3],[3,4],[0,5],[5,6],[6,7],[7,8],[5,9],[9,13],[13,17],[0,17],[13,14],[14,15],[15,16],[17,18],[18,19],[19,20]];
            for(let i of p) line(h.keypoints[i[0]].x*sx, h.keypoints[i[0]].y*sy, h.keypoints[i[1]].x*sx, h.keypoints[i[1]].y*sy);
        }
    </script>
</body>
</html>
