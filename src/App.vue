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
	Body,
	Events,
	Vertices,
	Common,
} = Matter;

Common.setDecomp(decomp);

// ─── Vue refs ─────────────────────────────────────────────────────────────────
const canvas = ref(null);
const targetX = ref(50);
const isDropping = ref(false);
const showSettings = ref(false);
const prizeScale = ref(0.45);
const message = ref('Ready!');
const wonPrizesCount = ref(0);

const prizeFiles = ['bear.png', 'bunny.png', 'cat.png', 'doggo.png', 'pengy.png', 'sheep.png'];

// ─── Matter.js globals ────────────────────────────────────────────────────────
let engine, render, runner, world;
let prizes = [];

// ─── Collision categories ─────────────────────────────────────────────────────
const PRIZE_CATEGORY = 0x0004;
const WALL_CATEGORY  = 0x0001;

// ─── Claw visual constants ────────────────────────────────────────────────────
const HOUSING_W       = 80;   // px — width of housing box
const HOUSING_H       = 28;   // px — height of housing box
const ARM_LENGTH      = 100;  // px — arm length from pivot to tip
const MAX_SPREAD      = 65;   // px — max lateral spread of arm tips from center
const SENSOR_Y_OFFSET = 42;   // px — sensor offset below housing bottom (upper part of opening)
const SENSOR_RADIUS   = 50;   // px — proximity detection radius
const GRAB_HALF_W     = MAX_SPREAD * 0.85; // px — horizontal grab tolerance
const INTERIOR_HALF_W = 18;   // px — half-width of claw interior (closed)
const INTERIOR_BOT    = 48;   // px — bottom of fake interior (relative to hold point)
const OPEN_ANIM_MS    = 420;  // ms — open/close animation duration

// ─── Claw runtime state ───────────────────────────────────────────────────────
let clawX          = 0;
let clawY          = 0;
let clawOpenAmount = 1.0;  // 0 = fully closed, 1 = fully open

// Animation state
let openAnimActive = false;
let openAnimStart  = 0;
let openAnimFrom   = 1.0;
let openAnimTo     = 0.0;

// ─── Fake-physics state for grabbed prize ─────────────────────────────────────
let grabbedPrize = null;
let fakeRelX     = 0;  // prize x offset relative to claw hold-point
let fakeRelY     = 0;  // prize y offset relative to claw hold-point
let fakeVelX     = 0;
let fakeVelY     = 0;

// ─── Init ─────────────────────────────────────────────────────────────────────

/**
 * Creates the physics engine, renderer, world boundaries, and kicks off the game.
 */
const initPhysics = () => {
	clawX = window.innerWidth / 2;
	clawY = 100;

	engine = Engine.create();
	world  = engine.world;
	world.gravity.y = 1.0;

	render = Render.create({
		canvas: canvas.value,
		engine,
		options: {
			width:      window.innerWidth,
			height:     window.innerHeight,
			wireframes: false,
			background: 'transparent',
		},
	});

	Render.run(render);
	runner = Runner.create();
	Runner.run(runner, engine);

	const W = window.innerWidth;
	const H = window.innerHeight;

	const wallOpts = {
		isStatic: true,
		collisionFilter: { category: WALL_CATEGORY },
		render: { visible: false },
	};

	Composite.add(world, [
		Bodies.rectangle(W / 2, H + 25, W, 50, wallOpts),
		Bodies.rectangle(-25,   H / 2,  50, H,  wallOpts),
		Bodies.rectangle(W + 25, H / 2, 50, H,  wallOpts),
		// Chute divider wall
		Bodies.rectangle(180, H - 150, 15, 400, {
			isStatic: true,
			collisionFilter: { category: WALL_CATEGORY },
			render: { fillStyle: '#2c3e50' },
		}),
	]);

	spawnPrizes();

	Events.on(engine, 'afterUpdate', onPhysicsUpdate);
	Events.on(render,  'afterRender', drawClaw);

	window.addEventListener('resize', handleResize);
	window.addEventListener('keydown', handleKeyDown);
};

