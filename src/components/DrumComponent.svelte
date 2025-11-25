<script>
	import {
		start,
		getTransport,
		MembraneSynth,
		NoiseSynth,
		Synth,
		Loop,
		Channel,
		Gain,
	} from "tone"
	import { createEventDispatcher, onMount } from "svelte"
	import Icon from "@iconify/svelte"
	import ModuleContainer from "./ModuleContainer.svelte"

	const dispatch = createEventDispatcher()

	export let isPlaying = false
	export let bpm = 120
	export let mixerChannels = []
	export let globalReverb = null

	// Drum voice configs (polymetric ready)
	let drums = [
		{
			name: "kick",
			type: "kick",
			pan: 0,
			pitch: 50,
			decay: 0.6,
			isMuted: false,
			isActive: false,
			glowIntensity: 0,
			glowDuration: 0.3,
			steps: 16,
			stepInterval: "16n",
			currentStep: 0,
			pattern: [1, 0, 0.5, 0.2, 1, 0, 0.25, 0, 1, 0, 0.6, 0.2, 1, 0.2, 0.3, 0],
			loop: null,
			channel: null,
			reverbSend: 0,
			sendGain: null,
			synths: { kick: null },
		},
		{
			name: "snare",
			type: "snare",
			pan: 0,
			snappy: 0.5,
			isMuted: false,
			isActive: false,
			glowIntensity: 0,
			glowDuration: 0.3,
			steps: 16,
			stepInterval: "16n",
			currentStep: 0,
			pattern: [
				0.4, 0, 0.25, 0, 0.9, 0, 0.35, 0, 0.6, 0, 0.3, 0, 0.5, 0, 0.2, 0,
			],
			loop: null,
			channel: null,
			reverbSend: 0,
			sendGain: null,
			synths: { noise: null, body: null },
		},
		{
			name: "HiHat",
			type: "closedHat",
			pan: 0.5,
			decay: 0.08,
			isMuted: false,
			isActive: false,
			glowIntensity: 0,
			glowDuration: 0.3,
			steps: 16,
			stepInterval: "16n",
			currentStep: 0,
			pattern: [
				0.8, 0.6, 0.7, 0.5, 0.9, 0.4, 0.6, 0.3, 0.8, 0.5, 0.7, 0.4, 0.9, 0.6,
				0.5, 0.2,
			],
			loop: null,
			channel: null,
			reverbSend: 0,
			sendGain: null,
			synths: { hat: null },
		},
	]

	onMount(() => {
		mixerChannels = drums.map((d) => ({
			name: d.name,
			channel: null,
			volume: 0,
			pan: d.pan ?? 0,
			reverbSend: d.reverbSend,
			sendGain: null,
			isMuted: d.isMuted,
			drumRef: d,
		}))
	})

	async function ensureDrums() {
		await start()
		for (const d of drums) {
			if (!d.channel) {
				d.channel = new Channel().toDestination()
				d.channel.volume.value = 0
				d.channel.pan.value = d.pan ?? 0
				if (globalReverb && !d.sendGain)
					d.sendGain = new Gain(d.reverbSend).connect(globalReverb)
			}
			if (d.type === "kick" && !d.synths.kick) {
				d.synths.kick = new MembraneSynth({
					pitchDecay: 0.008,
					octaves: 10,
					envelope: {
						attack: 0.001,
						decay: d.decay,
						sustain: 0.0,
						release: 0.2,
					},
				}).connect(d.channel)
				if (d.sendGain) d.synths.kick.connect(d.sendGain)
			}
			if (d.type === "snare" && (!d.synths.noise || !d.synths.body)) {
				d.synths.noise = new NoiseSynth({
					noise: { type: "white" },
					envelope: { attack: 0.001, decay: 0.2, sustain: 0, release: 0.1 },
				}).connect(d.channel)
				d.synths.body = new Synth({
					oscillator: { type: "sine" },
					envelope: { attack: 0.001, decay: 0.18, sustain: 0, release: 0.08 },
				}).connect(d.channel)
				d.synths.noise.volume.value = -8
				d.synths.body.volume.value = -12
				if (d.sendGain) {
					d.synths.noise.connect(d.sendGain)
					d.synths.body.connect(d.sendGain)
				}
			}
			if ((d.type === "closedHat" || d.type === "openHat") && !d.synths.hat) {
				d.synths.hat = new NoiseSynth({
					noise: { type: "pink" },
					envelope: {
						attack: 0.001,
						decay: d.decay,
						sustain: 0,
						release: 0.01,
					},
				}).connect(d.channel)
				d.synths.hat.volume.value = -6
				if (d.sendGain) d.synths.hat.connect(d.sendGain)
			}
			if (d.sendGain) d.sendGain.gain.value = d.reverbSend
		}
		mixerChannels.forEach((mixerCh) => {
			const drum = mixerCh.drumRef
			if (drum && drum.channel) {
				mixerCh.channel = drum.channel
				mixerCh.sendGain = drum.sendGain
				mixerCh.reverbSend = drum.reverbSend
			}
		})
		mixerChannels = mixerChannels
	}

	// reactive propagation of mixer reverb send changes
	$: mixerChannels &&
		mixerChannels.forEach((ch) => {
			if (ch.drumRef && ch.sendGain) {
				ch.sendGain.gain.value = ch.reverbSend ?? 0
				ch.drumRef.reverbSend = ch.reverbSend ?? 0
			}
		})

	function triggerDrum(d, time, stepIndex) {
		if (d.isMuted) return
		const stepProbability = d.pattern[stepIndex]
		if (stepProbability === 0) return
		if (Math.random() > stepProbability) return
		const velocity = stepProbability
		switch (d.type) {
			case "kick":
				d.synths.kick?.triggerAttackRelease(d.pitch, "8n", time, velocity)
				break
			case "snare":
				d.synths.noise?.triggerAttackRelease("8n", time, velocity * 0.9)
				d.synths.body?.triggerAttackRelease("A3", "8n", time, velocity * 0.85)
				break
			case "closedHat":
				d.synths.hat?.triggerAttackRelease("32n", time, velocity)
				break
			case "openHat":
				d.synths.hat?.triggerAttackRelease("8n", time, velocity)
				break
		}
		let offDelay
		switch (d.type) {
			case "kick":
				offDelay = d.decay + 0.1
				break
			case "snare":
				offDelay = 0.25
				break
			case "closedHat":
				offDelay = 0.15
				break
			case "openHat":
				offDelay = Math.min(d.decay + 0.2, 0.6)
				break
			default:
				offDelay = 0.3
		}
		if (d.isActive) {
			d.isActive = false
			drums = drums
		}
		setTimeout(() => {
			d.glowIntensity = velocity
			d.glowDuration = 0.05
			d.isActive = true
			drums = drums
		}, 0)
		try {
			getTransport().scheduleOnce(() => {
				d.isActive = false
				d.glowDuration = Math.min(offDelay, 0.5)
				drums = drums
			}, "+" + offDelay)
		} catch {
			setTimeout(() => {
				d.isActive = false
				d.glowDuration = Math.min(offDelay, 0.5)
				drums = drums
			}, offDelay * 1000)
		}
	}

	function buildLoop(d, startStep = 0) {
		if (startStep === 0) d.currentStep = -1
		let step = startStep
		d.loop = new Loop((time) => {
			const stepIndex = step % d.steps
			triggerDrum(d, time, stepIndex)
			d.currentStep = stepIndex
			step++
			drums = drums
		}, d.stepInterval)
		d.loop.start(0)
	}

	function disposeLoop(d) {
		if (d.loop) {
			d.loop.stop()
			d.loop.dispose()
			d.loop = null
		}
	}

	function rebuildLoop(d) {
		const wasRunning = d.loop !== null
		const currentStepPosition = d.currentStep >= 0 ? d.currentStep + 1 : 0
		disposeLoop(d)
		if (wasRunning && isPlaying) buildLoop(d, currentStepPosition)
	}

	export async function startDrums() {
		await ensureDrums()
		for (const d of drums) buildLoop(d)
	}

	export function stopDrums() {
		for (const d of drums) disposeLoop(d)
	}

	$: if (isPlaying && bpm) {
		try {
			getTransport().bpm.value = bpm
		} catch {}
	}

	function handleParamChange(d) {
		if (d.decay && d.decay < 0.01) d.decay = 0.01
		if (d.type === "kick" && d.synths.kick)
			d.synths.kick.envelope.decay = d.decay
		if ((d.type === "closedHat" || d.type === "openHat") && d.synths.hat)
			d.synths.hat.envelope.decay = d.decay
		dispatch("change", d)
	}

	function handleStepChange(d, stepIndex, value) {
		d.pattern[stepIndex] = value
		dispatch("change", d)
	}
