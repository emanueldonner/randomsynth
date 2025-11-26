<script>
	import ModuleContainer from "./ModuleContainer.svelte"
	import { createEventDispatcher } from "svelte"
	import Icon from "@iconify/svelte"
	import PixelCheckbox from "./PixelCheckbox.svelte"
	export let synthConfig

	const dispatch = createEventDispatcher()
	const SYNTH_WAVEFORMS = ["sine", "square", "triangle", "sawtooth"]
	const NOTE_LENGTHS = ["1n", "2n", "4n", "8n", "16n"]
	const STEP_INTERVALS = ["1n", "2n", "4n", "8n", "16n"]

	// Track base waveform separately so we can add/remove "fat" prefix
	let baseShape = synthConfig.shape?.replace(/^fat/, "") || "sine"
	// Reactive: update shape when oscBoost or baseShape changes
	$: {
		const newShape = synthConfig.oscBoost ? "fat" + baseShape : baseShape
		if (synthConfig.shape !== newShape) {
			synthConfig.shape = newShape
			notifyChange()
		}
	}

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

	// Logarithmic scale for filter cutoff (20Hz to 20kHz)
	// Slider value 0-1, mapped to 20-20000 Hz exponentially
	let cutoffSlider = 0.5
	$: {
		// Convert actual cutoff to slider position (inverse of the mapping below)
		cutoffSlider = Math.log(synthConfig.filterCutoff / 20) / Math.log(1000)
	}
	function handleCutoffChange(e) {
		// Map slider 0-1 to 20-20000 Hz exponentially
		const sliderVal = parseFloat(e.target.value)
		synthConfig.filterCutoff = Math.round(20 * Math.pow(1000, sliderVal))
		notifyChange()
	}

	function notifyChange() {
		clampOctaves()
		dispatch("change", synthConfig)
	}

	// Ensure default LFO depth field exists
	if (synthConfig.lfoCutoffDepth === undefined) synthConfig.lfoCutoffDepth = 0
</script>

<ModuleContainer
	type="synth"
	active={synthConfig.isActive}
	isMuted={synthConfig.isMuted}
	isSoloed={synthConfig.isSoloed}
	glowIntensity={synthConfig.glowIntensity || 0}
	glowDuration={synthConfig.glowDuration || 0.3}