// ─── Per-frame callbacks ──────────────────────────────────────────────────────

/**
 * Runs each physics tick: ticks claw open/close animation,
 * updates fake-physics for held prize, and checks the win zone.
 */
const onPhysicsUpdate = () => {
	if (openAnimActive) {
		const elapsed = performance.now() - openAnimStart;
		const t       = Math.min(elapsed / OPEN_ANIM_MS, 1.0);
		// Ease-in-out quad
		const ease    = t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;
		clawOpenAmount = openAnimFrom + (openAnimTo - openAnimFrom) * ease;
		if (t >= 1.0) {
			openAnimActive = false;
			clawOpenAmount = openAnimTo;
		}
	}

	if (grabbedPrize) tickFakePhysics();

	checkWin();
};

/**
 * Simulates the held prize swinging inside claw-local space using simple
 * Euler integration with spring, gravity, and damping. Overrides the body's
 * real physics position each frame so Matter.js doesn't interfere.
 */
const tickFakePhysics = () => {
	const GRAVITY  = 0.30;
	const DAMPING  = 0.88;
	const SPRING_K = 0.038;

	fakeVelY += GRAVITY;
	fakeVelX -= fakeRelX * SPRING_K;
	fakeVelY -= fakeRelY * SPRING_K;
	fakeVelX *= DAMPING;
	fakeVelY *= DAMPING;
	fakeRelX += fakeVelX;
	fakeRelY += fakeVelY;

	// Bounce off claw interior walls — wider when open so it can feel loose
	const halfW = INTERIOR_HALF_W + clawOpenAmount * 28;
	if (fakeRelX >  halfW) { fakeRelX =  halfW; fakeVelX *= -0.45; }
	if (fakeRelX < -halfW) { fakeRelX = -halfW; fakeVelX *= -0.45; }
	if (fakeRelY < 0)           { fakeRelY = 0;           fakeVelY *= -0.30; }
	if (fakeRelY > INTERIOR_BOT) { fakeRelY = INTERIOR_BOT; fakeVelY *= -0.50; }

	// Clamp prize to computed world position
	const holdY = clawY + HOUSING_H / 2 + ARM_LENGTH * 0.62;
	Body.setPosition(grabbedPrize, { x: clawX + fakeRelX, y: holdY + fakeRelY });
	Body.setVelocity(grabbedPrize, { x: 0, y: 0 });
	Body.setAngularVelocity(grabbedPrize, 0);
};

// ─── Win detection ────────────────────────────────────────────────────────────

/**
 * Checks whether any free prize has entered the win chute area.
 */
const checkWin = () => {
	const H = window.innerHeight;
	for (let i = prizes.length - 1; i >= 0; i--) {
		const p = prizes[i];
		if (p === grabbedPrize) continue;
		if (p.position.x < 180 && p.position.y > H - 200) {
			Composite.remove(world, p);
			prizes.splice(i, 1);
			wonPrizesCount.value++;
			message.value = 'WINNER!';
			setTimeout(() => { if (!isDropping.value) message.value = 'Ready!'; }, 2000);
		}
	}
};

// ─── Claw rendering ───────────────────────────────────────────────────────────

/**
 * Draws the full claw assembly (cable, housing, motor, arms) on top of the
 * Matter.js canvas each render frame. Called via Events.on(render, 'afterRender').
 */