</script>

<div class="sequencer">
	{#each drums as d}
		<ModuleContainer
			type="drum"
			active={d.isActive}
			isMuted={d.isMuted}
			glowIntensity={d.glowIntensity || 0}
			glowDuration={d.glowDuration || 0.3}
		>
			<div class="drum-row">
				<div class="drum-header">
					<h3>{d.name}</h3>
					<div class="randomiser">
						<button
							title="Randomise Pattern"
							on:click={() => {
								d.pattern = d.pattern.map(() => Math.random())
								handleParamChange(d)
								rebuildLoop(d)
							}}
						>
							<Icon icon="mdi:dice-multiple" />
						</button>
					</div>
				</div>

				<div class="step-grid">
					{#each Array(16) as _, stepIndex}
						<div
							class="step-cell"
							class:active={stepIndex === d.currentStep}
							class:inactive={stepIndex >= d.steps}
						>
							<input
								type="range"
								orient="vertical"
								min="0"
								max="1"
								step="0.01"
								value={d.pattern[stepIndex] || 0}
								disabled={stepIndex >= d.steps}
								on:input={(e) =>
									handleStepChange(d, stepIndex, parseFloat(e.target.value))}
								title="Step {stepIndex + 1}: {Math.round(
									(d.pattern[stepIndex] || 0) * 100
								)}%"
							/>
							<span class="step-number">{stepIndex + 1}</span>
						</div>
					{/each}
				</div>

				<div class="drum-params">
					<label>
						Active Steps: {d.steps}
						<input
							type="range"
							min="1"
							max="16"
							step="1"
							bind:value={d.steps}
							on:input={() => {
								while (d.pattern.length < 16) d.pattern.push(0)
								d.pattern = d.pattern.slice(0, 16)
								handleParamChange(d)
								rebuildLoop(d)
							}}
						/>
					</label>
					{#if d.type === "closedHat" || d.type === "openHat"}
						<label>
							Decay: {d.decay.toFixed(2)}
							<input
								type="range"
								min="0.05"
								max="0.8"
								step="0.01"
								bind:value={d.decay}
								on:input={() => handleParamChange(d)}
							/>
						</label>
					{/if}
				</div>
			</div>
		</ModuleContainer>
	{/each}
</div>

<style>
	.sequencer {
		display: flex;
		flex-direction: row;
		flex-wrap: wrap;
		justify-content: space-around;
		max-width: fit-content;
		gap: var(--gap-lg);
		margin: var(--gap-md) auto 0;
	}

	/* .drum-row removed (empty ruleset) */

	.drum-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: var(--gap-md);
		border-bottom: 1px solid var(--border-secondary);
		padding-bottom: var(--gap-sm);
	}

	.drum-header h3 {
		color: var(--accent-drum);
		text-shadow: 0 0 8px var(--accent-drum);
	}

	.drum-header .randomiser button {
		background: none;
		border-color: var(--accent-drum);
		padding: 0.4rem;
		width: 32px;
		height: 32px;
		display: flex;
		justify-content: center;
		align-items: center;
		color: var(--accent-drum);
		font-size: 2rem;
	}

	.step-grid {
		display: grid;
		grid-template-columns: repeat(16, auto);
		gap: 4px;
		margin-bottom: var(--gap-md);
		padding: var(--gap-sm);
		background: var(--bg-primary);
		border: 1px solid var(--border-secondary);
		border-radius: var(--radius-sm);
	}

	.step-cell {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 4px;
	}

	.step-cell.inactive {
		opacity: 0.4;
	}
	.step-cell input[type="range"] {
		writing-mode: bt-lr; /* for Firefox */
		-webkit-appearance: slider-vertical; /* for WebKit */
		appearance: slider-vertical;
		width: 14px;
		height: 60px;
		background: var(--bg-secondary);
		border: 1px solid var(--border-secondary);
		margin: 0;
		padding: 0;
	}

	.step-cell input[type="range"]::-webkit-slider-thumb {
		background: var(--accent-drum);
		border-color: var(--accent-drum);
		box-shadow: 0 0 4px var(--accent-drum);
	}

	.step-cell input[type="range"]::-moz-range-thumb {
		background: var(--accent-drum);
		border-color: var(--accent-drum);
		box-shadow: 0 0 4px var(--accent-drum);
	}

	.step-cell.active {
		background: var(--accent-drum);
		border-radius: var(--radius-sm);
		box-shadow: 0 0 6px var(--accent-drum);
	}

	.step-number {
		font-size: 0.55rem;
		color: var(--text-dim);
		user-select: none;
	}
</style>
