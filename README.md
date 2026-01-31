HTML
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <meta name="theme-color" content="#2980b9">
    <title>Azure Odyssey: The Pearl Hunter</title>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Bubblegum+Sans&family=Roboto:wght@300;700&display=swap');

        body {
            margin: 0;
            padding: 0;
            background-color: #000;
            overflow: hidden;
            font-family: 'Roboto', sans-serif;
            touch-action: none; /* Отключает стандартные жесты браузера */
            user-select: none;
            -webkit-user-select: none;
        }

        #game-container {
            position: fixed; /* Используем fixed для предотвращения скролла */
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: linear-gradient(to bottom, #2980b9, #2c3e50); 
            display: flex;
            justify-content: center;
            align-items: center;
        }

        canvas {
            display: block;
            width: 100%;
            height: 100%;
        }

        #ui-layer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            display: flex;
            flex-direction: column;
            justify-content: space-between;
            z-index: 5;
        }

        .hud-panel {
            padding: 20px;
            color: white;
            text-shadow: 2px 2px 0px rgba(0,0,0,0.5);
            display: flex;
            justify-content: space-between;
            width: 100%;
            box-sizing: border-box;
        }

        .score-display {
            font-family: 'Bubblegum Sans', cursive;
            font-size: 2.5rem;
            color: #ffd700;
            filter: drop-shadow(0 0 5px rgba(255, 215, 0, 0.5));
        }

        #start-screen, #game-over-screen {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.65);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            pointer-events: auto; /* Важно: разрешаем клики по интерфейсу */
            backdrop-filter: blur(8px);
            transition: opacity 0.3s ease;
            z-index: 10;
        }

        h1 {
            font-family: 'Bubblegum Sans', cursive;
            font-size: 5rem;
            color: #00ffff;
            text-shadow: 0 0 20px #0099ff;
            margin: 0;
            animation: float 3s ease-in-out infinite;
            text-align: center;
            line-height: 1;
            padding: 0 10px;
        }

        p {
            color: #f0f0f0;
            font-size: 1.3rem;
            margin-top: 15px;
            text-align: center;
            max-width: 600px;
            line-height: 1.5;
            padding: 0 20px;
        }

        .btn {
            margin-top: 40px;
            padding: 15px 50px;
            font-size: 1.8rem;
            background: linear-gradient(45deg, #ff6b6b, #feca57); 
            border: none;
            border-radius: 50px;
            color: white;
            cursor: pointer;
            font-family: 'Bubblegum Sans', cursive;
            box-shadow: 0 10px 20px rgba(0,0,0,0.3);
            transition: transform 0.1s cubic-bezier(0.175, 0.885, 0.32, 1.275);
            text-transform: uppercase;
            letter-spacing: 1px;
            /* Улучшение для тач-скринов: убираем подсветку при тапе */
            -webkit-tap-highlight-color: transparent;
            z-index: 20;
        }

        .btn:active {
            transform: scale(0.95);
        }

        .hidden {
            display: none !important;
            opacity: 0;
            pointer-events: none;
        }

        @keyframes float {
            0% { transform: translateY(0px); }
            50% { transform: translateY(-15px); }
            100% { transform: translateY(0px); }
        }

        /* Адаптив для телефонов */
        @media (max-width: 600px) {
            h1 { font-size: 3rem; }
            .score-display { font-size: 1.5rem; }
            .btn { padding: 15px 40px; font-size: 1.5rem; margin-top: 30px; }
            p { font-size: 1rem; }
        }
    </style>
</head>
<body>

<div id="game-container">
    <canvas id="gameCanvas"></canvas>
    
    <div id="ui-layer">
        <div class="hud-panel">
            <div class="score-display">Pearls: <span id="score-val">0</span></div>
            <div class="score-display" style="font-size: 1.5rem; color: #fff;">Distance: <span id="dist-val">0</span>m</div>
        </div>
    </div>

    <div id="start-screen">
        <h1>Azure Odyssey</h1>
        <p>Join Delphinus on a journey through the Royal Reefs.</p>
        <p>Tap to swim. Collect Pearls.</p>
        <button class="btn" id="start-btn">Dive In</button>
    </div>

    <div id="game-over-screen" class="hidden">
        <h1 style="color: #ff6b6b;">Oh No!</h1>
        <p>The ocean is a dangerous place.</p>
        <div class="score-display" style="margin-top: 20px;">Score: <span id="final-score">0</span></div>
        <button class="btn" id="restart-btn">Try Again</button>
    </div>
</div>

<script>
// CONFIGURATION
const CONFIG = {
    gravity: 0.25,
    jumpForce: -6.5,
    drag: 0.99,
    gameSpeed: 3,
    maxGameSpeed: 9,
    speedIncrement: 0.001,
    sharkSpawnRate: 150,
    pearlSpawnRate: 60,
    colors: {
        dolphin: '#00ced1',
        dolphinBelly: '#e0ffff',
        shark: '#2c3e50',
        pearl: '#fff0f5',
        bubble: 'rgba(255, 255, 255, 0.4)',
        reef: '#5D4037',
        seaweed: '#2ecc71',
        starfish: '#ff7675',
        goldfish: '#e67e22',
        whale: '#34495e'
    }
};

// GAME STATE
let canvas, ctx;
let frames = 0;
let score = 0;
let distance = 0;
let gameRunning = false;
let entities = {
    backgrounds: [],
    reefs: [],
    fishes: [],
    sharks: [],
    pearls: [],
    bubbles: []
};
let dolphin;
let currentSpeed = CONFIG.gameSpeed;
let dpr = 1; // Device Pixel Ratio

// DOM ELEMENTS
const scoreEl = document.getElementById('score-val');
const distEl = document.getElementById('dist-val');
const finalScoreEl = document.getElementById('final-score');
const startScreen = document.getElementById('start-screen');
const gameOverScreen = document.getElementById('game-over-screen');
const startBtn = document.getElementById('start-btn');
const restartBtn = document.getElementById('restart-btn');

// AUDIO SYSTEM
const AudioSys = {
    ctx: null,
    init: function() {
        window.AudioContext = window.AudioContext || window.webkitAudioContext;
        if (!this.ctx) {
            this.ctx = new AudioContext();
        }
        if (this.ctx.state === 'suspended') {
            this.ctx.resume();
        }
    },
    playTone: function(type, startFreq, endFreq, duration, vol) {
        if (!this.ctx) return;
        const osc = this.ctx.createOscillator();
        const gain = this.ctx.createGain();
        osc.type = type;
        osc.frequency.setValueAtTime(startFreq, this.ctx.currentTime);
        osc.frequency.exponentialRampToValueAtTime(endFreq, this.ctx.currentTime + duration);
        gain.gain.setValueAtTime(vol, this.ctx.currentTime);
        gain.gain.exponentialRampToValueAtTime(0.01, this.ctx.currentTime + duration);
        osc.connect(gain);
        gain.connect(this.ctx.destination);
        osc.start();
        osc.stop(this.ctx.currentTime + duration);
    },
    playSwim: function() { this.playTone('sine', 150, 300, 0.1, 0.1); },
    playCollect: function() { this.playTone('triangle', 600, 1200, 0.2, 0.1); },
    playCrash: function() { this.playTone('sawtooth', 100, 10, 0.5, 0.2); }
};

// CLASSES
class Whale {
    constructor() {
        this.x = (canvas.width / dpr) + 200;
        this.y = Math.random() * ((canvas.height / dpr) * 0.6) + 50;
        this.speed = currentSpeed * 0.3;
        this.size = Math.random() * 0.5 + 0.8;
    }
    update() { this.x -= this.speed; }
    draw() {
        ctx.save();
        ctx.translate(this.x, this.y);
        ctx.scale(this.size, this.size);
        ctx.fillStyle = CONFIG.colors.whale;
        ctx.beginPath(); ctx.ellipse(0, 0, 100, 40, 0, 0, Math.PI * 2); ctx.fill();
        ctx.beginPath(); ctx.moveTo(80, 0); ctx.lineTo(120, -20); ctx.lineTo(120, 20); ctx.fill();
        ctx.restore();
    }
}

class Goldfish {
    constructor(x, y) {
        this.x = x; this.y = y;
        this.speed = currentSpeed * 1.5 + Math.random();
        this.color = CONFIG.colors.goldfish;
    }
    update() {
        this.x -= this.speed;
        this.y += Math.sin(frames * 0.1) * 0.5;
    }
    draw() {
        ctx.save(); ctx.translate(this.x, this.y); ctx.fillStyle = this.color;
        ctx.beginPath(); ctx.ellipse(0, 0, 8, 4, 0, 0, Math.PI * 2); ctx.fill();
        ctx.beginPath(); ctx.moveTo(4, 0); ctx.lineTo(8, -4); ctx.lineTo(8, 4); ctx.fill();
        ctx.restore();
    }
}

class Reef {
    constructor(x, width) {
        this.x = x; this.width = width;
        this.points = []; this.decorations = [];
        let segments = 5; let segWidth = width / segments;
        for(let i=0; i<=segments; i++) {
            let h = Math.random() * 50 + 50;
            this.points.push({x: i * segWidth, h: h});
            if (i < segments && Math.random() > 0.3) {
                this.decorations.push({
                    type: Math.random() > 0.5 ? 'seaweed' : 'starfish',
                    x: (i * segWidth) + Math.random() * segWidth,
                    y: -h + 10, size: Math.random() * 10 + 10, offset: Math.random() * Math.PI * 2
                });
            }
        }
    }
    update() { this.x -= currentSpeed; }
    draw() {
        ctx.save();
        ctx.translate(this.x, (canvas.height / dpr));
        ctx.fillStyle = CONFIG.colors.reef;
        ctx.beginPath(); ctx.moveTo(0, 0);
        for(let p of this.points) ctx.lineTo(p.x, -p.h);
        ctx.lineTo(this.width, 0); ctx.fill();
        for(let d of this.decorations) {
            if (d.type === 'seaweed') this.drawSeaweed(d.x, -d.y, d.size, d.offset);
            else this.drawStarfish(d.x, -d.y, d.size);
        }
        ctx.restore();
    }
    drawSeaweed(x, y, h, offset) {
        ctx.save(); ctx.translate(x, y); ctx.strokeStyle = CONFIG.colors.seaweed;
        ctx.lineWidth = 3; ctx.lineCap = 'round';
        let sway = Math.sin((frames * 0.05) + offset) * 10;
        ctx.beginPath(); ctx.moveTo(0, 0); ctx.quadraticCurveTo(sway, -h/2, sway/2, -h * 2); ctx.stroke();
        ctx.restore();
    }
    drawStarfish(x, y, r) {
        ctx.save(); ctx.translate(x, y); ctx.fillStyle = CONFIG.colors.starfish;
        ctx.beginPath();
        for (let i = 0; i < 5; i++) {
            ctx.lineTo(Math.cos((18 + i * 72) * Math.PI / 180) * r, -Math.sin((18 + i * 72) * Math.PI / 180) * r);
            ctx.lineTo(Math.cos((54 + i * 72) * Math.PI / 180) * (r/2), -Math.sin((54 + i * 72) * Math.PI / 180) * (r/2));
        }
        ctx.fill(); ctx.restore();
    }
}

class Dolphin {
    constructor(x, y) {
        this.x = x; this.y = y;
        this.velocity = 0; this.radius = 20; this.angle = 0;
    }
    update() {
        this.velocity += CONFIG.gravity;
        this.velocity *= CONFIG.drag;
        this.y += this.velocity;
        const targetAngle = Math.min(Math.PI / 4, Math.max(-Math.PI / 4, (this.velocity * 0.1)));
        this.angle += (targetAngle - this.angle) * 0.1;
        
        let floor = (canvas.height / dpr) - this.radius;
        if (this.y + this.radius > (canvas.height / dpr)) { this.y = floor; this.velocity = 0; }
        if (this.y - this.radius < 0) { this.y = this.radius; this.velocity = 0; }
    }
    flap() {
        this.velocity = CONFIG.jumpForce;
        AudioSys.playSwim();
        for(let i=0; i<5; i++) entities.bubbles.push(new Bubble(this.x - 10, this.y));
    }
    draw() {
        ctx.save(); ctx.translate(this.x, this.y); ctx.rotate(this.angle);
        ctx.beginPath(); ctx.ellipse(0, 0, 30, 15, 0, 0, Math.PI * 2); ctx.fillStyle = CONFIG.colors.dolphin; ctx.fill();
        ctx.beginPath(); ctx.ellipse(0, 5, 20, 8, 0, 0, Math.PI * 2); ctx.fillStyle = CONFIG.colors.dolphinBelly; ctx.fill();
        ctx.fillStyle = 'black'; ctx.beginPath(); ctx.arc(10, -5, 2, 0, Math.PI * 2); ctx.fill();
        ctx.beginPath(); ctx.moveTo(-25, 0); ctx.lineTo(-35, -10); ctx.lineTo(-35, 10); ctx.fillStyle = CONFIG.colors.dolphin; ctx.fill();
        ctx.restore();
    }
}

class Shark {
    constructor() {
        this.x = (canvas.width / dpr) + 50;
        this.y = Math.random() * ((canvas.height / dpr) - 100) + 50;
        this.speed = currentSpeed * (1 + Math.random() * 0.2);
    }
    update() { this.x -= this.speed; }
    draw() {
        ctx.save(); ctx.translate(this.x, this.y);
        ctx.fillStyle = CONFIG.colors.shark;
        ctx.beginPath(); ctx.moveTo(30, 0); ctx.lineTo(-30, -15); ctx.lineTo(-30, 15); ctx.fill();
        ctx.beginPath(); ctx.moveTo(-10, -10); ctx.lineTo(0, -25); ctx.lineTo(10, -10); ctx.fill();
        ctx.fillStyle = 'white'; ctx.beginPath(); ctx.arc(15, -5, 3, 0, Math.PI * 2); ctx.fill();
        ctx.restore();
    }
}

class Pearl {
    constructor() {
        this.x = (canvas.width / dpr) + 50;
        this.y = Math.random() * ((canvas.height / dpr) - 100) + 50;
        this.t = 0; this.radius = 12;
    }
    update() {
        this.x -= currentSpeed; this.t += 0.1;
        this.y += Math.sin(this.t) * 0.5;
    }
    draw() {
        ctx.save(); ctx.translate(this.x, this.y);
        ctx.shadowBlur = 15; ctx.shadowColor = "white";
        ctx.fillStyle = CONFIG.colors.pearl; ctx.beginPath(); ctx.arc(0, 0, this.radius, 0, Math.PI * 2); ctx.fill();
        ctx.restore();
    }
}

class Bubble {
    constructor(x, y) {
        this.x = x; this.y = y;
        this.radius = Math.random() * 3 + 1; this.speed = Math.random() * 2 + 1;
    }
    update() { this.x -= currentSpeed * 0.5; this.y -= this.speed; }
    draw() {
        ctx.fillStyle = CONFIG.colors.bubble; ctx.beginPath(); ctx.arc(this.x, this.y, this.radius, 0, Math.PI*2); ctx.fill();
    }
}

// LOGIC
function init() {
    canvas = document.getElementById('gameCanvas');
    ctx = canvas.getContext('2d');
    resize();
    window.addEventListener('resize', resize);
    
    // Универсальный обработчик событий
    const gameArea = document.getElementById('game-container');
    gameArea.addEventListener('mousedown', handleInput);
    gameArea.addEventListener('touchstart', handleInput, {passive: false});
    window.addEventListener('keydown', (e) => { if (e.code === 'Space') handleInput(e); });

    startBtn.addEventListener('click', (e) => {
        // Запрос полноэкранного режима при старте
        toggleFullScreen(); 
        startGame();
    });
    
    // Для тач-устройств дублируем на touchend, чтобы срабатывало быстрее
    startBtn.addEventListener('touchend', (e) => {
        e.preventDefault(); // Предотвращаем двойной клик
        toggleFullScreen();
        startGame();
    });

    restartBtn.addEventListener('click', resetGame);
    restartBtn.addEventListener('touchend', (e) => { e.preventDefault(); resetGame(); });
}

function resize() {
    dpr = window.devicePixelRatio || 1;
    canvas.width = window.innerWidth * dpr;
    canvas.height = window.innerHeight * dpr;
    // Принудительно задаем CSS размеры, чтобы canvas не растягивался некорректно
    canvas.style.width = window.innerWidth + 'px';
    canvas.style.height = window.innerHeight + 'px';
    // Масштабируем контекст
    ctx.scale(dpr, dpr);
}

function handleInput(e) {
    // 1. Не блокируем системные кнопки или клики по интерфейсу
    if (e.target.tagName === 'BUTTON' || e.target.closest('.btn')) {
        return; 
    }

    // 2. Блокируем скролл и зум только если это тач внутри игры
    if (e.type === 'touchstart') {
        e.preventDefault(); 
    }

    // 3. Управление дельфином
    if (gameRunning) {
        dolphin.flap();
    }
}

function toggleFullScreen() {
    if (!document.fullscreenElement) {
        document.documentElement.requestFullscreen().catch(err => {
            console.log(`Error attempting to enable full-screen mode: ${err.message}`);
        });
    }
}

function startGame() {
    AudioSys.init(); // Инициализация звука по жесту
    startScreen.classList.add('hidden');
    gameOverScreen.classList.add('hidden');
    resetGameVariables();
    gameRunning = true;
    loop();
}

function resetGame() {
    gameOverScreen.classList.add('hidden');
    resetGameVariables();
    gameRunning = true;
    loop();
}

function resetGameVariables() {
    // Учитываем масштаб при спавне
    const w = canvas.width / dpr;
    const h = canvas.height / dpr;
    
    dolphin = new Dolphin(100, h / 2);
    entities = { backgrounds: [], reefs: [], fishes: [], sharks: [], pearls: [], bubbles: [] };
    
    for (let i = 0; i < w / 100 + 2; i++) {
        entities.reefs.push(new Reef(i * 100, 100));
    }
    score = 0; distance = 0; frames = 0; currentSpeed = CONFIG.gameSpeed;
    scoreEl.innerText = score; distEl.innerText = distance;
}

function gameOver() {
    gameRunning = false;
    AudioSys.playCrash();
    finalScoreEl.innerText = score;
    gameOverScreen.classList.remove('hidden');
}

function checkCollisions() {
    for (let i = entities.pearls.length - 1; i >= 0; i--) {
        let p = entities.pearls[i];
        let dx = dolphin.x - p.x; let dy = dolphin.y - p.y;
        if (Math.sqrt(dx*dx + dy*dy) < dolphin.radius + p.radius) {
            entities.pearls.splice(i, 1);
            score++; scoreEl.innerText = score;
            AudioSys.playCollect();
            for(let j=0; j<8; j++) entities.bubbles.push(new Bubble(p.x, p.y));
        }
    }
    for (let s of entities.sharks) {
        if (dolphin.x + 15 > s.x - 20 && dolphin.x - 15 < s.x + 20 && 
            dolphin.y + 10 > s.y - 10 && dolphin.y - 10 < s.y + 10) {
            gameOver();
        }
    }
}

function loop() {
    if (!gameRunning) return;
    const w = canvas.width / dpr;
    const h = canvas.height / dpr;

    ctx.clearRect(0, 0, w, h);

    if (currentSpeed < CONFIG.maxGameSpeed) currentSpeed += CONFIG.speedIncrement;
    frames++;
    if (frames % 10 === 0) { distance++; distEl.innerText = distance; }

    // Spawning logic using scaled dimensions
    let lastReef = entities.reefs[entities.reefs.length - 1];
    if (lastReef.x + lastReef.width < w + 100) entities.reefs.push(new Reef(lastReef.x + lastReef.width, 100));
    if (frames % CONFIG.sharkSpawnRate === 0) {
        entities.sharks.push(new Shark());
        if (CONFIG.sharkSpawnRate > 60) CONFIG.sharkSpawnRate--;
    }
    if (frames % CONFIG.pearlSpawnRate === 0) entities.pearls.push(new Pearl());
    if (frames % 1000 === 0) entities.backgrounds.push(new Whale());
    if (frames % 300 === 0) {
        let startY = Math.random() * h;
        for(let i=0; i<5; i++) entities.fishes.push(new Goldfish(w + (i*30), startY + (Math.random()*50 - 25)));
    }
    if (Math.random() < 0.1) entities.bubbles.push(new Bubble(w, Math.random() * h));

    // Update & Draw
    [entities.backgrounds, entities.fishes, entities.reefs, entities.bubbles, entities.pearls, entities.sharks].forEach(arr => {
        for (let i = arr.length - 1; i >= 0; i--) {
            let e = arr[i]; e.update(); e.draw();
            if (e.x < -300 || (e.width && e.x + e.width < -50) || (e.y < -50)) arr.splice(i, 1);
        }
    });

    dolphin.update();
    dolphin.draw();
    checkCollisions();
    requestAnimationFrame(loop);
}

init();
</script>
</body>
</html>