const drawClaw = () => {
	const ctx = render.context;
	const cx  = clawX;
	const cy  = clawY;

	ctx.save();

	// ── Cable ──
	const cableGrad = ctx.createLinearGradient(cx - 3, 0, cx + 3, 0);
	cableGrad.addColorStop(0,    '#333');
	cableGrad.addColorStop(0.40, '#bbb');
	cableGrad.addColorStop(1,    '#2a2a2a');
	ctx.strokeStyle = cableGrad;
	ctx.lineWidth   = 5;
	ctx.beginPath();
	ctx.moveTo(cx, 0);
	ctx.lineTo(cx, cy - HOUSING_H / 2);
	ctx.stroke();

	// ── Housing box ──
	const hx      = cx - HOUSING_W / 2;
	const hy      = cy - HOUSING_H / 2;
	const boxGrad = ctx.createLinearGradient(hx, hy, hx, hy + HOUSING_H);
	boxGrad.addColorStop(0,    '#a0a0a0');
	boxGrad.addColorStop(0.35, '#e8e8e8');
	boxGrad.addColorStop(1,    '#505050');
	ctx.fillStyle   = boxGrad;
	ctx.strokeStyle = '#1a1a1a';
	ctx.lineWidth   = 1.5;

	// Rounded rect (manual for broad browser compat)
	const R = 5;
	ctx.beginPath();
	ctx.moveTo(hx + R, hy);
	ctx.lineTo(hx + HOUSING_W - R, hy);
	ctx.quadraticCurveTo(hx + HOUSING_W, hy, hx + HOUSING_W, hy + R);
	ctx.lineTo(hx + HOUSING_W, hy + HOUSING_H - R);
	ctx.quadraticCurveTo(hx + HOUSING_W, hy + HOUSING_H, hx + HOUSING_W - R, hy + HOUSING_H);
	ctx.lineTo(hx + R, hy + HOUSING_H);
	ctx.quadraticCurveTo(hx, hy + HOUSING_H, hx, hy + HOUSING_H - R);
	ctx.lineTo(hx, hy + R);
	ctx.quadraticCurveTo(hx, hy, hx + R, hy);
	ctx.closePath();
	ctx.fill();
	ctx.stroke();

	// Rivet detail lines on housing
	ctx.strokeStyle = 'rgba(0,0,0,0.25)';
	ctx.lineWidth   = 1;
	for (let i = 1; i < 4; i++) {
		const lx = hx + (HOUSING_W / 4) * i;
		ctx.beginPath();
		ctx.moveTo(lx, hy + 4);
		ctx.lineTo(lx, hy + HOUSING_H - 4);
		ctx.stroke();
	}

	// Motor nub
	const motorGrad = ctx.createRadialGradient(cx - 2, cy - 2, 1, cx, cy, 9);
	motorGrad.addColorStop(0, '#888');
	motorGrad.addColorStop(1, '#222');
	ctx.fillStyle = motorGrad;
	ctx.beginPath();
	ctx.arc(cx, cy, 9, 0, Math.PI * 2);
	ctx.fill();
	ctx.fillStyle = '#aaa';
	ctx.beginPath();
	ctx.arc(cx, cy, 4, 0, Math.PI * 2);
	ctx.fill();

	// ── Arms ──
	drawClawArm(ctx, cx, cy, -1, clawOpenAmount);
	drawClawArm(ctx, cx, cy,  1, clawOpenAmount);

	ctx.restore();
};

/**
 * Draws one curved claw arm with a metallic gradient and hooked tip.
 *
 * @param {CanvasRenderingContext2D} ctx
 * @param {number} cx   - claw center x (world)
 * @param {number} cy   - claw center y, housing center (world)
 * @param {number} side - -1 = left arm, 1 = right arm
 * @param {number} open - 0 = fully closed, 1 = fully open
 */
