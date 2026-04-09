<script setup>
import { ref, computed, nextTick, onMounted, onUnmounted } from 'vue';
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
/** back_grip.png: 65×305, pivot at (31,14) — no rotation, shifts vertically */
const BACK_GRIP_META  = { w: 65,  h: 305, pivX: 31,  pivY: 14 };

/** Native grip length in pixels (pivot-to-tip) */
const GRIP_LENGTH_NATIVE   = 332;
/** Native distance from claw_top pivot down to grip attach point */
const GRIP_ATTACH_NATIVE_Y = 212;
/** Max closing angle for left/right grips (degrees) */
const CLOSE_ANGLE_DEG = 20;
/** How far (px) back_grip shifts down when fully closed */
const BACK_GRIP_CLOSE_OFFSET = 20;

// ─── Vue refs ─────────────────────────────────────────────────────────────────
const canvas        = ref(null);
const bgCanvas      = ref(null);
const bgVideo       = ref(null);
const cmdInputEl    = ref(null);   // for auto-focus after drop
const nameInputEl   = ref(null);   // for auto-focus on name modal open

const targetX       = ref(50);     // internal — set by !drop command
const isDropping    = ref(false);
const showSettings  = ref(false);
const prizeScale    = ref(0.9);
const message       = ref('Ready!');
const wonPrizesCount = ref(0);

// Slip mechanic
const slipChance  = ref(50);
const slipMinTime = ref(1.5);
const slipMaxTime = ref(3);

// UI / dev options
const cmdInput     = ref('');
const showHUD      = ref(false);
const pushOnMiss   = ref(true);
const pushOnGrab   = ref(true);
const pushStrength = ref(100);

// Player name
const playerName      = ref('');
const playerNameInput = ref('');
const showNameModal   = ref(true);

const prizeFiles = ['bear.png', 'bunny.png', 'cat.png', 'doggo.png', 'pengy.png', 'sheep.png'];

// ─── Command history (up/down like a shell) ───────────────────────────────────
const cmdHistory = [];
let historyIndex = -1;

// ─── Matter.js globals ────────────────────────────────────────────────────────
let engine, render, runner, world;
let prizes = [];

// ─── Collision categories ─────────────────────────────────────────────────────
const PRIZE_CATEGORY = 0x0004;
const WALL_CATEGORY  = 0x0001;

// ─── Claw base constants (unscaled at prizeScale = 0.45) ─────────────────────
const ARM_LENGTH_BASE      = 100;
const MAX_SPREAD_BASE      = 65;
const SENSOR_Y_OFFSET_BASE = 42;
const INTERIOR_HALF_W_BASE = 18;
const INTERIOR_BOT_BASE    = 48;
const OPEN_ANIM_MS         = 420;

// ─── Dynamic chute width ──────────────────────────────────────────────────────
const chuteWidth = computed(() => {
	const cs = Math.max(0.8, prizeScale.value / 0.45);
	return Math.round(180 * cs);
});

/**
 * Returns all scaled claw dimensions for the current prizeScale.
 * @returns {{ cs, armLen, maxSpread, sensorYOff, grabHalfW, intHalfW, intBot, clawRenderScale, attachOffset }}
 */
