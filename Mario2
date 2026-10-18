/* ============================================================
 * 网页版超级玛丽 v0.1
 * 严格按 PRD（v0.1 / 2026-10-18）实现
 * ------------------------------------------------------------
 * 技术约定：
 *  - 原生 HTML + CSS + JS，单页，零构建、零依赖
 *  - Canvas 2D，imageSmoothingEnabled = false
 *  - 逻辑分辨率 256×240，CSS 整数倍放大
 *  - 1 格 = 16px
 *  - requestAnimationFrame + 固定步长 1/60 累加器
 *  - 速度单位 px/s，物理步长恒定
 * ============================================================ */
(function () {
  'use strict';

  /* ==========================================================
   * 0. 调参面板（PRD 2 节：物理参数集中放在 CONFIG 顶部）
   * ========================================================== */
  const CONFIG = {
    // ---- 移动（3.1）----
    // 数值反解：到全速时间 = v/a，滑行距离 = v²/(2f)
    //   走速 90 / 加速度 180 → 0.5s（目标 0.4~0.6s）
    //   跑速 155 / 加速度 260（见 getAccel）→ 0.6s
    //   走速 90 / 摩擦 90 → 45px（目标 40~60px）
    walkSpeed: 90,          // 走速上限 px/s
    runSpeed: 155,          // 跑速上限 px/s（约走速 1.7 倍）
    groundAccel: 170,       // 地面走速加速度 px/s²
    runAccel: 260,          // 地面跑速加速度 px/s²
    airAccel: 300,          // 空中加速度 px/s²（约地面 70%~100%，保证能救回）
    turnAccel: 900,         // 转向加速度 px/s²
    groundFriction: 115,     // 地面摩擦 px/s²（全速滑行 40~60px）

    // ---- 重力（3.2）----
    gravityUp: 2000,        // 上升阶段重力 px/s²
    gravityDown: 3400,      // 下落阶段重力 px/s²（1.7 倍）
    maxFallSpeed: 420,      // 最大下落速度 px/s

    // ---- 跳跃（3.3）----
    // 说明（用数值反解得出，见 PRD 8 节验收）：
    //   长按高度 64px  → v0 = sqrt(2*2000*64) ≈ 506
    //   轻点高度 24px  → 松键后剩速 = sqrt(2*2000*24) ≈ 310 → 310/506 ≈ 0.59
    // 因此 cutMultiplier 取 0.59 才能让「轻点 1.5 格 / 长按 4 格」同时成立，
    // 若严格取 PRD 参考值 0.4，轻点只有 16px（约 1 格）。
    jumpSpeed: 506,         // 基础起跳竖直初速 px/s（向上，代码里取负）
    runJumpBonus: 66,       // 全速跑时额外初速 px/s（506+60=566 → 80px = 5 格）
    jumpCutMultiplier: 0.59,// 松键截断倍率
    jumpCutMinSpeed: 120,   // 截断后的最低上升速度 px/s（防止"完全刹死"）
    jumpCutDelay: 0.034,    // 起跳后 2 帧内不响应松键（保证轻点也有 1.5 格）
    coyoteTime: 0.100,      // 土狼时间 100ms
    jumpBufferTime: 0.120,  // 跳跃缓冲 120ms
    stompBounce: 253,       // 踩怪弹起速度 px/s（约 0.5 倍跳跃高度）

    // ---- 金币与分数（3.4）----
    coinScore: 200,
    stompScore: 100,
    coinsPerLife: 100,
    floatTextDuration: 0.6,
    floatTextRise: 12,      // 飘字上浮像素
    coinPopHeight: 24,      // 金币上浮高度
    coinPopDuration: 0.45,

    // ---- 敌人（3.5）----
    enemySpeed: 30,
    enemySquashTime: 0.3,

    // ---- 游戏流程（3.6）----
    startLives: 3,
    deathHopSpeed: 420,
    deathHopGravity: 2000,
    deathPause: 1.1,        // 死亡后到重生的停顿（秒）
  };

  /* ==========================================================
   * 1. 调色板（PRD 6 节：全局唯一色源，不许零散硬编码色值）
   * ========================================================== */
  const PALETTE = {
    sky: '#5C94FC',
    cloud: '#FFFFFF',
    bush: '#FFFFFF',
    hill: '#00A800',
    groundBrick: '#C84C0C',
    groundGrass: '#00A800',
    brick: '#C84C0C',
    brickLine: '#000000',
    question: '#FCA044',
    usedBlock: '#8B4513',
    marioRed: '#D82800',
    marioSkin: '#FCBCB0',
    overalls: '#0058F8',
    goombaBrown: '#C84C0C',
    coin: '#FCA044',
    text: '#FFFFFF',
  };

  /* 同色降明度工具（PRD：需要暗部时用同色降低明度，不引入新色相） */
  function shade(hex, factor) {
    const n = parseInt(hex.slice(1), 16);
    const r = Math.round(((n >> 16) & 255) * factor);
    const g = Math.round(((n >> 8) & 255) * factor);
    const b = Math.round((n & 255) * factor);
    return 'rgb(' + r + ',' + g + ',' + b + ')';
  }
  const DARK = {
    brick: shade(PALETTE.brick, 0.72),
    groundBrick: shade(PALETTE.groundBrick, 0.72),
    question: shade(PALETTE.question, 0.78),
    usedBlock: shade(PALETTE.usedBlock, 0.78),
    goomba: shade(PALETTE.goombaBrown, 0.72),
    hill: shade(PALETTE.hill, 0.8),
    red: shade(PALETTE.marioRed, 0.7),
    overalls: shade(PALETTE.overalls, 0.7),
    skin: shade(PALETTE.marioSkin, 0.85),
    coin: shade(PALETTE.coin, 0.75),
    sky: shade(PALETTE.sky, 0.88),
  };

  /* 同色提亮（高光用，同样不引入新色相） */
  function tint(hex, amount) {
    const n = parseInt(hex.slice(1), 16);
    const r = Math.min(255, Math.round(((n >> 16) & 255) + (255 - ((n >> 16) & 255)) * amount));
    const g = Math.min(255, Math.round(((n >> 8) & 255) + (255 - ((n >> 8) & 255)) * amount));
    const b = Math.min(255, Math.round((n & 255) + (255 - (n & 255)) * amount));
    return 'rgb(' + r + ',' + g + ',' + b + ')';
  }
  const LIGHT = {
    hill: tint(PALETTE.hill, 0.55),
    groundGrass: tint(PALETTE.groundGrass, 0.55),
  };

  /* ==========================================================
   * 2. 基础常量（PRD 2 节）
   * ========================================================== */
  const VIEW_W = 256;
  const VIEW_H = 240;
  const TILE = 16;
  const COLS = VIEW_W / TILE;          // 16 列可视
  const GROUND_ROWS = 2;               // 关卡底部连续两行地面
  const ROWS = VIEW_H / TILE;          // 15 行
  const SOLID_TOP_ROW = ROWS - GROUND_ROWS; // 地面踩踏面所在行
  const GROUND_TOP = SOLID_TOP_ROW * TILE;  // 玩家踩踏面 y 坐标 = 208
  const FIXED_DT = 1 / 60;
  const MAX_STEPS = 5;

  /* ==========================================================
   * 3. 关卡数据（PRD 5 节：字符地图，一行字符串 = 一行图块）
   * ----------------------------------------------------------
   * 图例：. 空 / X 地面 / B 砖块 / ? 问号砖 / P 管道
   *      G 板栗仔 / M 玩家出生点 / C 金币 / F 旗杆（终点）
   *
   * 节奏（教学关）：起点留空 → 首只板栗仔 → 弧线顶点问号砖
   * → 一排 3~4 块砖 → 递增管道 → 有底小坑 → 致死大坑
   * → 2~3 只板栗仔组合 → 终点旗杆
   * ========================================================== */
  const LEVEL = [
    // col 0                                                                        120
    '................................................................................', // 0
    '................................................................................', // 1
    '................................................................................', // 2
    '................................................................................', // 3
    '................................................................................', // 4
    '................................................................................', // 5
    '..........................................C.....................................', // 6
    '.......................?....BBB....................CCC..........................', // 7
    '.........................................GG.....................................', // 8
    '....................................................PPP.........................', // 9
    '..........................................G.....PPPPPP..........................', // 10
    '...................G.........?.BB..G...........PPPPPPPP.....G...GG...............', // 11
    '.....M..........XXX........XXX...........X.XX...X...XXXXX...X...XX....XXX....F...', // 12
    'XXXXXXXXXXXXXXXX...XXXXXXXXXXXXXXXXXXXXXXXXXXX...XXXXXXXXXXXXXXXXXXXXXX...XXXX...', // 13
    'XXXXXXXXXXXXXXXX...XXXXXXXXXXXXXXXXXXXXXXXXXXX...XXXXXXXXXXXXXXXXXXXXXX...XXXX...', // 14
  ];

  const TILE_CHARS = {
    ground: 'X',
    brick: 'B',
    question: '?',
    pipe: 'P',
    coin: 'C',
    flag: 'F',
  };
  const SOLID_CHARS = 'XBP?';

  /* 远景装饰（PRD 4.1：云 0.3 倍、山丘 0.5 倍视差） */
  const CLOUDS = [
    { x: 60, y: 40 }, { x: 210, y: 28 }, { x: 355, y: 48 },
    { x: 520, y: 32 }, { x: 660, y: 44 }, { x: 810, y: 26 },
    { x: 950, y: 42 }, { x: 1100, y: 34 },
  ];
  const HILLS = [
    { x: 40, w: 5, h: 2 }, { x: 178, w: 4, h: 1 }, { x: 270, w: 5, h: 2 },
    { x: 400, w: 4, h: 1 }, { x: 500, w: 5, h: 2 }, { x: 600, w: 4, h: 2 },
    { x: 720, w: 5, h: 1 }, { x: 840, w: 4, h: 2 }, { x: 960, w: 5, h: 2 },
    { x: 1090, w: 4, h: 1 },
  ];
  const BUSHES = [
    { x: 130, w: 3 }, { x: 320, w: 3 }, { x: 480, w: 3 },
    { x: 640, w: 3 }, { x: 800, w: 3 }, { x: 1000, w: 3 },
  ];

  /* ==========================================================
   * 4. 画布与整数倍缩放（PRD 2 节）
   * ========================================================== */
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  ctx.imageSmoothingEnabled = false;

  function resizeCanvas() {
    const availW = window.innerWidth - 40;
    const availH = window.innerHeight - 110;
    // 整数倍放大，至少 1 倍
    let scale = Math.min(Math.floor(availW / VIEW_W), Math.floor(availH / VIEW_H));
    if (scale < 1) scale = 1;
    canvas.style.width = (VIEW_W * scale) + 'px';
    canvas.style.height = (VIEW_H * scale) + 'px';
  }
  window.addEventListener('resize', resizeCanvas);

  /* ==========================================================
   * 5. 输入（PRD 3.1 / 3.3 / 7 节键位）
   * ========================================================== */
  const Input = {
    left: false,
    right: false,
    jumpFired: false,   // 本帧是否有新的跳跃按下（边沿）
    jumpHeld: false,
    run: false,
    _jumpDown: false,

    bind: function () {
      const that = this;
      document.addEventListener('keydown', function (e) { that.onKey(e, true); });
      document.addEventListener('keyup', function (e) { that.onKey(e, false); });
    },

    onKey: function (e, down) {
      const k = e.code;
      let handled = true;
      if (k === 'ArrowLeft' || k === 'KeyA') this.left = down;
      else if (k === 'ArrowRight' || k === 'KeyD') this.right = down;
      else if (k === 'Space' || k === 'ArrowUp' || k === 'KeyW') {
        if (down && !this._jumpDown) this.jumpFired = true;
        this._jumpDown = down;
        this.jumpHeld = down;
      } else if (k === 'ShiftLeft' || k === 'ShiftRight' || k === 'KeyJ') this.run = down;
      else if (k === 'KeyR' && down) { Game.restart(); handled = true; }
      else handled = false;

      if (handled && (k === 'Space' || k.indexOf('Arrow') === 0)) e.preventDefault();
    },

    /** 每个物理步结束后清掉一次性的按键边沿 */
    endFrame: function () { this.jumpFired = false; },
  };

  /* ==========================================================
   * 6. 游戏世界
   * ========================================================== */
  const Game = {
    state: 'play',        // play | dying | gameover | clear
    lives: CONFIG.startLives,
    coins: 0,
    score: 0,
    time: 400,            // PRD 9：本期固定 400，不递减
    deathTimer: 0,
    clearTimer: 0,
    frame: 0,
    tiles: [],            // { char, solid, bumpT }
    tileW: 0,
    tileH: 0,
    worldW: 0,
    enemies: [],
    coinPops: [],
    floatTexts: [],
    spawn: { x: 0, y: 0 },
    flagX: 0,
    player: null,
    camera: 0,
    constructor_: function () {},
  };

  const Player = {
    x: 0, y: 0,
    w: 12, h: 16,
    vx: 0, vy: 0,
    onGround: false,
    facing: 1,
    coyote: 0,
    jumpBuffer: 0,
    jumpCutLock: 0,     // 起跳后的截断保护计时
    jumping: false,     // 本次跳跃是否仍在"上升可控"阶段
    walkTimer: 0,
    animFrame: 0,
  };

  /* ---------------- 关卡解析 ---------------- */
  function loadLevel() {
    Game.tileH = LEVEL.length;
    Game.tileW = 0;
    for (let r = 0; r < LEVEL.length; r++) {
      if (LEVEL[r].length > Game.tileW) Game.tileW = LEVEL[r].length;
    }
    // PRD 5 节：关卡长度约 200 格 → 不足的部分补空
    if (Game.tileW < 200) Game.tileW = 200;
    Game.worldW = Game.tileW * TILE;

    Game.tiles = [];
    Game.enemies = [];
    Game.coinPops = [];
    Game.floatTexts = [];

    for (let r = 0; r < Game.tileH; r++) {
      const row = LEVEL[r] || '';
      const arr = [];
      for (let c = 0; c < Game.tileW; c++) {
        const ch = c < row.length ? row[c] : '.';
        arr.push(makeTile(ch, r));

        if (ch === 'M') {
          Game.spawn.x = c * TILE + 2;
          Game.spawn.y = r * TILE + (TILE - Player.h);
        } else if (ch === 'G') {
          Game.enemies.push({
            x: c * TILE, y: r * TILE + (TILE - 14),
            w: 14, h: 14, vx: -CONFIG.enemySpeed, vy: 0,
            alive: true, squash: 0, animTimer: 0, animFrame: 0,
          });
        } else if (ch === 'C') {
          arr[c].coin = true;
        } else if (ch === 'F') {
          Game.flagX = c * TILE;
        }
      }
      Game.tiles.push(arr);
    }
  }

  /** 生成单块图块；装饰类字符（G/M/C/F）本身不占图块，等同空 */
  function makeTile(ch, r) {
    const DECOR = 'GMCF';
    const isDecor = DECOR.indexOf(ch) !== -1;
    const real = isDecor ? '.' : ch;
    return {
      char: real,
      solid: SOLID_CHARS.indexOf(real) !== -1,
      coin: ch === 'C',
      bumpT: 0,
      groundRow: real === 'X' ? (r - (ROWS - GROUND_ROWS)) : -1, // 0 = 草绿叶, 1 = 砖土
    };
  }

  function tileAt(c, r) {
    if (r < 0 || r >= Game.tileH) return null;
    if (c < 0 || c >= Game.tileW) return null;
    return Game.tiles[r][c];
  }
  function isSolid(c, r) {
    const t = tileAt(c, r);
    return !!t && t.solid;
  }

  /* ==========================================================
   * 7. 物理：移动 / 重力 / 跳跃（PRD 3.1 3.2 3.3）
   * ========================================================== */
  function updatePlayer(dt) {
    const p = Player;

    // ---------- 3.1 横向：加速度曲线 ----------
    const wantLeft = Input.left;
    const wantRight = Input.right;
    const dir = (wantRight ? 1 : 0) - (wantLeft ? 1 : 0);
    const targetSpeed = Input.run ? CONFIG.runSpeed : CONFIG.walkSpeed;
    const maxSpeed = targetSpeed;
    let accel;
    if (p.onGround) {
      accel = Input.run ? CONFIG.runAccel : CONFIG.groundAccel;
    } else {
      accel = CONFIG.airAccel;
    }

    if (dir !== 0) {
      p.facing = dir;
      const reversing = (p.vx !== 0) && (Math.sign(p.vx) !== dir);
      // 快速转身：反向按键时加速度更高，且不设速度墙（才有"打滑"再拉回的脚感）
      const a = reversing ? CONFIG.turnAccel : accel;
      p.vx += dir * a * dt;
      // 正向加速才受速度上限约束；转身时等速度过零后再收进上限
      if (!reversing || Math.sign(p.vx) === dir) {
        if (Math.abs(p.vx) > maxSpeed) p.vx = dir * maxSpeed;
      }
    } else if (p.onGround) {
      // 惯性滑行：松手靠摩擦减速
      const f = CONFIG.groundFriction * dt;
      if (Math.abs(p.vx) <= f) p.vx = 0;
      else p.vx -= Math.sign(p.vx) * f;
    } else {
      // 空中松手：无摩擦，保留惯性（只为可救回保留少量空气阻力）
      p.vx -= Math.sign(p.vx) * Math.min(Math.abs(p.vx), 40 * dt);
    }

    // ---------- 3.3 跳跃：buffer + coyote ----------
    if (Input.jumpFired) p.jumpBuffer = CONFIG.jumpBufferTime;
    if (p.jumpBuffer > 0) p.jumpBuffer -= dt;
    if (p.onGround) p.coyote = CONFIG.coyoteTime;
    else if (p.coyote > 0) p.coyote -= dt;

    if (p.jumpBuffer > 0 && p.coyote > 0) {
      // 跑跳加成：起跳时水平速度越快跳得越高
      const speedRatio = Math.min(1, Math.abs(p.vx) / CONFIG.runSpeed);
      const bonus = CONFIG.runJumpBonus * speedRatio;
      p.vy = -(CONFIG.jumpSpeed + bonus);
      p.onGround = false;
      p.coyote = 0;
      p.jumpBuffer = 0;
      p.jumping = true;
      p.jumpCutLock = CONFIG.jumpCutDelay;   // 起跳后这段时间内不响应松键截断
    }

    // 松键截断：上升途中松键 → 竖直速度按倍率截断
    // 注意：起跳当帧不截断（p.jumpCutLock 计数），否则轻点会退化成"半力起跳"
    if (p.jumpCutLock > 0) {
      p.jumpCutLock -= dt;
    } else if (p.jumping && p.vy < 0 && !Input.jumpHeld) {
      p.vy = Math.max(p.vy * CONFIG.jumpCutMultiplier, -CONFIG.jumpCutMinSpeed);
      p.jumping = false;
    }
    if (p.vy >= 0) p.jumping = false;

    // ---------- 3.2 重力：上升/下落分离 ----------
    const g = p.vy < 0 ? CONFIG.gravityUp : CONFIG.gravityDown;
    p.vy += g * dt;
    if (p.vy > CONFIG.maxFallSpeed) p.vy = CONFIG.maxFallSpeed;

    // ---------- 横向移动 + 碰撞（分轴处理，防止穿模）----------
    p.x += p.vx * dt;
    if (p.x < 0) { p.x = 0; p.vx = 0; }
    if (p.x + p.w > Game.worldW) { p.x = Game.worldW - p.w; p.vx = 0; }
    resolveHorizontal(p);

    // ---------- 纵向移动 + 碰撞 ----------
    const wasOnGround = p.onGround;
    p.y += p.vy * dt;
    p.onGround = false;
    resolveVertical(p, wasOnGround);

    // ---------- 走路动画计时 ----------
    if (p.onGround && Math.abs(p.vx) > 5) {
      const speedRatio = Math.abs(p.vx) / CONFIG.runSpeed;
      p.walkTimer += dt * (6 + speedRatio * 14);
      if (p.walkTimer >= 1) { p.walkTimer = 0; p.animFrame = (p.animFrame + 1) % 3; }
    } else if (p.onGround) {
      p.walkTimer = 0;
      p.animFrame = 0;
    }
  }

  function resolveHorizontal(box) {
    const left = Math.floor(box.x / TILE);
    const right = Math.floor((box.x + box.w - 1) / TILE);
    const top = Math.floor(box.y / TILE);
    const bottom = Math.floor((box.y + box.h - 1) / TILE);
    for (let r = top; r <= bottom; r++) {
      for (let c = left; c <= right; c++) {
        if (!isSolid(c, r)) continue;
        if (box.vx > 0) box.x = c * TILE - box.w;
        else if (box.vx < 0) box.x = (c + 1) * TILE;
        box.vx = 0;
      }
    }
  }

  function resolveVertical(box, wasOnGround) {
    const left = Math.floor(box.x / TILE);
    const right = Math.floor((box.x + box.w - 1) / TILE);
    const top = Math.floor(box.y / TILE);
    const bottom = Math.floor((box.y + box.h - 1) / TILE);
    for (let r = top; r <= bottom; r++) {
      for (let c = left; c <= right; c++) {
        if (!isSolid(c, r)) continue;
        if (box.vy > 0) {
          box.y = r * TILE - box.h;
          box.vy = 0;
          box.onGround = true;
        } else if (box.vy < 0) {
          box.y = (r + 1) * TILE;
          box.vy = 0;
          onHeadHit(c, r);
        }
      }
    }
    // 兜底：永远不会掉出世界底部（掉坑判定在 updateGame 里做）
    if (box.y > VIEW_H + 32) box.onGround = false;
    else if (wasOnGround && box.onGround === false) { /* 走下平台边缘，交给 coyote */ }
  }

  /** 顶头：顶问号砖 / 顶碎砖（PRD 3.4） */
  function onHeadHit(c, r) {
    const t = tileAt(c, r);
    if (!t) return;
    t.bumpT = 0.18;
    if (t.char === '?') {
      t.char = 'u';          // 空块
      t.solid = true;
      spawnCoinPop(c, r);
      addCoin();
    } else if (t.char === 'B') {
      // 砖块可顶碎：变成空（保留视觉上的碎块飘散由 bump 动画承担）
      t.char = '.';
      t.solid = false;
      Sfx.play('brick');
    } else {
      Sfx.play('bump');
    }
  }

  /* ==========================================================
   * 8. 金币 / 分数 / 飘字（PRD 3.4）
   * ========================================================== */
  function spawnCoinPop(c, r) {
    Game.coinPops.push({ x: c * TILE + 4, y: r * TILE - 8, t: 0 });
    Sfx.play('coin');
  }

  function addCoin() {
    Game.coins += 1;
    Game.score += CONFIG.coinScore;
    addScoreText(Player.x + Player.w / 2, Player.y, '+' + CONFIG.coinScore);
    if (Game.coins >= CONFIG.coinsPerLife) {
      Game.coins -= CONFIG.coinsPerLife;
      Game.lives += 1;
      Sfx.play('life');
    }
  }

  function addScoreText(cx, y, text) {
    Game.floatTexts.push({ x: cx, y: y, t: 0, text: text });
  }

  /* ==========================================================
   * 9. 敌人（PRD 3.5）
   * ========================================================== */
  function updateEnemies(dt) {
    for (let i = 0; i < Game.enemies.length; i++) {
      const e = Game.enemies[i];
      if (!e.alive) {
        e.squash -= dt;
        continue;
      }
      e.animTimer += dt;
      if (e.animTimer > 0.16) { e.animTimer = 0; e.animFrame = 1 - e.animFrame; }

      // 水平移动
      e.x += e.vx * dt;
      const l = Math.floor(e.x / TILE);
      const r = Math.floor((e.x + e.w - 1) / TILE);
      const t = Math.floor(e.y / TILE);
      const b = Math.floor((e.y + e.h - 1) / TILE);
      for (let rr = t; rr <= b; rr++) {
        for (let cc = l; cc <= r; cc++) {
          if (!isSolid(cc, rr)) continue;
          if (e.vx > 0) e.x = cc * TILE - e.w;
          else e.x = (cc + 1) * TILE;
          e.vx = -e.vx;   // 撞墙掉头
        }
      }

      // 重力 + 落地
      e.vy += CONFIG.gravityDown * dt;
      if (e.vy > CONFIG.maxFallSpeed) e.vy = CONFIG.maxFallSpeed;
      e.y += e.vy * dt;
      e.onGround = false;
      const l2 = Math.floor(e.x / TILE);
      const r2 = Math.floor((e.x + e.w - 1) / TILE);
      const t2 = Math.floor(e.y / TILE);
      const b2 = Math.floor((e.y + e.h - 1) / TILE);
      for (let rr = t2; rr <= b2; rr++) {
        for (let cc = l2; cc <= r2; cc++) {
          if (!isSolid(cc, rr)) continue;
          if (e.vy > 0) { e.y = rr * TILE - e.h; e.vy = 0; e.onGround = true; }
          else if (e.vy < 0) { e.y = (rr + 1) * TILE; e.vy = 0; }
        }
      }

      // 走出平台边缘 → 自然掉落（不悬浮）
      if (e.y > VIEW_H + 48) e.alive = false; // 掉出画面就回收
    }

    // 清理已消失的敌人
    Game.enemies = Game.enemies.filter(function (e) {
      return e.alive || e.squash > 0;
    });
  }

  function checkEnemyCollision() {
    const p = Player;
    if (Game.state !== 'play') return;
    for (let i = 0; i < Game.enemies.length; i++) {
      const e = Game.enemies[i];
      if (!e.alive) continue;
      if (!overlap(p, e)) continue;

      const stomping = p.vy > 0 && (p.y + p.h - p.vy * FIXED_DT) <= e.y + 6;
      if (stomping) {
        // 踩死：变扁平，0.3 秒后消失，玩家弹起
        e.alive = false;
        e.squash = CONFIG.enemySquashTime;
        p.vy = -CONFIG.stompBounce;
        p.y = e.y - p.h;
        Game.score += CONFIG.stompScore;
        addScoreText(e.x + e.w / 2, e.y, '+' + CONFIG.stompScore);
        Sfx.play('stomp');
      } else {
        // 侧面 / 下方碰撞 → 死亡
        killPlayer();
        return;
      }
    }
  }

  function overlap(a, b) {
    return Math.abs(a.x + a.w / 2 - (b.x + b.w / 2)) < (a.w + b.w) / 2 &&
           Math.abs(a.y + a.h / 2 - (b.y + b.h / 2)) < (a.h + b.h) / 2;
  }

  /* ==========================================================
   * 10. 游戏流程（PRD 3.6）
   * ========================================================== */
  function killPlayer() {
    if (Game.state !== 'play') return;
    Game.state = 'dying';
    Game.deathTimer = 0;
    Player.vy = -CONFIG.deathHopSpeed; // 先向上一跳
    Player.vx = 0;
    Sfx.play('die');
  }

  function respawn() {
    Game.lives -= 1;
    if (Game.lives <= 0) {
      Game.lives = 0;
      Game.state = 'gameover';
      return;
    }
    // 位置回起点，敌人重置，金币/分数保留
    Game.state = 'play';
    loadLevelRuntimeReset();
    resetPlayerPosition();
  }

  /** 只重置敌人与砖块状态，不重建整关（保留金币/分数） */
  function loadLevelRuntimeReset() {
    // 重新解析一遍关卡（问号砖恢复成未顶状态）
    const savedScore = Game.score;
    const savedCoins = Game.coins;
    const savedLives = Game.lives;
    loadLevel();
    Game.score = savedScore;
    Game.coins = savedCoins;
    Game.lives = savedLives;
    Game.coinPops = [];
    Game.floatTexts = [];
  }

  function resetPlayerPosition() {
    Player.x = Game.spawn.x;
    Player.y = Game.spawn.y;
    Player.vx = 0;
    Player.vy = 0;
    Player.onGround = true;
    Player.facing = 1;
    Player.coyote = 0;
    Player.jumpBuffer = 0;
    Player.jumpCutLock = 0;
    Player.jumping = false;
    Player.animFrame = 0;
    Game.camera = 0;
  }

  Game.restart = function () {
    Game.lives = CONFIG.startLives;
    Game.coins = 0;
    Game.score = 0;
    Game.time = 400;
    Game.state = 'play';
    Game.deathTimer = 0;
    Game.clearTimer = 0;
    loadLevel();
    resetPlayerPosition();
  };

  function updateGame(dt) {
    Game.frame++;

    if (Game.state === 'play') {
      updatePlayer(dt);
      updateEnemies(dt);
      checkEnemyCollision();

      // 直接拾取金币（C）
      collectTileCoins();

      // 掉坑死亡
      if (Player.y > VIEW_H + 8) {
        Game.state = 'dying';
        Game.deathTimer = 0;
        Player.vy = -CONFIG.deathHopSpeed * 0.6;
        Sfx.play('die');
      }

      // 终点：走到旗杆即通关
      if (Player.x + Player.w >= Game.flagX) {
        Game.state = 'clear';
        Game.clearTimer = 0;
      }

      updateCamera();
    } else if (Game.state === 'dying') {
      Game.deathTimer += dt;
      // 死亡动画：只受重力，掉出画面
      Player.vy += CONFIG.deathHopGravity * dt;
      Player.y += Player.vy * dt;
      if (Game.deathTimer >= CONFIG.deathPause) respawn();
    } else if (Game.state === 'clear') {
      Game.clearTimer += dt;
      // 通关后小走两步
      Player.vx = CONFIG.walkSpeed;
      updatePlayer(dt);
    }

    // 动画计时
    for (let r = 0; r < Game.tileH; r++) {
      for (let c = 0; c < Game.tileW; c++) {
        const t = Game.tiles[r][c];
        if (t.bumpT > 0) t.bumpT = Math.max(0, t.bumpT - dt);
      }
    }
    for (let i = Game.coinPops.length - 1; i >= 0; i--) {
      Game.coinPops[i].t += dt;
      if (Game.coinPops[i].t >= CONFIG.coinPopDuration) Game.coinPops.splice(i, 1);
    }
    for (let i = Game.floatTexts.length - 1; i >= 0; i--) {
      Game.floatTexts[i].t += dt;
      if (Game.floatTexts[i].t >= CONFIG.floatTextDuration) Game.floatTexts.splice(i, 1);
    }
  }

  function collectTileCoins() {
    const p = Player;
    const c0 = Math.floor(p.x / TILE);
    const c1 = Math.floor((p.x + p.w - 1) / TILE);
    const r0 = Math.floor(p.y / TILE);
    const r1 = Math.floor((p.y + p.h - 1) / TILE);
    for (let r = r0; r <= r1; r++) {
      for (let c = c0; c <= c1; c++) {
        const t = tileAt(c, r);
        if (t && t.coin) {
          t.coin = false;
          t.char = '.';
          Game.coins += 1;
          Game.score += CONFIG.coinScore;
          addScoreText(c * TILE + 8, r * TILE, '+' + CONFIG.coinScore);
          Sfx.play('coin');
          if (Game.coins >= CONFIG.coinsPerLife) {
            Game.coins -= CONFIG.coinsPerLife;
            Game.lives += 1;
            Sfx.play('life');
          }
        }
      }
    }
  }

  function updateCamera() {
    if (Game.camLocked) return;
    const target = Player.x + Player.w / 2 - VIEW_W * 0.42;
    // 相机只向前推进，不回退（经典横版手感）
    if (target > Game.camera) {
      Game.camera += (target - Game.camera) * 0.18;
    }
    const maxCam = Game.worldW - VIEW_W;
    if (Game.camera > maxCam) Game.camera = maxCam;
    if (Game.camera < 0) Game.camera = 0;
  }

  /* ==========================================================
   * 11. 渲染（PRD 4 节）
   * ========================================================== */
  function render() {
    const camX = Math.round(Game.camera);

    drawSky();
    drawHills(camX);
    drawClouds(camX);
    drawTiles(camX);
    drawBushes(camX);
    drawFlag(camX);
    drawEnemies(camX);
    drawCoinPops(camX);
    drawPlayer(camX);
    drawFloatTexts(camX);
    drawHUD();
    drawOverlays();
  }

  /* ---------- 4.1 蓝天（纯色，无渐变） ---------- */
  function drawSky() {
    ctx.fillStyle = PALETTE.sky;
    ctx.fillRect(0, 0, VIEW_W, VIEW_H);
  }

  /* ---------- 远景山丘（视差 0.5） ---------- */
  function drawHills(camX) {
    const off = Math.round(camX * 0.5);
    for (let i = 0; i < HILLS.length; i++) {
      const h = HILLS[i];
      const baseX = h.x - off;
      if (baseX + h.w * TILE < -40 || baseX > VIEW_W + 40) continue;
      drawHill(baseX, GROUND_TOP, h.w, h.h);
    }
  }

  function drawHill(x, baseY, wTiles, hTiles) {
    const w = wTiles * TILE;
    const h = hTiles * TILE;
    ctx.fillStyle = PALETTE.hill;
    // 台阶式山丘（像素风）
    for (let i = 0; i < h; i++) {
      const inset = Math.floor(Math.pow(1 - (i + 1) / h, 1.6) * (w / 2));
      ctx.fillRect(Math.round(x + inset), baseY - h + i, Math.max(2, w - inset * 2), 1);
    }
    // 暗部（同色降明度）
    ctx.fillStyle = DARK.hill;
    for (let i = 0; i < Math.floor(h / 2); i++) {
      const inset = Math.floor(Math.pow(1 - (i + 1) / h, 1.6) * (w / 2));
      ctx.fillRect(Math.round(x + w - inset - Math.floor(w / 2)), baseY - h + i, Math.floor(w / 2) - 4, 1);
    }
  }

  /* ---------- 云（视差 0.3，块状） ---------- */
  function drawClouds(camX) {
    const off = Math.round(camX * 0.3);
    for (let i = 0; i < CLOUDS.length; i++) {
      const c = CLOUDS[i];
      const x = c.x - off;
      if (x < -60 || x > VIEW_W + 20) continue;
      drawCloud(x, c.y);
    }
  }

  function drawCloud(x, y) {
    ctx.fillStyle = PALETTE.cloud;
    // 块状云：复刻 NES 的"丘陵状"
    ctx.fillRect(x + 4, y + 4, 8, 4);
    ctx.fillRect(x, y + 8, 20, 4);
    ctx.fillRect(x + 8, y, 8, 4);
    ctx.fillRect(x + 16, y + 4, 8, 4);
    ctx.fillRect(x + 4, y + 12, 20, 4);
  }

  /* ---------- 灌木（与云同形的复用，贴地） ---------- */
  function drawBushes(camX) {
    const off = Math.round(camX);
    for (let i = 0; i < BUSHES.length; i++) {
      const b = BUSHES[i];
      const x = b.x - off;
      if (x < -60 || x > VIEW_W + 20) continue;
      const y = GROUND_TOP - 20;
      ctx.fillStyle = DARK.hill;
      ctx.fillRect(x + 4, y + 8, 8, 4);
      ctx.fillRect(x, y + 12, 20, 8);
      ctx.fillRect(x + 8, y + 4, 8, 4);
      ctx.fillRect(x + 16, y + 8, 8, 4);
    }
  }

  /* ---------- 4.2 图块 ---------- */
  function drawTiles(camX) {
    const startCol = Math.floor(camX / TILE) - 1;
    const endCol = startCol + COLS + 2;
    for (let r = 0; r < Game.tileH; r++) {
      for (let c = Math.max(0, startCol); c <= Math.min(Game.tileW - 1, endCol); c++) {
        const t = Game.tiles[r][c];
        if (!t) continue;
        let dy = 0;
        if (t.bumpT > 0) {
          dy = -Math.sin((t.bumpT / 0.18) * Math.PI) * 5;
        }
        const px = Math.round(c * TILE - camX);
        const py = Math.round(r * TILE + dy);
        switch (t.char) {
          case 'X': drawGroundTile(px, py, r); break;
          case 'B': drawBrickTile(px, py); break;
          case '?': drawQuestionTile(px, py); break;
          case 'u': drawUsedTile(px, py); break;
          case 'P': drawPipeTile(px, py, c, r); break;
          default:
            if (t.coin) drawCoin(px + 4, py + 4);
        }
      }
    }
  }

  /* 地面 16×16 程序绘制，无缝拼接 */
  function drawGroundTile(x, y, row) {
    const t = tileAtGroundRow(y);
    if (t === 0) {
      // 第一行：草绿顶 + 砖土
      ctx.fillStyle = PALETTE.groundGrass;
      ctx.fillRect(x, y, TILE, 4);
      ctx.fillStyle = PALETTE.groundBrick;
      ctx.fillRect(x, y + 4, TILE, TILE - 4);
    } else {
      ctx.fillStyle = PALETTE.groundBrick;
      ctx.fillRect(x, y, TILE, TILE);
    }
    // 砖缝（深色勾缝）
    ctx.fillStyle = DARK.groundBrick;
    ctx.fillRect(x, y + TILE - 1, TILE, 1);
    ctx.fillRect(x + TILE - 1, y, 1, TILE);
    // 内部错缝
    ctx.fillStyle = DARK.groundBrick;
    ctx.fillRect(x + 7, y + 6, 1, 9);
    ctx.fillRect(x, y + 11, 7, 1);
    ctx.fillRect(x + 8, y + 11, 7, 1);
  }

  function tileAtGroundRow(y) {
    const row = Math.floor(y / TILE);
    return row - (ROWS - GROUND_ROWS);
  }

  function drawBrickTile(x, y) {
    ctx.fillStyle = PALETTE.brick;
    ctx.fillRect(x, y, TILE, TILE);
    ctx.fillStyle = PALETTE.brickLine;
    // 外框
    ctx.fillRect(x, y, TILE, 1);
    ctx.fillRect(x, y + TILE - 1, TILE, 1);
    ctx.fillRect(x, y, 1, TILE);
    ctx.fillRect(x + TILE - 1, y, 1, TILE);
    // 砖缝：上下两排错缝
    ctx.fillRect(x, y + 7, TILE, 1);
    ctx.fillRect(x, y + 15, TILE, 1);
    ctx.fillRect(x + 7, y, 1, 7);
    ctx.fillRect(x + 3, y + 8, 1, 7);
    ctx.fillRect(x + 11, y + 8, 1, 7);
  }

  function drawQuestionTile(x, y) {
    ctx.fillStyle = PALETTE.question;
    ctx.fillRect(x, y, TILE, TILE);
    // 明暗边（同色）
    ctx.fillStyle = DARK.question;
    ctx.fillRect(x, y + TILE - 2, TILE, 2);
    ctx.fillRect(x + TILE - 2, y, 2, TILE);
    ctx.fillStyle = PALETTE.brickLine;
    // 外框
    ctx.fillRect(x, y, TILE, 1);
    ctx.fillRect(x, y, 1, TILE);
    // 四角铆钉
    ctx.fillRect(x + 2, y + 2, 2, 2);
    ctx.fillRect(x + TILE - 4, y + 2, 2, 2);
    ctx.fillRect(x + 2, y + TILE - 4, 2, 2);
    ctx.fillRect(x + TILE - 4, y + TILE - 4, 2, 2);
    // 问号
    ctx.fillRect(x + 6, y + 4, 4, 2);
    ctx.fillRect(x + 9, y + 5, 2, 2);
    ctx.fillRect(x + 7, y + 7, 2, 2);
    ctx.fillRect(x + 7, y + 9, 2, 2);
    ctx.fillRect(x + 7, y + 12, 2, 2);
  }

  function drawUsedTile(x, y) {
    ctx.fillStyle = PALETTE.usedBlock;
    ctx.fillRect(x, y, TILE, TILE);
    ctx.fillStyle = DARK.usedBlock;
    ctx.fillRect(x, y + TILE - 2, TILE, 2);
    ctx.fillRect(x + TILE - 2, y, 2, TILE);
    ctx.fillStyle = PALETTE.brickLine;
    ctx.fillRect(x, y, TILE, 1);
    ctx.fillRect(x, y, 1, TILE);
  }

  /* 管道：2 格宽，顶部加帽 */
  function drawPipeTile(x, y, c, r) {
    // 判断是否该管道的顶行（上方形不是管道）
    const above = tileAt(c, r - 1);
    const isTop = !above || above.char !== 'P';
    ctx.fillStyle = PALETTE.groundGrass;
    if (isTop) {
      // 管口（左右各外扩 2px 由相邻格共同呈现）
      ctx.fillRect(x, y, TILE, 5);
      ctx.fillStyle = shade(PALETTE.groundGrass, 0.78);
      ctx.fillRect(x, y + 5, TILE, 1);
      ctx.fillStyle = PALETTE.groundGrass;
      ctx.fillRect(x, y + 6, TILE, TILE - 6);
      // 高光
      ctx.fillStyle = LIGHT.groundGrass;
      ctx.fillRect(x + 2, y + 6, 2, TILE - 6);
      ctx.fillStyle = shade(PALETTE.groundGrass, 0.7);
      ctx.fillRect(x + TILE - 3, y + 6, 3, TILE - 6);
    } else {
      ctx.fillRect(x, y, TILE, TILE);
      ctx.fillStyle = LIGHT.groundGrass;
      ctx.fillRect(x + 3, y, 2, TILE);
      ctx.fillStyle = shade(PALETTE.groundGrass, 0.7);
      ctx.fillRect(x + TILE - 3, y, 3, TILE);
    }
  }

  /* ---------- 旗杆 ---------- */
  function drawFlag(camX) {
    const x = Game.flagX - camX;
    if (x < -20 || x > VIEW_W + 20) return;
    // 杆
    ctx.fillStyle = PALETTE.text;
    ctx.fillRect(x + 7, GROUND_TOP - 96, 2, 96);
    // 球顶
    ctx.fillStyle = PALETTE.coin;
    ctx.fillRect(x + 5, GROUND_TOP - 100, 6, 5);
    // 旗
    ctx.fillStyle = PALETTE.marioRed;
    ctx.fillRect(x, GROUND_TOP - 92, 7, 10);
    ctx.fillStyle = DARK.red;
    ctx.fillRect(x, GROUND_TOP - 84, 7, 2);
  }

  /* ---------- 4.3 金币 ---------- */
  function drawCoin(x, y) {
    ctx.fillStyle = PALETTE.coin;
    ctx.fillRect(x, y, 8, 12);
    ctx.fillStyle = PALETTE.brickLine;
    ctx.fillRect(x + 3, y + 2, 2, 8);
    ctx.fillStyle = DARK.coin;
    ctx.fillRect(x, y + 10, 8, 2);
  }

  function drawCoinPops(camX) {
    for (let i = 0; i < Game.coinPops.length; i++) {
      const p = Game.coinPops[i];
      const t = p.t / CONFIG.coinPopDuration;
      // 上升 24px 后消失
      const y = p.y - CONFIG.coinPopHeight * Math.sin(t * Math.PI * 0.85);
      const x = p.x - camX;
      ctx.globalAlpha = 1 - Math.max(0, (t - 0.75) / 0.25);
      drawCoin(Math.round(x), Math.round(y));
      ctx.globalAlpha = 1;
    }
  }

  /* ---------- 4.3 马里奥（16×16 程序绘制） ---------- */
  function drawPlayer(camX) {
    const p = Player;
    const x = Math.round(p.x - camX) - 2;   // 碰撞盒 12 宽，绘制 16 宽，居中
    const y = Math.round(p.y) - 0;           // 碰撞盒 16 高 = 绘制高
    if (!p.onGround && Game.state === 'play') {
      drawMarioJump(x, y, p.facing);
    } else if (Math.abs(p.vx) > 5) {
      drawMarioWalk(x, y, p.facing, p.animFrame);
    } else {
      drawMarioStand(x, y, p.facing);
    }
  }

  function marioColors() {
    return { r: PALETTE.marioRed, s: PALETTE.marioSkin, o: PALETTE.overalls, k: PALETTE.brickLine, w: PALETTE.text };
  }

  /* 站立帧：红帽、肤色脸、蓝背带裤 */
  function drawMarioStand(x, y, facing) {
    const C = marioColors();
    const f = facing >= 0;
    // 帽子
    ctx.fillStyle = C.r;
    ctx.fillRect(x + 3, y, 10, 3);
    ctx.fillRect(x + (f ? 9 : 2), y + 3, 5, 2);
    // 脸
    ctx.fillStyle = C.s;
    ctx.fillRect(x + 3, y + 3, 8, 5);
    ctx.fillStyle = C.k;
    ctx.fillRect(x + (f ? 7 : 5), y + 4, 1, 2);   // 眼睛
    ctx.fillStyle = C.r;
    ctx.fillRect(x + (f ? 10 : 2), y + 6, 4, 1);  // 胡子
    // 身体：红衣 + 蓝背带裤
    ctx.fillStyle = C.r;
    ctx.fillRect(x + 2, y + 8, 12, 3);
    ctx.fillStyle = C.o;
    ctx.fillRect(x + 3, y + 9, 10, 5);
    ctx.fillStyle = C.r;
    ctx.fillRect(x + 1, y + 8, 2, 4);
    ctx.fillRect(x + 13, y + 8, 2, 4);
    // 手
    ctx.fillStyle = C.s;
    ctx.fillRect(x + 0, y + 10, 2, 3);
    ctx.fillRect(x + 14, y + 10, 2, 3);
    // 鞋
    ctx.fillStyle = C.k;
    ctx.fillRect(x + 2, y + 14, 5, 2);
    ctx.fillRect(x + 9, y + 14, 5, 2);
  }

  /* 行走帧：3 帧腿部交替 */
  function drawMarioWalk(x, y, facing, frame) {
    const C = marioColors();
    const f = facing >= 0;
    // 帽子 + 脸（与站立一致）
    ctx.fillStyle = C.r;
    ctx.fillRect(x + 3, y, 10, 3);
    ctx.fillRect(x + (f ? 9 : 2), y + 3, 5, 2);
    ctx.fillStyle = C.s;
    ctx.fillRect(x + 3, y + 3, 8, 5);
    ctx.fillStyle = C.k;
    ctx.fillRect(x + (f ? 7 : 5), y + 4, 1, 2);
    ctx.fillStyle = C.r;
    ctx.fillRect(x + (f ? 10 : 2), y + 6, 4, 1);
    // 身体
    ctx.fillStyle = C.r;
    ctx.fillRect(x + 2, y + 8, 12, 3);
    ctx.fillStyle = C.o;
    ctx.fillRect(x + 3, y + 9, 10, 4);
    ctx.fillStyle = C.r;
    ctx.fillRect(x + 1, y + 8, 2, 4);
    ctx.fillRect(x + 13, y + 8, 2, 4);
    ctx.fillStyle = C.s;
    ctx.fillRect(x + 0, y + 10, 2, 3);
    ctx.fillRect(x + 14, y + 10, 2, 3);
    // 腿：3 帧
    ctx.fillStyle = C.k;
    if (frame === 0) {
      ctx.fillRect(x + 2, y + 13, 5, 3);
      ctx.fillRect(x + 9, y + 14, 5, 2);
    } else if (frame === 1) {
      ctx.fillRect(x + 3, y + 14, 10, 2);
    } else {
      ctx.fillRect(x + 1, y + 14, 5, 2);
      ctx.fillRect(x + 8, y + 13, 6, 3);
    }
  }

  /* 跳跃帧：举手 / 伸腿单帧 */
  function drawMarioJump(x, y, facing) {
    const C = marioColors();
    const f = facing >= 0;
    ctx.fillStyle = C.r;
    ctx.fillRect(x + 3, y, 10, 3);
    ctx.fillRect(x + (f ? 9 : 2), y + 3, 5, 2);
    ctx.fillStyle = C.s;
    ctx.fillRect(x + 3, y + 3, 8, 5);
    ctx.fillStyle = C.k;
    ctx.fillRect(x + (f ? 7 : 5), y + 4, 1, 2);
    // 身体
    ctx.fillStyle = C.r;
    ctx.fillRect(x + 2, y + 8, 12, 3);
    ctx.fillStyle = C.o;
    ctx.fillRect(x + 3, y + 9, 10, 3);
    // 举手
    ctx.fillStyle = C.s;
    if (f) {
      ctx.fillRect(x + 13, y + 5, 3, 4);
      ctx.fillRect(x + 0, y + 11, 2, 3);
    } else {
      ctx.fillRect(x + 0, y + 5, 3, 4);
      ctx.fillRect(x + 14, y + 11, 2, 3);
    }
    // 伸腿
    ctx.fillStyle = C.k;
    if (f) {
      ctx.fillRect(x + 9, y + 12, 6, 2);
      ctx.fillRect(x + 1, y + 14, 5, 2);
    } else {
      ctx.fillRect(x + 1, y + 12, 6, 2);
      ctx.fillRect(x + 10, y + 14, 5, 2);
    }
  }

  /* ---------- 4.3 板栗仔 ---------- */
  function drawEnemies(camX) {
    for (let i = 0; i < Game.enemies.length; i++) {
      const e = Game.enemies[i];
      const x = Math.round(e.x - camX) - 1;
      if (x < -20 || x > VIEW_W + 20) continue;
      if (!e.alive) {
        drawGoombaSquashed(x, Math.round(e.y));
      } else {
        const y = Math.round(e.y) - 2;
        if (e.animFrame === 0) drawGoomba0(x, y);
        else drawGoomba1(x, y);
      }
    }
  }

  function goombaHead(x, y) {
    ctx.fillStyle = PALETTE.goombaBrown;
    // 蘑菇状身体
    ctx.fillRect(x + 2, y + 0, 12, 3);
    ctx.fillRect(x + 0, y + 3, 16, 6);
    ctx.fillRect(x + 1, y + 9, 14, 3);
    // 暗部
    ctx.fillStyle = DARK.goomba;
    ctx.fillRect(x + 0, y + 7, 16, 2);
    // 眼睛（白眼 + 黑瞳）
    ctx.fillStyle = PALETTE.text;
    ctx.fillRect(x + 3, y + 3, 4, 5);
    ctx.fillRect(x + 9, y + 3, 4, 5);
    ctx.fillStyle = PALETTE.brickLine;
    ctx.fillRect(x + 5, y + 4, 2, 4);
    ctx.fillRect(x + 9, y + 4, 2, 4);
    // 眉毛
    ctx.fillStyle = PALETTE.brickLine;
    ctx.fillRect(x + 4, y + 2, 3, 1);
    ctx.fillRect(x + 9, y + 2, 3, 1);
  }

  function drawGoomba0(x, y) {
    goombaHead(x, y);
    ctx.fillStyle = PALETTE.brickLine;
    ctx.fillRect(x + 1, y + 12, 6, 2);
    ctx.fillRect(x + 9, y + 12, 6, 2);
  }

  function drawGoomba1(x, y) {
    goombaHead(x, y);
    ctx.fillStyle = PALETTE.brickLine;
    ctx.fillRect(x + 2, y + 12, 5, 2);
    ctx.fillRect(x + 9, y + 12, 5, 2);
    ctx.fillRect(x + 5, y + 11, 6, 1);
  }

  function drawGoombaSquashed(x, y) {
    ctx.fillStyle = PALETTE.goombaBrown;
    ctx.fillRect(x + 0, y + 6, 16, 8);
    ctx.fillStyle = DARK.goomba;
    ctx.fillRect(x + 0, y + 12, 16, 2);
    ctx.fillStyle = PALETTE.text;
    ctx.fillRect(x + 3, y + 8, 4, 3);
    ctx.fillRect(x + 9, y + 8, 4, 3);
    ctx.fillStyle = PALETTE.brickLine;
    ctx.fillRect(x + 5, y + 9, 2, 2);
    ctx.fillRect(x + 9, y + 9, 2, 2);
  }

  /* ---------- 得分飘字 ---------- */
  function drawFloatTexts(camX) {
    ctx.font = '8px monospace';
    ctx.textBaseline = 'top';
    for (let i = 0; i < Game.floatTexts.length; i++) {
      const f = Game.floatTexts[i];
      const t = f.t / CONFIG.floatTextDuration;
      const x = Math.round(f.x - camX);
      const y = Math.round(f.y - t * CONFIG.floatTextRise);
      ctx.globalAlpha = 1 - Math.max(0, (t - 0.6) / 0.4);
      ctx.fillStyle = PALETTE.brickLine;
      ctx.fillText(f.text, x - 7, y + 1);
      ctx.fillStyle = PALETTE.text;
      ctx.fillText(f.text, x - 8, y);
      ctx.globalAlpha = 1;
    }
  }

  /* ---------- 4.4 HUD ---------- */
  function pad(n, len) {
    let s = String(Math.max(0, Math.floor(n)));
    while (s.length < len) s = '0' + s;
    return s;
  }

  function drawHUD() {
    ctx.font = '8px monospace';
    ctx.textBaseline = 'top';
    ctx.fillStyle = PALETTE.text;

    // 第一行：MARIO / 金币×NN / WORLD / TIME
    // 第二行：6 位分数 / LIFE×N / 1-1 / 400
    drawShadowText('MARIO', 12, 8);
    drawShadowText(pad(Game.score, 6), 12, 18);

    drawHUDCoin(70, 8);
    drawShadowText('x' + pad(Game.coins, 2), 78, 8);
    drawShadowText('LIFE x' + Game.lives, 66, 18);

    drawShadowText('WORLD', 132, 8);
    drawShadowText('1-1', 140, 18);

    drawShadowText('TIME', 202, 8);
    drawShadowText(pad(Game.time, 3), 206, 18);
  }

  function drawShadowText(text, x, y) {
    ctx.fillStyle = PALETTE.brickLine;
    ctx.fillText(text, x + 1, y + 1);
    ctx.fillStyle = PALETTE.text;
    ctx.fillText(text, x, y);
  }

  function drawHUDCoin(x, y) {
    ctx.fillStyle = PALETTE.coin;
    ctx.fillRect(x, y, 5, 8);
    ctx.fillStyle = DARK.coin;
    ctx.fillRect(x, y + 6, 5, 2);
    ctx.fillStyle = PALETTE.brickLine;
    ctx.fillRect(x + 2, y + 2, 1, 4);
  }

  /* ---------- 覆盖层：GAME OVER / 通关 ---------- */
  function drawOverlays() {
    if (Game.state === 'gameover') {
      ctx.fillStyle = PALETTE.brickLine;
      ctx.globalAlpha = 0.72;
      ctx.fillRect(0, 0, VIEW_W, VIEW_H);
      ctx.globalAlpha = 1;
      ctx.font = '8px monospace';
      ctx.textBaseline = 'top';
      drawCenter('GAME OVER', 100);
      drawCenter('PRESS R TO RESTART', 118);
      drawCenter('SCORE ' + pad(Game.score, 6), 134);
    } else if (Game.state === 'clear') {
      ctx.fillStyle = PALETTE.brickLine;
      ctx.globalAlpha = 0.6;
      ctx.fillRect(0, 90, VIEW_W, 60);
      ctx.globalAlpha = 1;
      ctx.font = '8px monospace';
      ctx.textBaseline = 'top';
      drawCenter('COURSE CLEAR!', 100);
      drawCenter('SCORE ' + pad(Game.score, 6), 116);
      drawCenter('PRESS R TO PLAY AGAIN', 132);
    } else if (Game.state === 'dying') {
      ctx.font = '8px monospace';
      ctx.textBaseline = 'top';
      // 死亡瞬间不显示额外文字，保持原版简洁
    }
  }

  function drawCenter(text, y) {
    const w = ctx.measureText(text).width;
    drawShadowText(text, Math.round((VIEW_W - w) / 2), y);
  }

  /* ==========================================================
   * 12. 音效占位接口（PRD 1 节：留接口，不做音效）
   * ========================================================== */
  const Sfx = {
    enabled: false,
    play: function (name) {
      if (!this.enabled) return;
      // v0.2 接入：用 Web Audio 合成方波音效
      // console.log('[sfx]', name);
    },
  };

  /* ==========================================================
   * 13. 主循环：固定步长 1/60 累加器（PRD 2 节）
   * ========================================================== */
  let lastTime = 0;
  let accumulator = 0;

  function frame(now) {
    if (!lastTime) lastTime = now;
    let delta = (now - lastTime) / 1000;
    lastTime = now;
    // 切标签页回来时防止一次性追帧过多
    if (delta > 0.25) delta = 0.25;

    accumulator += delta;
    let steps = 0;
    while (accumulator >= FIXED_DT && steps < MAX_STEPS) {
      updateGame(FIXED_DT);
      Input.endFrame();
      accumulator -= FIXED_DT;
      steps++;
    }
    if (steps === MAX_STEPS) accumulator = 0;

    render();
    requestAnimationFrame(frame);
  }

  /* ==========================================================
   * 14. 启动
   * ========================================================== */
  function boot() {
    Input.bind();
    resizeCanvas();
    Game.restart();
    requestAnimationFrame(frame);
  }

  boot();
})();