const drawClawArm = (ctx, cx, cy, side, open) => {
	const s      = side;
	const spread = open * MAX_SPREAD;

	// Pivot sits at bottom corner of housing
	const pivotX = cx + s * (HOUSING_W * 0.28);
	const pivotY = cy + HOUSING_H / 2;

	// Tip sits below and laterally offset based on spread
	const tipX = cx + s * (8 + spread);
	const tipY = pivotY + ARM_LENGTH;

	// ── Outer edge bezier (wide side of arm) ──
	const oc1x = pivotX + s * (10 + spread * 0.45);
	const oc1y = pivotY + ARM_LENGTH * 0.38;
	const oc2x = tipX   + s * 14;
	const oc2y = tipY   - ARM_LENGTH * 0.18;

	// ── Inner edge bezier (narrow side — gives arm its body) ──
	const innerPivotX = cx + s * (HOUSING_W * 0.13);
	const ic1x = innerPivotX + s * (spread * 0.25);
	const ic1y = pivotY      + ARM_LENGTH * 0.30;
	const ic2x = cx + s * (spread * 0.70 + 2);
	const ic2y = tipY - ARM_LENGTH * 0.12;

	// Metallic diagonal gradient
	const grad = ctx.createLinearGradient(pivotX, pivotY, tipX + s * 10, tipY);
	grad.addColorStop(0,    '#c0c0c0');
	grad.addColorStop(0.28, '#e6e6e6');
	grad.addColorStop(0.62, '#848484');
	grad.addColorStop(1,    '#3a3a3a');

	ctx.fillStyle   = grad;
	ctx.strokeStyle = '#1a1a1a';
	ctx.lineWidth   = 1.2;

	ctx.beginPath();
	// Outer edge — pivot → tip
	ctx.moveTo(pivotX + s * 9, pivotY);
	ctx.bezierCurveTo(
		oc1x + s * 8, oc1y,
		oc2x + s * 5, oc2y,
		tipX  + s * 7, tipY - 2,
	);
	// Hooked tip curves inward
	ctx.lineTo(tipX + s * 3, tipY + 16);
	ctx.quadraticCurveTo(tipX - s * 2, tipY + 22, tipX - s * 10, tipY + 9);
	// Inner edge — tip → pivot
	ctx.bezierCurveTo(ic2x, ic2y, ic1x, ic1y, innerPivotX - s * 1, pivotY);
	ctx.closePath();
	ctx.fill();
	ctx.stroke();

	// Specular highlight stripe along outer edge
	ctx.save();
	ctx.globalAlpha = 0.30;
	ctx.strokeStyle = '#ffffff';
	ctx.lineWidth   = 2;
	ctx.beginPath();
	ctx.moveTo(pivotX + s * 6, pivotY + 5);
	ctx.bezierCurveTo(
		pivotX + s * (5 + spread * 0.35), pivotY + ARM_LENGTH * 0.32,
		tipX   + s * 9,                   tipY   - ARM_LENGTH * 0.22,
		tipX   + s * 5,                   tipY   - 6,
	);
	ctx.stroke();
	ctx.restore();
};

// ─── Sensor & grab ────────────────────────────────────────────────────────────

/**
 * Returns the closest prize that is within sensor range of the claw's
 * detection zone — a point positioned high inside the opening so the
 * claw doesn't trigger too early on objects below.
 *
 * @returns {Matter.Body|null}
 */
const checkSensor = () => {
	const sensorX = clawX;
	const sensorY = clawY + HOUSING_H / 2 + SENSOR_Y_OFFSET;
	let closest     = null;
	let closestDist = SENSOR_RADIUS;

	for (const p of prizes) {
		if (p === grabbedPrize) continue;
		const dx   = p.position.x - sensorX;
		const dy   = p.position.y - sensorY;
		const dist = Math.sqrt(dx * dx + dy * dy);
		if (dist < closestDist) {
			closestDist = dist;
			closest     = p;
		}
	}

	return closest;
};

/**
 * Returns true when the claw arms are realistically positioned on the
 * left and right sides of the prize (i.e. prize is between the tips
 * horizontally and within vertical reach).
 *
 * @param {Matter.Body} prize
 * @returns {boolean}
 */
const clawCanGrab = (prize) => {
	const px      = prize.position.x;
	const py      = prize.position.y;
	const horizOk = Math.abs(px - clawX) < GRAB_HALF_W;
	const vertTop = clawY + HOUSING_H / 2;
	const vertBot = clawY + HOUSING_H / 2 + ARM_LENGTH + 20;
	return horizOk && py > vertTop && py < vertBot;
};

