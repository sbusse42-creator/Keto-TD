/* Keto Drop Defense
   - Tap food to destroy (0 points)
   - Let food reach bottom: add/subtract points based on keto score
   - Win at +100, lose at -100
*/

const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

const scoreEl = document.getElementById("score");
const formLabelEl = document.getElementById("formLabel");
const intensityLabelEl = document.getElementById("intensityLabel");

const overlay = document.getElementById("overlay");
const resultTitle = document.getElementById("resultTitle");
const resultText = document.getElementById("resultText");
const btnRestart = document.getElementById("btnRestart");
const btnPlayAgain = document.getElementById("btnPlayAgain");

// SVG parts for "muscle" morphing
const avatarArmL = document.getElementById("armL");
const avatarArmR = document.getElementById("armR");
const avatarBody = document.querySelector("#body rect");

function clamp(n, a, b) { return Math.max(a, Math.min(b, n)); }
function lerp(a,b,t){ return a + (b-a)*t; }

// Food catalog: emoji + name + ketoScore (-10..+10)
const FOODS = [
  // Sehr keto-tauglich (+)
  { name: "Avocado", icon: "🥑", keto: +9 },
  { name: "Lachs", icon: "🐟", keto: +8 },
  { name: "Rind", icon: "🥩", keto: +8 },
  { name: "Eier", icon: "🥚", keto: +7 },
  { name: "Brokkoli", icon: "🥦", keto: +7 },
  { name: "Olivenöl", icon: "🫒", keto: +8 },
  { name: "Käse", icon: "🧀", keto: +6 },
  { name: "Nüsse", icon: "🥜", keto: +5 },
  { name: "Spinat", icon: "🥬", keto: +6 },

  // Neutral / eher bedingt
  { name: "Joghurt", icon: "🥣", keto: +2 },
  { name: "Beeren", icon: "🫐", keto: +1 },

  // Schlechter für Keto (-)
  { name: "Banane", icon: "🍌", keto: -4 },
  { name: "Apfel", icon: "🍎", keto: -3 },
  { name: "Kartoffeln", icon: "🥔", keto: -7 },
  { name: "Reis", icon: "🍚", keto: -8 },
  { name: "Brot", icon: "🍞", keto: -8 },
  { name: "Nudeln", icon: "🍝", keto: -9 },
  { name: "Kuchen", icon: "🍰", keto: -10 },
  { name: "Limo", icon: "🥤", keto: -9 },
  { name: "Süßigkeiten", icon: "🍬", keto: -10 },
];

function ketoToColor(ketoScore){
  // -10 (rot) .. +10 (grün)
  const t = (ketoScore + 10) / 20; // 0..1
  const r = Math.round(lerp(220, 34, t));
  const g = Math.round(lerp(38, 197, t));
  const b = Math.round(lerp(38, 94, t));
  return `rgb(${r},${g},${b})`;
}

let state;

function resetGame(){
  state = {
    running: true,
    score: 0,
    items: [],
    lastSpawnAt: 0,
    spawnEveryMs: 650,
    speedMin: 120,
    speedMax: 260,
    targetWin: 100,
    targetLose: -100,
    lastFrameTs: performance.now(),
  };

  hideOverlay();
  updateHud();
  applyAvatarMorph();
}

function showOverlay(title, text){
  resultTitle.textContent = title;
  resultText.textContent = text;
  overlay.classList.remove("hidden");
  state.running = false;
}
function hideOverlay(){
  overlay.classList.add("hidden");
}

function updateHud(){
  scoreEl.textContent = String(state.score);

  if (state.score >= 70) formLabelEl.textContent = "Bodybuilder";
  else if (state.score >= 30) formLabelEl.textContent = "Athletisch";
  else if (state.score >= -20) formLabelEl.textContent = "Durchschnitt";
  else if (state.score >= -60) formLabelEl.textContent = "Schmächtig";
  else formLabelEl.textContent = "Ausgemergelt";

  if (state.score >= 0) intensityLabelEl.textContent = "Anabol 🔥";
  else intensityLabelEl.textContent = "Katabol 🥶";
}

function applyAvatarMorph(){
  const t = (clamp(state.score, -100, 100) + 100) / 200;

  // Body width: from thin to wide
  // avatarBody base: x=70, width=80
  const bodyWidth = lerp(58, 112, t);
  const bodyX = 110 - bodyWidth/2;

  avatarBody.setAttribute("x", bodyX.toFixed(1));
  avatarBody.setAttribute("width", bodyWidth.toFixed(1));

  // Arms thickness: width from 22..46
  const armW = lerp(22, 46, t);
  avatarArmL.setAttribute("x", (58 - armW/2).toFixed(1));
  avatarArmL.setAttribute("width", armW.toFixed(1));

  avatarArmR.setAttribute("x", (162 - armW/2).toFixed(1));
  avatarArmR.setAttribute("width", armW.toFixed(1));

  const bodyColor = `rgb(${Math.round(lerp(148, 120, t))},${Math.round(lerp(163, 210, t))},${Math.round(lerp(184, 160, t))})`;
  avatarBody.style.fill = bodyColor;
  avatarArmL.style.fill = bodyColor;
  avatarArmR.style.fill = bodyColor;
}

function randInt(a,b){ return Math.floor(a + Math.random()*(b-a+1)); }
function pick(arr){ return arr[Math.floor(Math.random()*arr.length)]; }

function spawnItem(){
  const f = pick(FOODS);
  const r = 26; // hit radius
  const x = randInt(r+10, canvas.width - r - 10);
  const y = -r - 10;
  const v = lerp(state.speedMin, state.speedMax, Math.random());

  state.items.push({
    food: f,
    x, y,
    r,
    vy: v,
  });
}

