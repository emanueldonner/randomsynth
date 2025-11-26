<script>
	import {
		Frequency,
		Synth,
		start,
		getTransport,
		Channel,
		Loop,
		LFO,
		Filter,
		Reverb,
		PingPongDelay,
		BitCrusher,
		Compressor,
		WaveShaper,
		Gain,
		getDestination,
		Add,
		Multiply,
	} from "tone"
	import { onMount } from "svelte"
	import SynthComponent from "./SynthComponent.svelte"
	import DrumComponent from "./DrumComponent.svelte"
	import Mixer from "./Mixer.svelte"
	import ReverbControl from "./effects/ReverbControl.svelte"
	import DelayControl from "./effects/DelayControl.svelte"
	import BitCrusherControl from "./effects/BitCrusherControl.svelte"
	import LFOControl from "./LFOControl.svelte"
	import SynthEngine from "./SynthEngine.svelte"

	// Per-waveform perceived loudness compensation (in dB)
	// Adjust these by ear if needed. More complex psychoacoustic weighting would
	// require RMS measurement; this static table keeps it simple.
	const SHAPE_GAIN_COMP = {
		// baseline
		sine: 0,
		triangle: -1.5, // slightly brighter than sine
		square: -10, // strong odd harmonics
		sawtooth: -6, // richest harmonic content
		// "fat" variants (3 detuned oscillators)
		fatsine: 0,
		fattriangle: 0,
		fatsquare: 0,
		fatsawtooth: 0,
	}

	function applyShapeComp(config) {
		if (!config.instance) return
		const comp = SHAPE_GAIN_COMP[config.shape] ?? 0
		// Set synth intrinsic volume; keep channel volume for user control.
		config.instance.volume.value = comp
	}

	let name = "synth"
	let bpm = 120
	let rootNote = "C"
	let isPlaying = false
	let notePlaying = ""
	let drumComponentRef

	// Mixer channel data - initialize immediately
	let mixerSynthChannels = []
	let mixerDrumChannels = []

	// Global reverb and delay bus (created lazily on first ensure)
	let globalReverb = null
	let globalDelay = null
	let globalBitCrusher = null
	let masterCompressor = null
	let globalLFO = null
	// Reverb configuration (editable before generation)
	let reverbConfig = {
		decay: 2.8,
		preDelay: 0.02,
	}
	let delayConfig = {
		delayTime: "8n", // eighth note (synced to BPM)
		feedback: 0.5,
	}
	let bitCrusherConfig = {
		bits: 4, // global bit depth
		wet: 1, // global wet mix (BitCrusher's internal wet)
	}
	let lfoConfig = {
		frequency: 0.5,
		type: "sine",
	}

	// Initialize mixer channels on mount
	onMount(() => {
		// Pre-populate mixer with synth configs (channels will be created later)
		mixerSynthChannels = synths.map((config) => ({
			name: config.name,
			channel: null, // Will be populated in ensureSynths
			volume: 0,
			pan: config.pan,
			reverbSend: 0,
			delaySend: 0,
			bitCrusherSend: config.bitCrusherSend ?? 0,
			sendGain: null,
			delayGain: null,
			bitCrusherGain: null,
			isMuted: config.isMuted,
			isSoloed: config.isSoloed,
			configRef: config, // Keep reference to update later
		}))
	})

	let synths = [
		{
			name: "synth1",
			pan: -1,
			shape: "triangle",
			oscBoost: false,
			noteLength: "8n",
			stepInterval: "16n",
			minOctave: 3,
			maxOctave: 5,
			probability: 0.4,
			velocityIsProbability: false,
			velocity: 0.5,
			filterCutoff: 20000,
			filterQ: 1,
			filter: null,
			bitCrusherSend: 0,
			envelope: { attack: 0.01, decay: 0.2, sustain: 0.5, release: 0.3 },
			isMuted: false,
			isSoloed: false,
			isActive: false,
			glowIntensity: 0,
			glowDuration: 0.3,
			reverbSend: 0,
			delaySend: 0,
			bitCrusherSend: 0,
			instance: null,
			channel: null,
			sendGain: null,
			loop: null,
		},
		{
			name: "synth2",
			pan: 0,
			shape: "sawtooth",
			oscBoost: false,
			noteLength: "8n",
			stepInterval: "4n",
			minOctave: 2,
			maxOctave: 3,
			probability: 0.8,
			velocityIsProbability: false,
			velocity: 0.5,
			filterCutoff: 20000,
			filterQ: 1,
			filter: null,
			bitCrusherSend: 0,
			envelope: { attack: 0.005, decay: 0.15, sustain: 0.4, release: 0.25 },
			isMuted: false,
			isSoloed: false,
			isActive: false,
			glowIntensity: 0,
			glowDuration: 0.3,
			reverbSend: 0,
			delaySend: 0,
			bitCrusherSend: 0,
			instance: null,
			channel: null,
			sendGain: null,
			loop: null,
		},
		{
			name: "synth3",
			pan: 1,
			shape: "sine",
			oscBoost: false,
			noteLength: "16n",
			stepInterval: "1n",
			minOctave: 4,
			maxOctave: 6,
			probability: 0.7,
			velocityIsProbability: false,
			velocity: 0.5,
			filterCutoff: 20000,
			filterQ: 1,
			filter: null,
			bitCrusherSend: 0,
			envelope: { attack: 0.02, decay: 0.3, sustain: 0.6, release: 0.4 },
			isMuted: false,
			isSoloed: false,
			isActive: false,
			glowIntensity: 0,
			glowDuration: 0.3,
			reverbSend: 0,
			delaySend: 0,
			bitCrusherSend: 0,
			instance: null,
			channel: null,
			sendGain: null,
			loop: null,
		},
	]

	// pentatonic minor in semitones relative to root
	const pentaMinor = [0, 3, 5, 7, 10]

	// precompute scale notes
	const notesFromScale = (root, scale, octaves) => {
		const notes = []
		for (const octave of octaves) {
			for (const semis of scale) {
				const note = Frequency(`${root}${octave}`).transpose(semis).toNote()
				notes.push(note)
			}
		}
		return notes
	}

	const octaves = [3, 4, 5]
	let scaleNotes = notesFromScale(rootNote, pentaMinor, octaves)

	// recreate scale if root changes
	$: scaleNotes = notesFromScale(rootNote, pentaMinor, octaves)

	// Handle reverb config changes from child component
	function handleReverbChange() {
		console.log("handleReverbChange called with config:", reverbConfig)
		if (engineRef) engineRef.handleReverbChange()
	}
	// Handle delay config changes from child component
	function handleDelayChange() {
		console.log("handleDelayChange called with config:", delayConfig)
		console.log("engineRef:", engineRef)
		console.log("engineRef.handleDelayChange:", engineRef?.handleDelayChange)
		if (engineRef && typeof engineRef.handleDelayChange === "function") {
			engineRef.handleDelayChange()
		} else {
			console.warn("engineRef.handleDelayChange not available")
		}
	}
	// Handle bitCrusher config changes from child component
	function handleBitCrusherChange() {
		if (engineRef) engineRef.handleBitCrusherChange()
	}
	function handleLFOChange() {
		if (engineRef) engineRef.handleLFOChange()
	}

	// Audio graph now handled by SynthEngine

	// Handle changes from child components
	function handleSynthChange(event) {
		const config = event.detail
		// Update pan
		if (config.channel && config.pan !== undefined) {
			config.channel.pan.value = config.pan
		}
		// Clamp octave range if user inverted it
		if (config.minOctave > config.maxOctave) {
			config.maxOctave = config.minOctave
		}
		if (config.maxOctave < config.minOctave) {
			config.minOctave = config.maxOctave
		}
		// Update oscillator & envelope via set (Tone handles immutable envelope object internally)
		if (config.instance) {
			try {
				const setObj = {}
				if (config.shape) setObj.oscillator = { type: config.shape }
				if (config.envelope)
					setObj.envelope = {
						attack: config.envelope.attack,
						decay: config.envelope.decay,
						sustain: config.envelope.sustain,
						release: config.envelope.release,
					}
				if (Object.keys(setObj).length) config.instance.set(setObj)
				if (config.shape) applyShapeComp(config)
			} catch (e) {
				console.error("Error updating synth params:", e)
			}
		}
		// Update / (re)create filter if parameters changed significantly
		if (config.filterCutoff !== undefined && config.filterQ !== undefined) {
			// If filter node exists update params; else if cutoff > 20 create it and rewire
			if (config.filter) {
				// Only set frequency directly if NOT using LFO modulation
				if (!config.lfoMod) {
					config.filter.frequency.value = Math.max(config.filterCutoff, 20)
				}
				config.filter.Q.value = config.filterQ
			} else if (config.filterCutoff > 20 && config.instance) {
				config.filter = new Filter(config.filterCutoff, "lowpass")
				config.filter.Q.value = config.filterQ
				// Disconnect direct paths (if previously bypassed) and rewire
				try {
					config.instance.disconnect()
				} catch {}
				config.instance.connect(config.filter)
				config.filter.connect(config.channel)
				if (config.sendGain) config.filter.connect(config.sendGain)
				if (config.delayGain) config.filter.connect(config.delayGain)
				console.log(`Filter dynamically added to ${config.name}`)
			}
		}
		// Update BitCrusher send gain
		if (config.bitCrusherGain)
			config.bitCrusherGain.gain.value = config.bitCrusherSend || 0

		// LFO modulation is now handled by SynthEngine

		// Update filter Q (always safe)
		if (config.filter) {
			config.filter.Q.value = config.filterQ
		}
		// Reverb send: update gain node directly (mixer will handle UI sync)
		if (config.sendGain) {
			config.sendGain.gain.value = config.reverbSend
		}
		// Delay send: update gain node directly (mixer will handle UI sync)
		if (config.delayGain) {
			config.delayGain.gain.value = config.delaySend
			console.log(
				`handleSynthChange: ${config.name} delaySend = ${config.delaySend}`
			)
		}
		// BitCrusher send update
		if (config.bitCrusherGain) {
			config.bitCrusherGain.gain.value = config.bitCrusherSend || 0
		}
		// Engine will handle loop rebuilds

		// Forward changes to audio engine so live playback reflects updates
		if (engineRef && typeof engineRef.handleSynthChange === "function") {
			engineRef.handleSynthChange(event)
		}
	}

	const playRandomNoteForSynth = (config, timeOverride) => {
		if (config.isMuted) return

		// probability check
		if (Math.random() > config.probability) return

		// choose scale degree and octave within config range
		const semis = pentaMinor[Math.floor(Math.random() * pentaMinor.length)]
		const octave =
			Math.floor(Math.random() * (config.maxOctave - config.minOctave + 1)) +
			config.minOctave
		const randomNote = Frequency(`${rootNote}${octave}`)
			.transpose(semis)
			.toNote()
		notePlaying = randomNote

		// velocity is either from config or probability value
		const velocity = config.velocityIsProbability
			? config.probability
			: config.velocity

		config.instance.triggerAttackRelease(
			randomNote,
			config.noteLength,
			timeOverride,
			velocity
		)

		// Reset glow to force transition restart (interrupt previous)
		if (config.isActive) {
			config.isActive = false
			synths = synths
		}
		// Use requestAnimationFrame to ensure the reset is processed before re-triggering
		setTimeout(() => {
			config.glowIntensity = velocity
			config.glowDuration = 0.05 // fast attack
			config.isActive = true
			synths = synths
		}, 0)

		try {
			getTransport().scheduleOnce(() => {
				config.isActive = false
				config.glowDuration = 0.3 // slower fade out
				synths = synths // trigger reactivity when turning off
			}, "+" + config.noteLength)
		} catch (e) {
			// fallback: clear after ~500ms if transport not ready
			setTimeout(() => {
				config.isActive = false
				config.glowDuration = 0.3
				synths = synths
			}, 500)
		}
	}

	const startLoopForSynth = (config) => {
		// Create a Tone.Loop for perfectly synced timing
		config.loop = new Loop((time) => {
			playRandomNoteForSynth(config, time)
		}, config.stepInterval)

		config.loop.start(0)
	}

	const stopLoopForSynth = (config) => {
		if (config.loop) {
			config.loop.stop()
			config.loop.dispose()
			config.loop = null
		}
	}

	let engineRef
	const startStopLoop = async () => {
		if (engineRef) engineRef.startStopLoop(drumComponentRef)
		isPlaying = !isPlaying
	}

	// update BPM dynamically when it changes
	// BPM syncing handled inside SynthEngine

	// dynamically update loop interval when stepInterval changes without restarting
	function rebuildLoop(config) {}

	// (Removed automatic polling of stepInterval; handled via explicit change events for precision)