/**
 * Moves a prize from the real physics layer into fake-physics grab mode.
 * Its collision is ghosted out so it can't interact with anything while held.
 *
 * @param {Matter.Body} prize
 */
const grabPrize = (prize) => {
	grabbedPrize = prize;
	const holdY  = clawY + HOUSING_H / 2 + ARM_LENGTH * 0.62;
	fakeRelX = prize.position.x - clawX;
	fakeRelY = prize.position.y - holdY;
	fakeVelX = 0;
	fakeVelY = 0;
	Body.set(prize, { collisionFilter: { category: 0, mask: 0 } });
};

/**
 * Returns the held prize to the normal physics layer, restoring collisions
 * and giving it an exit velocity based on its fake-physics momentum.
 */
const releasePrize = () => {
	if (!grabbedPrize) return;
	Body.set(grabbedPrize, {
		collisionFilter: { category: PRIZE_CATEGORY, mask: WALL_CATEGORY | PRIZE_CATEGORY },
	});
	Body.setVelocity(grabbedPrize, { x: fakeVelX * 2.5, y: fakeVelY * 2.5 });
	grabbedPrize = null;
};

// ─── Movement helpers ─────────────────────────────────────────────────────────

/**
 * Smoothly moves the claw to (tx, ty) at the given speed in px/frame.
 * Returns a Promise that resolves when the claw arrives.
 *
 * @param {number} tx
 * @param {number} ty
 * @param {number} speed
 * @returns {Promise<void>}
 */
const moveClaw = (tx, ty, speed) => {
	return new Promise(resolve => {
		const step = () => {
			const dx   = tx - clawX;
			const dy   = ty - clawY;
			const dist = Math.sqrt(dx * dx + dy * dy);
			if (dist < speed) {
				clawX = tx;
				clawY = ty;
				resolve();
				return;
			}
			clawX += (dx / dist) * speed;
			clawY += (dy / dist) * speed;
			requestAnimationFrame(step);
		};
		step();
	});
};

/**
 * Triggers the open/close animation and returns a Promise that resolves
 * once the animation completes.
 *
 * @param {number} toAmount - 0 = closed, 1 = open
 * @returns {Promise<void>}
 */
const animateClawTo = (toAmount) => {
	return new Promise(resolve => {
		openAnimFrom   = clawOpenAmount;
		openAnimTo     = toAmount;
		openAnimStart  = performance.now();
		openAnimActive = true;
		setTimeout(resolve, OPEN_ANIM_MS + 50);
	});
};

// ─── Drop sequence ────────────────────────────────────────────────────────────

/**
 * Executes the full claw-machine drop cycle:
 *   open → position → descend (sensor) → close → grab? → ascend → deliver → release
 */
