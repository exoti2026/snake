<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Snake Arena</title>
<style>
  :root{
    --bg:#0d1117; --grid:#161b22; --text:#e6edf3; --accent:#3fb950; --danger:#f85149;
  }
  @media (prefers-color-scheme: light){
    :root:not([data-theme="dark"]){ --bg:#eef2f6; --grid:#dbe3ea; --text:#0d1117; }
  }
  *{box-sizing:border-box;}
  html,body{height:100%;margin:0;background:var(--bg);color:var(--text);
    font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;overflow:hidden;
    padding-top:env(safe-area-inset-top,0px);padding-bottom:env(safe-area-inset-bottom,0px);}
  #wrap{position:relative;width:100%;height:100%;}
  canvas{display:block;width:100%;height:100%;touch-action:none;}
  #hud{position:absolute;top:calc(12px + env(safe-area-inset-top,0px));left:12px;
    font-size:14px;background:rgba(0,0,0,0.35);padding:8px 12px;border-radius:8px;
    backdrop-filter:blur(4px);}
  #hud b{color:var(--accent);}
  #leaderboard{position:absolute;top:calc(12px + env(safe-area-inset-top,0px));right:12px;
    font-size:13px;background:rgba(0,0,0,0.35);padding:8px 12px;border-radius:8px;
    min-width:140px;backdrop-filter:blur(4px);}
  #leaderboard div{display:flex;justify-content:space-between;gap:12px;padding:1px 0;}
  #overlay{position:absolute;inset:0;display:flex;flex-direction:column;align-items:center;
    justify-content:center;background:rgba(0,0,0,0.55);text-align:center;gap:14px;}
  #overlay h1{margin:0;font-size:clamp(24px,6vw,40px);}
  #overlay p{margin:0;opacity:0.85;max-width:320px;}
  button{background:var(--accent);color:#04140a;border:none;padding:12px 28px;border-radius:999px;
    font-size:16px;font-weight:700;cursor:pointer;}
  button:active{transform:scale(0.97);}
  #hint{position:absolute;bottom:calc(14px + env(safe-area-inset-bottom,0px));left:50%;
    transform:translateX(-50%);font-size:12px;opacity:0.6;}
  .hidden{display:none !important;}
</style>
</head>
<body>
<div id="wrap">
  <canvas id="c"></canvas>
  <div id="hud">Length: <b id="len">10</b></div>
  <div id="leaderboard"><b>Leaderboard</b><div id="lb"></div></div>
  <div id="hint">Move mouse / drag to steer &middot; hold click to boost</div>
  <div id="overlay">
    <h1>Snake Arena</h1>
    <p>Grow by eating orbs. Bigger snakes crush smaller ones on contact &mdash; avoid heads bigger than yours!</p>
    <button id="startBtn">Play</button>
  </div>
