<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Stellar Vanguard</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #000;
            color: white;
            font-family: 'Courier New', Courier, monospace;
        }
        canvas {
            display: block;
        }
        #ui {
            position: absolute;
            top: 10px;
            left: 20px;
            right: 20px;
            display: flex;
            justify-content: space-between;
            pointer-events: none;
            text-shadow: 2px 2px 4px #000;
        }
        .stat { font-size: 24px; font-weight: bold; color: #00ffff; }
        
        #boss-ui {
            position: absolute;
            top: 50px;
            left: 50%;
            transform: translateX(-50%);
            width: 400px;
            display: none;
            text-align: center;
        }
        #boss-bar-container {
            width: 100%;
            height: 20px;
            border: 2px solid #ff0000;
            background: rgba(255,0,0,0.2);
            margin-top: 5px;
        }
        #boss-bar {
            width: 100%;
            height: 100%;
            background: #ff0000;
            transition: width 0.1s;
        }
        
        #messages {
            position: absolute;
            top: 40%;
            left: 50%;
            transform: translate(-50%, -50%);
            text-align: center;
        }
        h1 { font-size: 50px; color: #ff0055; margin-bottom: 10px; text-shadow: 0 0 10px #ff0055; }
        button {
            padding: 15px 30px;
            font-size: 20px;
            background: transparent;
            color: #00ffff;
            border: 2px solid #00ffff;
            cursor: pointer;
            text-transform: uppercase;
            font-weight: bold;
        }
        button:hover { background: #00ffff; color: #000; }
        .hidden { display: none; }
        #controls {
            position: absolute;
            bottom: 20px;
            width: 100%;
            text-align: center;
            color: #888;
            pointer-events: none;
        }
    </style>
</head>
<body>

    <div id="ui">
        <div class="stat">PUNTI: <span id="score">0</span></div>
        <div class="stat">VITA: <span id="hp">100</span>%</div>
    </div>

    <div id="boss-ui">
        <div style="color: #ff0000; font-weight: bold; font-size: 20px;">NAVE MADRE INIMICA</div>
        <div id="boss-bar-container"><div id="boss-bar"></div></div>
    </div>

    <div id="messages">
        <div id="start-screen">
            <h1 style="color: #00ffff;">STELLAR VANGUARD</h1>
            <p>Sopravvivi, potenziati e sconfiggi il Boss.</p>
            <button onclick="startGame()">INIZIA MISSIONE</button>
        </div>
        <div id="game-over" class="hidden">
            <h1>NAVE DISTRUTTA</h1>
            <p style="font-size: 24px;">Punti Finali: <span id="final-score">0</span></p>
            <button onclick="startGame()">RIPROVA</button>
        </div>
        <div id="victory" class="hidden">
            <h1 style="color: #00ff00; text-shadow: 0 0 10px #00ff00;">VITTORIA!</h1>
            <p style="font-size: 24px;">Hai salvato la galassia!</p>
            <button onclick="startGame()">GIOCA ANCORA</button>
        </div>
    </div>

    <div id="controls" class="hidden">WASD / Frecce: Muovi | SPAZIO: Spara</div>

    <canvas id="gameCanvas"></canvas>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;

        // Gestione Input (Tastiera fluida)
        const keys = { w: false, a: false, s: false, d: false, space: false };
        window.addEventListener('keydown', e => {
            if(e.code === 'KeyW' || e.code === 'ArrowUp') keys.w = true;
            if(e.code === 'KeyA' || e.code === 'ArrowLeft') keys.a = true;
            if(e.code === 'KeyS' || e.code === 'ArrowDown') keys.s = true;
            if(e.code === 'KeyD' || e.code === 'ArrowRight') keys.d = true;
            if(e.code === 'Space') keys.space = true;
        });
        window.addEventListener('keyup', e => {
            if(e.code === 'KeyW' || e.code === 'ArrowUp') keys.w = false;
            if(e.code === 'KeyA' || e.code === 'ArrowLeft') keys.a = false;
            if(e.code === 'KeyS' || e.code === 'ArrowDown') keys.s = false;
            if(e.code === 'KeyD' || e.code === 'ArrowRight') keys.d = false;
            if(e.code === 'Space') keys.space = false;
        });

        // Variabili Globali
        let gameActive = false;
        let score = 0;
        let frameCount = 0;
        let entities = { player: null, bullets: [], enemies: [], particles: [], powerups: [], boss: null };
        let stars = [];

        // Generazione Stelle Parallax
        for(let i=0; i<150; i++) {
            stars.push({
                x: Math.random() * canvas.width,
                y: Math.random() * canvas.height,
                size: Math.random() * 2,
                speed: Math.random() * 3 + 0.5
            });
        }

        // --- CLASSI DEL GIOCO ---

        class Player {
            constructor() {
                this.x = canvas.width / 2;
                this.y = canvas.height - 100;
                this.speed = 7;
                this.radius = 15;
                this.hp = 100;
                this.weaponLevel = 1;
                this.cooldown = 0;
                this.color = '#00ffff';
            }
            update() {
                // Movimento
                if (keys.w && this.y > 30) this.y -= this.speed;
                if (keys.s && this.y < canvas.height - 30) this.y += this.speed;
                if (keys.a && this.x > 30) this.x -= this.speed;
                if (keys.d && this.x < canvas.width - 30) this.x += this.speed;

                // Fuoco
                if (this.cooldown > 0) this.cooldown--;
                if (keys.space && this.cooldown === 0) this.shoot();
            }
            shoot() {
                this.cooldown = 15;
                if (this.weaponLevel === 1) {
                    entities.bullets.push(new Bullet(this.x, this.y - 20, 0, -10, this.color, 'player'));
                } else if (this.weaponLevel === 2) {
                    entities.bullets.push(new Bullet(this.x - 10, this.y - 20, 0, -10, this.color, 'player'));
                    entities.bullets.push(new Bullet(this.x + 10, this.y - 20, 0, -10, this.color, 'player'));
                } else {
                    entities.bullets.push(new Bullet(this.x, this.y - 20, 0, -10, this.color, 'player'));
                    entities.bullets.push(new Bullet(this.x - 15, this.y - 15, -3, -9, this.color, 'player'));
                    entities.bullets.push(new Bullet(this.x + 15, this.y - 15, 3, -9, this.color, 'player'));
                }
            }
            draw() {
                ctx.save();
                ctx.translate(this.x, this.y);
                ctx.beginPath();
                ctx.moveTo(0, -20);
                ctx.lineTo(15, 15);
                ctx.lineTo(0, 5);
                ctx.lineTo(-15, 15);
                ctx.closePath();
                ctx.fillStyle = this.color;
                ctx.shadowBlur = 15;
                ctx.shadowColor = this.color;
                ctx.fill();
                ctx.restore();
            }
            takeDamage(amount) {
                this.hp -= amount;
                if (this.hp < 0) this.hp = 0;
                document.getElementById('hp').innerText = this.hp;
                createExplosion(this.x, this.y, this.color, 10);
                if (this.hp <= 0) endGame(false);
            }
        }

        class Bullet {
            constructor(x, y, vx, vy, color, owner) {
                this.x = x; this.y = y;
                this.vx = vx; this.vy = vy;
                this.color = color;
                this.owner = owner; // 'player' o 'enemy'
                this.radius = 4;
            }
            update() {
                this.x += this.vx;
                this.y += this.vy;
            }
            draw() {
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.radius, 0, Math.PI*2);
                ctx.fillStyle = this.color;
                ctx.shadowBlur = 10;
                ctx.shadowColor = this.color;
                ctx.fill();
            }
        }

        class Enemy {
            constructor(type) {
                this.x = Math.random() * (canvas.width - 100) + 50;
                this.y = -50;
                this.type = type; // 1: Dritto, 2: Zig-Zag
                this.hp = type === 1 ? 20 : 30;
                this.radius = 15;
                this.color = type === 1 ? '#ff0055' : '#ff9900';
                this.startX = this.x;
                this.time = 0;
            }
            update() {
                this.time += 0.05;
                if (this.type === 1) {
                    this.y += 3; // Scende dritto
                } else {
                    this.y += 2;
                    this.x = this.startX + Math.sin(this.time) * 100; // Zig-zag
                }

                // Spara casualmente
                if (Math.random() < 0.01) {
                    entities.bullets.push(new Bullet(this.x, this.y + 20, 0, 5, '#ff0000', 'enemy'));
                }
            }
            draw() {
                ctx.save();
                ctx.translate(this.x, this.y);
                ctx.beginPath();
                ctx.moveTo(0, 15);
                ctx.lineTo(15, -15);
                ctx.lineTo(-15, -15);
                ctx.closePath();
                ctx.fillStyle = this.color;
                ctx.shadowBlur = 10;
                ctx.shadowColor = this.color;
                ctx.fill();
                ctx.restore();
            }
        }

        class Boss {
            constructor() {
                this.x = canvas.width / 2;
                this.y = -100;
                this.hp = 1000;
                this.maxHp = 1000;
                this.radius = 40;
                this.color = '#ff0000';
                this.phase = 'entering'; // entering, attacking
                this.direction = 1;
                this.attackTimer = 0;
            }
            update() {
                if (this.phase === 'entering') {
                    this.y += 1;
                    if (this.y >= 150) this.phase = 'attacking';
                } else if (this.phase === 'attacking') {
                    // Movimento laterale
                    this.x += 3 * this.direction;
                    if (this.x > canvas.width - 100 || this.x < 100) this.direction *= -1;

                    // Pattern d'attacco a raggiera
                    this.attackTimer++;
                    if (this.attackTimer > 60) {
                        this.attackTimer = 0;
                        for(let i=0; i<8; i++) {
                            let angle = (i * Math.PI / 4) + (Math.random() * 0.5);
                            let vx = Math.cos(angle) * 5;
                            let vy = Math.sin(angle) * 5;
                            entities.bullets.push(new Bullet(this.x, this.y, vx, vy, '#ff00ff', 'enemy'));
                        }
                    }
                }
                // Aggiorna UI Boss
                document.getElementById('boss-bar').style.width = (this.hp / this.maxHp * 100) + '%';
            }
            draw() {
                ctx.save();
                ctx.translate(this.x, this.y);
                ctx.beginPath();
                ctx.arc(0, 0, this.radius, 0, Math.PI*2);
                ctx.fillStyle = '#220000';
                ctx.fill();
                ctx.lineWidth = 5;
                ctx.strokeStyle = this.color;
                ctx.stroke();
                
                // Occhio del boss
                ctx.beginPath();
                ctx.arc(0, 0, 15, 0, Math.PI*2);
                ctx.fillStyle = this.color;
                ctx.shadowBlur = 20;
                ctx.shadowColor = this.color;
                ctx.fill();
                ctx.restore();
            }
        }

        class PowerUp {
            constructor(x, y) {
                this.x = x; this.y = y;
                this.radius = 12;
                this.color = '#ffff00';
            }
            update() { this.y += 2; }
            draw() {
                ctx.beginPath();
                ctx.rect(this.x - 10, this.y - 10, 20, 20);
                ctx.fillStyle = this.color;
                ctx.shadowBlur = 10;
                ctx.shadowColor = this.color;
                ctx.fill();
                ctx.fillStyle = '#000';
                ctx.font = '16px Arial';
                ctx.fillText('W', this.x - 7, this.y + 6);
            }
        }

        class Particle {
            constructor(x, y, color, speed) {
                this.x = x; this.y = y; this.color = color;
                this.radius = Math.random() * 3 + 1;
                let angle = Math.random() * Math.PI * 2;
                let v = Math.random() * speed;
                this.vx = Math.cos(angle) * v;
                this.vy = Math.sin(angle) * v;
                this.alpha = 1;
            }
            update() {
                this.x += this.vx; this.y += this.vy;
                this.alpha -= 0.03;
            }
            draw() {
                ctx.globalAlpha = Math.max(0, this.alpha);
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.radius, 0, Math.PI*2);
                ctx.fillStyle = this.color;
                ctx.fill();
                ctx.globalAlpha = 1;
            }
        }

        // --- FUNZIONI DI UTILITÀ ---

        function createExplosion(x, y, color, count) {
            for(let i=0; i<count; i++) entities.particles.push(new Particle(x, y, color, 6));
        }

        function checkCollision(obj1, obj2) {
            const dist = Math.hypot(obj1.x - obj2.x, obj1.y - obj2.y);
            return dist < obj1.radius + obj2.radius;
        }

        function startGame() {
            document.getElementById('messages').querySelectorAll('div').forEach(el => el.classList.add('hidden'));
            document.getElementById('controls').classList.remove('hidden');
            document.getElementById('boss-ui').style.display = 'none';
            
            entities = { player: new Player(), bullets: [], enemies: [], particles: [], powerups: [], boss: null };
            score = 0;
            frameCount = 0;
            document.getElementById('score').innerText = score;
            document.getElementById('hp').innerText = 100;
            gameActive = true;
            animate();
        }

        function endGame(victory) {
            gameActive = false;
            document.getElementById('controls').classList.add('hidden');
            document.getElementById('boss-ui').style.display = 'none';
            if (victory) {
                document.getElementById('victory').classList.remove('hidden');
            } else {
                document.getElementById('final-score').innerText = score;
                document.getElementById('game-over').classList.remove('hidden');
            }
        }

        // --- LOOP PRINCIPALE ---

        function animate() {
            if (!gameActive) return;
            requestAnimationFrame(animate);
            frameCount++;

            // Sfondo Parallax
            ctx.fillStyle = 'rgba(0, 0, 10, 0.5)'; // Effetto scia
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            ctx.fillStyle = '#ffffff';
            stars.forEach(star => {
                star.y += star.speed;
                if (star.y > canvas.height) { star.y = 0; star.x = Math.random() * canvas.width; }
                ctx.beginPath(); ctx.arc(star.x, star.y, star.size, 0, Math.PI*2); ctx.fill();
            });

            const p = entities.player;
            p.update();
            p.draw();

            // Spawn Nemici Regolari (se il boss non c'è)
            if (!entities.boss && frameCount % 60 === 0) {
                entities.enemies.push(new Enemy(Math.random() < 0.5 ? 1 : 2));
            }

            // Spawn Boss
            if (score >= 500 && !entities.boss) {
                entities.boss = new Boss();
                entities.enemies = []; // Pulisce i nemici minori
                document.getElementById('boss-ui').style.display = 'block';
            }

            // Aggiorna Boss
            if (entities.boss) {
                let b = entities.boss;
                b.update(); b.draw();
                // Collisione giocatore-boss
                if (checkCollision(p, b)) p.takeDamage(1);
            }

            // Aggiorna Nemici Minori
            for (let i = entities.enemies.length - 1; i >= 0; i--) {
                let e = entities.enemies[i];
                e.update(); e.draw();
                if (e.y > canvas.height + 50) entities.enemies.splice(i, 1);
                else if (checkCollision(p, e)) {
                    p.takeDamage(20);
                    createExplosion(e.x, e.y, e.color, 15);
                    entities.enemies.splice(i, 1);
                }
            }

            // Aggiorna Proiettili e Collisioni
            for (let i = entities.bullets.length - 1; i >= 0; i--) {
                let b = entities.bullets[i];
                b.update(); b.draw();

                // Rimuovi se fuori schermo
                if (b.y < -50 || b.y > canvas.height + 50 || b.x < -50 || b.x > canvas.width + 50) {
                    entities.bullets.splice(i, 1);
                    continue;
                }

                if (b.owner === 'player') {
                    // Controllo proiettili giocatore contro Boss
                    if (entities.boss && entities.boss.phase === 'attacking' && checkCollision(b, entities.boss)) {
                        entities.boss.hp -= 10;
                        createExplosion(b.x, b.y, '#ffff00', 3);
                        entities.bullets.splice(i, 1);
                        if (entities.boss.hp <= 0) {
                            createExplosion(entities.boss.x, entities.boss.y, '#ff0000', 100);
                            entities.boss = null;
                            score += 1000;
                            document.getElementById('score').innerText = score;
                            setTimeout(() => endGame(true), 2000); // Vittoria!
                        }
                        continue;
                    }

                    // Controllo proiettili giocatore contro Nemici
                    for (let j = entities.enemies.length - 1; j >= 0; j--) {
                        let e = entities.enemies[j];
                        if (checkCollision(b, e)) {
                            e.hp -= 10;
                            entities.bullets.splice(i, 1);
                            if (e.hp <= 0) {
                                createExplosion(e.x, e.y, e.color, 15);
                                score += 10;
                                document.getElementById('score').innerText = score;
                                // 10% di probabilità di droppare un powerup
                                if(Math.random() < 0.1 && p.weaponLevel < 3) {
                                    entities.powerups.push(new PowerUp(e.x, e.y));
                                }
                                entities.enemies.splice(j, 1);
                            }
                            break;
                        }
                    }
                } else if (b.owner === 'enemy') {
                    // Proiettili nemici contro giocatore
                    if (checkCollision(b, p)) {
                        p.takeDamage(10);
                        createExplosion(b.x, b.y, p.color, 5);
                        entities.bullets.splice(i, 1);
                    }
                }
            }

            // Aggiorna Powerups
            for (let i = entities.powerups.length - 1; i >= 0; i--) {
                let pup = entities.powerups[i];
                pup.update(); pup.draw();
                if (pup.y > canvas.height) {
                    entities.powerups.splice(i, 1);
                } else if (checkCollision(p, pup)) {
                    p.weaponLevel++;
                    if(p.weaponLevel > 3) p.weaponLevel = 3; // Max livello 3
                    createExplosion(pup.x, pup.y, pup.color, 20);
                    entities.powerups.splice(i, 1);
                }
            }

            // Aggiorna Particelle
            for (let i = entities.particles.length - 1; i >= 0; i--) {
                let pt = entities.particles[i];
                pt.update(); pt.draw();
                if (pt.alpha <= 0) entities.particles.splice(i, 1);
            }
        }
    </script>
</body>
</html>
