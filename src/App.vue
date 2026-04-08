<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
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

// ─── Claw PNG metadata (native pixel dimensions and pivot/attach coords) ───────
/** claw_top.png: 290×299, pivot at (145,16), grip attach points at y=228 */
const CLAW_TOP_META = {
	w: 290, h: 299,
	pivX: 145, pivY: 16,
	lgAttachX: 10,  lgAttachY: 228,  // left grip attach
	rgAttachX: 279, rgAttachY: 228,  // right grip attach
	bgAttachX: 145, bgAttachY: 228,  // back grip attach (center)
};
/** left_grip.png: 190×357, pivot at (165,25) — rotates CCW when closing */
const LEFT_GRIP_META  = { w: 190, h: 357, pivX: 165, pivY: 25 };
/** right_grip.png: 190×357, pivot at (25,25) — rotates CW when closing */
const RIGHT_GRIP_META = { w: 190, h: 357, pivX: 25,  pivY: 25 };
/** back_grip.png: 65×305, pivot at (31,14) — no rotation */
const BACK_GRIP_META  = { w: 65,  h: 305, pivX: 31,  pivY: 14 };

/** Native grip length in pixels (pivot-to-tip) */
const GRIP_LENGTH_NATIVE = 332;   // RIGHT_GRIP_META.h - RIGHT_GRIP_META.pivY
/** Native distance from claw_top pivot down to grip attachment point */
const GRIP_ATTACH_NATIVE_Y = 212; // CLAW_TOP_META.lgAttachY - CLAW_TOP_META.pivY
/** Maximum closing angle for left/right grips in degrees */
const CLOSE_ANGLE_DEG = 20;

// ─── Vue refs ─────────────────────────────────────────────────────────────────
const canvas    = ref(null);
const bgCanvas  = ref(null);
const targetX   = ref(50);
const isDropping = ref(false);
const showSettings = ref(false);
const prizeScale = ref(0.45);
const message = ref('Ready!');
const wonPrizesCount = ref(0);

// Slip mechanic settings
const slipChance  = ref(60);  // percent chance a grabbed prize will slip
const slipMinTime = ref(3);   // seconds (minimum slip duration)
const slipMaxTime = ref(6);   // seconds (maximum slip duration)

const prizeFiles = ['bear.png', 'bunny.png', 'cat.png', 'doggo.png', 'pengy.png', 'sheep.png'];

// ─── Matter.js globals ────────────────────────────────────────────────────────
let engine, render, runner, world;
let prizes = [];

// ─── Collision categories ─────────────────────────────────────────────────────
const PRIZE_CATEGORY = 0x0004;
const WALL_CATEGORY  = 0x0001;

// ─── Claw base constants (unscaled, at prizeScale = 0.45) ────────────────────
const ARM_LENGTH_BASE      = 100;
const MAX_SPREAD_BASE      = 65;
const SENSOR_Y_OFFSET_BASE = 42;
const INTERIOR_HALF_W_BASE = 18;
const INTERIOR_BOT_BASE    = 48;
const OPEN_ANIM_MS         = 420;

// ─── Dynamic chute width (scales with prize size) ─────────────────────────────
/**
 * Width of the win chute in pixels. Scales proportionally with prizeScale so
 * larger prizes can still fall into it.
 * @type {import('vue').ComputedRef<number>}
 */
const chuteWidth = computed(() => {
	const cs = Math.max(0.8, prizeScale.value / 0.45);
	return Math.round(180 * cs);
});

/**
 * Returns all claw dimensions scaled proportionally to the current prize size.
 * At default prizeScale (0.45) every value equals its BASE counterpart.
 *
 * @returns {{
 *   cs: number, armLen: number, maxSpread: number,
 *   sensorYOff: number, grabHalfW: number,
 *   intHalfW: number, intBot: number,
 *   clawRenderScale: number, attachOffset: number
 * }}
 */
