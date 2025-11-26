<script>
	import ModuleContainer from "./ModuleContainer.svelte"
	import { createEventDispatcher, onMount, onDestroy } from "svelte"

	export let lfoConfig

	const dispatch = createEventDispatcher()
	let canvas
	let animationFrame
	let time = 0

	function emit() {
		dispatch("change", lfoConfig)
	}

	// Trigger emit when frequency or type changes
	$: if (lfoConfig) {
		lfoConfig.frequency
		lfoConfig.type
		emit()
	}

	// FIXME: Frequency changes are not reflected in audio (cutoff)

	// Waveform generation functions
	function getWaveformValue(t, type) {
		const phase = t % 1 // 0 to 1
		switch (type) {
			case "sine":
				return Math.sin(phase * Math.PI * 2)
			case "triangle":
				return phase < 0.5 ? phase * 4 - 1 : 3 - phase * 4
			case "square":
				return phase < 0.5 ? 1 : -1
			case "sawtooth":
				return phase * 2 - 1
			case "inverseSawtooth":
				return 1 - phase * 2
			default:
				return 0
		}
	}

	function drawLFO() {
		if (!canvas) return
		const ctx = canvas.getContext("2d")
		const width = canvas.width
		const height = canvas.height

		// Clear
		ctx.fillStyle = "#0a0a0a"
		ctx.fillRect(0, 0, width, height)

		// Draw waveform
		ctx.strokeStyle = "#00ff88"
		ctx.lineWidth = 2
		ctx.beginPath()

		const cycles = 2 // Show 2 cycles
		const points = width
		for (let i = 0; i < points; i++) {
			const x = i
			const t = (i / points) * cycles + time * lfoConfig.frequency * 0.1
			const value = getWaveformValue(t, lfoConfig.type)
			const y = height / 2 - value * height * 0.4
			if (i === 0) {
				ctx.moveTo(x, y)
			} else {
				ctx.lineTo(x, y)
			}
		}
		ctx.stroke()

		// Draw LED indicator
		// const ledX = 10
		// const ledY = height / 2
		// const ledValue = getWaveformValue(
		// 	time * lfoConfig.frequency * 0.1,
		// 	lfoConfig.type
		// )
		// const ledRadius = 6 + Math.abs(ledValue) * 4

		// ctx.beginPath()
		// ctx.arc(ledX, ledY, ledRadius, 0, Math.PI * 2)
		// ctx.fillStyle = `rgba(0, 255, 136, ${0.3 + Math.abs(ledValue) * 0.7})`
		// ctx.fill()
		// ctx.strokeStyle = "#00ff88"
		// ctx.lineWidth = 1
		// ctx.stroke()

		// Continue animation
		time += 0.016 // ~60fps
		animationFrame = requestAnimationFrame(drawLFO)
	}

	onMount(() => {
		if (canvas) {
			canvas.width = 200
			canvas.height = 60
			drawLFO()
		}
	})

	onDestroy(() => {
		if (animationFrame) {
			cancelAnimationFrame(animationFrame)
		}
	})
</script>

<ModuleContainer type="modulation">
	<h3>LFO</h3>
	<canvas bind:this={canvas} class="lfo-display"></canvas>
	<label>
		Freq: {lfoConfig.frequency.toFixed(2)} Hz
		<input
			type="range"
			min="0.1"
			max="132"
			step="0.01"
			bind:value={lfoConfig.frequency}
		/>
	</label>
	<label>
		Shape: {lfoConfig.type}
		<select bind:value={lfoConfig.type}>
			<option value="sine">sine</option>
			<option value="triangle">triangle</option>
			<option value="square">square</option>
			<option value="sawtooth">saw</option>
			<option value="inverseSawtooth">invSaw</option>
		</select>
	</label>
</ModuleContainer>

<style>
	.lfo-display {
		width: 200px;
		height: 60px;
		border: 1px solid var(--border-secondary);
		background: #0a0a0a;
		border-radius: var(--radius-sm);
		margin: var(--gap-sm) 0;
	}
</style>
