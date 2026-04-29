# ダースベーダーゲーム Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 横スクロールサバイバル型ダースベーダーゲームを単一HTMLファイルで実装する

**Architecture:** HTML5 Canvas + バニラJS のゲームループ。単一 `darth-vader-game.html` に全コードを収める。ゲーム状態 (START/PLAYING/GAMEOVER) を中心に、星・ベーダー・ビーム・敵・パーティクルをオブジェクト配列で管理する。

**Tech Stack:** HTML5 Canvas, Vanilla JavaScript (ES6), requestAnimationFrame

---

## File Structure

- Create: `darth-vader-game.html` — ゲーム全体（HTML + CSS + JS）

---

### Task 1: HTML シェル + Canvas + ゲームループ

**Files:**
- Create: `darth-vader-game.html`

- [ ] **Step 1: ファイルを作成する**

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>Darth Vader Game</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { background: #000; display: flex; justify-content: center; align-items: center; height: 100vh; overflow: hidden; }
    canvas { display: block; border: 1px solid #1a1a3e; }
  </style>
</head>
<body>
<canvas id="gameCanvas" width="800" height="500"></canvas>
<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const W = canvas.width;
const H = canvas.height;

// ゲーム状態: 'START' | 'PLAYING' | 'GAMEOVER'
let state = 'START';

// キー入力
const keys = {};
window.addEventListener('keydown', e => { keys[e.code] = true; });
window.addEventListener('keyup', e => { keys[e.code] = false; });

function update(dt) {
  // 各フェーズで処理を分岐（後のタスクで埋める）
}

function draw() {
  ctx.fillStyle = '#05050f';
  ctx.fillRect(0, 0, W, H);
}

let lastTime = 0;
function loop(timestamp) {
  const dt = Math.min((timestamp - lastTime) / 1000, 0.05); // 秒、最大50ms
  lastTime = timestamp;
  update(dt);
  draw();
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);
</script>
</body>
</html>
```

- [ ] **Step 2: ブラウザで開いて確認**

`darth-vader-game.html` をブラウザで開く。
期待値：黒いキャンバス (800×500) が中央に表示される。コンソールエラーなし。

- [ ] **Step 3: コミット**

```bash
git init
git add darth-vader-game.html
git commit -m "feat: add HTML shell and canvas game loop"
```

---

### Task 2: 星空背景（視差スクロール）

**Files:**
- Modify: `darth-vader-game.html`

- [ ] **Step 1: 星データと更新処理を追加する**

`const keys = {};` の直後に以下を追加：

```js
// --- Starfield ---
const STARS = [];
(function initStars() {
  for (let i = 0; i < 80; i++) {
    STARS.push({
      x: Math.random() * W,
      y: Math.random() * H,
      r: Math.random() * 1.5 + 0.3,
      speed: Math.random() * 0.5 + 0.2, // layer1: slow
    });
  }
  for (let i = 0; i < 40; i++) {
    STARS.push({
      x: Math.random() * W,
      y: Math.random() * H,
      r: Math.random() * 1.0 + 0.5,
      speed: Math.random() * 1.0 + 0.8, // layer2: fast
    });
  }
})();

function updateStars(dt) {
  for (const s of STARS) {
    s.x -= s.speed * 60 * dt;
    if (s.x < 0) { s.x = W; s.y = Math.random() * H; }
  }
}

function drawStars() {
  for (const s of STARS) {
    ctx.fillStyle = `rgba(255,255,255,${s.speed > 0.7 ? 0.9 : 0.5})`;
    ctx.beginPath();
    ctx.arc(s.x, s.y, s.r, 0, Math.PI * 2);
    ctx.fill();
  }
}
```

- [ ] **Step 2: update/draw から呼び出す**

`update` 関数を以下に変更：

```js
function update(dt) {
  updateStars(dt);
}
```

`draw` 関数を以下に変更：

```js
function draw() {
  ctx.fillStyle = '#05050f';
  ctx.fillRect(0, 0, W, H);
  drawStars();
}
```

- [ ] **Step 3: ブラウザで確認**

ブラウザをリロード。期待値：速度の違う星が右から左に流れる。

- [ ] **Step 4: コミット**

```bash
git add darth-vader-game.html
git commit -m "feat: add parallax starfield background"
```

---

### Task 3: ダースベーダー（描画 + 上下移動）

**Files:**
- Modify: `darth-vader-game.html`

- [ ] **Step 1: ベーダーオブジェクトを追加する**

`updateStars` の定義の直前に追加：

```js
// --- Vader ---
const vader = {
  x: 100,
  y: H / 2,
  w: 60,
  h: 80,
  speed: 5,       // px/frame at 60fps
  hp: 3,
  invincible: 0,  // 無敵時間（秒）
  flashTimer: 0,  // 白フラッシュタイマー（秒）
};

function updateVader(dt) {
  if (state !== 'PLAYING') return;
  if (keys['ArrowLeft'] || keys['ArrowUp']) vader.y -= vader.speed * 60 * dt;
  if (keys['ArrowRight'] || keys['ArrowDown']) vader.y += vader.speed * 60 * dt;
  vader.y = Math.max(vader.h / 2, Math.min(H - vader.h / 2, vader.y));
  if (vader.invincible > 0) vader.invincible -= dt;
  if (vader.flashTimer > 0) vader.flashTimer -= dt;
}

function drawVader() {
  const x = vader.x;
  const y = vader.y;
  const w = vader.w;
  const h = vader.h;

  ctx.save();
  if (vader.flashTimer > 0) {
    ctx.globalCompositeOperation = 'source-over';
    ctx.filter = 'brightness(5)';
  }

  // 体（台形）
  ctx.fillStyle = '#1a1a1a';
  ctx.beginPath();
  ctx.moveTo(x - w * 0.3, y - h * 0.5);
  ctx.lineTo(x + w * 0.3, y - h * 0.5);
  ctx.lineTo(x + w * 0.4, y + h * 0.5);
  ctx.lineTo(x - w * 0.4, y + h * 0.5);
  ctx.closePath();
  ctx.fill();

  // ヘルメット（楕円）
  ctx.fillStyle = '#111';
  ctx.beginPath();
  ctx.ellipse(x, y - h * 0.25, w * 0.28, h * 0.3, 0, 0, Math.PI * 2);
  ctx.fill();

  // マスク（赤い目）
  ctx.fillStyle = '#ff2020';
  ctx.shadowColor = '#ff0000';
  ctx.shadowBlur = 8;
  ctx.fillRect(x - 10, y - h * 0.3, 6, 4);
  ctx.fillRect(x + 4, y - h * 0.3, 6, 4);
  ctx.shadowBlur = 0;

  // マント
  ctx.fillStyle = '#0a0a0a';
  ctx.beginPath();
  ctx.moveTo(x - w * 0.4, y - h * 0.1);
  ctx.lineTo(x - w * 0.55, y + h * 0.5);
  ctx.lineTo(x + w * 0.55, y + h * 0.5);
  ctx.lineTo(x + w * 0.4, y - h * 0.1);
  ctx.closePath();
  ctx.fill();

  ctx.restore();
}
```

- [ ] **Step 2: update/draw から呼び出す**

`update` を変更：

```js
function update(dt) {
  updateStars(dt);
  updateVader(dt);
}
```

`draw` を変更：

```js
function draw() {
  ctx.fillStyle = '#05050f';
  ctx.fillRect(0, 0, W, H);
  drawStars();
  drawVader();
}
```

- [ ] **Step 3: state を PLAYING に仮設定して確認**

`let state = 'START';` を一時的に `let state = 'PLAYING';` に変更。
ブラウザでリロード。期待値：ベーダーが表示され ↑↓（または ←→）キーで上下に動く。

- [ ] **Step 4: state を START に戻してコミット**

`let state = 'PLAYING';` を `let state = 'START';` に戻す。

```bash
git add darth-vader-game.html
git commit -m "feat: add Vader character with keyboard movement"
```

---

### Task 4: ライトセーバービーム（発射 + 連射制限）

**Files:**
- Modify: `darth-vader-game.html`

- [ ] **Step 1: ビーム配列と発射ロジックを追加する**

`updateVader` の直後に追加：

```js
// --- Beams ---
const beams = [];
let beamCooldown = 0; // 秒

function fireBeam() {
  beams.push({
    x: vader.x + vader.w * 0.5,
    y: vader.y,
    w: 30,
    h: 4,
    speed: 600, // px/秒
  });
}

function updateBeams(dt) {
  if (state !== 'PLAYING') return;
  beamCooldown -= dt;
  if (keys['Space'] && beamCooldown <= 0) {
    fireBeam();
    beamCooldown = 0.3;
  }
  for (const b of beams) b.x += b.speed * dt;
  // 画面外を除去
  for (let i = beams.length - 1; i >= 0; i--) {
    if (beams[i].x > W + 50) beams.splice(i, 1);
  }
}

function drawBeams() {
  for (const b of beams) {
    ctx.save();
    ctx.shadowColor = '#ff0000';
    ctx.shadowBlur = 12;
    ctx.fillStyle = '#ff4040';
    ctx.fillRect(b.x - b.w / 2, b.y - b.h / 2, b.w, b.h);
    // 内側の明るいコア
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(b.x - b.w / 2 + 2, b.y - 1, b.w - 4, 2);
    ctx.restore();
  }
}
```

- [ ] **Step 2: update/draw から呼び出す**

```js
function update(dt) {
  updateStars(dt);
  updateVader(dt);
  updateBeams(dt);
}

function draw() {
  ctx.fillStyle = '#05050f';
  ctx.fillRect(0, 0, W, H);
  drawStars();
  drawVader();
  drawBeams();
}
```

- [ ] **Step 3: 確認（一時的に PLAYING に）**

`state = 'PLAYING'` に変更してリロード。スペースキーで赤いグロービームが右に飛ぶことを確認。0.3秒の連射制限を確認。

- [ ] **Step 4: state を START に戻してコミット**

```bash
git add darth-vader-game.html
git commit -m "feat: add lightsaber beam with cooldown"
```

---

### Task 5: 敵（X翼）の出現・移動・消滅

**Files:**
- Modify: `darth-vader-game.html`

- [ ] **Step 1: 敵システムを追加する**

`updateBeams` の直後に追加：

```js
// --- Enemies ---
const enemies = [];
let spawnTimer = 0;
let playTime = 0; // PLAYING 中の経過秒

function getDifficulty() {
  if (playTime < 30) return { interval: 2.0, speed: 2, types: ['xwing'] };
  if (playTime < 60) return { interval: 1.5, speed: 3, types: ['xwing', 'blaster'] };
  return { interval: 1.0, speed: 4, types: ['xwing', 'blaster'] };
}

function spawnEnemy() {
  const diff = getDifficulty();
  const type = diff.types[Math.floor(Math.random() * diff.types.length)];
  if (type === 'xwing') {
    enemies.push({
      type: 'xwing',
      x: W + 30,
      y: Math.random() * (H - 80) + 40,
      w: 50,
      h: 30,
      speed: diff.speed * 60, // px/秒換算
    });
  } else {
    enemies.push({
      type: 'blaster',
      x: W + 10,
      y: Math.random() * (H - 40) + 20,
      w: 10,
      h: 6,
      speed: (diff.speed + 2) * 60,
    });
  }
}

function updateEnemies(dt) {
  if (state !== 'PLAYING') return;
  playTime += dt;
  const diff = getDifficulty();
  spawnTimer -= dt;
  if (spawnTimer <= 0) {
    spawnEnemy();
    spawnTimer = diff.interval;
  }
  for (const e of enemies) e.x -= e.speed * dt;
  for (let i = enemies.length - 1; i >= 0; i--) {
    if (enemies[i].x < -100) enemies.splice(i, 1);
  }
}

function drawXWing(e) {
  ctx.save();
  // 機体
  ctx.fillStyle = '#aaaacc';
  ctx.fillRect(e.x - e.w / 2, e.y - 5, e.w, 10);
  // コックピット
  ctx.fillStyle = '#334466';
  ctx.fillRect(e.x - 8, e.y - 8, 16, 16);
  // 上翼
  ctx.fillStyle = '#8888aa';
  ctx.beginPath();
  ctx.moveTo(e.x - 5, e.y - 5);
  ctx.lineTo(e.x - e.w * 0.6, e.y - e.h / 2);
  ctx.lineTo(e.x + e.w * 0.3, e.y - 5);
  ctx.closePath();
  ctx.fill();
  // 下翼
  ctx.beginPath();
  ctx.moveTo(e.x - 5, e.y + 5);
  ctx.lineTo(e.x - e.w * 0.6, e.y + e.h / 2);
  ctx.lineTo(e.x + e.w * 0.3, e.y + 5);
  ctx.closePath();
  ctx.fill();
  // エンジン（赤グロー）
  ctx.shadowColor = '#ff6600';
  ctx.shadowBlur = 10;
  ctx.fillStyle = '#ff8800';
  ctx.fillRect(e.x + e.w / 2 - 4, e.y - 3, 6, 6);
  ctx.restore();
}

function drawBlaster(e) {
  ctx.save();
  ctx.shadowColor = '#00ff44';
  ctx.shadowBlur = 8;
  ctx.fillStyle = '#44ff66';
  ctx.fillRect(e.x - e.w / 2, e.y - e.h / 2, e.w, e.h);
  ctx.fillStyle = '#ffffff';
  ctx.fillRect(e.x - e.w / 2 + 1, e.y - 1, e.w - 2, 2);
  ctx.restore();
}

function drawEnemies() {
  for (const e of enemies) {
    if (e.type === 'xwing') drawXWing(e);
    else drawBlaster(e);
  }
}
```

- [ ] **Step 2: update/draw から呼び出す**

```js
function update(dt) {
  updateStars(dt);
  updateVader(dt);
  updateBeams(dt);
  updateEnemies(dt);
}

function draw() {
  ctx.fillStyle = '#05050f';
  ctx.fillRect(0, 0, W, H);
  drawStars();
  drawEnemies();
  drawVader();
  drawBeams();
}
```

- [ ] **Step 3: 確認（一時的に PLAYING に）**

X翼が右から飛んでくることを確認。30秒後にブラスター弾が混じることを確認（`playTime = 31` に一時変更でも可）。

- [ ] **Step 4: state を START に戻してコミット**

```bash
git add darth-vader-game.html
git commit -m "feat: add X-Wing and blaster bolt enemies with difficulty scaling"
```

---

### Task 6: 衝突判定

**Files:**
- Modify: `darth-vader-game.html`

- [ ] **Step 1: AABB 衝突判定ヘルパーを追加する**

`// --- Enemies ---` の直前に追加：

```js
// --- Collision ---
function rectsOverlap(ax, ay, aw, ah, bx, by, bw, bh) {
  return ax - aw / 2 < bx + bw / 2 &&
         ax + aw / 2 > bx - bw / 2 &&
         ay - ah / 2 < by + bh / 2 &&
         ay + ah / 2 > by - bh / 2;
}
```

- [ ] **Step 2: スコアとパーティクルのプレースホルダーを追加する**

`const enemies = [];` の直前に追加：

```js
// --- Score ---
let score = 0;

// パーティクル（Task 7 で実装）
function spawnParticles(x, y) { /* later */ }
```

- [ ] **Step 3: checkCollisions 関数を追加する**

`updateEnemies` の直後に追加：

```js
function checkCollisions() {
  if (state !== 'PLAYING') return;

  // ビーム vs 敵
  for (let bi = beams.length - 1; bi >= 0; bi--) {
    const b = beams[bi];
    for (let ei = enemies.length - 1; ei >= 0; ei--) {
      const e = enemies[ei];
      if (rectsOverlap(b.x, b.y, b.w, b.h, e.x, e.y, e.w, e.h)) {
        spawnParticles(e.x, e.y);
        score += e.type === 'xwing' ? 50 : 20;
        enemies.splice(ei, 1);
        beams.splice(bi, 1);
        break;
      }
    }
  }

  // 敵 vs ベーダー
  if (vader.invincible <= 0) {
    for (let ei = enemies.length - 1; ei >= 0; ei--) {
      const e = enemies[ei];
      if (rectsOverlap(vader.x, vader.y, vader.w * 0.6, vader.h * 0.8, e.x, e.y, e.w, e.h)) {
        vader.hp -= 1;
        vader.invincible = 1.0;
        vader.flashTimer = 0.3;
        enemies.splice(ei, 1);
        if (vader.hp <= 0) state = 'GAMEOVER';
      }
    }
  }
}
```

- [ ] **Step 4: update から呼び出す**

```js
function update(dt) {
  updateStars(dt);
  updateVader(dt);
  updateBeams(dt);
  updateEnemies(dt);
  checkCollisions();
}
```

- [ ] **Step 5: 確認（一時的に PLAYING に）**

ビームが当たると敵が消えスコアが増えること（後のタスクで表示）、体当たりで state が GAMEOVER になることをコンソールで確認。

- [ ] **Step 6: state を START に戻してコミット**

```bash
git add darth-vader-game.html
git commit -m "feat: add AABB collision detection for beams and enemies"
```

---

### Task 7: パーティクル爆発エフェクト

**Files:**
- Modify: `darth-vader-game.html`

- [ ] **Step 1: パーティクル配列と実装を追加する**

`// --- Score ---` の直後（`let score = 0;` の直後）に追加：

```js
// --- Particles ---
const particles = [];

function spawnParticles(x, y) {
  const count = 12 + Math.floor(Math.random() * 6);
  for (let i = 0; i < count; i++) {
    const angle = (Math.PI * 2 / count) * i + Math.random() * 0.3;
    const speed = 80 + Math.random() * 120;
    particles.push({
      x, y,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed,
      life: 0.5 + Math.random() * 0.2,
      maxLife: 0.5 + Math.random() * 0.2,
      r: 2 + Math.random() * 3,
      color: Math.random() < 0.6 ? '#ff8800' : '#ffee00',
    });
  }
}

function updateParticles(dt) {
  for (const p of particles) {
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.vx *= 0.95;
    p.vy *= 0.95;
    p.life -= dt;
  }
  for (let i = particles.length - 1; i >= 0; i--) {
    if (particles[i].life <= 0) particles.splice(i, 1);
  }
}

function drawParticles() {
  for (const p of particles) {
    const alpha = p.life / p.maxLife;
    ctx.save();
    ctx.globalAlpha = alpha;
    ctx.shadowColor = p.color;
    ctx.shadowBlur = 6;
    ctx.fillStyle = p.color;
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.r * alpha, 0, Math.PI * 2);
    ctx.fill();
    ctx.restore();
  }
}
```

- [ ] **Step 2: 以前の spawnParticles プレースホルダーを削除する**

```js
// パーティクル（Task 7 で実装）
function spawnParticles(x, y) { /* later */ }
```
この行を削除する（新しい実装で置き換えられた）。

- [ ] **Step 3: update/draw から呼び出す**

```js
function update(dt) {
  updateStars(dt);
  updateVader(dt);
  updateBeams(dt);
  updateEnemies(dt);
  checkCollisions();
  updateParticles(dt);
}

function draw() {
  ctx.fillStyle = '#05050f';
  ctx.fillRect(0, 0, W, H);
  drawStars();
  drawEnemies();
  drawParticles();
  drawVader();
  drawBeams();
}
```

- [ ] **Step 4: 確認（PLAYING に）**

敵撃墜時にオレンジ・黄色の爆発パーティクルが出ることを確認。

- [ ] **Step 5: state を START に戻してコミット**

```bash
git add darth-vader-game.html
git commit -m "feat: add particle explosion effects on enemy kill"
```

---

### Task 8: HUD（スコア + HP）

**Files:**
- Modify: `darth-vader-game.html`

- [ ] **Step 1: HUD 描画関数を追加する**

`drawParticles` の直後に追加：

```js
// --- HUD ---
function drawHUD() {
  if (state !== 'PLAYING') return;

  ctx.save();
  ctx.font = 'bold 18px "Courier New"';
  ctx.textAlign = 'right';
  ctx.fillStyle = '#ccccff';
  ctx.shadowColor = '#6666ff';
  ctx.shadowBlur = 6;
  ctx.fillText(`SCORE: ${score}`, W - 20, 30);

  // HP ハート（赤い四角で表現）
  ctx.textAlign = 'right';
  ctx.font = '14px "Courier New"';
  ctx.fillStyle = '#aaaaaa';
  ctx.shadowBlur = 0;
  ctx.fillText('HP', W - 20, 54);
  for (let i = 0; i < 3; i++) {
    const filled = i < vader.hp;
    ctx.shadowColor = filled ? '#ff2020' : 'transparent';
    ctx.shadowBlur = filled ? 8 : 0;
    ctx.fillStyle = filled ? '#ff3030' : '#333333';
    ctx.fillRect(W - 80 + i * 22, 58, 16, 16);
  }

  ctx.restore();
}
```

- [ ] **Step 2: draw から呼び出す（最後に描く）**

```js
function draw() {
  ctx.fillStyle = '#05050f';
  ctx.fillRect(0, 0, W, H);
  drawStars();
  drawEnemies();
  drawParticles();
  drawVader();
  drawBeams();
  drawHUD();
}
```

- [ ] **Step 3: 確認（PLAYING に）**

右上にスコアと HP が表示されることを確認。敵に当たると HP が減ることを確認。

- [ ] **Step 4: state を START に戻してコミット**

```bash
git add darth-vader-game.html
git commit -m "feat: add HUD with score and HP display"
```

---

### Task 9: ゲーム状態（START / GAMEOVER 画面）

**Files:**
- Modify: `darth-vader-game.html`

- [ ] **Step 1: 状態リセット関数を追加する**

`drawHUD` の直後に追加：

```js
// --- Game State Screens ---
function resetGame() {
  vader.y = H / 2;
  vader.hp = 3;
  vader.invincible = 0;
  vader.flashTimer = 0;
  beams.length = 0;
  enemies.length = 0;
  particles.length = 0;
  score = 0;
  playTime = 0;
  spawnTimer = 0;
  beamCooldown = 0;
  state = 'PLAYING';
}
```

- [ ] **Step 2: スペースキーでの遷移を update に追加する**

`update` 関数の先頭（`updateStars` の前）に追加：

```js
function update(dt) {
  // 状態遷移
  if (state === 'START' && keys['Space']) {
    keys['Space'] = false;
    resetGame();
  }
  if (state === 'GAMEOVER' && keys['Space']) {
    keys['Space'] = false;
    resetGame();
  }

  updateStars(dt);
  updateVader(dt);
  updateBeams(dt);
  updateEnemies(dt);
  checkCollisions();
  updateParticles(dt);
}
```

- [ ] **Step 3: 生存時間スコアを update に追加する**

`checkCollisions();` の直後に追加：

```js
  if (state === 'PLAYING') score += Math.floor(10 * dt);
```

- [ ] **Step 4: START / GAMEOVER 画面の描画関数を追加する**

`resetGame` の直後に追加：

```js
function drawStartScreen() {
  ctx.save();
  // タイトル
  ctx.font = 'bold 48px "Courier New"';
  ctx.textAlign = 'center';
  ctx.fillStyle = '#cc0000';
  ctx.shadowColor = '#ff0000';
  ctx.shadowBlur = 24;
  ctx.fillText('DARTH VADER', W / 2, H / 2 - 60);

  ctx.font = 'bold 24px "Courier New"';
  ctx.fillStyle = '#888899';
  ctx.shadowColor = '#4444aa';
  ctx.shadowBlur = 8;
  ctx.fillText('SURVIVAL', W / 2, H / 2 - 20);

  ctx.font = '16px "Courier New"';
  ctx.fillStyle = '#666688';
  ctx.shadowBlur = 0;
  ctx.fillText('↑↓ で移動 / SPACE で攻撃', W / 2, H / 2 + 30);

  // 点滅テキスト
  if (Math.floor(Date.now() / 500) % 2 === 0) {
    ctx.fillStyle = '#aaaaff';
    ctx.shadowColor = '#6666ff';
    ctx.shadowBlur = 10;
    ctx.fillText('[ SPACE ] START', W / 2, H / 2 + 80);
  }
  ctx.restore();
}

function drawGameOverScreen() {
  ctx.save();
  ctx.fillStyle = 'rgba(0,0,0,0.6)';
  ctx.fillRect(0, 0, W, H);

  ctx.font = 'bold 52px "Courier New"';
  ctx.textAlign = 'center';
  ctx.fillStyle = '#ff2020';
  ctx.shadowColor = '#ff0000';
  ctx.shadowBlur = 30;
  ctx.fillText('GAME OVER', W / 2, H / 2 - 50);

  ctx.font = 'bold 24px "Courier New"';
  ctx.fillStyle = '#ccccff';
  ctx.shadowColor = '#6666ff';
  ctx.shadowBlur = 10;
  ctx.fillText(`SCORE: ${score}`, W / 2, H / 2 + 10);

  if (Math.floor(Date.now() / 500) % 2 === 0) {
    ctx.font = '16px "Courier New"';
    ctx.fillStyle = '#aaaaff';
    ctx.shadowColor = '#6666ff';
    ctx.shadowBlur = 8;
    ctx.fillText('[ SPACE ] RETRY', W / 2, H / 2 + 60);
  }
  ctx.restore();
}
```

- [ ] **Step 5: draw から呼び出す**

```js
function draw() {
  ctx.fillStyle = '#05050f';
  ctx.fillRect(0, 0, W, H);
  drawStars();
  drawEnemies();
  drawParticles();
  drawVader();
  drawBeams();
  drawHUD();
  if (state === 'START') drawStartScreen();
  if (state === 'GAMEOVER') drawGameOverScreen();
}
```

- [ ] **Step 6: ブラウザで最終確認**

1. スタート画面が表示される
2. スペースでゲーム開始
3. 敵に3回当たるとゲームオーバー画面
4. スペースでリトライ

- [ ] **Step 7: コミット**

```bash
git add darth-vader-game.html
git commit -m "feat: add START and GAMEOVER screens with state transitions"
```

---

## セルフレビュー（スペック対照）

| スペック要件 | タスク |
|-------------|--------|
| 横スクロールサバイバル | Task 5 |
| 単一HTMLファイル | Task 1 |
| 60fps ゲームループ | Task 1 |
| 視差スクロール星 | Task 2 |
| ベーダー ← → 上下移動 | Task 3 |
| スペースでビーム発射、0.3秒制限 | Task 4 |
| X翼・ブラスター弾 | Task 5 |
| 衝突判定 | Task 6 |
| パーティクル爆発 | Task 7 |
| スコア（時間+撃墜） | Task 8, 9 |
| HP 3、ダメージフラッシュ、無敵時間 | Task 3, 6 |
| 難易度スケーリング（30s/60s） | Task 5 |
| グローエフェクト | Task 4, 5, 7 |
| START/GAMEOVER 画面 | Task 9 |