const getDims = () => {
	const cs             = Math.max(0.8, prizeScale.value / 0.45);
	const armLen         = ARM_LENGTH_BASE * cs;
	const clawRenderScale = armLen / GRIP_LENGTH_NATIVE;
	const attachOffset   = GRIP_ATTACH_NATIVE_Y * clawRenderScale;
	const maxSpread      = MAX_SPREAD_BASE * cs;
	return {
		cs,
		armLen,
		maxSpread,
		sensorYOff: SENSOR_Y_OFFSET_BASE * cs,
		grabHalfW:  maxSpread * 0.85,
		intHalfW:   INTERIOR_HALF_W_BASE * cs,
		intBot:     INTERIOR_BOT_BASE    * cs,
		clawRenderScale,
		attachOffset,
	};
};

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
let fakeRelX     = 0;
let fakeRelY     = 0;
let fakeVelX     = 0;
let fakeVelY     = 0;

// ─── Slip state ───────────────────────────────────────────────────────────────
let isSlipping    = false;
let slipStartTime = 0;
let slipDuration  = 0;  // seconds

// ─── Loaded sprite images ─────────────────────────────────────────────────────
let clawTopImg   = null;
let leftGripImg  = null;
let rightGripImg = null;
let backGripImg  = null;

// ─── Chute physics wall ───────────────────────────────────────────────────────
let chuteWall = null;

// ─── Image loading ────────────────────────────────────────────────────────────

/**
 * Preloads all four claw sprite images before the game starts.
 * @returns {Promise<void>}
 */
const loadImages = () => {
	return new Promise(resolve => {
		let loaded = 0;
		const onLoad = () => { if (++loaded === 4) resolve(); };

		clawTopImg   = new Image();
		leftGripImg  = new Image();
		rightGripImg = new Image();
		backGripImg  = new Image();

		clawTopImg.onload   = onLoad;
		leftGripImg.onload  = onLoad;
		rightGripImg.onload = onLoad;
		backGripImg.onload  = onLoad;

		clawTopImg.src   = '/claw/claw_top.png';
		leftGripImg.src  = '/claw/left_grip.png';
		rightGripImg.src = '/claw/right_grip.png';
		backGripImg.src  = '/claw/back_grip.png';
	});
};

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
		Bodies.rectangle(-25,    H / 2, 50, H,  wallOpts),
		Bodies.rectangle(W + 25, H / 2, 50, H,  wallOpts),
	]);

	// Size the background canvas
	if (bgCanvas.value) {
		bgCanvas.value.width  = W;
		bgCanvas.value.height = H;
	}

	rebuildChute();
	spawnPrizes();

	Events.on(engine, 'afterUpdate', onPhysicsUpdate);
	Events.on(render,  'afterRender', drawClaw);

	window.addEventListener('resize', handleResize);
	window.addEventListener('keydown', handleKeyDown);
};

// ─── Chute management ─────────────────────────────────────────────────────────

/**
 * Removes the old chute divider wall (if any) and creates a new one sized to
 * match the current chuteWidth. Called on init and whenever prizes are re-spawned.
 */