function scoreForFood(keto){
  const mag = clamp(Math.abs(keto), 1, 10);
  return keto >= 0 ? mag : -mag;
}

function endIfNeeded(){
  if (state.score >= state.targetWin){
    showOverlay("Gewonnen ✅", `Du hast ${state.score} Punkte erreicht. Keto-Form auf Maximum.`);
  } else if (state.score <= state.targetLose){
    showOverlay("Verloren ❌", `Du bist bei ${state.score} Punkten. Zu viele Carbs durchgelassen.`);
  }
}

function step(dt){
  // Increase difficulty slightly over time: faster spawn
  state.spawnEveryMs = clamp(state.spawnEveryMs - dt*0.01*1000, 380, 650);
  state.speedMin = clamp(state.speedMin + dt*0.01*1000, 120, 240);
  state.speedMax = clamp(state.speedMax + dt*0.01*1000, 260, 420);

  // spawn (dt in seconds => use ms accumulator)
  state.lastSpawnAt += dt*1000;
  if (state.lastSpawnAt >= state.spawnEveryMs){
    state.lastSpawnAt = 0;
    spawnItem();
  }

  // move items
  for (const it of state.items){
    it.y += it.vy * dt;
  }

  // bottom collision => score
  const bottomY = canvas.height - 22; // floor margin
  const remaining = [];
  for (const it of state.items){
    if (it.y + it.r >= bottomY){
      const delta = scoreForFood(it.food.keto);
      state.score += delta;
      updateHud();
      applyAvatarMorph();
      endIfNeeded();
      continue;
    }
    remaining.push(it);
  }
  state.items = remaining;
}

function draw(){
  ctx.clearRect(0,0,canvas.width,canvas.height);

  // Background grid
  ctx.save();
  ctx.globalAlpha = 0.12;
  ctx.strokeStyle = "#94a3b8";
  ctx.lineWidth = 1;
  for (let y=40; y<canvas.height; y+=40){
    ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(canvas.width,y); ctx.stroke();
  }
  for (let x=40; x<canvas.width; x+=40){
    ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,canvas.height); ctx.stroke();
  }
  ctx.restore();

  // Floor line
  ctx.save();
  ctx.strokeStyle = "rgba(148,163,184,.35)";
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(0, canvas.height - 22);
  ctx.lineTo(canvas.width, canvas.height - 22);
  ctx.stroke();
  ctx.restore();

  // Items
  for (const it of state.items){
    const { food } = it;
    const c = ketoToColor(food.keto);

    ctx.save();
    ctx.beginPath();
    ctx.arc(it.x, it.y, it.r+10, 0, Math.PI*2);
    ctx.fillStyle = "rgba(15,23,42,.55)";
    ctx.fill();
    ctx.strokeStyle = "rgba(148,163,184,.22)";
    ctx.lineWidth = 2;
    ctx.stroke();
    ctx.restore();

    // emoji icon
    ctx.save();
    ctx.font = "28px system-ui, Apple Color Emoji, Segoe UI Emoji";
    ctx.textAlign = "center";
    ctx.textBaseline = "middle";
    ctx.fillText(food.icon, it.x, it.y - 6);
    ctx.restore();

    // name colored
    ctx.save();
    ctx.font = "12px system-ui";
    ctx.textAlign = "center";
    ctx.textBaseline = "top";
    ctx.fillStyle = c;
    ctx.fillText(food.name, it.x, it.y + 18);
    ctx.restore();

    // small hint score (+/-)
    const delta = scoreForFood(food.keto);
    ctx.save();
    ctx.font = "11px system-ui";
    ctx.textAlign = "center";
    ctx.textBaseline = "top";
    ctx.fillStyle = "rgba(226,232,240,.9)";
    ctx.fillText(delta > 0 ? `+${delta}` : `${delta}`, it.x, it.y + 33);
    ctx.restore();
  }

  // Top instructions inside canvas
  ctx.save();
  ctx.fillStyle = "rgba(226,232,240,.75)";
  ctx.font = "12px system-ui";
  ctx.fillText("Tippe Food weg (0 Punkte) · Durchlassen = +/-", 12, 14);
  ctx.restore();
}

function loop(ts){
  const dt = Math.min(0.033, (ts - state.lastFrameTs)/1000);
  state.lastFrameTs = ts;

  if (state.running){
    step(dt);
  }
  draw();

  requestAnimationFrame(loop);
}

// Input: tap/click destroy if hit
function getCanvasPoint(evt){
  const rect = canvas.getBoundingClientRect();
  const clientX = evt.touches ? evt.touches[0].clientX : evt.clientX;
  const clientY = evt.touches ? evt.touches[0].clientY : evt.clientY;

  const sx = canvas.width / rect.width;
  const sy = canvas.height / rect.height;
  return {
    x: (clientX - rect.left) * sx,
    y: (clientY - rect.top) * sy
  };
}

function onTap(evt){
  evt.preventDefault();
  if (!state.running) return;

  const p = getCanvasPoint(evt);

  let hitIndex = -1;
  let best = Infinity;
  for (let i=0; i<state.items.length; i++){
    const it = state.items[i];
    const dx = it.x - p.x;
    const dy = it.y - p.y;
    const d2 = dx*dx + dy*dy;
    if (d2 <= (it.r+14)*(it.r+14)){
      if (d2 < best){
        best = d2;
        hitIndex = i;
      }
    }
  }

  if (hitIndex >= 0){
    state.items.splice(hitIndex, 1);
  }
}

canvas.addEventListener("click", onTap, { passive: false });
canvas.addEventListener("touchstart", onTap, { passive: false });

btnRestart.addEventListener("click", resetGame);
btnPlayAgain.addEventListener("click", resetGame);

resetGame();
requestAnimationFrame(loop);
