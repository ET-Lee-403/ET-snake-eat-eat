<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>氙基原點：三部曲 - 能量補完版</title>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@400;900&family=Noto+Sans+TC:wght@500;900&display=swap');
        body, html { margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #010103; color: #00ff9d; font-family: 'Noto Sans TC', sans-serif; touch-action: none; }
        
        .overlay { 
            position: absolute; top: 0; left: 0; width: 100%; height: 100%; 
            display: flex; flex-direction: column; align-items: center; justify-content: center; 
            z-index: 100; transition: opacity 0.8s ease-in-out;
            background: rgba(1, 1, 3, 0.75); backdrop-filter: blur(15px);
        }

        h1 { font-size: clamp(40px, 8vw, 60px); margin: 0; text-shadow: 0 0 30px #00ff9d; letter-spacing: 10px; text-align: center; font-family: 'Orbitron', sans-serif; }
        .sub-title { color: #ffaa00; font-size: 16px; margin-bottom: 30px; letter-spacing: 2px; }
        
        .btn-group { display: flex; flex-direction: column; gap: 15px; width: 280px; }
        button { 
            padding: 15px; font-size: 18px; font-weight: 900; background: rgba(0, 255, 157, 0.05); 
            color: #00ff9d; border: 2px solid #00ff9d; cursor: pointer; letter-spacing: 4px; 
            font-family: 'Noto Sans TC'; transition: 0.3s;
        }
        button:hover { background: #00ff9d; color: #000; box-shadow: 0 0 25px #00ff9d; }
        
        #leaderboard-ui { margin-top: 30px; color: #fff; font-size: 14px; width: 260px; border-top: 1px solid #444; padding-top: 20px; }
        .leader-item { display: flex; justify-content: space-between; margin: 8px 0; font-family: 'Orbitron'; background: rgba(255,255,255,0.03); padding: 5px 10px; border-radius: 4px; }

        #ui-layer { position: absolute; top: 30px; left: 30px; pointer-events: none; z-index: 10; display: none; }
        #progress-bar-container { width: 220px; height: 8px; background: rgba(255,255,255,0.1); border-radius: 4px; overflow: hidden; margin-top: 10px; border: 1px solid rgba(0,255,157,0.3); }
        #progress-bar { width: 0%; height: 100%; background: linear-gradient(90deg, #00ff9d, #00f2ff); transition: width 0.3s; }
        
        canvas { display: block; }
        .hidden { opacity: 0; pointer-events: none; }
        
        #name-input-ui { display: none; position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); background: #111; padding: 40px; border: 2px solid #ffaa00; z-index: 200; text-align: center; box-shadow: 0 0 50px rgba(255,170,0,0.3); }
        input { background: #000; border: 1px solid #00ff9d; color: #fff; padding: 12px; font-size: 20px; text-align: center; width: 200px; margin-bottom: 25px; outline: none; }
    </style>
</head>
<body>

<div id="game-overlay" class="overlay">
    <h1>氙基原點</h1>
    <p class="sub-title">XENOBASE ORIGIN</p>
    <div class="btn-group">
        <button onclick="startGame('LEVEL')">關卡模式</button>
        <button onclick="startGame('INFINITE')">無限挑戰</button>
    </div>
    <div id="leaderboard-ui">
        <div style="text-align:center; color:#ffaa00; margin-bottom:12px; font-weight:900;">榮譽排行榜 (TOP 5)</div>
        <div id="leader-list"></div>
    </div>
</div>

<div id="name-input-ui">
    <h2 style="color:#ffaa00; margin-top:0;">成就達成！</h2>
    <p>輸入你的生物代碼以記錄能量：</p>
    <input type="text" id="player-name" maxlength="10" placeholder="你的名字">
    <br>
    <button onclick="saveScore()">確認上傳</button>
</div>

<div id="ui-layer">
    <div id="level-display" style="color:#ffaa00; font-weight:bold; font-size: 14px; letter-spacing:1px;"></div>
    <div id="progress-bar-container"><div id="progress-bar"></div></div>
    <p id="score-text" style="font-size: 32px; font-weight: 900; margin: 10px 0 0 0;">能量: 0</p>
</div>

<canvas id="gameCanvas"></canvas>

<script>
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");
    const overlay = document.getElementById("game-overlay");
    const uiLayer = document.getElementById("ui-layer");
    const progressBar = document.getElementById("progress-bar");
    const scoreText = document.getElementById("score-text");
    const levelDisplay = document.getElementById("level-display");
    const leaderList = document.getElementById("leader-list");
    const nameInputUI = document.getElementById("name-input-ui");

    let gameMode = 'LEVEL', currentLevelIdx = 0, score = 0, dx = 0, dy = 0, snake = [], foods = [], obstacles = [], enemies = [];
    let isPlaying = false, frameCount = 0;
    
    const levels = [
        { id: 1, name: "星系覺醒", target: 10, speed: 100, obs: 3, bigProb: 0.2 },
        { id: 2, name: "能量風暴", target: 20, speed: 85, obs: 5, bigProb: 0.4 },
        { id: 3, name: "終極演化", target: 25, speed: 75, obs: 8, bigProb: 0.7 } 
    ];

    function res() { canvas.width = window.innerWidth; canvas.height = window.innerHeight; }
    window.addEventListener('resize', res); res();

    function getLeaders() { return JSON.parse(localStorage.getItem('xenoLeaders_v2') || '[]'); }
    function updateLeaderUI() {
        const leaders = getLeaders();
        leaderList.innerHTML = leaders.length ? '' : '<div style="text-align:center; opacity:0.4">等待勇者挑戰...</div>';
        leaders.forEach((l, i) => {
            leaderList.innerHTML += `<div class="leader-item"><span>${i+1}. ${l.name}</span><span>${l.score}</span></div>`;
        });
    }
    updateLeaderUI();

    function startGame(mode) {
        gameMode = mode;
        overlay.classList.add('hidden');
        uiLayer.style.display = "block";
        currentLevelIdx = 0;
        isPlaying = true;
        initGame();
        gameLoop();
    }

    function initGame() {
        const tx = Math.floor(canvas.width/30), ty = Math.floor(canvas.height/30);
        snake = [{x: Math.floor(tx/2), y: Math.floor(ty/2)}];
        score = 0; dx = 0; dy = 0; obstacles = []; enemies = [];
        if (gameMode === 'LEVEL') {
            const lv = levels[currentLevelIdx];
            for(let i=0; i<lv.obs; i++) spawnObstacle();
            if(lv.id >= 2) spawnEnemies(lv.id === 2 ? 4 : 6);
        } else {
            for(let i=0; i<4; i++) spawnObstacle();
            spawnEnemies(3);
        }
        foods = Array.from({length: 8}, spawnFood);
        updateUI();
    }

    function spawnObstacle() {
        const tx = Math.floor(canvas.width/30), ty = Math.floor(canvas.height/30);
        obstacles.push({ x: Math.floor(Math.random()*(tx-4)+2), y: Math.floor(Math.random()*(ty-4)+2), size: Math.random()*1.1+1.3, hue: Math.random()*360 });
    }

    function spawnEnemies(count) {
        const tx = Math.floor(canvas.width/30), ty = Math.floor(canvas.height/30);
        for(let i=0; i<count; i++) enemies.push({ x: Math.floor(Math.random()*tx), y: Math.floor(Math.random()*ty) });
    }

    function spawnFood() {
        const tx = Math.floor(canvas.width/30), ty = Math.floor(canvas.height/30);
        const prob = gameMode === 'LEVEL' ? levels[currentLevelIdx].bigProb : 0.35;
        return { x: Math.floor(Math.random()*tx), y: Math.floor(Math.random()*ty), type: Math.random() < prob ? 'BIG' : 'SMALL', hue: Math.random()*360 };
    }

    function gameLoop() {
        if (!isPlaying) return;
        if (checkHit()) { isPlaying = false; handleGameOver(); return; }
        if (gameMode === 'LEVEL' && score >= levels[currentLevelIdx].target) {
            if (currentLevelIdx < 2) { currentLevelIdx++; initGame(); } 
            else { alert("恭喜通關！你是星系的主宰！"); location.reload(); return; }
        }
        if (gameMode === 'INFINITE' && frameCount % 1200 === 0) { spawnEnemies(1); if(frameCount % 2400 === 0) spawnObstacle(); }
        
        setTimeout(() => { frameCount++; draw(); moveSnake(); if(gameMode==='INFINITE'||currentLevelIdx===2) moveEnemies(); gameLoop(); }, gameMode === 'LEVEL' ? levels[currentLevelIdx].speed : 80);
    }

    function moveSnake() {
        let nx = snake[0].x+dx, ny = snake[0].y+dy;
        const tx = Math.floor(canvas.width/30), ty = Math.floor(canvas.height/30);
        if (nx<0) nx=tx-1; if (nx>=tx) nx=0; if (ny<0) ny=ty-1; if (ny>=ty) ny=0;

        enemies.forEach((en, ei) => {
            if (snake.some(p => p.x === en.x && p.y === en.y)) {
                score = Math.max(0, score - 5); updateUI();
                enemies[ei] = { x: Math.floor(Math.random()*tx), y: Math.floor(Math.random()*ty) };
            }
        });

        snake.unshift({x: nx, y: ny});
        let fi = foods.findIndex(f => f.x===nx && f.y===ny);
        if (fi !== -1) {
            score += (foods[fi].type === 'BIG' ? 5 : 1);
            updateUI(); foods[fi] = spawnFood();
        } else if (snake.length > 3) snake.pop();
    }

    function moveEnemies() {
        if (frameCount % 6 !== 0) return;
        const head = snake[0];
        enemies.forEach(en => { if (Math.random() < 0.3) { if (en.x < head.x) en.x++; else if (en.x > head.x) en.x--; if (en.y < head.y) en.y++; else if (en.y > head.y) en.y--; } });
    }

    function checkHit() {
        if(dx===0 && dy===0) return false;
        const h = snake[0];
        return obstacles.some(o => Math.sqrt((o.x+0.5-h.x-0.5)**2 + (o.y+0.5-h.y-0.5)**2) < o.size * 0.75) || snake.slice(4).some(p => p.x===h.x && p.y===h.y);
    }

    function handleGameOver() {
        if (gameMode === 'INFINITE') {
            const leaders = getLeaders();
            if (leaders.length < 5 || score > leaders[leaders.length-1].score) { nameInputUI.style.display = "block"; return; }
        }
        alert(`能量崩潰！得分: ${score}`); location.reload();
    }

    function saveScore() {
        const name = document.getElementById("player-name").value || "匿名星塵";
        let leaders = getLeaders();
        leaders.push({ name, score });
        leaders.sort((a, b) => b.score - a.score);
        localStorage.setItem('xenoLeaders_v2', JSON.stringify(leaders.slice(0, 5)));
        location.reload();
    }

    function updateUI() {
        scoreText.innerText = `能量: ${score}`;
        if (gameMode === 'LEVEL') {
            const lv = levels[currentLevelIdx];
            levelDisplay.innerText = `[關卡] ${lv.name}`;
            progressBar.style.width = (score / lv.target * 100) + "%";
        } else {
            levelDisplay.innerText = `[無限] 穩定演化中...`;
            progressBar.style.width = "100%";
        }
    }

    function draw() {
        ctx.fillStyle = "#010103"; ctx.fillRect(0, 0, canvas.width, canvas.height);
        
        obstacles.forEach(o => {
            const px = o.x*30+15, py = o.y*30+15, r = 30*o.size;
            ctx.save();
            let g = ctx.createRadialGradient(px-r*0.2, py-r*0.2, r*0.1, px, py, r);
            g.addColorStop(0, '#fff'); g.addColorStop(0.3, `hsl(${o.hue},70%,40%)`); g.addColorStop(1, '#000');
            ctx.shadowBlur = 40; ctx.shadowColor = `hsl(${o.hue},80%,50%)`;
            ctx.fillStyle = g; ctx.beginPath(); ctx.arc(px, py, r, 0, Math.PI*2); ctx.fill();
            ctx.restore();
        });

        // --- 敵人繪製 ---
        enemies.forEach(en => {
            const px = en.x*30+15, py = en.y*30+15;
            ctx.save(); ctx.translate(px, py); ctx.rotate(frameCount * 0.1);
            let g = ctx.createRadialGradient(0, 0, 0, 0, 0, 22);
            g.addColorStop(0, 'rgba(255, 0, 0, 1)'); g.addColorStop(1, 'transparent');
            ctx.shadowBlur = 20; ctx.shadowColor = "red"; ctx.fillStyle = g;
            ctx.beginPath(); ctx.arc(0, 0, 22, 0, Math.PI*2); ctx.fill();
            ctx.strokeStyle = "#fff"; ctx.lineWidth = 2; ctx.beginPath();
            for(let i=0; i<8; i++) {
                let a = i * Math.PI / 4;
                let r = i % 2 === 0 ? 14 : 6;
                ctx.lineTo(Math.cos(a)*r, Math.sin(a)*r);
            }
            ctx.closePath(); ctx.stroke(); ctx.restore();
        });

        // --- 食物繪製 (特別修復大能量球) ---
        foods.forEach(f => {
            const isBig = f.type === 'BIG';
            const px = f.x*30+15, py = f.y*30+15;
            ctx.save();
            if (isBig) {
                // 大能量球：強烈發光、彩色漸變、動態縮放
                let pulse = Math.sin(frameCount / 5) * 3;
                let r = 10 + pulse;
                let g = ctx.createRadialGradient(px, py, 2, px, py, r + 10);
                g.addColorStop(0, '#fff');
                g.addColorStop(0.4, `hsl(${f.hue}, 100%, 70%)`);
                g.addColorStop(1, 'transparent');
                ctx.shadowBlur = 30;
                ctx.shadowColor = `hsl(${f.hue}, 100%, 50%)`;
                ctx.fillStyle = g;
                ctx.beginPath(); ctx.arc(px, py, r + 10, 0, Math.PI*2); ctx.fill();
                // 核心
                ctx.fillStyle = "#fff";
                ctx.beginPath(); ctx.arc(px, py, 6, 0, Math.PI*2); ctx.fill();
            } else {
                // 小能量球：簡單閃爍
                ctx.shadowBlur = 10;
                ctx.shadowColor = `hsl(${f.hue}, 100%, 60%)`;
                ctx.fillStyle = `hsl(${f.hue}, 100%, 75%)`;
                ctx.beginPath(); ctx.arc(px, py, 5 + Math.sin(frameCount/8)*2, 0, Math.PI*2); ctx.fill();
            }
            ctx.restore();
        });

        snake.forEach((p, i) => {
            ctx.save();
            ctx.shadowBlur = i===0 ? 20 : 5; ctx.shadowColor = "#00ff9d";
            ctx.fillStyle = i===0 ? "#00ff9d" : "rgba(0, 255, 157, 0.4)";
            ctx.beginPath(); ctx.roundRect(p.x*30+3, p.y*30+3, 24, 24, i===0?8:4); ctx.fill();
            ctx.restore();
        });
    }

    window.addEventListener("keydown", e => {
        const k = { 37: [-1,0], 38: [0,-1], 39: [1,0], 40: [0,1] };
        if (k[e.keyCode]) {
            const [nx, ny] = k[e.keyCode];
            if (nx!==-dx || ny!==-dy || snake.length===1) { dx=nx; dy=ny; }
        }
    });

    // 觸控方向判斷
    let tsX = 0, tsY = 0;
    document.addEventListener('touchstart', e => { tsX = e.touches[0].clientX; tsY = e.touches[0].clientY; }, {passive: false});
    document.addEventListener('touchend', e => {
        let dx_raw = e.changedTouches[0].clientX - tsX, dy_raw = e.changedTouches[0].clientY - tsY;
        if (Math.abs(dx_raw) > 20 || Math.abs(dy_raw) > 20) {
            if (Math.abs(dx_raw) > Math.abs(dy_raw)) {
                let nDx = dx_raw > 0 ? 1 : -1;
                if (nDx !== -dx || snake.length === 1) { dx = nDx; dy = 0; }
            } else {
                let nDy = dy_raw > 0 ? 1 : -1;
                if (nDy !== -dy || snake.length === 1) { dx = 0; dy = nDy; }
            }
        }
    }, {passive: false});
</script>
</body>
</html>
