<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import Matter from 'matter-js';
import decomp from 'poly-decomp';

// Required for concave shapes in Matter.js
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
} = Matter;

const canvas = ref(null);
const targetX = ref(50); // 0-100
const isDropping = ref(false);
const showSettings = ref(false);
const prizeScale = ref(0.5);
const message = ref("Ready!");
const wonPrizesCount = ref(0);

const prizeFiles = [
  'bear.png',
  'bunny.png',
  'cat.png',
  'doggo.png',
  'pengy.png',
  'sheep.png'
];

let engine, render, runner, world;
let clawBase = null;
let leftPincer = null;
let rightPincer = null;
let leftSpring = null;
let rightSpring = null;
let prizes = [];

const initPhysics = () => {
  engine = Engine.create();
  world = engine.world;
  world.gravity.y = 1.2;

  const width = window.innerWidth;
  const height = window.innerHeight;

  render = Render.create({
    canvas: canvas.value,
    engine: engine,
    options: {
      width: width,
      height: height,
      wireframes: false,
      background: 'transparent',
    },
  });

  Render.run(render);
  runner = Runner.create();
  Runner.run(runner, engine);

  const ground = Bodies.rectangle(width / 2, height + 25, width, 50, { isStatic: true, render: { visible: false } });
  const leftWall = Bodies.rectangle(-25, height / 2, 50, height, { isStatic: true, render: { visible: false } });
  const rightWall = Bodies.rectangle(width + 25, height / 2, 50, height, { isStatic: true, render: { visible: false } });
  const shootWall = Bodies.rectangle(120, height - 100, 10, 200, { isStatic: true, render: { fillStyle: 'rgba(255,255,255,0.2)' } });
  
  Composite.add(world, [ground, leftWall, rightWall, shootWall]);

  initClaw();
  spawnPrizes();
  
  Events.on(engine, 'afterUpdate', checkWin);

  window.addEventListener('resize', handleResize);
  window.addEventListener('keydown', handleKeyDown);
};

const handleResize = () => {
  const width = window.innerWidth;
  const height = window.innerHeight;
  render.canvas.width = width;
  render.canvas.height = height;
  render.options.width = width;
  render.options.height = height;
};

const handleKeyDown = (e) => {
  if (e.key === '`' || e.key === '~') {
    showSettings.value = !showSettings.value;
  }
};

const initClaw = () => {
  const startX = 150;
  const startY = 100;

  clawBase = Bodies.rectangle(startX, startY, 60, 20, {
    label: 'clawBase',
    isStatic: true,
    collisionFilter: { group: -1 },
    render: { fillStyle: '#444' }
  });

  const pincerPath = [
    { x: 0, y: 0 },
    { x: 15, y: 50 },
    { x: 45, y: 80 },
    { x: 55, y: 75 },
    { x: 25, y: 45 },
    { x: 10, y: 0 }
  ];

  leftPincer = Bodies.fromVertices(startX - 25, startY + 40, [pincerPath.map(p => ({ x: -p.x, y: p.y }))], {
    label: 'leftPincer',
    friction: 0.8,
    density: 0.01,
    render: { fillStyle: '#888' }
  });

  rightPincer = Bodies.fromVertices(startX + 25, startY + 40, [pincerPath], {
    label: 'rightPincer',
    friction: 0.8,
    density: 0.01,
    render: { fillStyle: '#888' }
  });

  const leftHinge = Constraint.create({
    bodyA: clawBase,
    pointA: { x: -25, y: 10 },
    bodyB: leftPincer,
    pointB: { x: 5, y: -35 },
    stiffness: 1,
    length: 0,
    render: { visible: false }
  });

  const rightHinge = Constraint.create({
    bodyA: clawBase,
    pointA: { x: 25, y: 10 },
    bodyB: rightPincer,
    pointB: { x: -5, y: -35 },
    stiffness: 1,
    length: 0,
    render: { visible: false }
  });

  leftSpring = Constraint.create({
    bodyA: clawBase,
    pointA: { x: -50, y: -10 },
    bodyB: leftPincer,
    pointB: { x: 0, y: 30 },
    stiffness: 0.02,
    length: 80,
    render: { strokeStyle: '#aaa', lineWidth: 1 }
  });

  rightSpring = Constraint.create({
    bodyA: clawBase,
    pointA: { x: 50, y: -10 },
    bodyB: rightPincer,
    pointB: { x: 0, y: 30 },
    stiffness: 0.02,
    length: 80,
    render: { strokeStyle: '#aaa', lineWidth: 1 }
  });

  Composite.add(world, [clawBase, leftPincer, rightPincer, leftHinge, rightHinge, leftSpring, rightSpring]);
};

