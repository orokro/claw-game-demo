<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import Matter from 'matter-js';
import decomp from 'poly-decomp';

window.decomp = decomp;

const {
  Engine,
  Render,
  Runner,
  Bodies,
  Composite,
  Constraint,
  Vertices,
  Body,
  Events,
  Common
} = Matter;

Common.setDecomp(decomp);

const canvas = ref(null);
const targetX = ref(50);
const isDropping = ref(false);
const showSettings = ref(false);
const prizeScale = ref(0.45);
const message = ref("Ready!");
const wonPrizesCount = ref(0);

const prizeFiles = ['bear.png', 'bunny.png', 'cat.png', 'doggo.png', 'pengy.png', 'sheep.png'];

let engine, render, runner, world;
let clawBase = null;
let leftPincer = null;
let rightPincer = null;
let leftSpring = null;
let rightSpring = null;
let cable = null;
let prizes = [];

const CLAW_CATEGORY = 0x0002;
const PRIZE_CATEGORY = 0x0004;
const WALL_CATEGORY = 0x0001;

const initPhysics = () => {
  engine = Engine.create();
  world = engine.world;
  world.gravity.y = 1.0;

  render = Render.create({
    canvas: canvas.value,
    engine: engine,
    options: {
      width: window.innerWidth,
      height: window.innerHeight,
      wireframes: false,
      background: 'transparent',
    },
  });

  Render.run(render);
  runner = Runner.create();
  Runner.run(runner, engine);

  const wallOpts = { isStatic: true, collisionFilter: { category: WALL_CATEGORY }, render: { visible: false } };
  const ground = Bodies.rectangle(window.innerWidth / 2, window.innerHeight + 25, window.innerWidth, 50, wallOpts);
  const leftWall = Bodies.rectangle(-25, window.innerHeight / 2, 50, window.innerHeight, wallOpts);
  const rightWall = Bodies.rectangle(window.innerWidth + 25, window.innerHeight / 2, 50, window.innerHeight, wallOpts);
  const shootWall = Bodies.rectangle(180, window.innerHeight - 150, 15, 400, { 
    isStatic: true, 
    collisionFilter: { category: WALL_CATEGORY },
    render: { fillStyle: '#2c3e50' } 
  });
  
  Composite.add(world, [ground, leftWall, rightWall, shootWall]);

  initClaw();
  spawnPrizes();
  
  Events.on(engine, 'afterUpdate', () => {
    checkWin();
    updateCable();
  });

  window.addEventListener('resize', handleResize);
  window.addEventListener('keydown', handleKeyDown);
};