>
	<div class="synth-controls">
		<h3>{synthConfig.name}</h3>

		<div class="module-grid">
			<div class="module">
				<div class="module-title">TONE</div>
				<div class="">
					<span class="control-label">Wave</span>

					<div class="button-group small">
						{#each SYNTH_WAVEFORMS as waveform}
							<button
								class="icon-button"
								class:active={baseShape === waveform}
								on:click={() => {
									baseShape = waveform
								}}
								title={waveform}
							>
								<Icon icon={WAVEFORM_TO_SYMBOL[waveform]} />
							</button>
						{/each}
					</div>
					<div class="button-group small">
						<PixelCheckbox
							label="Boost"
							bind:checked={synthConfig.oscBoost}
							on:change={notifyChange}
						/>
					</div>
				</div>
				<div class="">
					<span class="control-label">Oct</span>
					<div class="button-group octave-selector small">
						{#each [1, 2, 3, 4, 5, 6, 7, 8] as octave}
							<button
								class="octave-button"
								class:in-range={octave >= synthConfig.minOctave &&
									octave <= synthConfig.maxOctave}
								class:min={octave === synthConfig.minOctave}
								class:max={octave === synthConfig.maxOctave}
								on:click={() => {
									if (octave < synthConfig.minOctave) {
										synthConfig.minOctave = octave
									} else if (octave > synthConfig.maxOctave) {
										synthConfig.maxOctave = octave
									} else if (
										octave === synthConfig.minOctave &&
										octave === synthConfig.maxOctave
									) {
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
				<div class="row spread">
					<div class="group">
						<span class="control-label">Len</span>
						<div class="button-group small">
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
					<div class="group">
						<span class="control-label">Step</span>
						<div class="button-group small">
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
					<div class="fader">
						<label class="mini">
							Prob ({(synthConfig.probability * 100).toFixed(0)}%)
							<input
								type="range"
								min="0"
								max="1"
								step="0.01"
								bind:value={synthConfig.probability}
							/>
						</label>
						<label class="mini">
							Vel ({(synthConfig.velocity * 100).toFixed(0)}%)
							<input
								type="range"
								min="0"
								max="1"
								step="0.01"
								bind:value={synthConfig.velocity}
							/>
						</label>
					</div>
				</div>
			</div>

			<div class="module">
				<div class="module-title">ENV</div>
				<div class="fader-row">
					<label class="mini">
						A

						<input
							type="range"
							min="0.01"
							max="2"
							step="0.01"
							orient="vertical"
							bind:value={synthConfig.envelope.attack}
							on:input={notifyChange}
						/>
						{synthConfig.envelope.attack.toFixed(2)}
					</label>
					<label class="mini">
						D
						<input
							type="range"
							min="0.01"
							max="2"
							step="0.01"
							orient="vertical"
							bind:value={synthConfig.envelope.decay}
							on:input={notifyChange}
						/>
						{synthConfig.envelope.decay.toFixed(2)}
					</label>
					<label class="mini">
						S
						<input
							type="range"
							min="0"
							max="1"
							step="0.01"
							orient="vertical"
							bind:value={synthConfig.envelope.sustain}
							on:input={notifyChange}
						/>
						{synthConfig.envelope.sustain.toFixed(2)}
					</label>
					<label class="mini">
						R
						<input
							type="range"
							min="0.01"
							max="5"
							step="0.01"
							orient="vertical"
							bind:value={synthConfig.envelope.release}
							on:input={notifyChange}
						/>
						{synthConfig.envelope.release.toFixed(2)}
					</label>
				</div>
				<!-- </div>

			<div class="module"> -->
				<div class="module-title">FILTER</div>
				<div class="fader-row">
					<label class="mini">
						LFO
						<input
							type="range"
							min="0"
							max="1"
							step="0.01"
							orient="vertical"
							bind:value={synthConfig.lfoCutoffDepth}
							on:input={notifyChange}
						/>
						{(synthConfig.lfoCutoffDepth * 100).toFixed(0)}%
					</label>
					<label class="mini">
						Cutoff
						<input
							type="range"
							min="0"
							max="1"
							step="0.001"
							orient="vertical"
							value={cutoffSlider}
							on:input={handleCutoffChange}
						/>
						{synthConfig.filterCutoff.toFixed(0)}
					</label>
					<label class="mini">
						Q
						<input
							type="range"
							min="0.1"
							max="20"
							step="0.1"
							orient="vertical"
							bind:value={synthConfig.filterQ}
							on:input={notifyChange}
						/>
						{synthConfig.filterQ.toFixed(1)}
					</label>
				</div>
				<!-- </div>

			<div class="module"> -->
				<!-- <div class="module-title">DYN</div>
				
			</div> -->
			</div>
		</div>
	</div></ModuleContainer
>

<style>
	.synth-controls {
		display: flex;
		flex-direction: column;
		gap: var(--gap-md);
		align-items: center;
	}

	.module-grid {
		display: grid;
		grid-template-columns: repeat(2, minmax(120px, 1fr));
		gap: var(--gap-sm);
	}

	.module {
		border: 1px solid var(--border-secondary);
		background: var(--bg-secondary);
		padding: 6px;
		border-radius: var(--radius-sm);
		display: flex;
		flex-direction: column;
		gap: 6px;
		min-width: 120px;
	}

	.module-title {
		font-size: 0.8rem;
		text-transform: uppercase;
		letter-spacing: 0.12em;
		color: var(--text-secondary);
		border-bottom: 1px dashed var(--border-secondary);
		padding-bottom: 4px;
	}

	/* compact rows for merged TONE module */
	.row {
		display: flex;
		align-items: center;
		gap: 8px;
		justify-content: space-between;
	}

	.row.spread {
		justify-content: center;
		gap: var(--gap-sm);
		flex-wrap: wrap;
	}

	/* .inline-group removed (unused after layout refactor) */

	.fader-row {
		display: flex;
		gap: var(--gap-sm);
		justify-content: center;
		width: fit-content;
		margin: 0 auto;
	}

	label.mini {
		display: flex;
		flex-direction: column;
		gap: 6px;
		align-items: center;
		font-size: 0.7rem;
		color: var(--text-secondary);
		flex: 1;
	}

	label.mini input[orient="vertical"] {
		writing-mode: bt-lr;
		-webkit-appearance: slider-vertical;
		appearance: slider-vertical;
		height: 70px;
		width: 18px;
		background: var(--bg-primary);
		border: 1px solid var(--border-secondary);
	}

	.control-label {
		font-size: 0.75rem;
		color: var(--text-secondary);
		text-transform: uppercase;
		letter-spacing: 0.05em;
	}

	.button-group {
		display: flex;
		justify-content: center;
		gap: 4px;
		flex-wrap: wrap;
	}

	.button-group.small .icon-button {
		min-width: 28px;
		padding: 0.25rem 0.35rem;
		font-size: 1rem;
	}

	.icon-button {
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

	/* .text-button removed (unused) */

	.icon-button:hover {
		border-color: var(--accent-synth);
		color: var(--text-secondary);
	}

	.icon-button.active {
		background: var(--accent-synth);
		color: var(--bg-primary);
		border-color: var(--accent-synth);
		box-shadow: 0 0 6px var(--accent-synth);
		font-weight: bold;
	}
	/* toggle-button styles removed (unused) */

	.octave-selector {
		display: flex;
		gap: 1px;
	}

	.octave-selector.small .octave-button {
		min-width: 18px;
		font-size: 0.65rem;
		padding: 0.2rem 0.1rem;
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
