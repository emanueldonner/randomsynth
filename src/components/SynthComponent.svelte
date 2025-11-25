<script>
	import ModuleContainer from "./ModuleContainer.svelte"
	import { createEventDispatcher } from "svelte"
	import Icon from "@iconify/svelte"
	export let synthConfig

	const dispatch = createEventDispatcher()
	const SYNTH_WAVEFORMS = ["sine", "square", "triangle", "sawtooth"]
	const NOTE_LENGTHS = ["1n", "2n", "4n", "8n", "16n"]
	const STEP_INTERVALS = ["1n", "2n", "4n", "8n", "16n"]

	const WAVEFORM_TO_SYMBOL = {
		sine: "mdi:sine-wave",
		square: "mdi:square-wave",
		triangle: "mdi:triangle-wave",
		sawtooth: "mdi:sawtooth-wave",
	}
	const NOTE_VALUE_TO_SYMBOL = {
		"1n": "mdi:music-note-whole", // whole note (circle)
		"2n": "mdi:music-note-half", // half note (half circle)
		"4n": "mdi:music-note-quarter", // quarter note
		"8n": "mdi:music-note-eighth", // eighth note
		"16n": "mdi:music-note-sixteenth", // sixteenth note
	}
	function clampOctaves() {
		if (synthConfig.minOctave > synthConfig.maxOctave) {
			synthConfig.maxOctave = synthConfig.minOctave
		}
		if (synthConfig.maxOctave < synthConfig.minOctave) {
			synthConfig.minOctave = synthConfig.maxOctave
		}
	}

	function notifyChange() {
		clampOctaves()
		dispatch("change", synthConfig)
	}
</script>

<ModuleContainer
	type="synth"
	active={synthConfig.isActive}
	isMuted={synthConfig.isMuted}
	glowIntensity={synthConfig.glowIntensity || 0}
	glowDuration={synthConfig.glowDuration || 0.3}