</script>

<div class="header">
	<h1>┌─ RandomProbabilityOnlineSynth RPOS v0.1.0 ─┐</h1>
	<div class="global-settings">
		<label for="bpm">BPM: {bpm}</label>
		<input type="range" id="bpm" min="50" max="280" bind:value={bpm} />
		<button on:click={startStopLoop}>
			{isPlaying ? "■ STOP" : "▶ START"}
		</button>
	</div>
</div>

<div class="drum-section">
	<h2>┌─ DRUMS ─┐</h2>
	<DrumComponent
		{bpm}
		{isPlaying}
		bind:this={drumComponentRef}
		bind:mixerChannels={mixerDrumChannels}
		{globalReverb}
		{globalDelay}
		{globalBitCrusher}
		{masterCompressor}
	/>
</div>
<div class="synth-section">
	<h2>┌─ SYNTHS ─┐</h2>
	<div class="synth-grid">
		{#each synths as synthConfig}
			<SynthComponent {synthConfig} on:change={handleSynthChange} />
		{/each}
	</div>
</div>
<div class="bottom-section">
	<div class="mixer-section">
		<h2>┌─ MIXER ─┐</h2>
		<Mixer
			synthChannels={mixerSynthChannels}
			drumChannels={mixerDrumChannels}
			{masterCompressor}
		/>
	</div>
	<div class="group-column">
		<div class="effects-section">
			<h2>┌─ GLOBAL EFFECTS ─┐</h2>
			<div class="effects-container">
				{#if bitCrusherConfig}
					<BitCrusherControl
						{bitCrusherConfig}
						on:change={handleBitCrusherChange}
					/>
				{/if}
				{#if delayConfig}
					<DelayControl {delayConfig} on:change={handleDelayChange} />
				{/if}
				{#if reverbConfig}
					<ReverbControl {reverbConfig} on:change={handleReverbChange} />
				{/if}
			</div>
		</div>
		<div class="modulation-section">
			<h2>┌─ MODULATION ─┐</h2>
			<LFOControl {lfoConfig} on:change={handleLFOChange} />
		</div>
	</div>
</div>

<!-- Invisible audio engine manager -->
<SynthEngine
	bind:this={engineRef}
	{bpm}
	{synths}
	{reverbConfig}
	{delayConfig}
	{bitCrusherConfig}
	{lfoConfig}
	bind:mixerSynthChannels
	on:engine-ready={(e) => {
		globalReverb = e.detail.globalReverb
		globalDelay = e.detail.globalDelay
		globalBitCrusher = e.detail.globalBitCrusher
		masterCompressor = e.detail.masterCompressor
		globalLFO = e.detail.globalLFO
	}}
/>

<style>
	.header {
		margin-bottom: var(--gap-lg);
		padding-bottom: var(--gap-md);
		/* border-bottom: 1px dashed var(--border-primary); */
	}

	h1 {
		font-size: 1.5rem;
		margin: 0 0 var(--gap-md) 0;
		color: var(--text-primary);
		text-shadow: 0 0 10px var(--accent-synth);
		letter-spacing: 0.2em;
	}

	h2 {
		font-size: 1.1rem;
		margin: var(--gap-lg) 0 var(--gap-md) 0;
		color: var(--text-primary);
	}

	.global-settings {
		max-width: fit-content;
		display: flex;
		align-items: center;
		gap: var(--gap-md);
		margin: 0 auto;
		padding: var(--gap-md);
		background: var(--bg-secondary);
		border: 1px solid var(--border-primary);
		border-radius: var(--radius-md);
	}

	.global-settings label {
		min-width: 80px;
	}

	.global-settings input[type="range"] {
		flex: 1;
		min-width: 200px;
	}

	.drum-section,
	.synth-section,
	.mixer-section {
		margin-bottom: var(--gap-lg);
		flex: 1;
	}

	.bottom-section {
		display: flex;
		gap: var(--gap-lg);
		flex-wrap: wrap;
		justify-content: center;
		width: fit-content;
		margin: 0 auto;
		/* align-items: center; */
	}
	.effects-container {
		display: flex;
		gap: var(--gap-md);
		justify-content: space-around;
		width: fit-content;
		margin: 0 auto;
	}
	.synth-grid {
		display: flex;
		gap: var(--gap-md);
		flex-wrap: wrap;
	}
</style>
