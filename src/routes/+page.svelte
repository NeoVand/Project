<script lang="ts">
	import { onMount } from 'svelte';
	import { scale } from 'svelte/transition';
	import { backOut } from 'svelte/easing';
	import Matter from 'matter-js';

	let isStarted = $state(false);
	let buttonElements: ButtonData[] = $state([]);
	let buttonIdCounter = 0;
	
	// Store engine reference outside of $state (Matter objects aren't meant to be reactive)
	let engine: Matter.Engine;
	let runner: Matter.Runner;
	let buttonBodies: Map<number, Matter.Body> = new Map();

	type ButtonData = {
		id: number;
		style: ButtonStyle;
		x: number;
		y: number;
		angle: number;
		width: number;
		height: number;
	};

	type ButtonStyle = {
		background: string;
		color: string;
		border: string;
		borderRadius: string;
		boxShadow: string;
		fontFamily: string;
		fontSize: string;
		fontWeight: string;
		textTransform: string;
		filter: string;
	};

	// ============================================
	// 🎨 BUTTON STYLE GENERATORS
	// ============================================
	
	const styleGenerators: (() => ButtonStyle)[] = [
		// Windows 95
		() => ({ background: 'linear-gradient(180deg, #dfdfdf 0%, #c0c0c0 100%)', color: '#000', border: '2px outset #fff', borderRadius: '0', boxShadow: 'inset -1px -1px 0 #808080, inset 1px 1px 0 #fff', fontFamily: 'Tahoma, sans-serif', fontSize: '12px', fontWeight: '400', textTransform: 'none', filter: 'none' }),
		// macOS Aqua
		() => ({ background: 'linear-gradient(180deg, #7cb8ff 0%, #2e7ad1 50%, #1a5fb4 100%)', color: '#fff', border: '1px solid #1a4d8c', borderRadius: '8px', boxShadow: '0 1px 3px rgba(0,0,0,0.3)', fontFamily: 'system-ui', fontSize: '13px', fontWeight: '500', textTransform: 'none', filter: 'none' }),
		// Rainbow
		() => ({ background: `linear-gradient(${Math.random()*360}deg, hsl(${Math.random()*360},100%,50%), hsl(${Math.random()*360},100%,50%))`, color: Math.random()>0.5?'#000':'#fff', border: `3px solid hsl(${Math.random()*360},100%,30%)`, borderRadius: `${Math.random()*30}px`, boxShadow: `0 0 15px hsl(${Math.random()*360},100%,50%)`, fontFamily: 'Impact', fontSize: `${11+Math.random()*6}px`, fontWeight: '900', textTransform: Math.random()>0.5?'uppercase':'none', filter: 'none' }),
		// Brutalist
		() => ({ background: '#000', color: '#fff', border: '3px solid #fff', borderRadius: '0', boxShadow: '6px 6px 0 rgba(255,255,255,0.3)', fontFamily: 'monospace', fontSize: '13px', fontWeight: '700', textTransform: 'uppercase', filter: 'none' }),
		// Neumorphic
		() => ({ background: '#e0e5ec', color: '#566573', border: 'none', borderRadius: '12px', boxShadow: '6px 6px 12px #b8bec7, -6px -6px 12px #fff', fontFamily: 'system-ui', fontSize: '13px', fontWeight: '600', textTransform: 'none', filter: 'none' }),
		// Hacker
		() => ({ background: '#0a0a0a', color: '#0f0', border: '1px solid #0f0', borderRadius: '0', boxShadow: '0 0 8px #0f0', fontFamily: 'monospace', fontSize: '11px', fontWeight: '400', textTransform: 'none', filter: 'none' }),
		// Kawaii
		() => ({ background: 'linear-gradient(135deg, #ff9a9e, #fecfef)', color: '#ff6b9d', border: '2px solid #ffb6c1', borderRadius: '25px', boxShadow: '0 4px 15px rgba(255,107,157,0.3)', fontFamily: 'cursive', fontSize: '12px', fontWeight: '700', textTransform: 'none', filter: 'none' }),
		// Neon
		() => { const c = ['#f0f','#0ff','#ff0080','#80ff00','#ff8000'][Math.floor(Math.random()*5)]; return { background: '#111', color: c, border: `2px solid ${c}`, borderRadius: '4px', boxShadow: `0 0 10px ${c}, 0 0 20px ${c}`, fontFamily: 'sans-serif', fontSize: '11px', fontWeight: '700', textTransform: 'uppercase', filter: 'none' }; },
		// Vintage
		() => ({ background: '#f4e4bc', color: '#2c1810', border: '2px solid #8b4513', borderRadius: '2px', boxShadow: '3px 3px 0 #8b4513', fontFamily: 'monospace', fontSize: '13px', fontWeight: '400', textTransform: 'none', filter: 'sepia(20%)' }),
		// Glass
		() => ({ background: 'rgba(255,255,255,0.15)', color: '#fff', border: '1px solid rgba(255,255,255,0.25)', borderRadius: '10px', boxShadow: '0 8px 32px rgba(31,38,135,0.3)', fontFamily: 'system-ui', fontSize: '12px', fontWeight: '500', textTransform: 'none', filter: 'none' }),
		// Vegas
		() => ({ background: 'linear-gradient(45deg, gold, #ff6b35, gold)', color: '#800000', border: '3px double gold', borderRadius: '8px', boxShadow: '0 0 20px gold', fontFamily: 'serif', fontSize: '14px', fontWeight: '900', textTransform: 'uppercase', filter: 'saturate(1.5)' }),
		// Modern
		() => ({ background: 'linear-gradient(135deg, #667eea, #764ba2)', color: '#fff', border: 'none', borderRadius: '8px', boxShadow: '0 4px 15px rgba(102,126,234,0.4)', fontFamily: 'system-ui', fontSize: '13px', fontWeight: '600', textTransform: 'none', filter: 'none' }),
		// Clown
		() => ({ background: 'repeating-linear-gradient(45deg, #f00, #f00 8px, #ff0 8px, #ff0 16px)', color: '#00f', border: '3px dashed #0f0', borderRadius: `${Math.random()*30}px`, boxShadow: '4px 4px 0 #f0f, -4px -4px 0 #0ff', fontFamily: 'cursive', fontSize: `${12+Math.random()*8}px`, fontWeight: '900', textTransform: 'uppercase', filter: `hue-rotate(${Math.random()*360}deg)` }),
		// Corporate
		() => ({ background: '#f8f9fa', color: '#495057', border: '1px solid #dee2e6', borderRadius: '4px', boxShadow: '0 2px 4px rgba(0,0,0,0.1)', fontFamily: 'Arial', fontSize: '13px', fontWeight: '400', textTransform: 'none', filter: 'none' }),
		// Midnight
		() => ({ background: 'linear-gradient(180deg, #1a1a2e, #16213e)', color: '#e8d5b7', border: '1px solid #e8d5b7', borderRadius: '2px', boxShadow: '0 4px 20px rgba(232,213,183,0.1)', fontFamily: 'serif', fontSize: '14px', fontWeight: '500', textTransform: 'none', filter: 'none' }),
	];

	function randomStyle(): ButtonStyle {
		return styleGenerators[Math.floor(Math.random() * styleGenerators.length)]();
	}

	// ============================================
	// 🎮 PHYSICS
	// ============================================

	onMount(() => {
		const { Engine, Bodies, Composite, Runner: MatterRunner, Body } = Matter;

		engine = Engine.create({ gravity: { x: 0, y: 1 } });

		// Walls and floor
		const floor = Bodies.rectangle(window.innerWidth/2, window.innerHeight + 25, window.innerWidth * 2, 50, { isStatic: true, restitution: 0.5, friction: 0.8 });
		const leftWall = Bodies.rectangle(-25, window.innerHeight/2, 50, window.innerHeight * 2, { isStatic: true, restitution: 0.6 });
		const rightWall = Bodies.rectangle(window.innerWidth + 25, window.innerHeight/2, 50, window.innerHeight * 2, { isStatic: true, restitution: 0.6 });
		
		Composite.add(engine.world, [floor, leftWall, rightWall]);

		runner = MatterRunner.create();
		MatterRunner.run(runner, engine);

		// Sync loop
		function sync() {
			buttonElements = buttonElements.map(btn => {
				const body = buttonBodies.get(btn.id);
				if (body) {
					return { ...btn, x: body.position.x, y: body.position.y, angle: body.angle };
				}
				return btn;
			});
			requestAnimationFrame(sync);
		}
		sync();

		return () => {
			MatterRunner.stop(runner);
			Composite.clear(engine.world, false);
			Engine.clear(engine);
		};
	});

	function spawnButtons(count: number) {
		const { Bodies, Composite, Body } = Matter;
		
		for (let i = 0; i < count; i++) {
			const id = buttonIdCounter++;
			const style = randomStyle();
			const width = 70 + Math.random() * 40;
			const height = 28 + Math.random() * 12;
			const x = 60 + Math.random() * (window.innerWidth - 120);
			const y = -40 - Math.random() * 400 - i * 25;

			const body = Bodies.rectangle(x, y, width, height, {
				restitution: 0.3 + Math.random() * 0.4,
				friction: 0.4,
				frictionAir: 0.008,
				chamfer: { radius: 2 }
			});

			Body.setVelocity(body, { x: (Math.random() - 0.5) * 8, y: Math.random() * 2 });
			Body.setAngularVelocity(body, (Math.random() - 0.5) * 0.1);

			Composite.add(engine.world, body);
			buttonBodies.set(id, body);

			buttonElements.push({ id, style, x, y, angle: 0, width, height });
		}
		buttonElements = buttonElements; // trigger update
	}

	function handleClick() {
		if (!isStarted) {
			isStarted = true;
			spawnButtons(1); // Start with just 1 button
		} else {
			// 10× MORE BUTTONS EVERY CLICK! 🔥
			const count = buttonElements.length * 10;
			
			// Cap at 2000 to prevent browser crash (but still chaotic!)
			const cappedCount = Math.min(count, 2000);
			
			spawnButtons(cappedCount);
		}
	}