const dropClaw = async () => {
	if (isDropping.value) return;
	isDropping.value = true;

	// Safety: release anything stuck from a previous run
	if (grabbedPrize) releasePrize();

	const W      = window.innerWidth;
	const H      = window.innerHeight;
	const topY   = 100;
	const floorY = H - 200;
	const destX  = (targetX.value / 100) * (W - 500) + 400;

	// 1. Open claw
	await animateClawTo(1.0);

	// 2. Lateral positioning
	message.value = 'LOCKING ON...';
	await moveClaw(destX, topY, 8);

	// 3. Descend — stop the moment the sensor picks up a nearby prize
	message.value = 'DESCENDING...';
	let detectedPrize = null;

	await new Promise(resolve => {
		const descend = () => {
			clawY += 4;
			if (clawY >= floorY) { clawY = floorY; resolve(); return; }
			detectedPrize = checkSensor();
			if (detectedPrize) { resolve(); return; }
			requestAnimationFrame(descend);
		};
		descend();
	});

	// 4. Close claw
	message.value = 'GRAB!';
	await animateClawTo(0.0);
	await new Promise(r => setTimeout(r, 130));

	// 5. Check if grab is geometrically valid before committing
	if (detectedPrize && clawCanGrab(detectedPrize)) {
		grabPrize(detectedPrize);
	}

	// 6. Ascend
	message.value = 'ASCENDING...';
	await moveClaw(clawX, topY, 4);

	// 7. Travel to chute
	message.value = 'DELIVERING...';
	await moveClaw(90, topY, 6);

	// 8. Open and release
	message.value = 'RELEASE!';
	await animateClawTo(1.0);

	if (grabbedPrize) {
		// If the prize is directly above the win zone, award it immediately
		const absX = clawX + fakeRelX;
		if (absX < 180) {
			wonPrizesCount.value++;
			Composite.remove(world, grabbedPrize);
			const idx = prizes.indexOf(grabbedPrize);
			if (idx !== -1) prizes.splice(idx, 1);
			grabbedPrize = null;
			message.value = 'WINNER!';
		} else {
			releasePrize();
		}
	}

	await new Promise(r => setTimeout(r, 900));
	isDropping.value = false;
	if (message.value !== 'WINNER!') message.value = 'Ready!';
};

// ─── Prize spawning ───────────────────────────────────────────────────────────

/**
 * Removes all existing prizes and spawns a fresh set above the viewport.
 */
const spawnPrizes = async () => {
	const W = window.innerWidth;
	prizes.forEach(p => Composite.remove(world, p));
	prizes = [];
	for (let i = 0; i < 18; i++) {
		const file = prizeFiles[i % prizeFiles.length];
		await createPrize(`/prizes/${file}`, Math.random() * (W - 450) + 350, -100 - (i * 120));
	}
};

/**
 * Loads a prize image, generates a convex-hull physics body from its
 * non-transparent pixels, and adds it to the world.
 *
 * @param {string} url - public image path
 * @param {number} x   - spawn x
 * @param {number} y   - spawn y (typically above the viewport)
 */
const createPrize = async (url, x, y) => {
	const img = new Image();
	img.src   = url;
	await new Promise(resolve => (img.onload = resolve));

	const targetWidth = (window.innerWidth / 9) * prizeScale.value;
	const scale       = targetWidth / img.width;
	const tc          = document.createElement('canvas');
	const ctx         = tc.getContext('2d');
	tc.width  = img.width;
	tc.height = img.height;
	ctx.drawImage(img, 0, 0);

	const data   = ctx.getImageData(0, 0, tc.width, tc.height);
	const points = [];
	for (let py = 0; py < tc.height; py += 12) {
		for (let px = 0; px < tc.width; px += 12) {
			if (data.data[((py * tc.width) + px) * 4 + 3] > 150) {
				points.push({ x: px * scale, y: py * scale });
			}
		}
	}

	const hull = Vertices.hull(points);
	const body = Bodies.fromVertices(x, y, [hull], {
		collisionFilter: { category: PRIZE_CATEGORY, mask: WALL_CATEGORY | PRIZE_CATEGORY },
		render:          { sprite: { texture: url, xScale: scale, yScale: scale } },
		friction:        0.9,
		density:         0.002,
	});
	if (body) {
		Composite.add(world, body);
		prizes.push(body);
	}
};

// ─── Misc ─────────────────────────────────────────────────────────────────────

const handleResize = () => {
	render.canvas.width  = window.innerWidth;
	render.canvas.height = window.innerHeight;
};

const handleKeyDown = (e) => {
	if (e.key === '`' || e.key === '~') showSettings.value = !showSettings.value;
};

onMounted(initPhysics);

onUnmounted(() => {
	window.removeEventListener('resize', handleResize);
	window.removeEventListener('keydown', handleKeyDown);
	if (runner) Runner.stop(runner);
	if (render) Render.stop(render);
});
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