</div>
<script>
(function(){
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');
let W, H, DPR = Math.min(window.devicePixelRatio||1, 2);
function resize(){
  W = window.innerWidth; H = window.innerHeight;
  canvas.width = W*DPR; canvas.height = H*DPR;
  canvas.style.width = W+'px'; canvas.style.height = H+'px';
  ctx.setTransform(DPR,0,0,DPR,0,0);
}
window.addEventListener('resize', resize); resize();

const WORLD = 3000;
const FOOD_COUNT = 260;
const BOT_COUNT = 9;
const colors = ['#ff6b6b','#4ecdc4','#ffe66d','#a29bfe','#fd79a8','#55efc4','#74b9ff','#fab1a0','#81ecec'];

let food = [], snakes = [], camera = {x:0,y:0,zoom:1};
let mouse = {x:0,y:0,down:false};
let running = false, myId = 0;

function rand(a,b){return a+Math.random()*(b-a);}
function dist2(ax,ay,bx,by){const dx=ax-bx,dy=ay-by;return dx*dx+dy*dy;}

function spawnFood(n){
  for(let i=0;i<n;i++){
    food.push({
      x: rand(0,WORLD), y: rand(0,WORLD),
      r: rand(3,6), color: colors[(Math.random()*colors.length)|0]
    });
  }
}

function makeSnake(isPlayer, name){
  const x = rand(WORLD*0.2,WORLD*0.8), y = rand(WORLD*0.2,WORLD*0.8);
  const segs = [];
  for(let i=0;i<10;i++) segs.push({x:x-i*6,y:y});
  return {
    segs, dir: rand(0,Math.PI*2), speed: 2.2, boosting:false,
    color: isPlayer ? '#3fb950' : colors[(Math.random()*colors.length)|0],
    name, isPlayer, alive:true, score:10, targetDir: 0
  };
}

function radiusFor(score){ return Math.min(6 + Math.sqrt(score)*1.4, 40); }

function initGame(){
  food = []; snakes = [];
  spawnFood(FOOD_COUNT);
  const me = makeSnake(true, 'You');
  snakes.push(me); myId = 0;
  for(let i=0;i<BOT_COUNT;i++) snakes.push(makeSnake(false, 'Bot '+(i+1)));
}

function headPos(s){ return s.segs[0]; }

function steerBot(s){
  // wander with slight bias toward nearest food, avoid world edge
  const h = headPos(s);
  let best=null, bd=Infinity;
  for(let i=0;i<food.length;i+=4){
    const f = food[i];
    const d = dist2(h.x,h.y,f.x,f.y);
    if(d<bd){bd=d;best=f;}
  }
  let target = s.targetDir;
  if(best) target = Math.atan2(best.y-h.y, best.x-h.x);
  if(h.x<150) target = 0;
  if(h.x>WORLD-150) target = Math.PI;
  if(h.y<150) target = Math.PI/2;
  if(h.y>WORLD-150) target = -Math.PI/2;
  let diff = ((target - s.dir + Math.PI*3) % (Math.PI*2)) - Math.PI;
  s.dir += diff*0.06;
}

function updateSnake(s){
  if(!s.alive) return;
  const h = headPos(s);
  const spd = s.boosting && s.score>12 ? s.speed*1.9 : s.speed;
  if(s.boosting && s.score>12){ s.score -= 0.05; }
  const nx = h.x + Math.cos(s.dir)*spd;
  const ny = h.y + Math.sin(s.dir)*spd;
  s.segs.unshift({x:nx,y:ny});
  const targetLen = Math.max(10, Math.floor(10 + s.score*1.2));
  while(s.segs.length > targetLen) s.segs.pop();
  // world bounds
  s.segs[0].x = Math.max(0,Math.min(WORLD,s.segs[0].x));
  s.segs[0].y = Math.max(0,Math.min(WORLD,s.segs[0].y));
}

function eatFood(s){
  const h = headPos(s);
  const r = radiusFor(s.score);
  for(let i=food.length-1;i>=0;i--){
    const f = food[i];
    if(dist2(h.x,h.y,f.x,f.y) < (r+f.r)*(r+f.r)){
      food.splice(i,1);
      s.score += 1;
    }
  }
  while(food.length < FOOD_COUNT) spawnFood(1);
}

function checkCollisions(){
  for(let i=0;i<snakes.length;i++){
    const a = snakes[i];
    if(!a.alive) continue;
    const ah = headPos(a);
    const ar = radiusFor(a.score);
    for(let j=0;j<snakes.length;j++){
      if(i===j) continue;
      const b = snakes[j];
      if(!b.alive) continue;
      const br = radiusFor(b.score);
      // check a's head against b's body segments (skip first couple segs of b to avoid instant self-adjacent kill)
      for(let k=2;k<b.segs.length;k+=2){
        const seg = b.segs[k];
        if(dist2(ah.x,ah.y,seg.x,seg.y) < (ar*0.6+br*0.5)*(ar*0.6+br*0.5)){
          killSnake(a);
          break;
        }
      }
      if(!a.alive) break;
    }
  }
}

function killSnake(s){
  if(!s.alive) return;
  s.alive = false;
  // scatter food from its body
  for(let i=0;i<s.segs.length;i+=2){
    food.push({x:s.segs[i].x+rand(-8,8), y:s.segs[i].y+rand(-8,8), r:rand(3,6), color:s.color});
  }
  if(s.isPlayer){
    endGame();
  } else {
    // respawn bot after a moment
    setTimeout(()=>{
      const idx = snakes.indexOf(s);
      if(idx>=0) snakes[idx] = makeSnake(false, s.name);
    }, 1500);
  }
}

function endGame(){
  running = false;
  document.getElementById('overlay').classList.remove('hidden');
  document.querySelector('#overlay h1').textContent = 'Game Over';
  document.querySelector('#overlay p').textContent = 'Final length: ' + Math.floor(snakes[myId].score+10) + '. Try again?';
  document.getElementById('startBtn').textContent = 'Play Again';
}

function updateHUD(){
  const me = snakes[myId];
  document.getElementById('len').textContent = Math.floor(10+me.score);
  const ranked = snakes.slice().sort((a,b)=>b.score-a.score).slice(0,5);
  const lb = document.getElementById('lb');
  lb.innerHTML = ranked.map(s=>`<div><span>${s===me?'<b>You</b>':s.name}</span><span>${Math.floor(10+s.score)}</span></div>`).join('');
}

function draw(){
  const me = snakes[myId];
  const h = headPos(me);
  camera.x = h.x; camera.y = h.y;

  ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--bg') || '#0d1117';
  ctx.fillRect(0,0,W,H);

  ctx.save();
  ctx.translate(W/2 - camera.x, H/2 - camera.y);

  // grid
  ctx.strokeStyle = 'rgba(128,128,128,0.15)';
  ctx.lineWidth = 1;
  const gStep = 60;
  const startX = Math.floor((camera.x-W/2)/gStep)*gStep;
  const startY = Math.floor((camera.y-H/2)/gStep)*gStep;
  for(let x=startX; x<camera.x+W/2; x+=gStep){
    ctx.beginPath(); ctx.moveTo(x,camera.y-H/2); ctx.lineTo(x,camera.y+H/2); ctx.stroke();
  }
  for(let y=startY; y<camera.y+H/2; y+=gStep){
    ctx.beginPath(); ctx.moveTo(camera.x-W/2,y); ctx.lineTo(camera.x+W/2,y); ctx.stroke();
  }

  // world border
  ctx.strokeStyle = '#f85149'; ctx.lineWidth = 6;
  ctx.strokeRect(0,0,WORLD,WORLD);

  // food
  for(const f of food){
    ctx.beginPath(); ctx.arc(f.x,f.y,f.r,0,Math.PI*2);
    ctx.fillStyle = f.color; ctx.fill();
  }

  // snakes
  for(const s of snakes){
    if(!s.alive) continue;
    const r = radiusFor(s.score);
    for(let i=s.segs.length-1;i>=0;i--){
      const seg = s.segs[i];
      ctx.beginPath(); ctx.arc(seg.x,seg.y, r*(0.85+0.15*(1-i/s.segs.length)), 0, Math.PI*2);
      ctx.fillStyle = s.color; ctx.fill();
    }
    // eyes on head
    const hpos = s.segs[0];
    ctx.fillStyle = '#fff';
    ctx.beginPath(); ctx.arc(hpos.x+Math.cos(s.dir+0.5)*r*0.5, hpos.y+Math.sin(s.dir+0.5)*r*0.5, r*0.22,0,Math.PI*2); ctx.fill();
    ctx.beginPath(); ctx.arc(hpos.x+Math.cos(s.dir-0.5)*r*0.5, hpos.y+Math.sin(s.dir-0.5)*r*0.5, r*0.22,0,Math.PI*2); ctx.fill();
  }

  ctx.restore();
}

function loop(){
  if(!running) return;
  const me = snakes[myId];
  if(me.alive){
    const dx = mouse.x - W/2, dy = mouse.y - H/2;
    if(Math.hypot(dx,dy) > 8) me.dir = Math.atan2(dy,dx);
    me.boosting = mouse.down;
  }
  for(const s of snakes){ if(s!==me && s.alive) steerBot(s); }
  for(const s of snakes) updateSnake(s);
  for(const s of snakes) if(s.alive) eatFood(s);
  checkCollisions();
  updateHUD();
  draw();
  requestAnimationFrame(loop);
}

canvas.addEventListener('mousemove', e=>{
  const rect = canvas.getBoundingClientRect();
  mouse.x = e.clientX - rect.left; mouse.y = e.clientY - rect.top;
});
canvas.addEventListener('mousedown', ()=> mouse.down = true);
window.addEventListener('mouseup', ()=> mouse.down = false);
canvas.addEventListener('touchmove', e=>{
  e.preventDefault();
  const t = e.touches[0]; const rect = canvas.getBoundingClientRect();
  mouse.x = t.clientX - rect.left; mouse.y = t.clientY - rect.top;
}, {passive:false});
canvas.addEventListener('touchstart', e=>{
  mouse.down = true;
  const t = e.touches[0]; const rect = canvas.getBoundingClientRect();
  mouse.x = t.clientX - rect.left; mouse.y = t.clientY - rect.top;
});
canvas.addEventListener('touchend', ()=> mouse.down = false);

document.getElementById('startBtn').addEventListener('click', ()=>{
  document.getElementById('overlay').classList.add('hidden');
  mouse.x = W/2; mouse.y = H/2;
  initGame();
  running = true;
  requestAnimationFrame(loop);
});
})();
</script>
</body>
</html>