</script>

<svelte:head>
	<title>For Jeff 💀</title>
	<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;700&display=swap" rel="stylesheet">
</svelte:head>

<main>
	{#if !isStarted}
		<div class="intro">
			<p class="intro-text">Hey Jeff, this button is just for you...</p>
			<button class="starter" onclick={handleClick} transition:scale={{ duration: 300, easing: backOut }}>
				For Jeff
			</button>
			<p class="hint">👆 Click it, Jeff. You know you want to.</p>
		</div>
	{:else}
		{#each buttonElements as btn (btn.id)}
			<button
				class="btn"
				style="left:{btn.x}px; top:{btn.y}px; width:{btn.width}px; height:{btn.height}px; transform:translate(-50%,-50%) rotate({btn.angle}rad); background:{btn.style.background}; color:{btn.style.color}; border:{btn.style.border}; border-radius:{btn.style.borderRadius}; box-shadow:{btn.style.boxShadow}; font-family:{btn.style.fontFamily}; font-size:{btn.style.fontSize}; font-weight:{btn.style.fontWeight}; text-transform:{btn.style.textTransform}; filter:{btn.style.filter};"
				onclick={handleClick}
			>
				For Jeff
			</button>
		{/each}
	{/if}
</main>

<style>
	main {
		position: fixed;
		inset: 0;
		background: linear-gradient(180deg, #0f0f23 0%, #1a1a2e 50%, #16213e 100%);
		overflow: hidden;
	}

	.intro {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		height: 100%;
		text-align: center;
	}

	.intro-text {
		font-family: 'Space Grotesk', sans-serif;
		font-size: 1.5rem;
		color: rgba(255, 255, 255, 0.8);
		margin-bottom: 2rem;
	}

	.hint {
		font-family: 'Space Grotesk', sans-serif;
		font-size: 0.9rem;
		color: rgba(255, 255, 255, 0.4);
		margin-top: 1.5rem;
	}

	.starter {
		font-family: 'Space Grotesk', sans-serif;
		font-size: 1.25rem;
		font-weight: 700;
		color: #0f0f23;
		background: linear-gradient(135deg, #e8d5b7 0%, #d4a574 100%);
		border: none;
		border-radius: 12px;
		padding: 1rem 3rem;
		cursor: pointer;
		transition: all 0.2s ease;
		box-shadow: 0 4px 15px rgba(232, 213, 183, 0.3);
	}

	.starter:hover {
		transform: scale(1.05);
		box-shadow: 0 6px 25px rgba(232, 213, 183, 0.4);
	}

	.btn {
		position: absolute;
		display: flex;
		align-items: center;
		justify-content: center;
		white-space: nowrap;
		cursor: pointer;
		user-select: none;
		padding: 0;
	}

	.btn:hover {
		z-index: 100;
		filter: brightness(1.15) !important;
	}
</style>