const initClaw = () => {
  const startX = window.innerWidth / 2;
  const startY = 150;

  const clawFilter = {
    group: Body.nextGroup(true),
    category: CLAW_CATEGORY,
    mask: PRIZE_CATEGORY | WALL_CATEGORY
  };

  clawBase = Bodies.rectangle(startX, startY, 100, 30, {
    isStatic: true,
    collisionFilter: clawFilter,
    render: { fillStyle: '#222' }
  });

  // Simple hook shapes that won't confuse poly-decomp
  const lPath = [{x:0,y:0}, {x:-30,y:40}, {x:-60,y:100}, {x:-40,y:110}, {x:0,y:50}, {x:10,y:0}];
  const rPath = [{x:0,y:0}, {x:30,y:40}, {x:60,y:100}, {x:40,y:110}, {x:0,y:50}, {x:-10,y:0}];

  leftPincer = Bodies.fromVertices(startX - 40, startY + 40, [lPath], {
    collisionFilter: clawFilter,
    friction: 1,
    density: 0.2,
    render: { fillStyle: '#555' }
  });

  rightPincer = Bodies.fromVertices(startX + 40, startY + 40, [rPath], {
    collisionFilter: clawFilter,
    friction: 1,
    density: 0.2,
    render: { fillStyle: '#555' }
  });

  // Helper to connect constraints using world positions
  const connect = (body1, body2, worldX, worldY, stiffness = 1, length = 0) => {
    return Constraint.create({
      bodyA: body1,
      bodyB: body2,
      pointA: { x: worldX - body1.position.x, y: worldY - body1.position.y },
      pointB: { x: worldX - body2.position.x, y: worldY - body2.position.y },
      stiffness,
      length,
      render: { visible: false }
    });
  };

  const leftHinge = connect(clawBase, leftPincer, startX - 40, startY + 15);
  const rightHinge = connect(clawBase, rightPincer, startX + 40, startY + 15);

  leftSpring = connect(clawBase, leftPincer, startX - 80, startY, 0.05, 100);
  rightSpring = connect(clawBase, rightPincer, startX + 80, startY, 0.05, 100);
  
  // Custom render for springs
  leftSpring.render.visible = true;
  leftSpring.render.strokeStyle = '#666';
  rightSpring.render.visible = true;
  rightSpring.render.strokeStyle = '#666';

  // Cable visual
  cable = Bodies.rectangle(startX, startY / 2, 4, startY, {
    isStatic: true,
    isSensor: true,
    render: { fillStyle: '#111' }
  });

  Composite.add(world, [clawBase, leftPincer, rightPincer, leftHinge, rightHinge, leftSpring, rightSpring, cable]);
};

const updateCable = () => {
  if (!clawBase || !cable) return;
  const y = clawBase.position.y;
  Body.setPosition(cable, { x: clawBase.position.x, y: y / 2 });
  Body.scale(cable, 1, 1); // Reset
  // We'll just move and set height manually by re-creating or scaling
  // For simplicity, let's just use a rectangle and set its scale
  const scaleY = y / (cable.bounds.max.y - cable.bounds.min.y);
  // Actually, scaling every frame is bad. Let's just update vertices.
  const top = 0;
  const bottom = y;
  const x = clawBase.position.x;
  Body.setVertices(cable, [
    { x: x - 2, y: top }, { x: x + 2, y: top },
    { x: x + 2, y: bottom }, { x: x - 2, y: bottom }
  ]);
};

const spawnPrizes = async () => {
  const width = window.innerWidth;
  prizes.forEach(p => Composite.remove(world, p));
  prizes = [];
  for (let i = 0; i < 18; i++) {
    const file = prizeFiles[i % prizeFiles.length];
    await createPrize(`/prizes/${file}`, Math.random() * (width - 450) + 350, -100 - (i * 120));
  }
};

const createPrize = async (url, x, y) => {
  const img = new Image();
  img.src = url;
  await new Promise(resolve => img.onload = resolve);
  const targetWidth = (window.innerWidth / 9) * prizeScale.value;
  const scale = targetWidth / img.width;
  const tempCanvas = document.createElement('canvas');
  const ctx = tempCanvas.getContext('2d');
  tempCanvas.width = img.width;
  tempCanvas.height = img.height;
  ctx.drawImage(img, 0, 0);
  const imageData = ctx.getImageData(0, 0, tempCanvas.width, tempCanvas.height);
  const points = [];
  for (let py = 0; py < tempCanvas.height; py += 12) {
    for (let px = 0; px < tempCanvas.width; px += 12) {
      if (imageData.data[((py * tempCanvas.width) + px) * 4 + 3] > 150) {
        points.push({ x: px * scale, y: py * scale });
      }
    }
  }
  const hull = Vertices.hull(points);
  const body = Bodies.fromVertices(x, y, [hull], {
    collisionFilter: { category: PRIZE_CATEGORY },
    render: { sprite: { texture: url, xScale: scale, yScale: scale } },
    friction: 0.9,
    density: 0.002 // Heavier so they don't fly away
  });
  if (body) {
    Composite.add(world, body);
    prizes.push(body);
  }
};

