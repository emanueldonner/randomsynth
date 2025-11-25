<script>
	import {
		Frequency,
		Synth,
		start,
		getTransport,
		Channel,
		Loop,
		Reverb,
		Gain,
	} from "tone"
	import { onMount } from "svelte"
	import SynthComponent from "./SynthComponent.svelte"
	import DrumComponent from "./DrumComponent.svelte"
	import Mixer from "./Mixer.svelte"

	// Per-waveform perceived loudness compensation (in dB)
	// Adjust these by ear if needed. More complex psychoacoustic weighting would
	// require RMS measurement; this static table keeps it simple.
	const SHAPE_GAIN_COMP = {
		// baseline
		sine: 0,
		triangle: -1.5, // slightly brighter than sine
		square: -10, // strong odd harmonics
		sawtooth: -6, // richest harmonic content
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

	// Global reverb bus (created lazily on first ensure)
	let globalReverb = null

	// Initialize mixer channels on mount
	onMount(() => {
		// Pre-populate mixer with synth configs (channels will be created later)
		mixerSynthChannels = synths.map((config) => ({
			name: config.name,
			channel: null, // Will be populated in ensureSynths
			volume: 0,
			pan: config.pan,
			reverbSend: 0,
			sendGain: null,
			isMuted: config.isMuted,
			configRef: config, // Keep reference to update later
		}))
	})

	let synths = [
		{
			name: "synth1",
			pan: -1,
			shape: "triangle",
			noteLength: "8n",
			stepInterval: "16n",
			minOctave: 3,
			maxOctave: 5,
			probability: 0.4,
			velocityIsProbability: false,
			velocity: 0.5,
			isMuted: false,
			isActive: false,
			glowIntensity: 0,
			glowDuration: 0.3,
			reverbSend: 0,
			instance: null,
			channel: null,
			sendGain: null,
			loop: null,
		},
		{
			name: "synth2",
			pan: 0,
			shape: "sawtooth",
			noteLength: "8n",
			stepInterval: "4n",
			minOctave: 2,
			maxOctave: 3,
			probability: 0.8,
			velocityIsProbability: false,
			velocity: 0.5,
			isMuted: false,
			isActive: false,
			glowIntensity: 0,
			glowDuration: 0.3,
			reverbSend: 0,
			instance: null,
			channel: null,
			sendGain: null,
			loop: null,
		},
		{
			name: "synth3",
			pan: 1,
			shape: "sine",
			noteLength: "2n",
			stepInterval: "1n",
			minOctave: 4,
			maxOctave: 6,
			probability: 0.7,
			velocityIsProbability: false,
			velocity: 0.5,
			isMuted: false,
			isActive: false,
			glowIntensity: 0,
			glowDuration: 0.3,
			reverbSend: 0,
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

	const ensureSynths = async () => {
		await start()
		if (!globalReverb) {
			globalReverb = new Reverb({
				decay: 2.8,
				preDelay: 0.02,
				wet: 1,
			}).toDestination()
			await globalReverb.generate()
		}
		// create synth instances if they don't exist and give it a channel so we can control pan/volume later
		for (const config of synths) {
			if (!config.instance) {
				console.log(`Creating synth ${config.name} with shape:`, config.shape)
				config.channel = new Channel().toDestination()
				config.channel.pan.value = config.pan
				config.sendGain = new Gain(config.reverbSend).connect(globalReverb)
				// Create synth with default oscillator type
				const oscillatorType = config.shape || "sine"
				config.instance = new Synth()
				config.instance.connect(config.channel)
				config.instance.connect(config.sendGain)
				// Set oscillator type after instance is created
				try {
					config.instance.set({ oscillator: { type: oscillatorType } })
					console.log(
						`Successfully set oscillator type to ${oscillatorType} for ${config.name}`
					)
					applyShapeComp(config)
				} catch (e) {
					console.error(
						`Error setting oscillator type for ${config.name}:`,
						e,
						config
					)
				}
			}
			// ensure send gain reflects current reverbSend
			if (config.sendGain) config.sendGain.gain.value = config.reverbSend
		}
		// Update mixer channel references with created channels
		mixerSynthChannels.forEach((mixerCh) => {
			const config = mixerCh.configRef
			if (config && config.channel) {
				mixerCh.channel = config.channel
				mixerCh.sendGain = config.sendGain
				mixerCh.reverbSend = config.reverbSend
			}
		})
		// Trigger reactivity
		mixerSynthChannels = mixerSynthChannels
	}

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
		// Update oscillator shape
		if (config.instance && config.shape) {
			try {
				config.instance.set({ oscillator: { type: config.shape } })
				applyShapeComp(config)
			} catch (e) {
				console.error("Error updating synth:", e)
			}
		}
		// Reverb send: update gain node directly (mixer will handle UI sync)
		if (config.sendGain) {
			config.sendGain.gain.value = config.reverbSend
		}
		// If stepInterval changed while playing, rebuild that loop for immediate effect
		if (isPlaying && config.loop) {
			// always rebuild on change event to guarantee applied
			rebuildLoop(config)
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

	const startStopLoop = async () => {
		const transport = getTransport()
		if (!isPlaying) {
			await ensureSynths()
			transport.bpm.value = bpm
			// start individual loops for each synth
			for (const config of synths) {
				startLoopForSynth(config)
			}
			// start drums
			if (drumComponentRef) {
				await drumComponentRef.startDrums()
			}
			transport.start()
			isPlaying = true
		} else {
			// stop all synth loops
			transport.stop()
			for (const config of synths) {
				stopLoopForSynth(config)
			}
			// stop drums
			if (drumComponentRef) {
				drumComponentRef.stopDrums()
			}
			isPlaying = false
		}
	}

	// update BPM dynamically when it changes (only if playing)
	$: if (isPlaying && bpm) {
		try {
			getTransport().bpm.value = bpm
		} catch (e) {
			// ignore if transport not ready
		}
	}

	// dynamically update loop interval when stepInterval changes without restarting
	function rebuildLoop(config) {
		// dispose existing loop and create a new one with updated interval
		stopLoopForSynth(config)
		startLoopForSynth(config)
	}

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
	/>
</div>
<div class="bottom-section">
	<div class="synth-section">
		<h2>┌─ SYNTHS ─┐</h2>
		<div class="synth-grid">
			{#each synths as synthConfig}
				<SynthComponent {synthConfig} on:change={handleSynthChange} />
			{/each}
		</div>
	</div>

	<div class="mixer-section">
		<h2>┌─ MIXER ─┐</h2>
		<Mixer
			synthChannels={mixerSynthChannels}
			drumChannels={mixerDrumChannels}
		/>
	</div>
</div>

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
		align-items: center;
	}
	.synth-grid {
		display: flex;
		gap: var(--gap-md);
		flex-wrap: wrap;
	}
</style>