const rebuildChute = () => {
	if (chuteWall) Composite.remove(world, chuteWall);
	const H = window.innerHeight;
	const w = chuteWidth.value;
	chuteWall = Bodies.rectangle(w, H - 150, 15, 400, {
		isStatic: true,
		collisionFilter: { category: WALL_CATEGORY },
		render: { fillStyle: '#2c3e50' },
	});
	Composite.add(world, chuteWall);
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
 *
 * When a slip is active the spring weakens and the effective floor grows,
 * allowing the prize to gradually slide out of the claw. At progress=1 the
 * prize is released back into real physics.
 */
const tickFakePhysics = () => {
	const GRAVITY      = 0.30;
	const DAMPING      = 0.88;
	const BASE_SPRING_K = 0.038;
	const { cs, attachOffset, armLen, intHalfW, intBot } = getDims();

	// Slip: weaken spring and expand the bottom clamp over time
	let springK     = BASE_SPRING_K;
	let effectiveBot = intBot;

	if (isSlipping) {
		const elapsed  = (performance.now() - slipStartTime) / 1000;
		const progress = Math.min(elapsed / slipDuration, 1.0);
		// Spring fades to ~5% of normal so gravity takes over
		springK      = BASE_SPRING_K * (1 - progress * 0.95);
		effectiveBot = intBot + progress * armLen * 1.5;
		if (progress >= 1.0) {
			isSlipping = false;
			releasePrize();
			return;
		}
	}

	fakeVelY += GRAVITY;
	fakeVelX -= fakeRelX * springK;
	fakeVelY -= fakeRelY * springK;
	fakeVelX *= DAMPING;
	fakeVelY *= DAMPING;
	fakeRelX += fakeVelX;
	fakeRelY += fakeVelY;

	// Bounce off scaled claw interior walls — wider when open so it feels loose
	const halfW = intHalfW + clawOpenAmount * 28 * cs;
	if (fakeRelX >  halfW)  { fakeRelX =  halfW;  fakeVelX *= -0.45; }
	if (fakeRelX < -halfW)  { fakeRelX = -halfW;  fakeVelX *= -0.45; }
	if (fakeRelY < 0)       { fakeRelY = 0;        fakeVelY *= -0.30; }
	if (fakeRelY > effectiveBot) { fakeRelY = effectiveBot; fakeVelY *= -0.50; }

	const holdY = clawY + attachOffset + armLen * 0.62;
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
	const cw = chuteWidth.value;
	for (let i = prizes.length - 1; i >= 0; i--) {
		const p = prizes[i];
		if (p === grabbedPrize) continue;
		if (p.position.x < cw && p.position.y > H - 200) {
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
 * Draws the full claw assembly using pre-loaded PNG sprites.
 *
 * Layer order (back → front):
 *   bgCanvas  : back_grip (behind prizes)
 *   main canvas (afterRender, on top of prizes):
 *     left_grip  — rotates CCW when closing
 *     right_grip — rotates CW  when closing
 *     claw_top   — static housing, drawn last so it sits above the grips
 *
 * Called via Events.on(render, 'afterRender').
 */
const drawClaw = () => {
	if (!clawTopImg || !leftGripImg || !rightGripImg || !backGripImg) return;

	const ctx   = render.context;
	const bgCtx = bgCanvas.value?.getContext('2d');
	if (!bgCtx) return;

	const { clawRenderScale: s } = getDims();
	const cx = clawX;
	const cy = clawY;

	// World-space attachment positions relative to claw_top pivot
	const lgX = cx + (CLAW_TOP_META.lgAttachX - CLAW_TOP_META.pivX) * s;
	const lgY = cy + (CLAW_TOP_META.lgAttachY - CLAW_TOP_META.pivY) * s;
	const rgX = cx + (CLAW_TOP_META.rgAttachX - CLAW_TOP_META.pivX) * s;
	const rgY = cy + (CLAW_TOP_META.rgAttachY - CLAW_TOP_META.pivY) * s;
	const bgX = cx + (CLAW_TOP_META.bgAttachX - CLAW_TOP_META.pivX) * s;
	const bgY = cy + (CLAW_TOP_META.bgAttachY - CLAW_TOP_META.pivY) * s;

	// Grip close angle: 0 rad when fully open, CLOSE_ANGLE_DEG when closed
	const closeAngle = (1 - clawOpenAmount) * CLOSE_ANGLE_DEG * (Math.PI / 180);

	// ── Cable ──
	ctx.save();
	ctx.strokeStyle = '#555';
	ctx.lineWidth   = Math.max(2, 4 * s);
	ctx.beginPath();
	ctx.moveTo(cx, 0);
	ctx.lineTo(cx, cy - CLAW_TOP_META.pivY * s);
	ctx.stroke();
	ctx.restore();

	// ── back_grip on background canvas (renders behind prizes) ──
	bgCtx.clearRect(0, 0, bgCtx.canvas.width, bgCtx.canvas.height);
	bgCtx.save();
	bgCtx.translate(bgX, bgY);
	bgCtx.drawImage(
		backGripImg,
		-BACK_GRIP_META.pivX * s, -BACK_GRIP_META.pivY * s,
		BACK_GRIP_META.w * s,      BACK_GRIP_META.h * s,
	);
	bgCtx.restore();

	// ── left_grip (CCW rotation) ──
	ctx.save();
	ctx.translate(lgX, lgY);
	ctx.rotate(-closeAngle);
	ctx.drawImage(
		leftGripImg,
		-LEFT_GRIP_META.pivX * s, -LEFT_GRIP_META.pivY * s,
		LEFT_GRIP_META.w * s,      LEFT_GRIP_META.h * s,
	);
	ctx.restore();

	// ── right_grip (CW rotation) ──
	ctx.save();
	ctx.translate(rgX, rgY);
	ctx.rotate(closeAngle);
	ctx.drawImage(
		rightGripImg,
		-RIGHT_GRIP_META.pivX * s, -RIGHT_GRIP_META.pivY * s,
		RIGHT_GRIP_META.w * s,      RIGHT_GRIP_META.h * s,
	);
	ctx.restore();

	// ── claw_top housing (drawn on top of grips) ──
	ctx.save();
	ctx.translate(cx, cy);
	ctx.drawImage(
		clawTopImg,
		-CLAW_TOP_META.pivX * s, -CLAW_TOP_META.pivY * s,
		CLAW_TOP_META.w * s,      CLAW_TOP_META.h * s,
	);
	ctx.restore();
};

// ─── Sensor & grab ────────────────────────────────────────────────────────────

/**
 * Detects prizes inside the claw opening using a hybrid test:
 *
 *  • Vertical   — uses the prize CENTER. The claw must descend until the
 *                 prize center is within the opening band (sensorYOff to
 *                 armLen below the grip attach point).
 *  • Horizontal — uses the prize BOUNDS so large or off-center prizes are
 *                 still detected even if their center is displaced.
 *
 * @returns {Matter.Body|null} Closest qualifying prize, or null.
 */
const checkSensor = () => {
	const { attachOffset, armLen, maxSpread, sensorYOff } = getDims();
	const pivotY     = clawY + attachOffset;
	const openingTop = pivotY + sensorYOff;
	const openingBot = pivotY + armLen + 10;

	let closest     = null;
	let closestDist = Infinity;

	for (const p of prizes) {
		if (p === grabbedPrize) continue;

		const cy = p.position.y;
		if (cy < openingTop || cy > openingBot) continue;

		const b = p.bounds;
		if (b.max.x < clawX - maxSpread * 1.1 || b.min.x > clawX + maxSpread * 1.1) continue;

		const dx   = p.position.x - clawX;
		const dy   = cy - (openingTop + openingBot) / 2;
		const dist = Math.sqrt(dx * dx + dy * dy);
		if (dist < closestDist) {
			closestDist = dist;
			closest     = p;
		}
	}

	return closest;
};

/**
 * Returns true when the closed claw arms are realistically on the left and
 * right sides of the prize.
 *
 * @param {Matter.Body} prize
 * @returns {boolean}
 */
const clawCanGrab = (prize) => {
	const { attachOffset, armLen, grabHalfW } = getDims();
	const px      = prize.position.x;
	const py      = prize.position.y;
	const horizOk = Math.abs(px - clawX) < grabHalfW;
	const vertTop = clawY + attachOffset;
	const vertBot = clawY + attachOffset + armLen + 20;
	return horizOk && py > vertTop && py < vertBot;
};

/**
 * Moves a prize into fake-physics grab mode. Ghosts its collision layer so it
 * cannot interact with other bodies while held.
 *
 * If the slip chance roll succeeds, a slip is begun immediately: the prize
 * will slowly slide out of the claw over [slipMinTime, slipMaxTime] seconds.
 *
 * @param {Matter.Body} prize
 */
const grabPrize = (prize) => {
	const { attachOffset, armLen } = getDims();
	grabbedPrize = prize;
	const holdY  = clawY + attachOffset + armLen * 0.62;
	fakeRelX     = prize.position.x - clawX;
	fakeRelY     = prize.position.y - holdY;
	fakeVelX     = 0;
	fakeVelY     = 0;
	isSlipping   = false;
	Body.set(prize, { collisionFilter: { category: 0, mask: 0 } });

	// Roll for slip
	if (Math.random() * 100 < slipChance.value) {
		const minT = Math.min(slipMinTime.value, slipMaxTime.value);
		const maxT = Math.max(slipMinTime.value, slipMaxTime.value);
		slipDuration  = minT + Math.random() * (maxT - minT);
		isSlipping    = true;
		slipStartTime = performance.now();
	}
};

/**
 * Returns the held prize to the normal physics layer, restoring collisions
 * and giving it an exit velocity based on its fake-physics momentum.
 */
const releasePrize = () => {
	if (!grabbedPrize) return;
	isSlipping = false;
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

	if (grabbedPrize) releasePrize();

	const W      = window.innerWidth;
	const H      = window.innerHeight;
	const topY   = 100;
	const floorY = H - 110;
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
	await moveClaw(chuteWidth.value / 2, topY, 6);

	// 8. Open and release
	message.value = 'RELEASE!';
	await animateClawTo(1.0);

	if (grabbedPrize) releasePrize();

	await new Promise(r => setTimeout(r, 900));
	isDropping.value = false;
	if (message.value !== 'WINNER!') message.value = 'Ready!';
};

// ─── Prize spawning ───────────────────────────────────────────────────────────

/**
 * Removes all existing prizes, rebuilds the chute wall for the current scale,
 * and spawns a fresh set above the viewport. Prizes are kept to the right of
 * the chute so they don't block the win area on spawn.
 */
const spawnPrizes = async () => {
	const W = window.innerWidth;
	prizes.forEach(p => Composite.remove(world, p));
	prizes = [];
	rebuildChute();
	const spawnMin = chuteWidth.value + 50;
	const spawnMax = W - spawnMin;
	for (let i = 0; i < 18; i++) {
		const file = prizeFiles[i % prizeFiles.length];
		const x    = spawnMin + Math.random() * Math.max(0, spawnMax - spawnMin);
		await createPrize(`/prizes/${file}`, x, -100 - (i * 120));
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
	if (bgCanvas.value) {
		bgCanvas.value.width  = window.innerWidth;
		bgCanvas.value.height = window.innerHeight;
	}
};

const handleKeyDown = (e) => {
	if (e.key === '`' || e.key === '~') showSettings.value = !showSettings.value;
};

onMounted(async () => {
	await loadImages();
	initPhysics();
});

onUnmounted(() => {
	window.removeEventListener('resize', handleResize);
	window.removeEventListener('keydown', handleKeyDown);
	if (runner) Runner.stop(runner);
	if (render) Render.stop(render);
});
</script>

<template>
  <div class="game-wrapper">
    <!-- Background canvas: renders back_grip behind the prizes -->
    <canvas ref="bgCanvas" class="bg-canvas"></canvas>
    <!-- Matter.js canvas: prizes + foreground claw sprites -->
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
        <label>PRIZE SCALE: {{ prizeScale }}</label>
        <input type="range" v-model.number="prizeScale" min="0.2" max="1.0" step="0.05" />
      </div>
      <div class="field">
        <label>SLIP CHANCE: {{ slipChance }}%</label>
        <input type="range" v-model.number="slipChance" min="0" max="100" step="5" />
      </div>
      <div class="field">
        <label>SLIP MIN TIME: {{ slipMinTime }}s</label>
        <input type="range" v-model.number="slipMinTime" min="1" max="10" step="0.5" />
      </div>
      <div class="field">
        <label>SLIP MAX TIME: {{ slipMaxTime }}s</label>
        <input type="range" v-model.number="slipMaxTime" min="1" max="15" step="0.5" />
      </div>
      <button @click="spawnPrizes" class="btn-cyan">RE-SPAWN ALL</button>
      <p class="small">Press ~ to exit</p>
    </div>

    <div class="shoot" :style="{ width: chuteWidth + 'px' }">
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

/* Both canvases share base positioning */
canvas {
  position: absolute; top: 0; left: 0; width: 100%; height: 100%;
}

/* bg-canvas sits behind Matter.js prizes */
.bg-canvas { z-index: 1; pointer-events: none; }

/* Matter.js canvas (main) sits in front of bg-canvas */
canvas:not(.bg-canvas) { z-index: 2; }

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

.field { margin: 1.5rem 0; display: flex; flex-direction: column; gap: 0.75rem; }
.small { font-size: 0.8rem; opacity: 0.5; margin-top: 2rem; }

.shoot {
  position: absolute; bottom: 0; left: 0; height: 300px;
  background: rgba(34, 211, 238, 0.05); border-right: 2px solid rgba(34, 211, 238, 0.3);
  z-index: 5; display: flex; align-items: center; justify-content: center;
  transition: width 0.3s ease;
}

.shoot-text {
  color: rgba(34, 211, 238, 0.2); transform: rotate(-90deg);
  font-weight: bold; font-size: 2rem; letter-spacing: 8px;
}
</style>