const spawnPrizes = async () => {
  const width = window.innerWidth;
  // Clear existing prizes
  prizes.forEach(p => Composite.remove(world, p));
  prizes = [];
  
  for (let i = 0; i < 24; i++) {
    const file = prizeFiles[Math.floor(Math.random() * prizeFiles.length)];
    await createPrize(`/prizes/${file}`, Math.random() * (width - 300) + 200, -100 - (i * 100));
  }
};

const createPrize = async (url, x, y) => {
  const img = new Image();
  img.src = url;
  await new Promise(resolve => img.onload = resolve);

  const targetWidth = (window.innerWidth / 8) * prizeScale.value;
  const scale = targetWidth / img.width;
  
  const tempCanvas = document.createElement('canvas');
  const ctx = tempCanvas.getContext('2d');
  tempCanvas.width = img.width;
  tempCanvas.height = img.height;
  ctx.drawImage(img, 0, 0);

  const imageData = ctx.getImageData(0, 0, tempCanvas.width, tempCanvas.height);
  const points = [];
  const step = 8;
  for (let py = 0; py < tempCanvas.height; py += step) {
    for (let px = 0; px < tempCanvas.width; px += step) {
      const alpha = imageData.data[((py * tempCanvas.width) + px) * 4 + 3];
      if (alpha > 150) {
        points.push({ x: px * scale, y: py * scale });
      }
    }
  }

  const hull = Vertices.hull(points);
  const prizeBody = Bodies.fromVertices(x, y, [hull], {
    render: {
      sprite: {
        texture: url,
        xScale: scale,
        yScale: scale,
      }
    },
    friction: 0.6,
    frictionAir: 0.01,
    restitution: 0.2,
    density: 0.001
  });

  Composite.add(world, prizeBody);
  prizes.push(prizeBody);
};

const dropClaw = async () => {
  if (isDropping.value) return;
  isDropping.value = true;
  message.value = "Positioning...";

  const width = window.innerWidth;
  const height = window.innerHeight;
  const destX = (targetX.value / 100) * (width - 300) + 200;

  leftSpring.length = 90;
  rightSpring.length = 90;
  leftSpring.stiffness = 0.02;
  rightSpring.stiffness = 0.02;

  await animateToX(destX);
  
  message.value = "Going down...";
  await animateToY(height - 250);

  message.value = "GRAB!!";
  leftSpring.length = 10;
  rightSpring.length = 10;
  leftSpring.stiffness = 0.6;
  rightSpring.stiffness = 0.6;

  await new Promise(r => setTimeout(r, 1500));

  message.value = "Lifting prize...";
  await animateToY(150);

  message.value = "Heading to delivery...";
  await animateToX(80);

  message.value = "DROP!";
  leftSpring.length = 100;
  rightSpring.length = 100;
  leftSpring.stiffness = 0.01;
  rightSpring.stiffness = 0.01;

  await new Promise(r => setTimeout(r, 2000));
  
  isDropping.value = false;
  message.value = "Ready!";
};

const animateToX = (target) => {
  return new Promise(resolve => {
    const speed = 4;
    const step = () => {
      const dx = target - clawBase.position.x;
      if (Math.abs(dx) < speed) {
        Body.setPosition(clawBase, { x: target, y: clawBase.position.y });
        resolve();
      } else {
        Body.setPosition(clawBase, { x: clawBase.position.x + Math.sign(dx) * speed, y: clawBase.position.y });
        requestAnimationFrame(step);
      }
    };
    step();
  });
};

const animateToY = (target) => {
  return new Promise(resolve => {
    const speed = 3;
    const step = () => {
      const dy = target - clawBase.position.y;
      if (Math.abs(dy) < speed) {
        Body.setPosition(clawBase, { x: clawBase.position.x, y: target });
        resolve();
      } else {
        Body.setPosition(clawBase, { x: clawBase.position.x, y: clawBase.position.y + Math.sign(dy) * speed });
        requestAnimationFrame(step);
      }
    };
    step();
  });
};

const checkWin = () => {
  for (let i = prizes.length - 1; i >= 0; i--) {
    const prize = prizes[i];
    if (prize.position.x < 120 && prize.position.y > window.innerHeight - 150) {
      Composite.remove(world, prize);
      prizes.splice(i, 1);
      wonPrizesCount.value++;
      message.value = "YOU WON A PRIZE!";
      setTimeout(() => {
        if (!isDropping.value) message.value = "Ready!";
      }, 3000);
    }
  }
};

onMounted(() => {
  initPhysics();
});

onUnmounted(() => {
  if (render) Render.stop(render);
  if (runner) Runner.stop(runner);
  if (engine) Engine.clear(engine);
  window.removeEventListener('resize', handleResize);
  window.removeEventListener('keydown', handleKeyDown);
});

