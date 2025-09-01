<!doctype html>
<html lang="id">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Heading Sepak Bola (Otomatis)</title>
<style>
  :root{
    --bg-top: #157a2f;
    --bg-bot: #0b5e0b;
    --panel: #ffffffcc;
  }
  html,body{height:100%;margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial;background:linear-gradient(180deg,var(--bg-top),var(--bg-bot));color:#fff}
  .center{display:flex;align-items:center;justify-content:center;height:100%}
  .wrap{width:min(920px,96vw);max-width:920px;padding:12px;box-sizing:border-box;position:relative}
  canvas{width:100%;height:auto;display:block;border-radius:12px;background:linear-gradient(180deg,#1a8a39,#0b5e0b);box-shadow:0 18px 50px #0008}
  .hud{display:flex;justify-content:space-between;align-items:center;margin:10px 6px;gap:10px}
  .tag{background:var(--panel);color:#0b3d0b;padding:8px 12px;border-radius:999px;font-weight:700}
  button{cursor:pointer}
  .btn-muted{background:rgba(255,255,255,0.12);color:#fff;border:0;padding:8px 12px;border-radius:8px}
  /* virtual controls */
  .touch-row{position:absolute;left:0;right:0;bottom:18px;display:flex;justify-content:center;pointer-events:none}
  .vc{display:flex;gap:14px;pointer-events:auto}
  .vc-btn{width:72px;height:72px;border-radius:50%;border:2px solid rgba(255,255,255,0.22);background:rgba(255,255,255,0.06);color:#fff;font-weight:800;font-size:20px;display:grid;place-items:center}
  @media(max-width:420px){ .vc-btn{width:60px;height:60px;font-size:18px} }
  /* menu overlay */
  .overlay{position:absolute;inset:0;display:flex;align-items:center;justify-content:center;backdrop-filter: blur(4px)}
  .menu{background:rgba(0,0,0,0.45);padding:18px;border-radius:12px;text-align:center}
  .menu h1{margin:0 0 8px;font-size:20px}
  .menu p{margin:0 0 12px;opacity:0.9}
  .menu .mrow{display:flex;gap:8px;justify-content:center;margin-top:8px}
  .small{font-size:13px;opacity:0.9;margin-top:8px}
</style>
</head>
<body>
  <div class="center">
    <div class="wrap">
      <div class="hud">
        <div style="display:flex;gap:10px;align-items:center">
          <div class="tag" id="scoreTag">Skor: 0</div>
          <div class="tag" id="bestTag">Rekor: 0</div>
        </div>
        <div style="display:flex;gap:10px;align-items:center">
          <div class="tag" id="livesTag">❤️❤️❤️</div>
          <button id="restartBtn" class="btn-muted">Mulai Ulang</button>
        </div>
      </div>

      <canvas id="game" width="800" height="500" aria-label="Game Heading Sepak Bola Otomatis"></canvas>

      <!-- virtual buttons (HP) -->
      <div class="touch-row" id="touchRow">
        <div class="vc">
          <div class="vc-btn" id="vc-left">◀</div>
          <div style="width:16px"></div>
          <div class="vc-btn" id="vc-right">▶</div>
        </div>
      </div>

      <!-- menu overlay -->
      <div class="overlay" id="menuOverlay">
        <div class="menu">
          <h1>Heading Sepak Bola (Otomatis)</h1>
          <p>Pilih tingkat kesulitan — bola akan mulai tepat di atas kepala pemain. Kontrol: ← → atau A/D (laptop) | tombol kiri/kanan (HP).</p>
          <div class="mrow">
            <button onclick="start('easy')" style="padding:8px 12px;border-radius:8px;border:0;background:#fff;color:#0b5e0b;font-weight:700">Mudah</button>
            <button onclick="start('normal')" style="padding:8px 12px;border-radius:8px;border:0;background:#ffce00;color:#123;font-weight:700">Sedang</button>
            <button onclick="start('hard')" style="padding:8px 12px;border-radius:8px;border:0;background:#e74c3c;color:#fff;font-weight:700">Sulit</button>
          </div>
          <div class="small">Tidak perlu tombol sundul — jika bola menyentuh kepala, bola otomatis memantul.</div>
        </div>
      </div>

    </div>
  </div>

<script>
/* ====== Setup ====== */
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;

const scoreTag = document.getElementById('scoreTag');
const bestTag = document.getElementById('bestTag');
const livesTag = document.getElementById('livesTag');
const restartBtn = document.getElementById('restartBtn');
const menuOverlay = document.getElementById('menuOverlay');

let state = {
  running: false,
  score: 0,
  best: Number(localStorage.getItem('heading_best') || 0),
  lives: 3
};
bestTag.textContent = `Rekor: ${state.best}`;

let player, ball;
let gravity = 0.15;
let keys = { left:false, right:false };
let vcLeft = document.getElementById('vc-left');
let vcRight = document.getElementById('vc-right');
let headCooldown = 0;

/* Helpers */
const clamp = (v,a,b) => Math.max(a, Math.min(b, v));
const rand = (a,b) => Math.random()*(b-a)+a;

/* ====== Entities ====== */
function spawnPlayer(){
  player = {
    x: W/2,
    y: H - 80,
    w: 60,
    h: 18,
    speed: 5,
    headR: 20
  };
}

function spawnBallAboveHead(){
  ball = {
    x: player.x + rand(-20,20),
    y: player.y - 150,
    r: 16,
    vx: rand(-1.0,1.0),
    vy: rand(0.6,1.3),
    touchedThisFall: false
  };
}

/* ====== Reset & HUD ====== */
function resetGame(){
  state.running = true;
  state.score = 0;
  state.lives = 3;
  updateHUD();
  spawnPlayer();
  spawnBallAboveHead();
  headCooldown = 0;
}

function updateHUD(){
  scoreTag.textContent = `Skor: ${state.score}`;
  bestTag.textContent = `Rekor: ${state.best}`;
  livesTag.textContent = '❤️'.repeat(state.lives) + '🖤'.repeat(3-state.lives);
}

/* ====== Input ====== */
window.addEventListener('keydown', e=>{
  if(e.code === 'ArrowLeft' || e.code === 'KeyA') keys.left = true;
  if(e.code === 'ArrowRight' || e.code === 'KeyD') keys.right = true;
});
window.addEventListener('keyup', e=>{
  if(e.code === 'ArrowLeft' || e.code === 'KeyA') keys.left = false;
  if(e.code === 'ArrowRight' || e.code === 'KeyD') keys.right = false;
});

/* Virtual controls touch/hold */
function bindHold(el, cb){
  let holding = false;
  const start = (ev) => { ev.preventDefault(); holding = true; cb(true); };
  const end = (ev) => { if(holding){ holding=false; cb(false); } };
  el.addEventListener('touchstart', start, {passive:false});
  el.addEventListener('mousedown', start);
  el.addEventListener('touchend', end);
  el.addEventListener('mouseup', end);
  el.addEventListener('mouseleave', end);
  el.addEventListener('touchcancel', end);
}
bindHold(vcLeft, v => keys.left = v);
bindHold(vcRight, v => keys.right = v);

/* Restart button */
restartBtn.addEventListener('click', () => {
  resetGame();
  menuOverlay.style.display = 'none';
});

/* ====== Collision: automatic heading ====== */
function checkHeadHit(){
  // head center coordinates
  const headX = player.x;
  const headY = player.y - player.h/2 - player.headR + 6; // visual adjustment
  const dx = ball.x - headX;
  const dy = ball.y - headY;
  const dist = Math.hypot(dx, dy);
  if(dist <= ball.r + player.headR + 6){
    // avoid repeated triggers due to frames: use cooldown
    if(headCooldown > 0) return;
    headCooldown = 12; // frames
    // mark touched while falling or anytime (easier)
    ball.touchedThisFall = true;
    // reflect velocity using simple normal reflection
    const nx = dx / (dist || 1);
    const ny = dy / (dist || 1);
    // relative velocity along normal
    const rel = ball.vx * nx + ball.vy * ny;
    // reflect
    ball.vx = ball.vx - 2 * rel * nx;
    ball.vy = ball.vy - 2 * rel * ny;
    // boost upward to simulate heading
    ball.vy -= 5 + Math.abs(ball.vy) * 0.15;
    // add horizontal kick based on where ball hits relative to player
    ball.vx += clamp((ball.x - player.x) * 0.05, -3, 3) + rand(-0.8, 0.8);
    // score
    state.score++;
    if(state.score > state.best){
      state.best = state.score;
      localStorage.setItem('heading_best', state.best);
    }
    updateHUD();
  }
}

/* ====== Update physics ====== */
function update(){
  if(!state.running) return;

  // move player
  if(keys.left && !keys.right) player.x -= player.speed;
  if(keys.right && !keys.left) player.x += player.speed;
  player.x = clamp(player.x, 30, W - 30);

  // cooldown
  if(headCooldown > 0) headCooldown--;

  // ball physics
  ball.vy += gravity;
  ball.x += ball.vx;
  ball.y += ball.vy;

  // walls
  if(ball.x - ball.r < 10){
    ball.x = 10 + ball.r;
    ball.vx *= -0.88;
  }
  if(ball.x + ball.r > W - 10){
    ball.x = W - 10 - ball.r;
    ball.vx *= -0.88;
  }

  // head collision check (automatic)
  checkHeadHit();

  // ground handling
  if(ball.y + ball.r >= H - 16){
    if(!ball.touchedThisFall){
      // missed
      state.lives--;
      if(state.lives <= 0){
        state.running = false;
        updateHUD();
        return;
      } else {
        // respawn ball above head to give next chance
        spawnBallAboveHead();
        updateHUD();
        return;
      }
    } else {
      // if already touched (sundul sebelumnya), let it bounce
      ball.y = H - 16 - ball.r;
      ball.vy *= -0.7;
      ball.vx *= 0.98;
      ball.touchedThisFall = false; // next fall can be missed
    }
  }

  // friction
  ball.vx *= 0.999;
}

/* ====== Render ====== */
function drawField(){
  // background gradient already set by canvas CSS, draw subtle stripes
  ctx.clearRect(0,0,W,H);
  const g = ctx.createLinearGradient(0,0,0,H);
  g.addColorStop(0,'#197b2f'); g.addColorStop(1,'#0b5e0b');
  ctx.fillStyle = g;
  ctx.fillRect(0,0,W,H);

  // baseline
  ctx.strokeStyle = 'rgba(255,255,255,0.12)';
  ctx.lineWidth = 2;
  ctx.setLineDash([10,12]);
  ctx.beginPath();
  ctx.moveTo(10,H-16); ctx.lineTo(W-10,H-16);
  ctx.stroke();
  ctx.setLineDash([]);
}

function drawPlayer(){
  ctx.save();
  ctx.translate(player.x, player.y);
  // simple stick figure
  ctx.strokeStyle = '#fff'; ctx.lineWidth = 6; ctx.lineCap = 'round';
  // legs
  ctx.beginPath(); ctx.moveTo(0,14); ctx.lineTo(-14,34); ctx.moveTo(0,14); ctx.lineTo(14,34); ctx.stroke();
  // body
  ctx.beginPath(); ctx.moveTo(0,-8); ctx.lineTo(0,14); ctx.stroke();
  // arms
  ctx.beginPath(); ctx.moveTo(0,-2); ctx.lineTo(-18,6); ctx.moveTo(0,-2); ctx.lineTo(18,6); ctx.stroke();
  // head
  const headY = -player.h/2 - player.headR + 6;
  ctx.beginPath(); ctx.fillStyle = '#ffe1bd'; ctx.arc(0, headY, player.headR, 0, Math.PI*2); ctx.fill();
  // cooldown halo
  if(headCooldown > 0){
    ctx.beginPath(); ctx.strokeStyle = 'rgba(255,235,59,0.95)'; ctx.lineWidth = 3;
    ctx.arc(0, headY, player.headR + 6, 0, Math.PI*2); ctx.stroke();
  }
  ctx.restore();
}

function drawBall(){
  ctx.save();
  ctx.translate(ball.x, ball.y);
  // shadow
  ctx.globalAlpha = 0.22;
  ctx.beginPath(); ctx.ellipse(0, 26, 22, 8, 0, 0, Math.PI*2); ctx.fillStyle = '#000'; ctx.fill();
  ctx.globalAlpha = 1;
  // ball
  ctx.beginPath(); ctx.fillStyle = '#fff'; ctx.arc(0,0,ball.r,0,Math.PI*2); ctx.fill();
  ctx.lineWidth = 2; ctx.strokeStyle = '#111'; ctx.stroke();
  // simple panels
  for(let i=0;i<6;i++){ const a = i*Math.PI/3; ctx.beginPath(); ctx.moveTo(0,0); ctx.lineTo(Math.cos(a)*ball.r, Math.sin(a)*ball.r); ctx.stroke(); }
  ctx.restore();
}

function drawHUD(){
  // already reflected by HTML tags; optionally draw small text
  ctx.fillStyle = 'rgba(255,255,255,0.95)';
  ctx.font = '16px system-ui';
  ctx.fillText(`Skor: ${state.score}`, 18, 28);
}

/* ====== Loop ====== */
function loop(){
  update();
  drawField();
  drawPlayer();
  drawBall();
  drawHUD();

  if(!state.running){
    // overlay game over
    ctx.fillStyle = 'rgba(0,0,0,0.55)';
    ctx.fillRect(0,0,W,H);
    ctx.fillStyle = '#fff'; ctx.textAlign = 'center';
    ctx.font = 'bold 36px system-ui';
    ctx.fillText('GAME OVER', W/2, H/2 - 12);
    ctx.font = '16px system-ui';
    ctx.fillText(`Skor: ${state.score}`, W/2, H/2 + 18);
    ctx.textAlign = 'start';
  }

  requestAnimationFrame(loop);
}

/* ====== Start / Difficulty ====== */
function start(mode){
  menuOverlay.style.display = 'none';
  if(mode === 'easy'){ gravity = 0.05; player && (player.speed = 5); }
  else if(mode === 'normal'){ gravity = 0.15; player && (player.speed = 5); }
  else { gravity = 0.28; player && (player.speed = 5); }
  resetGame();
}

/* ====== Init ====== */
spawnPlayer();
spawnBallAboveHead();
loop();

/* show overlay initially (menu) already visible by default */
</script>
</body>
</html>