const getDims = () => {
	const cs              = Math.max(0.8, prizeScale.value / 0.45);
	const armLen          = ARM_LENGTH_BASE * cs;
	const clawRenderScale = armLen / GRIP_LENGTH_NATIVE;
	const attachOffset    = GRIP_ATTACH_NATIVE_Y * clawRenderScale;
	const maxSpread       = MAX_SPREAD_BASE * cs;
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
let clawOpenAmount = 1.0;

let openAnimActive = false;
let openAnimStart  = 0;
let openAnimFrom   = 1.0;
let openAnimTo     = 0.0;

// ─── Fake-physics state ───────────────────────────────────────────────────────
let grabbedPrize = null;
let fakeRelX     = 0;
let fakeRelY     = 0;
let fakeVelX     = 0;
let fakeVelY     = 0;

// ─── Slip state ───────────────────────────────────────────────────────────────
let isSlipping    = false;
let slipStartTime = 0;
let slipDuration  = 0;

// ─── Loaded sprite images ─────────────────────────────────────────────────────
let clawTopImg   = null;
let leftGripImg  = null;
let rightGripImg = null;
let backGripImg  = null;

// ─── Chute physics wall ───────────────────────────────────────────────────────
let chuteWall = null;

// ─── Canvas drawing helpers ───────────────────────────────────────────────────

/**
 * Draws a rounded rectangle path onto ctx (no fill/stroke, caller does that).
 *
 * @param {CanvasRenderingContext2D} ctx
 * @param {number} x
 * @param {number} y
 * @param {number} w
 * @param {number} h
 * @param {number} r - corner radius
 */
const roundRectPath = (ctx, x, y, w, h, r) => {
	ctx.beginPath();
	ctx.moveTo(x + r, y);
	ctx.lineTo(x + w - r, y);
	ctx.arcTo(x + w, y,     x + w, y + r,     r);
	ctx.lineTo(x + w, y + h - r);
	ctx.arcTo(x + w, y + h, x + w - r, y + h, r);
	ctx.lineTo(x + r, y + h);
	ctx.arcTo(x,      y + h, x,      y + h - r, r);
	ctx.lineTo(x, y + r);
	ctx.arcTo(x,      y,     x + r,  y,         r);
	ctx.closePath();
};

// ─── Image loading ────────────────────────────────────────────────────────────

/**
 * Preloads all four claw sprite images.
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
 * Creates the physics engine, renderer, world boundaries, and starts the game.
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
 * Removes the old chute divider wall and recreates it at the current chuteWidth.
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
 * Simulates the held prize swinging in claw-local space (spring + gravity + damping).
 * During a slip the spring weakens and floor expands until the prize falls free.
 */
const tickFakePhysics = () => {
	const GRAVITY       = 0.30;
	const DAMPING       = 0.88;
	const BASE_SPRING_K = 0.038;
	const { cs, attachOffset, armLen, intHalfW, intBot } = getDims();

	let springK      = BASE_SPRING_K;
	let effectiveBot = intBot;

	if (isSlipping) {
		const elapsed  = (performance.now() - slipStartTime) / 1000;
		const progress = Math.min(elapsed / slipDuration, 1.0);
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

const checkWin = () => {
	const H  = window.innerHeight;
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
 * Draws the full claw assembly using PNG sprites.
 *
 * Layers (back → front):
 *   bgCanvas  — back_grip (behind prizes); shifts down as claw closes
 *   main canvas (afterRender):
 *     left_grip  (CCW rotation when closing)
 *     right_grip (CW  rotation when closing)
 *     claw_top   (housing, on top of grips)
 *     player name pill (centered in claw_top housing area)
 */
const drawClaw = () => {
	if (!clawTopImg || !leftGripImg || !rightGripImg || !backGripImg) return;

	const ctx   = render.context;
	const bgCtx = bgCanvas.value?.getContext('2d');
	if (!bgCtx) return;

	const { clawRenderScale: s } = getDims();
	const cx = clawX;
	const cy = clawY;

	const lgX = cx + (CLAW_TOP_META.lgAttachX - CLAW_TOP_META.pivX) * s;
	const lgY = cy + (CLAW_TOP_META.lgAttachY - CLAW_TOP_META.pivY) * s;
	const rgX = cx + (CLAW_TOP_META.rgAttachX - CLAW_TOP_META.pivX) * s;
	const rgY = cy + (CLAW_TOP_META.rgAttachY - CLAW_TOP_META.pivY) * s;
	const bgX = cx + (CLAW_TOP_META.bgAttachX - CLAW_TOP_META.pivX) * s;
	const bgY = cy + (CLAW_TOP_META.bgAttachY - CLAW_TOP_META.pivY) * s;

	const closeAngle      = (1 - clawOpenAmount) * CLOSE_ANGLE_DEG * (Math.PI / 180);
	const backGripOffsetY = (1 - clawOpenAmount) * BACK_GRIP_CLOSE_OFFSET;

	// ── Cable ──
	ctx.save();
	ctx.strokeStyle = '#555';
	ctx.lineWidth   = Math.max(6, 12 * s);
	ctx.beginPath();
	ctx.moveTo(cx, 0);
	ctx.lineTo(cx, cy - CLAW_TOP_META.pivY * s);
	ctx.stroke();
	ctx.restore();

	// ── back_grip on background canvas (behind prizes) ──
	bgCtx.clearRect(0, 0, bgCtx.canvas.width, bgCtx.canvas.height);
	bgCtx.save();
	bgCtx.translate(bgX, bgY + backGripOffsetY);
	bgCtx.drawImage(
		backGripImg,
		-BACK_GRIP_META.pivX * s, -BACK_GRIP_META.pivY * s,
		BACK_GRIP_META.w * s,      BACK_GRIP_META.h * s,
	);
	bgCtx.restore();

	// ── left_grip (CCW) ──
	ctx.save();
	ctx.translate(lgX, lgY);
	ctx.rotate(-closeAngle);
	ctx.drawImage(
		leftGripImg,
		-LEFT_GRIP_META.pivX * s, -LEFT_GRIP_META.pivY * s,
		LEFT_GRIP_META.w * s,      LEFT_GRIP_META.h * s,
	);
	ctx.restore();

	// ── right_grip (CW) ──
	ctx.save();
	ctx.translate(rgX, rgY);
	ctx.rotate(closeAngle);
	ctx.drawImage(
		rightGripImg,
		-RIGHT_GRIP_META.pivX * s, -RIGHT_GRIP_META.pivY * s,
		RIGHT_GRIP_META.w * s,      RIGHT_GRIP_META.h * s,
	);
	ctx.restore();

	// ── claw_top housing (on top of grips) ──
	ctx.save();
	ctx.translate(cx, cy);
	ctx.drawImage(
		clawTopImg,
		-CLAW_TOP_META.pivX * s, -CLAW_TOP_META.pivY * s,
		CLAW_TOP_META.w * s,      CLAW_TOP_META.h * s,
	);
	ctx.restore();

	// ── Player name pill (on top of claw_top, in the housing area) ──
	if (playerName.value) {
		// Vertical center of the claw_top housing body:
		// image spans cy - pivY*s → cy + (lgAttachY - pivY)*s
		// center = cy + ((lgAttachY/2) - pivY)*s
		const labelY  = cy + (CLAW_TOP_META.lgAttachY / 2 - CLAW_TOP_META.pivY) * s;
		const fontSize = Math.max(11, Math.round(10 + s * 14));

		ctx.save();
		ctx.font         = `bold ${fontSize}px Rajdhani, sans-serif`;
		ctx.textAlign    = 'center';
		ctx.textBaseline = 'middle';

		const textW  = ctx.measureText(playerName.value).width;
		const padX   = Math.max(6, 8 * s);
		const padY   = Math.max(3, 4 * s);
		const pillW  = textW + padX * 2;
		const pillH  = fontSize + padY * 2;
		const pillR  = pillH / 2;
		const pillX  = cx - pillW / 2;
		const pillY  = labelY - pillH / 2;

		// Pill background
		ctx.fillStyle = 'rgba(0, 0, 0, 0.60)';
		roundRectPath(ctx, pillX, pillY, pillW, pillH, pillR);
		ctx.fill();

		// White text with hard drop shadow
		ctx.shadowColor   = '#000';
		ctx.shadowBlur    = 0;
		ctx.shadowOffsetX = 1.5;
		ctx.shadowOffsetY = 1.5;
		ctx.fillStyle     = '#ffffff';
		ctx.fillText(playerName.value, cx, labelY);

		ctx.shadowColor   = 'transparent';
		ctx.shadowOffsetX = 0;
		ctx.shadowOffsetY = 0;
		ctx.restore();
	}
};

// ─── Sensor & grab ────────────────────────────────────────────────────────────

/**
 * Detects prizes inside the claw opening using a hybrid test:
 *   vertical = prize CENTER within opening band
 *   horizontal = prize BOUNDS within arm span
 *
 * @returns {Matter.Body|null}
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
 * Returns true when the closed arms are realistically around the prize.
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
 * Moves a prize into fake-physics grab mode and optionally starts a slip.
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

	if (Math.random() * 100 < slipChance.value) {
		const minT    = Math.min(slipMinTime.value, slipMaxTime.value);
		const maxT    = Math.max(slipMinTime.value, slipMaxTime.value);
		slipDuration  = minT + Math.random() * (maxT - minT);
		isSlipping    = true;
		slipStartTime = performance.now();
	}
};

/**
 * Returns the held prize to real physics with exit velocity.
 */
const releasePrize = () => {
	if (!grabbedPrize) return;
	isSlipping = false;
	Body.set(grabbedPrize, {
		collisionFilter: { category: PRIZE_CATEGORY, mask: WALL_CATEGORY | PRIZE_CATEGORY },
	});
	Body.setVelocity(grabbedPrize, { x: fakeVelX * 2.5, y: fakeVelY * 2.5 });
	// Gentle random tumble as the prize falls — biased toward the direction it was moving
	Body.setAngularVelocity(grabbedPrize, (Math.random() - 0.5) * 0.2);
	grabbedPrize = null;
};

/**
 * Applies a radial velocity nudge to all free prizes within `radius` px.
 * Simulates the claw disturbing the pile on grabs and misses.
 *
 * @param {number} cx
 * @param {number} cy
 * @param {number} radius
 * @param {number} strength - velocity magnitude applied at dead-center
 */
const pushNearbyPrizes = (cx, cy, radius, strength) => {
	for (const p of prizes) {
		if (p === grabbedPrize) continue;
		const dx   = p.position.x - cx;
		const dy   = p.position.y - cy;
		const dist = Math.sqrt(dx * dx + dy * dy);
		if (dist < radius && dist > 0) {
			const mag = strength * (1 - dist / radius);
			Body.setVelocity(p, {
				x: p.velocity.x + (dx / dist) * mag,
				y: p.velocity.y + (dy / dist) * mag,
			});
		}
	}
};

// ─── Movement helpers ─────────────────────────────────────────────────────────

/**
 * Smoothly moves the claw to (tx, ty) at the given speed in px/frame.
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
 * Animates the claw open/close to the given amount.
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
 * Full drop cycle:
 *   open → position → descend → close → grab/push → ascend → deliver → release
 * Auto-focuses the command input when complete.
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

	await animateClawTo(1.0);

	message.value = 'LOCKING ON...';
	await moveClaw(destX, topY, 8);

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

	message.value = 'GRAB!';
	await animateClawTo(0.0);
	await new Promise(r => setTimeout(r, 130));

	// Grab check + pile disturbance
	const { cs, attachOffset } = getDims();
	const pushCx = clawX;
	const pushCy = clawY + attachOffset;
	const pushR  = 130 * cs;

	if (detectedPrize && clawCanGrab(detectedPrize)) {
		grabPrize(detectedPrize);
		if (pushOnGrab.value) pushNearbyPrizes(pushCx, pushCy, pushR, pushStrength.value);
	} else {
		if (pushOnMiss.value) pushNearbyPrizes(pushCx, pushCy, pushR, pushStrength.value);
	}

	message.value = 'ASCENDING...';
	await moveClaw(clawX, topY, 4);

	message.value = 'DELIVERING...';
	await moveClaw(chuteWidth.value / 2, topY, 6);

	message.value = 'RELEASE!';
	await animateClawTo(1.0);

	if (grabbedPrize) releasePrize();

	await new Promise(r => setTimeout(r, 900));
	isDropping.value = false;
	if (message.value !== 'WINNER!') message.value = 'Ready!';

	// Auto-focus command input for the next round
	await nextTick();
	cmdInputEl.value?.focus();
};

// ─── Command parsing ──────────────────────────────────────────────────────────

/**
 * Parses and executes a chat-style command from the input field.
 * Recognized format: !drop <0-100>
 */
const handleCommand = () => {
	const val = cmdInput.value.trim();
	if (!val) return;

	if (cmdHistory[0] !== val) cmdHistory.unshift(val);
	if (cmdHistory.length > 50) cmdHistory.pop();
	historyIndex = -1;

	const match = val.match(/^!drop\s+(\d+(?:\.\d+)?)$/i);
	if (match) {
		targetX.value  = Math.max(0, Math.min(100, parseFloat(match[1])));
		cmdInput.value = '';
		dropClaw();
	} else {
		cmdInput.value = '';
	}
};

/**
 * Keyboard handler for the command input.
 * Enter submits; Up/Down cycle history like a shell.
 *
 * @param {KeyboardEvent} e
 */
const handleCmdKeyDown = (e) => {
	if (e.key === 'Enter') {
		handleCommand();
	} else if (e.key === 'ArrowUp') {
		e.preventDefault();
		if (historyIndex < cmdHistory.length - 1) {
			historyIndex++;
			cmdInput.value = cmdHistory[historyIndex];
		}
	} else if (e.key === 'ArrowDown') {
		e.preventDefault();
		if (historyIndex > 0) {
			historyIndex--;
			cmdInput.value = cmdHistory[historyIndex];
		} else if (historyIndex === 0) {
			historyIndex   = -1;
			cmdInput.value = '';
		}
	}
};

// ─── Name modal ───────────────────────────────────────────────────────────────

/**
 * Confirms the entered player name and closes the modal.
 * Focuses the command input after the modal is dismissed.
 */
const confirmName = async () => {
	const name = playerNameInput.value.trim();
	if (!name) return;
	playerName.value    = name;
	showNameModal.value = false;
	await nextTick();
	cmdInputEl.value?.focus();
};

// ─── Background video ─────────────────────────────────────────────────────────

/**
 * On first user interaction, unmute the background video.
 * Autoplay requires the video start muted; this lifts that restriction.
 */
const handleFirstInteraction = () => {
	if (bgVideo.value) {
		bgVideo.value.muted  = false;
		bgVideo.value.volume = 0.3;
	}
	document.removeEventListener('click', handleFirstInteraction);
	document.removeEventListener('keydown', handleFirstInteraction);
};

// ─── Prize spawning ───────────────────────────────────────────────────────────

/**
 * Removes all prizes, rebuilds the chute, spawns a fresh set.
 * Spawn range keeps prizes out of the chute area.
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
 * Loads a prize image, builds a convex-hull physics body, adds to world.
 *
 * @param {string} url
 * @param {number} x
 * @param {number} y
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

/**
 * Toggles the dev menu on tilde/backtick.
 * Skips when a text input is focused to avoid interfering with typing.
 *
 * @param {KeyboardEvent} e
 */
const handleKeyDown = (e) => {
	if (e.key === '`' || e.key === '~') {
		if (document.activeElement?.tagName === 'INPUT') return;
		showSettings.value = !showSettings.value;
	}
};

onMounted(async () => {
	// Register first-interaction handler for video audio
	document.addEventListener('click',   handleFirstInteraction);
	document.addEventListener('keydown', handleFirstInteraction);

	// Start the background video (muted for autoplay compliance)
	bgVideo.value?.play();

	// Auto-focus the name input while images and physics load in parallel
	await nextTick();
	nameInputEl.value?.focus();

	await loadImages();
	initPhysics();
});

onUnmounted(() => {
	window.removeEventListener('resize',  handleResize);
	window.removeEventListener('keydown', handleKeyDown);
	document.removeEventListener('click',   handleFirstInteraction);
	document.removeEventListener('keydown', handleFirstInteraction);
	if (runner) Runner.stop(runner);
	if (render) Render.stop(render);
});
</script>

<template>
  <div class="game-wrapper">

    <!-- ── Layer 0: background video ── -->
    <video ref="bgVideo" class="bg-video" loop playsinline muted>
      <source src="/bg/bg.mp4" type="video/mp4" />
    </video>

    <!-- ── Layer 1: back_grip (behind prizes) ── -->
    <canvas ref="bgCanvas" class="bg-canvas"></canvas>

    <!-- ── Layer 2: Matter.js prizes + foreground claw sprites ── -->
    <canvas ref="canvas"></canvas>

    <!-- ── Layer 3: UI overlay ── -->
    <div class="ui-overlay">
      <!-- Title + status: hidden by default, toggled by SHOW STATUS HUD -->
      <div class="header">
        <h1 class="neon-title" v-show="showHUD">CLAW MASTER 3000</h1>
        <div class="msg-box" v-show="showHUD">{{ message }}</div>
      </div>

      <!-- Command input -->
      <div class="control-panel" :class="{ locked: isDropping }">
        <input
          ref="cmdInputEl"
          type="text"
          class="cmd-input"
          v-model="cmdInput"
          @keydown="handleCmdKeyDown"
          placeholder="!drop 50"
          :disabled="isDropping"
          autocomplete="off"
          spellcheck="false"
        />
        <div class="cmd-hint">
          Type <code>!drop &lt;number&gt;</code> with a number between 0–100 to play!
        </div>
      </div>
    </div>

    <!-- ── Layer 3: Win chute ── -->
    <div class="shoot" :style="{ width: chuteWidth + 'px' }">
      <div class="shoot-text">WIN AREA</div>
      <div class="shoot-wins">PRIZES WON: {{ wonPrizesCount }}</div>
    </div>

    <!-- ── Layer 4: Dev menu (press ~) ── -->
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
        <input type="range" v-model.number="slipMinTime" min="0.5" max="10" step="0.5" />
      </div>
      <div class="field">
        <label>SLIP MAX TIME: {{ slipMaxTime }}s</label>
        <input type="range" v-model.number="slipMaxTime" min="0.5" max="15" step="0.5" />
      </div>
      <div class="field">
        <label>PUSH FORCE: {{ pushStrength }}</label>
        <input type="range" v-model.number="pushStrength" min="1" max="100" step="1" />
      </div>
      <div class="field checkbox-field">
        <label><input type="checkbox" v-model="pushOnMiss" /> PUSH TOYS ON MISSES</label>
        <label><input type="checkbox" v-model="pushOnGrab" /> PUSH TOYS ON GRABS</label>
        <label><input type="checkbox" v-model="showHUD" /> SHOW STATUS HUD</label>
      </div>
      <button @click="spawnPrizes" class="btn-cyan">RE-SPAWN ALL</button>
      <p class="small">Press ~ to exit</p>
    </div>

    <!-- ── Layer 5: Name modal (shown on first load) ── -->
    <div v-if="showNameModal" class="modal-overlay">
      <div class="name-modal">
        <h2 class="modal-title">CLAW MASTER 3000</h2>
        <p class="modal-sub">Enter your name to play</p>
        <input
          ref="nameInputEl"
          type="text"
          class="name-input"
          v-model="playerNameInput"
          @keydown.enter="confirmName"
          placeholder="Your name"
          maxlength="20"
          autocomplete="off"
          spellcheck="false"
        />
        <button class="btn-cyan" @click="confirmName">LET'S GO!</button>
      </div>
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
  position: relative;
  background: transparent;
}

/* ── Background video ── */
.bg-video {
  position: absolute; top: 0; left: 0;
  width: 100%; height: 100%;
  object-fit: cover;
  z-index: 0;
  pointer-events: none;
}

/* ── Canvas layers ── */
canvas {
  position: absolute; top: 0; left: 0; width: 100%; height: 100%;
  pointer-events: none;
}
.bg-canvas           { z-index: 1; }
canvas:not(.bg-canvas) { z-index: 2; }

/* ── UI overlay ── */
.ui-overlay {
  position: absolute; top: 0; left: 0; width: 100%; height: 100%;
  display: flex; flex-direction: column; justify-content: space-between;
  padding: 3rem; box-sizing: border-box; pointer-events: none; z-index: 3;
}

.header { display: flex; flex-direction: column; align-items: center; }

.neon-title {
  font-size: 4rem; color: #fff; text-align: center; margin: 0;
  text-shadow: 0 0 10px #22d3ee, 0 0 20px #22d3ee; letter-spacing: 4px;
}

.msg-box {
  color: #5eead4; font-size: 2rem; text-align: center; margin-top: 20px;
  font-weight: bold; height: 3rem;
}

/* ── Command input panel ── */
.control-panel {
  pointer-events: auto; align-self: center;
  background: rgba(9, 14, 30, 0.82); padding: 1.5rem 2.5rem;
  border-radius: 24px; border: 2px solid #334155;
  display: flex; flex-direction: column; align-items: center; gap: 0.9rem;
  box-shadow: 0 20px 50px rgba(0,0,0,0.7), inset 0 0 20px rgba(34, 211, 238, 0.08);
  backdrop-filter: blur(6px);
}

.control-panel.locked { opacity: 0.4; pointer-events: none; filter: grayscale(0.5); }

.cmd-input {
  width: 380px; padding: 0.85rem 1.4rem;
  background: #0a1020; color: #22d3ee;
  border: 2px solid #334155; border-radius: 12px;
  font-family: 'Rajdhani', monospace; font-size: 1.7rem; font-weight: bold;
  outline: none; box-sizing: border-box;
  transition: border-color 0.15s, box-shadow 0.15s;
  caret-color: #22d3ee;
}

.cmd-input:focus {
  border-color: #22d3ee;
  box-shadow: 0 0 14px rgba(34, 211, 238, 0.35);
}

.cmd-input::placeholder { color: #1e3a4a; }
.cmd-input:disabled { opacity: 0.4; }

.cmd-hint { color: #475569; font-size: 1rem; text-align: center; }

.cmd-hint code {
  color: #22d3ee; background: rgba(34, 211, 238, 0.1);
  padding: 0.1rem 0.4rem; border-radius: 4px;
}

/* ── Win chute ── */
.shoot {
  position: absolute; bottom: 0; left: 0; height: 300px; z-index: 3;
  background: rgba(34, 211, 238, 0.05);
  border-right: 2px solid rgba(34, 211, 238, 0.3);
  display: flex; flex-direction: column;
  align-items: center; justify-content: flex-end;
  padding-bottom: 1.2rem; box-sizing: border-box;
  transition: width 0.3s ease;
}

.shoot-text {
  position: absolute; top: 50%; left: 50%;
  transform: translate(-50%, -50%) rotate(-90deg);
  color: rgba(34, 211, 238, 0.15);
  font-weight: bold; font-size: 1.4rem; letter-spacing: 6px;
  white-space: nowrap; pointer-events: none;
}

.shoot-wins {
  color: #22d3ee; font-size: 1.1rem; font-weight: bold;
  letter-spacing: 1px;
  text-shadow: 0 0 10px rgba(34, 211, 238, 0.6);
  position: relative; z-index: 1;
}

/* ── Dev menu ── */
.dev-menu {
  position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
  background: rgba(10, 16, 32, 0.96); color: #22d3ee; padding: 3rem;
  border-radius: 24px; z-index: 20; border: 2px solid #db2777;
  min-width: 430px; pointer-events: auto; text-align: center;
  backdrop-filter: blur(8px);
}

.btn-cyan {
  width: 100%; padding: 1.2rem; background: #22d3ee; color: #0f172a;
  border: none; border-radius: 12px; font-weight: bold; cursor: pointer;
  font-family: inherit; font-size: 1.2rem; margin-top: 1rem;
  transition: opacity 0.15s;
}

.btn-cyan:hover { opacity: 0.85; }

.field { margin: 1.4rem 0; display: flex; flex-direction: column; gap: 0.7rem; }
.small { font-size: 0.8rem; opacity: 0.5; margin-top: 2rem; }

.checkbox-field { align-items: flex-start; text-align: left; gap: 0.65rem; padding: 0 0.3rem; }

.checkbox-field label {
  display: flex; align-items: center; gap: 0.75rem;
  cursor: pointer; font-size: 1.1rem;
}

.checkbox-field input[type="checkbox"] {
  width: 18px; height: 18px; cursor: pointer; accent-color: #22d3ee; flex-shrink: 0;
}

/* ── Name modal ── */
.modal-overlay {
  position: absolute; inset: 0; z-index: 50;
  display: flex; align-items: center; justify-content: center;
  background: rgba(0, 0, 0, 0.65);
  backdrop-filter: blur(4px);
}

.name-modal {
  background: rgba(10, 16, 32, 0.97); border: 2px solid #22d3ee;
  border-radius: 28px; padding: 3.5rem 4rem;
  display: flex; flex-direction: column; align-items: center; gap: 1.4rem;
  min-width: 400px;
  box-shadow: 0 0 60px rgba(34, 211, 238, 0.2), 0 30px 60px rgba(0,0,0,0.6);
}

.modal-title {
  font-size: 2.4rem; color: #fff; margin: 0;
  text-shadow: 0 0 10px #22d3ee, 0 0 22px #22d3ee; letter-spacing: 3px;
}

.modal-sub { color: #94a3b8; font-size: 1.2rem; margin: 0; }

.name-input {
  width: 100%; padding: 0.85rem 1.4rem;
  background: #0a1020; color: #fff;
  border: 2px solid #334155; border-radius: 12px;
  font-family: 'Rajdhani', sans-serif; font-size: 1.6rem; font-weight: bold;
  outline: none; box-sizing: border-box;
  transition: border-color 0.15s, box-shadow 0.15s;
  text-align: center; caret-color: #22d3ee;
}

.name-input:focus {
  border-color: #22d3ee;
  box-shadow: 0 0 14px rgba(34, 211, 238, 0.35);
}

.name-input::placeholder { color: #334155; }
</style>
