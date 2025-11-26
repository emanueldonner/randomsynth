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
	import ReverbControl from "./ReverbControl.svelte"
	import DelayControl from "./DelayControl.svelte"
	import BitCrusherControl from "./BitCrusherControl.svelte"
	import LFOControl from "./LFOControl.svelte"

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
	let masterSaturation = null
	let globalLFO = null
	// Effect chain order: "parallel" (both to destination) or "delay-reverb" or "reverb-delay"
	let effectChain = "parallel" // TODO: make effect-chain user-selectable
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
		// FIXME: see if we can fix the "grumble" when changing reverb params live
		if (globalReverb) {
			console.log("Updating global reverb with config:", reverbConfig)
			globalReverb.decay = reverbConfig.decay
			globalReverb.preDelay = reverbConfig.preDelay
		}
	}
	// Handle delay config changes from child component
	function handleDelayChange() {
		if (globalDelay) {
			console.log("Updating global delay with config:", delayConfig)
			globalDelay.delayTime.value = delayConfig.delayTime
			globalDelay.feedback.value = delayConfig.feedback
		}
	}
	// Handle bitCrusher config changes from child component
	function handleBitCrusherChange() {
		if (globalBitCrusher) {
			console.log("Updating global bitCrusher with config:", bitCrusherConfig)
			globalBitCrusher.bits = bitCrusherConfig.bits
			globalBitCrusher.wet.value = bitCrusherConfig.wet
		}
	}
	function handleLFOChange() {
		if (globalLFO) {
			// Use rampTo for smooth parameter changes
			globalLFO.frequency.rampTo(lfoConfig.frequency, 0.1)
			globalLFO.type = lfoConfig.type
			console.log(
				"Updated global LFO frequency to",
				lfoConfig.frequency,
				"Hz, type:",
				lfoConfig.type
			)
		} else {
			console.warn("globalLFO not initialized yet")
		}
	}

	const ensureSynths = async () => {
		await start()

		// Create effects based on chain order
		if (!globalReverb && !globalDelay) {
			if (effectChain === "parallel") {
				// Both effects go to destination
				globalReverb = new Reverb({
					decay: reverbConfig.decay,
					preDelay: reverbConfig.preDelay,
					wet: 1,
				}).toDestination()
				await globalReverb.generate()

				globalDelay = new PingPongDelay({
					delayTime: delayConfig.delayTime,
					feedback: delayConfig.feedback,
				}).toDestination()
				globalDelay.wet.value = 1
			} else if (effectChain === "delay-reverb") {
				// Delay feeds into reverb
				globalReverb = new Reverb({
					decay: reverbConfig.decay,
					preDelay: reverbConfig.preDelay,
					wet: 1,
				}).toDestination()
				await globalReverb.generate()

				globalDelay = new PingPongDelay({
					delayTime: delayConfig.delayTime,
					feedback: delayConfig.feedback,
				}).connect(globalReverb)
				globalDelay.wet.value = 1
			} else if (effectChain === "reverb-delay") {
				// Reverb feeds into delay
				globalDelay = new PingPongDelay({
					delayTime: delayConfig.delayTime,
					feedback: delayConfig.feedback,
				}).toDestination()
				globalDelay.wet.value = 1

				globalReverb = new Reverb({
					decay: reverbConfig.decay,
					preDelay: reverbConfig.preDelay,
					wet: 1,
				}).connect(globalDelay)
				await globalReverb.generate()
			}
			console.log(`Created effects chain: ${effectChain}`)
		}
		// Create global BitCrusher (always parallel) if not exists
		if (!globalBitCrusher) {
			globalBitCrusher = new BitCrusher(bitCrusherConfig.bits)
			globalBitCrusher.wet.value = bitCrusherConfig.wet
			globalBitCrusher.toDestination()
			console.log("Created global BitCrusher")
		}
		// Create master saturation (more audible soft-clipping for warmth)
		if (!masterSaturation) {
			// Soft saturation curve with more drive for audible warmth
			const makeSaturationCurve = (amount = 1.5) => {
				const samples = 1024
				const curve = new Float32Array(samples)
				for (let i = 0; i < samples; i++) {
					const x = (i * 2) / samples - 1
					// Aggressive tanh curve for warm saturation
					const drive = 1 + amount * 3
					curve[i] = Math.tanh(x * drive) / Math.tanh(drive)
				}
				return curve
			}
			masterSaturation = new WaveShaper(makeSaturationCurve(1.5))
			masterSaturation.oversample = "4x" // Higher oversampling for better quality
		}
		// Create master compressor on main output if not exists
		if (!masterCompressor) {
			masterCompressor = new Compressor({
				threshold: -30,
				ratio: 12,
				attack: 0.003,
				release: 0.1,
				knee: 10,
			})
				.connect(masterSaturation)
				.connect(getDestination())
			// Reconnect all effect buses through compressor
			globalReverb.disconnect()
			globalReverb.connect(masterCompressor)
			globalDelay.disconnect()
			globalDelay.connect(masterCompressor)
			globalBitCrusher.disconnect()
			globalBitCrusher.connect(masterCompressor)
			console.log("Created master compressor with saturation")
		}
		// Create global LFO (single modulation source) if not exists
		if (!globalLFO) {
			globalLFO = new LFO({
				frequency: lfoConfig.frequency,
				type: lfoConfig.type,
				min: -1,
				max: 1,
			})
			globalLFO.start()
			console.log("Created global LFO")
		}
		// create synth instances if they don't exist and give it a channel so we can control pan/volume later
		for (const config of synths) {
			if (!config.instance) {
				console.log(`Creating synth ${config.name} with shape:`, config.shape)
				config.channel = new Channel().connect(masterCompressor)
				config.channel.pan.value = config.pan
				// Initialize solo state on Tone.Channel if provided
				try {
					config.channel.solo = !!config.isSoloed
				} catch {}
				config.sendGain = new Gain().connect(globalReverb)
				config.sendGain.gain.value = config.reverbSend
				config.delayGain = new Gain().connect(globalDelay)
				config.delayGain.gain.value = config.delaySend
				console.log(
					`Created delayGain for ${config.name}:`,
					config.delayGain,
					"initial value:",
					config.delaySend,
					"connected to:",
					globalDelay
				)
				// Create synth with default oscillator type and envelope
				const oscillatorType = config.shape || "sine"
				config.instance = new Synth({
					oscillator: { type: oscillatorType },
					envelope: config.envelope,
				})

				// Create filter node (lowpass). If cutoff <= 20 treat as bypass (skip attaching).
				if (!config.filter) {
					if (config.filterCutoff && config.filterCutoff > 20) {
						config.filter = new Filter(config.filterCutoff, "lowpass")
						config.filter.Q.value = config.filterQ || 1
						// Route synth -> filter -> channel & sends
						config.instance.connect(config.filter)
					} else {
						// Bypass: no filter node
					}
				}
				// Build final chain (optional filter) and connect to outputs
				let sourceNode = config.filter || config.instance
				sourceNode.connect(config.channel)
				sourceNode.connect(config.sendGain)
				sourceNode.connect(config.delayGain)
				// BitCrusher send gain (parallel)
				if (!config.bitCrusherGain) {
					config.bitCrusherGain = new Gain(config.bitCrusherSend).connect(
						globalBitCrusher
					)
				}
				sourceNode.connect(config.bitCrusherGain)
				console.log(
					`Connected ${config.name} chain (filter=${!!config.filter}) to outputs + globalBitCrusher send`
				)
				// Per-synth LFO modulation setup (filter cutoff)
				if (config.lfoCutoffDepth === undefined) config.lfoCutoffDepth = 0
				if (config.filter && config.lfoCutoffDepth > 0 && !config.lfoMod) {
					const depthHz = Math.max(
						5,
						config.filterCutoff * config.lfoCutoffDepth
					)
					const mult = new Multiply(depthHz)
					const add = new Add(config.filterCutoff)
					globalLFO.connect(mult)
					mult.connect(add)
					add.connect(config.filter.frequency)
					config.lfoMod = { mult, add }
					console.log(`LFO modulation enabled for ${config.name}`)
				}
				// Apply loudness compensation after creation
				applyShapeComp(config)
			}
			// ensure send gain reflects current reverbSend and delaySend
			if (config.sendGain) config.sendGain.gain.value = config.reverbSend
			if (config.delayGain) {
				config.delayGain.gain.value = config.delaySend
				console.log(
					`${config.name} delayGain set to ${config.delaySend}`,
					config.delayGain
				)
			}
			if (config.bitCrusherGain)
				config.bitCrusherGain.gain.value = config.bitCrusherSend || 0
			// Update existing LFO modulation scaling if active
			if (config.filter) {
				if (config.lfoCutoffDepth > 0) {
					const depthHz = Math.max(
						5,
						config.filterCutoff * config.lfoCutoffDepth
					)
					if (!config.lfoMod) {
						const mult = new Multiply(depthHz)
						const add = new Add(config.filterCutoff)
						globalLFO.connect(mult)
						mult.connect(add)
						add.connect(config.filter.frequency)
						config.lfoMod = { mult, add }
						console.log(`LFO modulation enabled (late) for ${config.name}`)
					} else {
						config.lfoMod.mult.factor.value = depthHz
						config.lfoMod.add.addend.value = config.filterCutoff
					}
				} else if (config.lfoMod) {
					try {
						config.lfoMod.mult.dispose()
						config.lfoMod.add.dispose()
					} catch {}
					config.lfoMod = null
					console.log(`LFO modulation disabled for ${config.name}`)
				}
			}
		}
		// Update mixer channel references with created channels
		mixerSynthChannels.forEach((mixerCh) => {
			const config = mixerCh.configRef
			if (config && config.channel) {
				mixerCh.channel = config.channel
				mixerCh.sendGain = config.sendGain
				mixerCh.reverbSend = config.reverbSend
				mixerCh.delayGain = config.delayGain
				mixerCh.delaySend = config.delaySend
				mixerCh.bitCrusherGain = config.bitCrusherGain
				mixerCh.bitCrusherSend = config.bitCrusherSend
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

		// Handle LFO modulation for cutoff (create/update/destroy chain)
		if (config.filter && config.lfoCutoffDepth !== undefined) {
			if (config.lfoCutoffDepth > 0) {
				const depthHz = Math.max(5, config.filterCutoff * config.lfoCutoffDepth)
				if (!config.lfoMod && globalLFO) {
					// Create modulation chain
					const mult = new Multiply(depthHz)
					const add = new Add(config.filterCutoff)
					globalLFO.connect(mult)
					mult.connect(add)
					add.connect(config.filter.frequency)
					config.lfoMod = { mult, add }
					console.log(`LFO modulation enabled for ${config.name}`)
				} else if (config.lfoMod) {
					// Update existing modulation scaling
					config.lfoMod.mult.factor.value = depthHz
					config.lfoMod.add.addend.value = config.filterCutoff
				}
			} else if (config.lfoMod) {
				// Remove modulation chain and restore direct control
				try {
					config.lfoMod.mult.dispose()
					config.lfoMod.add.dispose()
				} catch {}
				config.lfoMod = null
				// Restore direct frequency control
				config.filter.frequency.value = config.filterCutoff
				console.log(`LFO modulation disabled for ${config.name}`)
			}
		}

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

	// update BPM dynamically when it changes
	$: if (isPlaying && bpm) {
		try {
			getTransport().bpm.value = bpm
			// Update delay time to sync with new BPM
			if (globalDelay && delayConfig.delayTime) {
				globalDelay.delayTime.value = delayConfig.delayTime
			}
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
