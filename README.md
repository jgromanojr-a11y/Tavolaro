<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no" />
<title>Space War</title>
<style>
  :root{
    --bg:#030617; --panel:rgba(255,255,255,0.04); --accent:#66e0ff; --danger:#ff7070; --ok:#7cff7a;
  }
  html,body{height:100%;margin:0;background:var(--bg);font-family:Inter,system-ui,Segoe UI,Roboto,Arial;color:#dff6ff;overflow:hidden}
  canvas{display:block;width:100vw;height:100vh}
  /* HUD */
  .hud{position:fixed;left:12px;right:12px;top:12px;display:flex;justify-content:space-between;gap:8px;pointer-events:none}
  .card{background:var(--panel);padding:.5rem .75rem;border-radius:10px;font-weight:700;pointer-events:auto}
  /* controls */
  .controls{position:fixed;left:12px;right:12px;bottom:12px;display:flex;justify-content:space-between;pointer-events:auto}
  button{appearance:none;border:0;background:linear-gradient(180deg,rgba(255,255,255,0.03),rgba(255,255,255,0.01));color:#eaffff;padding:.6rem .9rem;border-radius:12px;font-weight:800;box-shadow:0 8px 18px rgba(0,0,0,.4)}
  .overlay{position:fixed;inset:0;display:flex;align-items:center;justify-content:center;background:linear-gradient(180deg,rgba(0,0,0,.55),rgba(0,0,0,.3));backdrop-filter: blur(2px)}
  .menu{background:linear-gradient(180deg,#071427,#031026);padding:18px;border-radius:12px;border:1px solid rgba(255,255,255,.06);max-width:720px;width:92%;text-align:center}
  h1{margin:.2rem 0;color:var(--accent);letter-spacing:1px}
  p {color:#bfefff;margin:.2rem 0}
  .small{font-size:.85rem;color:#bfefff;opacity:.9}
  .hint{position:fixed;right:10px;top:60px;color:#9edff5;font-size:12px;opacity:.8}
  @media(min-width:900px){button{padding:.6rem 1.1rem}}
</style>
</head>
<body>
<canvas id="cv"></canvas>

<div class="hud">
  <div class="card" id="lblScore">Inimigos: 0</div>
  <div style="display:flex;gap:8px">
    <div class="card" id="lblLives">Vidas: 10</div>
    <div class="card" id="lblLevel">Nível: 1</div>
  </div>
</div>

<div class="controls">
  <div style="display:flex;gap:8px">
    <button id="left">◀</button>
    <button id="right">▶</button>
  </div>
  <div style="display:flex;gap:8px">
    <button id="fire">Fogo</button>
    <button id="quant">Quântico</button>
  </div>
</div>

<div class="hint">PC: ← → • Espaço = Fogo • Q = Quântico • P = Pausa</div>

<!-- Start -->
<div class="overlay" id="start">
  <div class="menu">
    <h1>Space War</h1>
    <p class="small">Controle a nave Solaris — destrua inimigos, colete corações e sobreviva.</p>
    <p style="margin-top:.6rem">Regras: a cada 10 inimigos destruídos +3 vidas e nível aumenta. Perde 1 vida a cada 10 tiros recebidos, 5 colisões com inimigos e 4 colisões com planetas.</p>
    <div style="margin-top:12px;display:flex;gap:10px;justify-content:center">
      <button id="btnStart">Iniciar</button>
      <button id="btnEasy">Fácil (10 vidas)</button>
    </div>
  </div>
</div>

<!-- Game Over -->
<div class="overlay" id="over" style="display:none">
  <div class="menu">
    <h1>Game Over</h1>
    <p id="final" class="small"></p>
    <div style="margin-top:12px">
      <button id="btnRestart">Jogar de novo</button>
    </div>
  </div>
</div>

<script>
(() => {
  const cvs = document.getElementById('cv'), ctx = cvs.getContext('2d');
  let DPR = devicePixelRatio || 1;
  function resize(){
    DPR = devicePixelRatio || 1;
    cvs.width = innerWidth * DPR; cvs.height = innerHeight * DPR;
    cvs.style.width = innerWidth + 'px'; cvs.style.height = innerHeight + 'px';
    ctx.setTransform(DPR,0,0,DPR,0,0);
  }
  addEventListener('resize', resize); resize();

  // audio simples (sintetizador)
  const AudioCtx = window.AudioContext || window.webkitAudioContext;
  let audio;
  function ensureAudio(){
    if(!audio) audio = new AudioCtx();
  }
  function beep(freq=440, time=0.08, type='sine', vol=0.12){
    try{
      ensureAudio();
      const o = audio.createOscillator();
      const g = audio.createGain();
      o.type = type; o.frequency.value = freq;
      g.gain.value = vol;
      o.connect(g); g.connect(audio.destination);
      o.start();
      g.gain.exponentialRampToValueAtTime(0.0001, audio.currentTime + time);
      o.stop(audio.currentTime + time + 0.02);
    }catch(e){}
  }

  // estado
  const S = {
    running:false, score:0, lives:10, level:1,
    enemies:[], bullets:[], ebullets:[], hearts:[], parts:[],
    shotsReceived:0, enemyCollisions:0, planetCollisions:0,
    lastShot:0, lastSpawn:0, quantumReadyAt:0
  };

  const player = { x: innerWidth/2, y: innerHeight - 90, w:46, h:56, speed:6, left:false, right:false };

  // util
  const R=(a,b)=>Math.random()*(b-a)+a, RI=(a,b)=>Math.floor(R(a,b));
  const clamp=(v,a,b)=>Math.max(a,Math.min(b,v));
  const el = id => document.getElementById(id);

  // HUD
  function updateHUD(){ el('lblScore').textContent = 'Inimigos: ' + S.score; el('lblLives').textContent = 'Vidas: ' + S.lives; el('lblLevel').textContent = 'Nível: ' + S.level; }

  // spawn inimigo
  function spawnEnemy(){
    const x = R(30, innerWidth-30);
    const size = RI(26,44);
    const speed = 0.9 + S.level*0.35 + R(0,0.6);
    const type = RI(0,3);
    S.enemies.push({x, y:-50, w:size, h:size*0.8, vx:R(-.7,.7), vy:speed, hp:1 + Math.floor(S.level/2), hue:RI(0,360), type, lastFire: performance.now(), fireDelay: R(900,1600)/(Math.max(1,S.level*0.8))});
  }

  // tiro jogador
  function shoot(quant=false){
    const now = performance.now();
    const cd = quant ? 420 : 140;
    if(now - S.lastShot < cd) return;
    if(quant && now < S.quantumReadyAt) return;
    S.lastShot = now;
    if(quant) S.quantumReadyAt = now + 2000;
    const speed = quant ? 13 : 10;
    const dmg = quant ? 3 : 1;
    const r = quant ? 7 : 4;
    S.bullets.push({x:player.x, y:player.y-28, vy:-speed, r, dmg, quant});
    beep(900,0.05,'triangle',0.07);
  }

  // colisão círculo/retângulo
  function hitCircleRect(cx,cy,cr, rx,ry,rw,rh){
    const tx = clamp(cx, rx, rx+rw);
    const ty = clamp(cy, ry, ry+rh);
    const dx = cx - tx, dy = cy - ty;
    return (dx*dx + dy*dy) <= cr*cr;
  }

  // dano por contagens
  function registerShotReceived(){
    S.shotsReceived++;
    if(S.shotsReceived % 10 === 0){ changeLives(-1); popup('-1 vida (tiros)'); beep(220,0.12,'sine',0.12); }
  }
  function registerEnemyCollision(){
    S.enemyCollisions++;
    if(S.enemyCollisions % 5 === 0){ changeLives(-1); popup('-1 vida (colisão)'); beep(220,0.12,'sine',0.12); }
  }
  function registerPlanetCollision(){
    S.planetCollisions++;
    if(S.planetCollisions % 4 === 0){ changeLives(-1); popup('-1 vida (planeta)'); beep(220,0.12,'sine',0.12); }
  }
  function changeLives(delta){
    S.lives += delta;
    if(delta>0) popup('+1 vida!');
    if(S.lives <= 0) gameOver();
    updateHUD();
  }

  function popup(text){
    S.parts.push({x:player.x, y:player.y-36, text, life:60, type:'text'});
  }

  function maybeHeart(x,y){
    if(Math.random() < 0.14) S.hearts.push({x,y,vy:2.2,r:10});
  }

  // explosão (partículas)
  function explosion(x,y,color,count=12){
    for(let i=0;i<count;i++){
      const ang = R(0,Math.PI*2), sp = R(1.2,4);
      S.parts.push({x,y,vx:Math.cos(ang)*sp, vy:Math.sin(ang)*sp, life:40+RI(0,20), c:color, r:R(1,3)});
    }
    beep(240,0.12,'sawtooth',0.09);
  }

  // loop
  let last = performance.now();
  function loop(now){
    if(!S.running) return;
    const dt = (now - last)/16.666; last = now;

    // player movement
    if(player.left) player.x -= player.speed * dt;
    if(player.right) player.x += player.speed * dt;
    player.x = clamp(player.x, 24, innerWidth-24);

    // spawn enemies pacing
    if(now - (S.lastSpawn||0) > Math.max(700 - S.level*50, 200)){
      spawnEnemy(); S.lastSpawn = now;
    }

    // update enemies
    for(let i=S.enemies.length-1;i>=0;i--){
      const e = S.enemies[i];
      e.x += e.vx * dt; e.y += e.vy * dt;
      if(e.x < 20 || e.x > innerWidth-20) e.vx *= -1;

      // inimigo atira
      if(now - e.lastFire > e.fireDelay && Math.random() < 0.6){
        e.lastFire = now;
        S.ebullets.push({x:e.x, y:e.y + e.h/2, vy: 3 + S.level*0.5, r:4});
      }

      // colisão com jogador
      if(hitCircleRect(player.x,player.y,20, e.x - e.w/2, e.y - e.h/2, e.w, e.h)){
        registerEnemyCollision();
        // empurra inimigo
        e.y += 28;
      }

      if(e.y - 80 > innerHeight) S.enemies.splice(i,1);
    }

    // bullets do jogador
    for(let i=S.bullets.length-1;i>=0;i--){
      const b = S.bullets[i];
      b.y += b.vy * dt;
      ctx.beginPath();
      // collision with enemies
      let hit = false;
      for(let j=S.enemies.length-1;j>=0;j--){
        const e = S.enemies[j];
        if(hitCircleRect(b.x,b.y,b.r, e.x - e.w/2, e.y - e.h/2, e.w, e.h)){
          e.hp -= b.dmg;
          hit = true;
          if(e.hp <= 0){
            explosion(e.x, e.y, `hsl(${e.hue} 70% 60%)`, 12);
            maybeHeart(e.x,e.y);
            S.enemies.splice(j,1);
            S.score++;
            if(S.score % 10 === 0){
              changeLives(+3);
              S.level++;
              popup('+3 vidas • nível ↑');
            }
            updateHUD();
          }
          break;
        }
      }
      if(hit || b.y < -20) S.bullets.splice(i,1);
    }

    // bullets inimigos
    for(let i=S.ebullets.length-1;i>=0;i--){
      const b = S.ebullets[i];
      b.y += b.vy * dt;
      if(hitCircleRect(b.x,b.y,b.r, player.x - player.w/2, player.y - player.h/2, player.w, player.h)){
        S.ebullets.splice(i,1);
        registerShotReceived();
        continue;
      }
      if(b.y > innerHeight + 30) S.ebullets.splice(i,1);
    }

    // hearts
    for(let i=S.hearts.length-1;i>=0;i--){
      const h = S.hearts[i];
      h.y += h.vy * dt;
      if(hitCircleRect(h.x,h.y,h.r, player.x - player.w/2, player.y - player.h/2, player.w, player.h)){
        S.hearts.splice(i,1); changeLives(+1); explosion(player.x, player.y-24, '--', 8);
      } else if(h.y > innerHeight + 30) S.hearts.splice(i,1);
    }

    // particles/text
    for(let i=S.parts.length-1;i>=0;i--){
      const p = S.parts[i];
      if(p.type === 'text'){
        p.life -= dt;
        if(p.life <= 0) S.parts.splice(i,1);
        continue;
      }
      p.x += p.vx * dt; p.y += p.vy * dt; p.vy += 0.05 * dt;
      p.life -= dt;
      if(p.life <= 0) S.parts.splice(i,1);
    }

    // render
    ctx.clearRect(0,0,innerWidth,innerHeight);

    // background stars
    for(let s=0;s<140;s++){
      const sx = (s * 97 + Math.floor(now/8)) % innerWidth;
      const sy = (s * 41 + (now/12)) % innerHeight;
      ctx.fillStyle = s%8===0 ? 'rgba(200,240,255,0.08)' : 'rgba(200,240,255,0.03)';
      ctx.fillRect(sx, sy, (s%11===0?2:1), (s%11===0?2:1));
    }

    // decorative planets
    for(let i=0;i<3;i++){
      ctx.beginPath();
      ctx.fillStyle = `rgba(60,90,120,${0.06 + i*0.02})`;
      ctx.ellipse(60 + i*200, (now/30 + i*140) % (innerHeight+140) - 140, 48 - i*8, 26 - i*6, 0, 0, Math.PI*2);
      ctx.fill();
    }

    // draw enemies
    for(const e of S.enemies){
      ctx.fillStyle = `hsl(${e.hue} 70% 60%)`;
      if(e.type === 0) ctx.fillRect(e.x - e.w/2, e.y - e.h/2, e.w, e.h);
      else if(e.type === 1){ ctx.beginPath(); ctx.arc(e.x, e.y, e.w/2, 0, Math.PI*2); ctx.fill(); }
      else { ctx.beginPath(); ctx.moveTo(e.x, e.y - e.h/2); ctx.lineTo(e.x - e.w/2, e.y + e.h/2); ctx.lineTo(e.x + e.w/2, e.y + e.h/2); ctx.closePath(); ctx.fill(); }
      // life bar
      ctx.fillStyle = 'rgba(0,0,0,0.45)'; ctx.fillRect(e.x - e.w/2, e.y - e.h/2 - 8, e.w, 4);
      ctx.fillStyle = 'rgba(255,255,255,0.85)'; ctx.fillRect(e.x - e.w/2, e.y - e.h/2 - 8, e.w * (e.hp / (1 + Math.floor(S.level/2))), 4);
    }

    // player bullets
    for(const b of S.bullets){
      ctx.beginPath();
      ctx.fillStyle = b.quant ? '#bff4ff' : '#e8fbff';
      ctx.arc(b.x, b.y, b.r, 0, Math.PI*2); ctx.fill();
      if(b.quant){
        ctx.beginPath(); ctx.arc(b.x,b.y,b.r+4,0,Math.PI*2); ctx.strokeStyle='rgba(180,255,255,0.08)'; ctx.stroke();
      }
    }

    // enemy bullets
    for(const b of S.ebullets){
      ctx.fillStyle = '#ff9aa8'; ctx.fillRect(b.x-3,b.y-8,6,16);
    }

    // hearts
    for(const h of S.hearts){
      ctx.save(); ctx.translate(h.x,h.y);
      ctx.fillStyle = '#ff6b8c';
      ctx.beginPath();
      const r = h.r;
      ctx.moveTo(0, r/1.6);
      ctx.bezierCurveTo(r, -r/2, r*1.2, r/1.4, 0, r*1.6);
      ctx.bezierCurveTo(-r*1.2, r/1.4, -r, -r/2, 0, r/1.6);
      ctx.fill(); ctx.restore();
    }

    // particles & text
    for(const p of S.parts){
      if(p.text){
        ctx.font = "bold 16px system-ui"; ctx.fillStyle = "#bff0ff";
        ctx.fillText(p.text, p.x - ctx.measureText(p.text).width/2, p.y);
      } else {
        ctx.fillStyle = p.c || '#fff'; ctx.globalAlpha = Math.max(0, p.life/40);
        ctx.beginPath(); ctx.arc(p.x,p.y,p.r,0,Math.PI*2); ctx.fill(); ctx.globalAlpha = 1;
      }
    }

    // draw player
    ctx.save(); ctx.translate(player.x, player.y);
    ctx.fillStyle = '#7cf5ff';
    ctx.beginPath();
    ctx.moveTo(0,-player.h/2);
    ctx.lineTo(player.w/2, player.h/2);
    ctx.lineTo(0, player.h/4);
    ctx.lineTo(-player.w/2, player.h/2);
    ctx.closePath(); ctx.fill();
    ctx.fillStyle = '#c4f6ff'; ctx.beginPath(); ctx.ellipse(0,-10,10,8,0,0,Math.PI*2); ctx.fill();
    ctx.restore();

    requestAnimationFrame(loop);
  }

  // input
  addEventListener('keydown', e=>{
    if(e.key === 'ArrowLeft') player.left = true;
    if(e.key === 'ArrowRight') player.right = true;
    if(e.code === 'Space'){ shoot(false); e.preventDefault(); }
    if(e.key.toLowerCase() === 'q'){ shoot(true); }
    if(e.key.toLowerCase() === 'p'){ S.running = !S.running; if(S.running){ last = performance.now(); loop(last); } }
  });
  addEventListener('keyup', e=>{
    if(e.key === 'ArrowLeft') player.left = false;
    if(e.key === 'ArrowRight') player.right = false;
  });

  function bindBtn(id, down, up=()=>{}){ const b = el(id); ['touchstart','mousedown'].forEach(ev=>b.addEventListener(ev, e=>{ e.preventDefault(); down(); })); ['touchend','mouseup','mouseleave','touchcancel'].forEach(ev=>b.addEventListener(ev, e=>{ e.preventDefault(); up(); })); }

  bindBtn('left', ()=>player.left=true, ()=>player.left=false);
  bindBtn('right', ()=>player.right=true, ()=>player.right=false);
  bindBtn('fire', ()=>shoot(false));
  bindBtn('quant', ()=>shoot(true));

  // UI buttons
  el('btnStart').addEventListener('click', ()=>{ el('start').style.display='none'; startGame(); });
  el('btnRestart').addEventListener('click', ()=>{ el('over').style.display='none'; startGame(); });
  el('btnEasy').addEventListener('click', ()=>{ S.lives = 10; updateHUD(); });

  function startGame(){
    // reset
    S.enemies.length = 0; S.bullets.length = 0; S.ebullets.length = 0; S.hearts.length = 0; S.parts.length = 0;
    S.score = 0; S.lives = 10; S.level = 1; S.shotsReceived = 0; S.enemyCollisions = 0; S.planetCollisions = 0;
    player.x = innerWidth/2; player.y = innerHeight - 90;
    updateHUD();
    S.running = true; last = performance.now(); ensureAudio();
    requestAnimationFrame(loop);
  }

  function gameOver(){
    S.running = false;
    el('final').innerHTML = `Inimigos destruídos: <b>${S.score}</b><br>Nível alcançado: <b>${S.level}</b>`;
    el('over').style.display = 'flex';
  }

  // ajustar player bottom on resize
  addEventListener('resize', ()=>{ player.y = innerHeight - 90; resize(); });

  // inicial
  updateHUD();

})();
</script>
</body>
</html>