const dropClaw = async () => {
  if (isDropping.value) return;
  isDropping.value = true;
  message.value = "LOCKING ON...";
  const width = window.innerWidth;
  const height = window.innerHeight;
  const destX = (targetX.value / 100) * (width - 500) + 400;

  leftSpring.length = 120;
  rightSpring.length = 120;

  await animateClaw(destX, 150, 6);
  message.value = "DESCENDING...";
  await animateClaw(destX, height - 280, 4);
  
  message.value = "GRAB!";
  leftSpring.length = 20;
  rightSpring.length = 20;
  leftSpring.stiffness = 0.5;
  rightSpring.stiffness = 0.5;
  await new Promise(r => setTimeout(r, 1200));

  message.value = "ASCENDING...";
  await animateClaw(destX, 150, 4);
  message.value = "DELIVERING...";
  await animateClaw(100, 150, 5);

  leftSpring.length = 150;
  rightSpring.length = 150;
  leftSpring.stiffness = 0.02;
  rightSpring.stiffness = 0.02;
  message.value = "RELEASE!";
  await new Promise(r => setTimeout(r, 1500));
  
  isDropping.value = false;
  message.value = "READY!";
};

const animateClaw = (tx, ty, speed) => {
  return new Promise(resolve => {
    const step = () => {
      const dx = tx - clawBase.position.x;
      const dy = ty - clawBase.position.y;
      const dist = Math.sqrt(dx*dx + dy*dy);
      if (dist < speed) {
        Body.setPosition(clawBase, { x: tx, y: ty });
        resolve();
      } else {
        Body.setPosition(clawBase, { 
          x: clawBase.position.x + (dx/dist)*speed, 
          y: clawBase.position.y + (dy/dist)*speed 
        });
        requestAnimationFrame(step);
      }
    };
    step();
  });
};

const checkWin = () => {
  for (let i = prizes.length - 1; i >= 0; i--) {
    const p = prizes[i];
    if (p.position.x < 180 && p.position.y > window.innerHeight - 200) {
      Composite.remove(world, p);
      prizes.splice(i, 1);
      wonPrizesCount.value++;
      message.value = "WINNER!";
      setTimeout(() => { if (!isDropping.value) message.value = "Ready!"; }, 2000);
    }
  }
};

const handleResize = () => {
  render.canvas.width = window.innerWidth;
  render.canvas.height = window.innerHeight;
};

const handleKeyDown = (e) => {
  if (e.key === '`' || e.key === '~') showSettings.value = !showSettings.value;
};

onMounted(initPhysics);
</script>

<template>
  <div class="game-wrapper">
    <canvas ref="canvas"></canvas>
    
    <div class="ui-overlay">
      <div class="header">
        <h1 class="neon-title">CLAW MASTER 3000</h1>
        <div class="win-counter">PRIZES WON: {{ wonPrizesCount }}</div>
        <div class="msg-box">{{ message }}</div>
      </div>

      <div class="control-panel" :class="{ locked: isDropping }">
        <div class="slider-container">
          <div class="coord-label">TARGET POSITION: {{ targetX }}</div>
          <input type="range" v-model="targetX" min="0" max="100" class="neon-range" />
        </div>
        <button class="drop-trigger" @click="dropClaw" :disabled="isDropping">
          <span class="btn-text">DROP</span>
        </button>
      </div>
    </div>

    <div v-if="showSettings" class="dev-menu">
      <h2>ENGINEER MODE</h2>
      <div class="field">
        <label>PRIZE SCALE</label>
        <input type="range" v-model="prizeScale" min="0.2" max="1.0" step="0.05" />
      </div>
      <button @click="spawnPrizes" class="btn-cyan">RE-SPAWN ALL</button>
      <p class="small">Press ~ to exit</p>
    </div>

    <div class="shoot">
      <div class="shoot-text">WIN AREA</div>
    </div>
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=Rajdhani:wght@500;700&display=swap');

