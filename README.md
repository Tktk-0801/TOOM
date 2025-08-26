<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<title>Raycast — Occlusion（壁で敵が見えないように）</title>
<style>
  html,body{height:100%;margin:0;background:#000;font-family:system-ui,-apple-system,Segoe UI,Roboto;}
  #hint { position:fixed; left:8px; top:8px; color:#ddd; font-size:13px; z-index:10; }
  canvas{display:block; margin:0 auto; touch-action:none; -webkit-user-select:none; user-select:none;}
  .btn { position:fixed; right:12px; bottom:12px; width:64px;height:64px;border-radius:32px; background:rgba(255,255,255,0.06); color:#fff; display:flex;align-items:center;justify-content:center; font-weight:700; z-index:11; user-select:none; -webkit-user-select:none;}
  #swap { right:12px; bottom:88px; width:48px;height:48px;border-radius:8px; font-size:14px;}
</style>
</head>
<body>
<div id="hint">左スティック：移動 • 右スティック：視点 • 右タップ：射撃</div>
<canvas id="game"></canvas>
<button id="swap" class="btn">WEAPON</button>
<script>
(() => {
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  function resize(){ canvas.width = Math.min(window.innerWidth, 960); canvas.height = Math.min(window.innerHeight, 640); }
  window.addEventListener('resize', resize); resize();

  // --- map (そのまま) ---
  const map = [
    [1,1,1,1,1,1,1,1,1,1,1,1],
    [1,0,0,0,0,1,0,0,0,0,0,1],
    [1,0,1,0,0,1,0,1,1,0,0,1],
    [1,0,1,0,0,0,0,0,1,0,0,1],
    [1,0,0,0,1,1,1,0,1,0,0,1],
    [1,0,1,0,0,0,0,0,1,0,0,1],
    [1,0,1,0,1,0,1,0,0,0,0,1],
    [1,0,0,0,1,0,0,0,1,0,0,1],
    [1,0,1,0,0,0,0,0,1,0,0,1],
    [1,0,0,0,0,0,1,0,0,0,0,1],
    [1,0,0,0,0,0,0,0,0,1,0,1],
    [1,1,1,1,1,1,1,1,1,1,1,1]
  ];
  const MAP_W = map[0].length, MAP_H = map.length;

  // --- player ---
  let posX = 5.5, posY = 5.5;
  let dirX = -1, dirY = 0;
  let planeX = 0, planeY = 0.66;
  const moveSpeedBase = 0.06;
  const rotSpeedBase = 0.045;

  // --- weapons ---
  const weapons = [
    { name:'PISTOL', cooldown:300, pellets:1, spread:0.01, speed:0.28, recoil:0.012 },
    { name:'SHOTGUN', cooldown:700, pellets:7, spread:0.18, speed:0.22, recoil:0.06 },
    { name:'RIFLE', cooldown:140, pellets:1, spread:0.006, speed:0.36, recoil:0.02 }
  ];
  let currentWeapon = 0;
  let lastShotTime = 0;
  let recoil = 0, shake = 0;

  // --- dynamic objects ---
  let enemies = [];
  let bullets = [];
  let pickups = [];
  const MAX_ENEMIES = 18;
  const SPAWN_INTERVAL = 1400;
  let spawnTimer = 0;

  // --- textures (small) ---
  function makeTex(color){
    const t = document.createElement('canvas'); t.width=64; t.height=64; const c=t.getContext('2d'); c.fillStyle=color; c.fillRect(0,0,64,64);
    const id=c.getImageData(0,0,64,64); for(let i=0;i<id.data.length;i+=4){ const v=(Math.random()*40-20)|0; id.data[i]=Math.max(0,Math.min(255,id.data[i]+v)); id.data[i+1]=Math.max(0,Math.min(255,id.data[i+1]+v)); id.data[i+2]=Math.max(0,Math.min(255,id.data[i+2]+v)); } c.putImageData(id,0,0); return t;
  }
  const wallTexA = makeTex('#8b6b44');
  const wallTexB = makeTex('#666666');

  // --- utilities ---
  function getEmptyCells(){ const arr=[]; for(let y=0;y<MAP_H;y++) for(let x=0;x<MAP_W;x++) if(map[y][x]===0) arr.push({x,y}); return arr; }
  function dist(ax,ay,bx,by){ return Math.hypot(ax-bx, ay-by); }

  // --- initial pickups ---
  function placeInitialPickups(){
    const spots = getEmptyCells();
    for(let i=0;i<4 && spots.length;i++){
      const s = spots.splice(Math.floor(Math.random()*spots.length),1)[0];
      pickups.push({x: s.x+0.5, y: s.y+0.5, weaponIndex: 1 + Math.floor(Math.random()*(weapons.length-1)), spawnedAt: performance.now()});
    }
  }
  placeInitialPickups();

  // --- audio (light) ---
  const AudioCtx = window.AudioContext || window.webkitAudioContext;
  const audioCtx = AudioCtx ? new AudioCtx() : null;
  function ensureAudio(){ if(!audioCtx) return; if(audioCtx.state === 'suspended') audioCtx.resume(); }
  window.addEventListener('touchstart', ensureAudio, {passive:true});
  window.addEventListener('mousedown', ensureAudio);

  function playPistolSound(){ if(!audioCtx) return; const g = audioCtx.createGain(); g.connect(audioCtx.destination); const o = audioCtx.createOscillator(); o.type='sawtooth'; o.frequency.setValueAtTime(900,audioCtx.currentTime); o.frequency.exponentialRampToValueAtTime(140,audioCtx.currentTime+0.08); o.connect(g); g.gain.setValueAtTime(0.0001,audioCtx.currentTime); g.gain.linearRampToValueAtTime(0.9,audioCtx.currentTime+0.01); g.gain.exponentialRampToValueAtTime(0.001,audioCtx.currentTime+0.18); o.start(); o.stop(audioCtx.currentTime+0.18); }
  function playShotgunSound(){ if(!audioCtx) return; const now=audioCtx.currentTime; const g = audioCtx.createGain(); g.connect(audioCtx.destination); for(let i=0;i<5;i++){ const o=audioCtx.createOscillator(); o.type='square'; const f=700+Math.random()*1200; o.frequency.setValueAtTime(f,now); o.frequency.exponentialRampToValueAtTime(120,now+0.12+Math.random()*0.05); o.connect(g); o.start(now + i*0.002); o.stop(now + 0.12 + Math.random()*0.05);} g.gain.setValueAtTime(0.0001,now); g.gain.linearRampToValueAtTime(0.9,now+0.006); g.gain.exponentialRampToValueAtTime(0.001,now+0.22); }
  function playPickup(){ if(!audioCtx) return; const now=audioCtx.currentTime; const o=audioCtx.createOscillator(); const g=audioCtx.createGain(); o.type='sine'; o.frequency.setValueAtTime(700,now); o.frequency.exponentialRampToValueAtTime(1100,now+0.06); g.gain.setValueAtTime(0.0001,now); g.gain.linearRampToValueAtTime(0.7,now+0.01); g.gain.exponentialRampToValueAtTime(0.001,now+0.12); o.connect(g); g.connect(audioCtx.destination); o.start(now); o.stop(now+0.12); }
  function playImpact(){ if(!audioCtx) return; const now=audioCtx.currentTime; const o=audioCtx.createOscillator(); const g=audioCtx.createGain(); o.type='triangle'; o.frequency.setValueAtTime(1200,now); g.gain.setValueAtTime(0.0001,now); g.gain.linearRampToValueAtTime(0.6,now+0.002); g.gain.exponentialRampToValueAtTime(0.001,now+0.08); o.connect(g); g.connect(audioCtx.destination); o.start(now); o.stop(now+0.08); }

  // --- input (touch + keyboard) ---
  const keys = {};
  window.addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
  window.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);
  let leftTouch = null, rightTouch = null, rightTouchMoved=false, rightTouchStart=0;
  canvas.addEventListener('touchstart', e=>{
    ensureAudio();
    for(const t of e.changedTouches){
      const x=t.clientX, y=t.clientY;
      if(x < canvas.width/2 && !leftTouch) leftTouch = {id:t.identifier, baseX:x, baseY:y, curX:x, curY:y};
      else if(x >= canvas.width/2 && !rightTouch){ rightTouch = {id:t.identifier, baseX:x, baseY:y, curX:x, curY:y}; rightTouchStart = Date.now(); rightTouchMoved=false; }
    }
  }, {passive:false});
  canvas.addEventListener('touchmove', e=>{ for(const t of e.changedTouches){ if(leftTouch && t.identifier===leftTouch.id){ leftTouch.curX=t.clientX; leftTouch.curY=t.clientY; } if(rightTouch && t.identifier===rightTouch.id){ rightTouch.curX=t.clientX; rightTouch.curY=t.clientY; if(dist(rightTouch.curX,rightTouch.curY,rightTouch.baseX,rightTouch.baseY)>12) rightTouchMoved=true; } } e.preventDefault?.(); }, {passive:false});
  canvas.addEventListener('touchend', e=>{ for(const t of e.changedTouches){ if(leftTouch && t.identifier===leftTouch.id) leftTouch=null; if(rightTouch && t.identifier===rightTouch.id){ const dt = Date.now()-rightTouchStart; if(!rightTouchMoved && dt<300) shoot(); rightTouch=null; } } });

  let mouseDown=false, mouseLastX=0;
  canvas.addEventListener('mousedown', e=>{ ensureAudio(); mouseDown=true; mouseLastX=e.clientX; if(e.button===0) shoot(); });
  window.addEventListener('mousemove', e=>{ if(mouseDown){ const dx = e.clientX - mouseLastX; mouseLastX = e.clientX; rotate(-dx * 0.003); }});
  window.addEventListener('mouseup', ()=>mouseDown=false);

  const swapBtn = document.getElementById('swap');
  swapBtn.addEventListener('click', ()=>{ currentWeapon=(currentWeapon+1)%weapons.length; swapBtn.textContent = weapons[currentWeapon].name; });
  swapBtn.textContent = weapons[currentWeapon].name;

  function rotate(a){
    const od = dirX;
    dirX = dirX*Math.cos(a) - dirY*Math.sin(a);
    dirY = od*Math.sin(a) + dirY*Math.cos(a);
    const op = planeX;
    planeX = planeX*Math.cos(a) - planeY*Math.sin(a);
    planeY = op*Math.sin(a) + planeY*Math.cos(a);
  }

  // --- shooting ---
  function shoot(){
    const now = performance.now();
    const w = weapons[currentWeapon];
    if(now - lastShotTime < w.cooldown) return;
    lastShotTime = now;
    recoil = Math.min(0.16, recoil + w.recoil);
    shake = Math.min(10, shake + 6);
    if(currentWeapon===0) playPistolSound(); else if(currentWeapon===1) playShotgunSound(); else playPistolSound();
    for(let p=0;p<w.pellets;p++){
      const spread = (Math.random()-0.5) * w.spread;
      const s = Math.sin(spread), c = Math.cos(spread);
      const pdx = dirX*c - dirY*s, pdy = dirX*s + dirY*c;
      bullets.push({ x: posX + dirX*0.28, y: posY + dirY*0.28, dirX: pdx, dirY: pdy, life: 220, speed: w.speed });
    }
  }

  // --- enemy spawn ---
  function spawnEnemyAt(x,y){ enemies.push({ x:x+0.5, y:y+0.5, hp: 2 + Math.floor(Math.random()*2), lastHit:0, spawnAt:performance.now() }); }
  function trySpawnEnemy(){ if(enemies.length >= MAX_ENEMIES) return; const empties = getEmptyCells().filter(s => dist(s.x+0.5,s.y+0.5,posX,posY) > 4); if(empties.length===0) return; const s = empties[Math.floor(Math.random()*empties.length)]; spawnEnemyAt(s.x,s.y); }

  // --- update/draw loop ---
  let last = performance.now();
  function update(dt){
    // movement
    let mf=0, ms=0;
    if(keys['w']) mf+=1;
    if(keys['s']) mf-=1;
    if(keys['q']) ms-=1;
    if(keys['e']) ms+=1;
    if(keys['a']) rotate(rotSpeedBase); if(keys['d']) rotate(-rotSpeedBase);
    if(leftTouch){ mf += -(leftTouch.curY - leftTouch.baseY)/80; ms += (leftTouch.curX - leftTouch.baseX)/80; }
    const moveSpeed = moveSpeedBase;
    if(Math.abs(mf) > 0.01){
      const nx = posX + dirX * mf * moveSpeed; if(map[Math.floor(posY)][Math.floor(nx)]===0) posX = nx;
      const ny = posY + dirY * mf * moveSpeed; if(map[Math.floor(ny)][Math.floor(posX)]===0) posY = ny;
    }
    if(Math.abs(ms) > 0.01){
      const nx = posX + dirY * ms * moveSpeed; if(map[Math.floor(posY)][Math.floor(nx)]===0) posX = nx;
      const ny = posY - dirX * ms * moveSpeed; if(map[Math.floor(ny)][Math.floor(posX)]===0) posY = ny;
    }
    if(rightTouch){ const dx = rightTouch.curX - rightTouch.baseX; rotate(-dx * 0.0035); }

    // bullets
    for(let i=bullets.length-1;i>=0;i--){
      const b = bullets[i]; b.x += b.dirX * b.speed; b.y += b.dirY * b.speed; b.life--;
      if(!map[Math.floor(b.y)] || map[Math.floor(b.y)][Math.floor(b.x)] > 0){ playImpact(); bullets.splice(i,1); continue; }
      let hit=false;
      for(let j=enemies.length-1;j>=0;j--){
        const e = enemies[j];
        if(Math.hypot(e.x-b.x,e.y-b.y) < 0.36){
          e.hp -= 1; e.lastHit = performance.now(); playImpact(); bullets.splice(i,1); hit=true;
          if(e.hp <= 0){
            if(Math.random() < 0.4){
              const widx = 0 | (1 + Math.floor(Math.random()*(weapons.length-1)));
              pickups.push({ x: e.x, y: e.y, weaponIndex: widx, spawnedAt: performance.now() });
            }
          }
          break;
        }
      }
      if(hit) continue;
      if(b.life <= 0) bullets.splice(i,1);
    }

    // pickups
    for(let i=pickups.length-1;i>=0;i--){
      const p = pickups[i];
      if(dist(p.x,p.y,posX,posY) < 0.7){
        currentWeapon = p.weaponIndex; swapBtn.textContent = weapons[currentWeapon].name; pickups.splice(i,1); playPickup();
      }
    }

    // enemies movement
    enemies.forEach(e=>{
      const vx = posX - e.x, vy = posY - e.y; const d = Math.hypot(vx,vy) || 1;
      if(d < 6){
        const nx = e.x + (vx/d) * (0.002 + Math.random()*0.003);
        const ny = e.y + (vy/d) * (0.002 + Math.random()*0.003);
        if(map[Math.floor(e.y)][Math.floor(nx)]===0) e.x = nx;
        if(map[Math.floor(ny)][Math.floor(e.x)]===0) e.y = ny;
      } else {
        if(!e._wanderDir || Math.random() < 0.01){ const ang=Math.random()*Math.PI*2; e._wanderDir={x:Math.cos(ang),y:Math.sin(ang),t:Date.now()}; }
        const nx = e.x + e._wanderDir.x * 0.001; const ny = e.y + e._wanderDir.y * 0.001;
        if(map[Math.floor(e.y)][Math.floor(nx)]===0) e.x = nx;
        if(map[Math.floor(ny)][Math.floor(e.x)]===0) e.y = ny;
        if(Date.now() - e._wanderDir.t > 2000) e._wanderDir = null;
      }
    });
    enemies = enemies.filter(e => e.hp > 0);

    // spawn
    spawnTimer += dt;
    if(spawnTimer > SPAWN_INTERVAL){ spawnTimer = 0; trySpawnEnemy(); }

    if(shake > 0) shake *= 0.92; else shake = 0;
    if(recoil > 0) recoil *= 0.86; else recoil = 0;
  }

  function draw(){
    const screenW = canvas.width, screenH = canvas.height;
    const shakeX = (Math.random()-0.5) * (shake*0.25);
    const shakeY = (Math.random()-0.5) * (shake*0.25);

    // background
    ctx.fillStyle = '#1b1b1b'; ctx.fillRect(0,0,screenW,screenH/2);
    ctx.fillStyle = '#2e2e2e'; ctx.fillRect(0,screenH/2,screenW,screenH/2);

    // Z-buffer: 各スクリーン列の壁距離を格納
    const zBuffer = new Array(screenW);
    const step = 2;
    for(let x=0;x<screenW;x+=step){
      const cameraX = 2*x/screenW -1;
      const rayDirX = dirX + (planeX + recoil) * cameraX;
      const rayDirY = dirY + (planeY + recoil) * cameraX;
      let mapX = Math.floor(posX), mapY = Math.floor(posY);
      const deltaDistX = (rayDirX === 0) ? 1e9 : Math.abs(1 / rayDirX);
      const deltaDistY = (rayDirY === 0) ? 1e9 : Math.abs(1 / rayDirY);
      let stepMapX, stepMapY, sideDistX, sideDistY;
      if(rayDirX < 0){ stepMapX=-1; sideDistX=(posX-mapX)*deltaDistX; } else { stepMapX=1; sideDistX=(mapX+1-posX)*deltaDistX; }
      if(rayDirY < 0){ stepMapY=-1; sideDistY=(posY-mapY)*deltaDistY; } else { stepMapY=1; sideDistY=(mapY+1-posY)*deltaDistY; }
      let hit=0, side=0, guard=0;
      while(hit===0 && guard++<1000){
        if(sideDistX < sideDistY){ sideDistX += deltaDistX; mapX += stepMapX; side=0; }
        else { sideDistY += deltaDistY; mapY += stepMapY; side=1; }
        if(mapY>=0 && mapY<MAP_H && mapX>=0 && mapX<MAP_W && map[mapY][mapX]>0) hit=1;
      }
      let perpWallDist = side===0 ? (mapX - posX + (1 - stepMapX)/2) / (rayDirX || 1e-9) : (mapY - posY + (1 - stepMapY)/2) / (rayDirY || 1e-9);
      if(perpWallDist <= 0) perpWallDist = 1e-6;
      // store perpWallDist into zBuffer for columns [x .. x+step-1]
      for(let cx = x; cx < Math.min(screenW, x+step); cx++) zBuffer[cx] = perpWallDist;

      const lineHeight = Math.floor(screenH / perpWallDist);
      const drawStart = Math.floor(-lineHeight/2 + screenH/2 + shakeY);
      const drawEnd = Math.floor(lineHeight/2 + screenH/2 + shakeY);
      const useA = ((mapY+mapX)%2===0);
      const tex = useA ? wallTexA : wallTexB;
      const texX = Math.floor(((side===0 ? (posY + perpWallDist*rayDirY) : (posX + perpWallDist*rayDirX)) * 32) % tex.width);
      try{ ctx.drawImage(tex, texX, 0, 1, tex.height, x + shakeX, drawStart, step, Math.max(2, drawEnd - drawStart)); }
      catch(e){ const shade = side===1?0.6:1.0; const color = Math.floor(200 * shade / (1 + perpWallDist*0.04)); ctx.fillStyle = `rgb(${color},${color},${color})`; ctx.fillRect(x+shakeX, drawStart, step, Math.max(2, drawEnd-drawStart)); }
    }

    // sprites: enemies & pickups & impacts, sorted by depth
    const sprites = [];
    enemies.forEach(e=>{
      const dx=e.x-posX, dy=e.y-posY;
      const invDet = 1.0/(planeX*dirY - dirX*planeY + 1e-9);
      const tx = invDet * (dirY*dx - dirX*dy);
      const ty = invDet * (-planeY*dx + planeX*dy);
      if(ty>0.01){
        const screenX = Math.floor((screenW/2)*(1 + tx/ty));
        const size = Math.abs(Math.floor(screenH/ty));
        sprites.push({type:'enemy', screenX, size, dist:ty, src:e});
      }
    });
    pickups.forEach(p=>{
      const dx=p.x-posX, dy=p.y-posY;
      const invDet = 1.0/(planeX*dirY - dirX*planeY + 1e-9);
      const tx = invDet * (dirY*dx - dirX*dy);
      const ty = invDet * (-planeY*dx + planeX*dy);
      if(ty>0.01){
        const screenX = Math.floor((screenW/2)*(1 + tx/ty));
        const size = Math.max(8, Math.floor(18/ty));
        sprites.push({type:'pickup', screenX, size, dist:ty, src:p});
      }
    });
    // sort far->near
    sprites.sort((a,b)=> b.dist - a.dist);

    // draw sprites but only if not occluded by walls (compare sprite.dist vs zBuffer at its screenX)
    sprites.forEach(s=>{
      const sx = s.screenX;
      if(sx < 0 || sx >= screenW) return; // out of screen
      const wallDistAtColumn = zBuffer[sx] || 1e9;
      // important: compare depth. draw only if sprite is closer than wall
      if(s.dist >= wallDistAtColumn - 0.01) return; // behind wall or clipped -> don't draw

      if(s.type==='enemy'){
        const e=s.src; const w=Math.max(12, Math.floor(s.size*0.5)); const h=Math.max(18, s.size);
        ctx.globalAlpha = (e.lastHit && (performance.now()-e.lastHit) < 220) ? 0.6 : 1.0;
        const grad = ctx.createLinearGradient(0,0,0,h); grad.addColorStop(0,'#ff8a8a'); grad.addColorStop(1,'#7a1f1f');
        ctx.fillStyle = grad; ctx.fillRect(sx - Math.floor(w/2) + shakeX, (canvas.height/2)-Math.floor(h/2)+shakeY, w, h);
        ctx.globalAlpha = 1.0;
        // HP bar
        ctx.fillStyle = 'black'; ctx.fillRect(sx-Math.floor(w/2)+shakeX, canvas.height/2-Math.floor(h/2)-8+shakeY, w, 4);
        ctx.fillStyle = 'lime'; const hpRate = Math.max(0, e.hp/4); ctx.fillRect(sx-Math.floor(w/2)+shakeX, canvas.height/2-Math.floor(h/2)-8+shakeY, Math.floor(w*hpRate), 4);
      } else if(s.type==='pickup'){
        const p = s.src; ctx.fillStyle = ['#ffd27f','#ff8a8a','#7fbfff'][p.weaponIndex % 3] || '#ffd27f';
        ctx.beginPath(); ctx.arc(sx + shakeX, canvas.height/2 + shakeY, Math.max(4, s.size/3), 0, Math.PI*2); ctx.fill();
        ctx.fillStyle = '#111'; ctx.font = '10px monospace'; ctx.fillText(weapons[p.weaponIndex].name[0], s.screenX - 5 + shakeX, canvas.height/2 + 4 + shakeY);
      }
    });

    // bullets projected
    bullets.forEach(b=>{
      const dx=b.x-posX, dy=b.y-posY; const invDet = 1.0/(planeX*dirY - dirX*planeY + 1e-9);
      const tx = invDet * (dirY*dx - dirX*dy); const ty = invDet * (-planeY*dx + planeX*dy);
      if(ty>0.01){ const screenX = Math.floor((screenW/2)*(1+tx/ty)); const size = Math.max(2, Math.floor(6/ty)); ctx.fillStyle='rgba(255,220,100,0.95)'; ctx.fillRect(screenX-1+shakeX, canvas.height/2 - size/2 + shakeY, 2, size); }
    });

    // hud
    ctx.fillStyle = 'rgba(255,255,255,0.6)'; ctx.fillRect(screenW/2 - 1 + shakeX, screenH/2 - 6 + shakeY, 2, 12); ctx.fillRect(screenW/2 - 6 + shakeX, screenH/2 - 1 + shakeY, 12, 2);

    // left/right ghost
    if(leftTouch){ ctx.globalAlpha=0.45; ctx.fillStyle='white'; ctx.beginPath(); ctx.arc(leftTouch.baseX,leftTouch.baseY,36,0,Math.PI*2); ctx.fill(); ctx.fillStyle='rgba(255,255,255,0.9)'; ctx.beginPath(); ctx.arc(leftTouch.curX,leftTouch.curY,18,0,Math.PI*2); ctx.fill(); ctx.globalAlpha=1.0; }
    else { ctx.globalAlpha=0.06; ctx.beginPath(); ctx.arc(80,canvas.height-120,36,0,Math.PI*2); ctx.fill(); ctx.globalAlpha=1.0; }
    if(rightTouch){ ctx.globalAlpha=0.45; ctx.fillStyle='white'; ctx.beginPath(); ctx.arc(rightTouch.baseX,rightTouch.baseY,36,0,Math.PI*2); ctx.fill(); ctx.fillStyle='rgba(255,255,255,0.9)'; ctx.beginPath(); ctx.arc(rightTouch.curX,rightTouch.curY,18,0,Math.PI*2); ctx.fill(); ctx.globalAlpha=1.0; }
    else { ctx.globalAlpha=0.06; ctx.beginPath(); ctx.arc(canvas.width-80,canvas.height-120,36,0,Math.PI*2); ctx.fill(); ctx.globalAlpha=1.0; }

    // HUD text
    ctx.fillStyle='#ddd'; ctx.font='14px monospace'; ctx.fillText(weapons[currentWeapon].name + '  ['+bullets.length+' bullets]', 12, 22); ctx.fillText('enemies: '+enemies.length, 12, 42); ctx.fillText('pickups: '+pickups.length, 12, 62);
  }

  // main loop
  function loop(){
    const now = performance.now(); const dt = now - last; last = now;
    update(dt); draw();
    requestAnimationFrame(loop);
  }
  loop();

  // keyboard shortcuts
  window.addEventListener('keydown', e=>{
    if(e.key === ' ') shoot();
    if(e.key === '1'){ currentWeapon=0; swapBtn.textContent=weapons[currentWeapon].name; }
    if(e.key === '2'){ currentWeapon=1; swapBtn.textContent=weapons[currentWeapon].name; }
    if(e.key === '3'){ currentWeapon=2; swapBtn.textContent=weapons[currentWeapon].name; }
  });

})();
</script>
</body>
</html>