>
	<div class="synth-controls">
		<h3>{synthConfig.name}</h3>

		<div class="control-group">
			<span class="control-label">Waveform:</span>
			<div class="button-group">
				{#each SYNTH_WAVEFORMS as waveform}
					<button
						class="icon-button"
						class:active={synthConfig.shape === waveform}
						on:click={() => {
							synthConfig.shape = waveform
							notifyChange()
						}}
						title={waveform}
					>
						<Icon icon={WAVEFORM_TO_SYMBOL[waveform]} />
					</button>
				{/each}
			</div>
		</div>

		<div class="control-group">
			<span class="control-label">Octave Range:</span>
			<div class="octave-selector">
				{#each [0, 1, 2, 3, 4, 5, 6, 7, 8] as octave}
					<button
						class="octave-button"
						class:in-range={octave >= synthConfig.minOctave &&
							octave <= synthConfig.maxOctave}
						class:min={octave === synthConfig.minOctave}
						class:max={octave === synthConfig.maxOctave}
						on:click={() => {
							// Click to set min, shift+click to set max, or toggle range
							if (octave < synthConfig.minOctave) {
								synthConfig.minOctave = octave
							} else if (octave > synthConfig.maxOctave) {
								synthConfig.maxOctave = octave
							} else if (
								octave === synthConfig.minOctave &&
								octave === synthConfig.maxOctave
							) {
								// Single octave selected - do nothing or could expand
							} else if (octave === synthConfig.minOctave) {
								synthConfig.minOctave = Math.min(
									octave + 1,
									synthConfig.maxOctave
								)
							} else if (octave === synthConfig.maxOctave) {
								synthConfig.maxOctave = Math.max(
									octave - 1,
									synthConfig.minOctave
								)
							}
							notifyChange()
						}}
						title="Octave {octave}"
					>
						{octave}
					</button>
				{/each}
			</div>
		</div>

		<div class="control-group">
			<span class="control-label">Note Length:</span>
			<div class="button-group">
				{#each NOTE_LENGTHS as length}
					<button
						class="icon-button"
						class:active={synthConfig.noteLength === length}
						on:click={() => {
							synthConfig.noteLength = length
						}}
						title={length}
					>
						<Icon icon={NOTE_VALUE_TO_SYMBOL[length]} />
					</button>
				{/each}
			</div>
		</div>

		<div class="control-group">
			<span class="control-label">Step Interval:</span>
			<div class="button-group">
				{#each STEP_INTERVALS as interval}
					<button
						class="icon-button"
						class:active={synthConfig.stepInterval === interval}
						on:click={() => {
							synthConfig.stepInterval = interval
							notifyChange()
						}}
					>
						<Icon icon={NOTE_VALUE_TO_SYMBOL[interval]} />
					</button>
				{/each}
			</div>
		</div>

		<div class="control-group">
			<span class="control-label">
				Attack: {synthConfig.envelope.attack.toFixed(2)} s
			</span>
			<input
				type="range"
				min="0.01"
				max="2"
				step="0.01"
				bind:value={synthConfig.envelope.attack}
				on:input={notifyChange}
			/>
			<span class="control-label">
				Decay: {synthConfig.envelope.decay.toFixed(2)} s
			</span>
			<input
				type="range"
				min="0.01"
				max="2"
				step="0.01"
				bind:value={synthConfig.envelope.decay}
				on:input={notifyChange}
			/>
			<span class="control-label">
				Sustain: {(synthConfig.envelope.sustain * 100).toFixed(0)} %
			</span>
			<input
				type="range"
				min="0"
				max="1"
				step="0.01"
				bind:value={synthConfig.envelope.sustain}
				on:input={notifyChange}
			/>
			<span class="control-label">
				Release: {synthConfig.envelope.release.toFixed(2)} s
			</span>
			<input
				type="range"
				min="0.01"
				max="5"
				step="0.01"
				bind:value={synthConfig.envelope.release}
				on:input={notifyChange}
			/>
		</div>

		<div class="control-group">
			<span class="control-label"
				>Filter Cutoff: {synthConfig.filterCutoff} Hz</span
			>
			<input
				type="range"
				min="20"
				max="20000"
				step="1"
				bind:value={synthConfig.filterCutoff}
				on:input={notifyChange}
			/>

			<span class="control-label"
				>Filter Q: {synthConfig.filterQ.toFixed(2)}</span
			>
			<input
				type="range"
				min="0.1"
				max="20"
				step="0.1"
				bind:value={synthConfig.filterQ}
				on:input={notifyChange}
			/>
		</div>

		<div class="control-group">
			<span class="control-label"
				>Probability: {(synthConfig.probability * 100).toFixed(0)}%</span
			>
			<input
				type="range"
				min="0"
				max="1"
				step="0.01"
				bind:value={synthConfig.probability}
			/>
		</div>

		<!-- <div class="control-group">
			<button
				class="toggle-button"
				class:active={synthConfig.velocityIsProbability}
				on:click={() => {
					synthConfig.velocityIsProbability = !synthConfig.velocityIsProbability
				}}
			>
				{synthConfig.velocityIsProbability ? "☑" : "☐"} Velocity = Probability
			</button>
		</div> -->

		<!-- {#if !synthConfig.velocityIsProbability} -->
		<div class="control-group">
			<span class="control-label"
				>Velocity: {(synthConfig.velocity * 100).toFixed(0)}%</span
			>
			<input
				type="range"
				min="0"
				max="1"
				step="0.01"
				bind:value={synthConfig.velocity}
			/>
		</div>
		<!-- {/if} -->
	</div>
</ModuleContainer>

<style>
	.synth-controls {
		display: flex;
		flex-direction: column;
		gap: var(--gap-md);
		align-items: center;
	}

	.control-group {
		display: flex;
		flex-direction: column;
		gap: var(--gap-sm);
	}

	.control-label {
		font-size: 0.75rem;
		color: var(--text-secondary);
		text-transform: uppercase;
		letter-spacing: 0.05em;
	}

	.button-group {
		display: flex;
		gap: 4px;
		flex-wrap: wrap;
	}

	.icon-button,
	.text-button {
		padding: 0.4rem 0.6rem;
		display: flex;
		align-items: center;
		justify-content: center;
		min-width: 40px;
		font-size: 1.2rem;
		background: var(--bg-secondary);
		border: 1px solid var(--border-secondary);
		color: var(--text-dim);
		cursor: pointer;
		transition: all 0.1s;
	}

	.text-button {
		font-size: 0.85rem;
		min-width: 35px;
	}

	.icon-button:hover,
	.text-button:hover {
		border-color: var(--accent-synth);
		color: var(--text-secondary);
	}

	.icon-button.active,
	.text-button.active {
		background: var(--accent-synth);
		color: var(--bg-primary);
		border-color: var(--accent-synth);
		box-shadow: 0 0 6px var(--accent-synth);
		font-weight: bold;
	}

	.toggle-button {
		padding: 0.5rem 1rem;
		background: var(--bg-secondary);
		border: 1px solid var(--border-secondary);
		color: var(--text-secondary);
		cursor: pointer;
		text-align: left;
		font-size: 0.85rem;
		transition: all 0.1s;
	}

	.toggle-button:hover {
		border-color: var(--accent-synth);
	}

	.toggle-button.active {
		border-color: var(--accent-synth);
		color: var(--accent-synth);
	}

	.octave-selector {
		display: flex;
		gap: 1px;
	}

	.octave-button {
		padding: 0.25rem 0.15rem;
		min-width: 20px;
		font-size: 0.7rem;
		background: var(--bg-secondary);
		border: 1px solid var(--border-secondary);
		color: var(--text-dim);
		cursor: pointer;
		transition: all 0.1s;
	}

	.octave-button:hover {
		border-color: var(--accent-synth);
		color: var(--text-secondary);
	}

	.octave-button.in-range {
		background: var(--accent-synth);
		color: var(--bg-primary);
		border-color: var(--accent-synth);
	}

	.octave-button.min {
		box-shadow: inset 2px 0 0 var(--bg-primary);
		font-weight: bold;
	}

	.octave-button.max {
		box-shadow: inset -2px 0 0 var(--bg-primary);
		font-weight: bold;
	}

	.octave-button.min.max {
		box-shadow:
			inset 2px 0 0 var(--bg-primary),
			inset -2px 0 0 var(--bg-primary);
	}
</style>