body, html {
  margin: 0; padding: 0; overflow: hidden;
  width: 100%; height: 100%;
  background: #000;
  font-family: 'Rajdhani', sans-serif;
}

.game-wrapper {
  width: 100vw; height: 100vh;
  background: linear-gradient(180deg, #09090b 0%, #1e1b4b 100%);
  position: relative;
}

canvas { position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 2; }

.ui-overlay {
  position: absolute; top: 0; left: 0; width: 100%; height: 100%;
  display: flex; flex-direction: column; justify-content: space-between;
  padding: 3rem; box-sizing: border-box; pointer-events: none; z-index: 10;
}

.neon-title {
  font-size: 4rem; color: #fff; text-align: center; margin: 0;
  text-shadow: 0 0 10px #22d3ee, 0 0 20px #22d3ee; letter-spacing: 4px;
}

.win-counter { color: #f472b6; font-size: 1.5rem; text-align: center; margin-top: 10px; font-weight: bold; }
.msg-box { color: #5eead4; font-size: 2rem; text-align: center; margin-top: 20px; font-weight: bold; height: 3rem; }

.control-panel {
  pointer-events: auto; align-self: center;
  background: rgba(15, 23, 42, 0.9); padding: 2.5rem;
  border-radius: 30px; border: 2px solid #334155;
  display: flex; gap: 4rem; align-items: center;
  box-shadow: 0 20px 50px rgba(0,0,0,0.6), inset 0 0 20px rgba(34, 211, 238, 0.1);
}

.control-panel.locked { opacity: 0.4; pointer-events: none; filter: grayscale(0.5); }

.neon-range {
  -webkit-appearance: none; width: 300px; height: 12px;
  background: #1e293b; border-radius: 6px; outline: none;
}

.neon-range::-webkit-slider-thumb {
  -webkit-appearance: none; width: 34px; height: 34px;
  background: #22d3ee; border-radius: 50%; cursor: pointer;
  box-shadow: 0 0 15px #22d3ee; border: 3px solid #fff;
}

.coord-label { color: #94a3b8; margin-bottom: 12px; text-align: center; font-size: 1.1rem; }

.drop-trigger {
  background: #db2777; border: none; padding: 6px; border-radius: 18px;
  cursor: pointer; transition: 0.2s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.btn-text {
  display: block; background: #db2777; color: #fff;
  padding: 1.2rem 4rem; font-size: 2.5rem; font-weight: bold;
  border-radius: 12px; border: 2px solid rgba(255,255,255,0.2);
  box-shadow: inset 0 0 20px rgba(255,255,255,0.3);
}

.drop-trigger:hover { transform: scale(1.08) rotate(2deg); }
.drop-trigger:active { transform: scale(0.92); }

.dev-menu {
  position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
  background: #0f172a; color: #22d3ee; padding: 3rem;
  border-radius: 24px; z-index: 100; border: 2px solid #db2777;
  min-width: 400px; pointer-events: auto; text-align: center;
}

.btn-cyan {
  width: 100%; padding: 1.2rem; background: #22d3ee; color: #0f172a;
  border: none; border-radius: 12px; font-weight: bold; cursor: pointer;
  font-family: inherit; font-size: 1.2rem; margin-top: 1rem;
}

.field { margin: 2rem 0; display: flex; flex-direction: column; gap: 1rem; }
.small { font-size: 0.8rem; opacity: 0.5; margin-top: 2rem; }

.shoot {
  position: absolute; bottom: 0; left: 0; width: 180px; height: 300px;
  background: rgba(34, 211, 238, 0.05); border-right: 2px solid rgba(34, 211, 238, 0.3);
  z-index: 5; display: flex; align-items: center; justify-content: center;
}

.shoot-text {
  color: rgba(34, 211, 238, 0.2); transform: rotate(-90deg);
  font-weight: bold; font-size: 2rem; letter-spacing: 8px;
}
</style>