</script>

<template>
  <div class="game-wrapper">
    <canvas ref="canvas"></canvas>
    
    <div class="ui-overlay">
      <div class="header">
        <h1>CLAW MASTER</h1>
        <div class="stats">PRIZES WON: {{ wonPrizesCount }}</div>
        <p v-if="message" class="message">{{ message }}</p>
      </div>

      <div class="controls" :class="{ disabled: isDropping }">
        <div class="input-group">
          <label>Position (0-100)</label>
          <input type="number" v-model="targetX" min="0" max="100" />
        </div>
        <button class="drop-btn" @click="dropClaw" :disabled="isDropping">
          DROP!
        </button>
      </div>
    </div>

    <!-- Hidden Settings -->
    <div v-if="showSettings" class="settings-modal">
      <h3>Hidden Settings</h3>
      <div class="setting">
        <label>Prize Scale: {{ prizeScale }}</label>
        <input type="range" v-model="prizeScale" min="0.1" max="1.0" step="0.1" />
      </div>
      <button @click="spawnPrizes" class="respawn-btn">Re-spawn Prizes</button>
      <p class="hint">Press ~ to hide</p>
    </div>

    <div class="shoot">
      <div class="shoot-label">WIN AREA</div>
    </div>
  </div>
</template>

<style>
body, html {
  margin: 0;
  padding: 0;
  overflow: hidden;
  width: 100%;
  height: 100%;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

#app {
  width: 100%;
  height: 100%;
}

.game-wrapper {
  width: 100vw;
  height: 100vh;
  background: linear-gradient(135deg, #008080 0%, #2e004b 100%);
  position: relative;
}

canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.ui-overlay {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  padding: 2rem;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  align-items: center;
  pointer-events: none;
  z-index: 10;
}

.header {
  position: absolute;
  top: 2rem;
  text-align: center;
  color: white;
  text-shadow: 0 4px 8px rgba(0,0,0,0.5);
}

.header h1 {
  margin: 0;
  font-size: 3rem;
  letter-spacing: 0.5rem;
}

.stats {
  font-size: 1.2rem;
  margin-top: 0.5rem;
  color: #ffde00;
  font-weight: bold;
}

.message {
  font-size: 1.5rem;
  color: #00ffcc;
  font-weight: bold;
  margin-top: 1rem;
}

.controls {
  pointer-events: auto;
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  padding: 1.5rem;
  border-radius: 20px;
  display: flex;
  gap: 2rem;
  align-items: center;
  border: 2px solid rgba(255,255,255,0.2);
  box-shadow: 0 10px 30px rgba(0,0,0,0.3);
}

.controls.disabled {
  opacity: 0.5;
  pointer-events: none;
}

.input-group {
  display: flex;
  flex-direction: column;
  color: white;
  gap: 0.5rem;
}

.input-group input {
  background: rgba(0,0,0,0.3);
  border: 1px solid rgba(255,255,255,0.3);
  color: white;
  padding: 0.5rem;
  border-radius: 8px;
  font-size: 1.2rem;
  width: 100px;
  text-align: center;
}

.drop-btn {
  background: #ff0080;
  color: white;
  border: none;
  padding: 1rem 2.5rem;
  font-size: 1.5rem;
  font-weight: bold;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 5px 0 #b30059;
}

.drop-btn:hover {
  background: #ff1a8c;
  transform: translateY(-2px);
  box-shadow: 0 7px 0 #b30059;
}

.drop-btn:active {
  transform: translateY(3px);
  box-shadow: 0 2px 0 #b30059;
}

.settings-modal {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(0, 0, 0, 0.9);
  color: white;
  padding: 2rem;
  border-radius: 20px;
  z-index: 100;
  border: 2px solid #ff0080;
  min-width: 300px;
}

.setting {
  margin: 1rem 0;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.respawn-btn {
  width: 100%;
  padding: 0.8rem;
  background: #00ffcc;
  color: #333;
  border: none;
  border-radius: 8px;
  font-weight: bold;
  cursor: pointer;
}

.hint {
  font-size: 0.8rem;
  opacity: 0.7;
  text-align: center;
  margin-top: 1rem;
}

.shoot {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 150px;
  height: 250px;
  background: rgba(0, 0, 0, 0.3);
  border-right: 4px solid rgba(255,255,255,0.1);
  z-index: 5;
  display: flex;
  align-items: center;
  justify-content: center;
}

.shoot-label {
  color: rgba(255,255,255,0.1);
  transform: rotate(-90deg);
  font-weight: bold;
  font-size: 1.5rem;
  white-space: nowrap;
}
</style>
