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
	import { createEventDispatcher } from "svelte"

	export let bpm = 120
	export let synths = []
	export let reverbConfig
	export let delayConfig
	export let bitCrusherConfig
	export let lfoConfig

	const dispatch = createEventDispatcher()

	let globalReverb = null
	let globalDelay = null
	let globalBitCrusher = null
	let masterCompressor = null
	let masterSaturation = null
	let globalLFO = null
	let isPlaying = false
	let effectChain = "parallel"

	// Expose references to parent
	export let mixerSynthChannels = []

	function applyShapeComp(config) {
		if (!config.instance) return
		const SHAPE_GAIN_COMP = {
			sine: 0,
			triangle: -1.5,
			square: -10,
			sawtooth: -6,
			fatsine: 0,
			fattriangle: 0,
			fatsquare: 0,
			fatsawtooth: 0,
		}
		const comp = SHAPE_GAIN_COMP[config.shape] ?? 0
		config.instance.volume.value = comp
	}

	export function handleReverbChange() {
		if (globalReverb) {
			console.log("SynthEngine updating reverb:", reverbConfig)
			globalReverb.decay = reverbConfig.decay
			globalReverb.preDelay = reverbConfig.preDelay
		}
	}
	export function handleDelayChange() {
		if (globalDelay) {
			console.log("SynthEngine updating delay:", delayConfig)
			globalDelay.delayTime.value = delayConfig.delayTime
			globalDelay.feedback.value = delayConfig.feedback
		}
	}
	export function handleBitCrusherChange() {
		if (globalBitCrusher) {
			console.log("SynthEngine updating bitCrusher:", bitCrusherConfig)
			globalBitCrusher.bits = bitCrusherConfig.bits
			globalBitCrusher.wet.value = bitCrusherConfig.wet
		}
	}
	export function handleLFOChange() {
		if (globalLFO) {
			console.log("SynthEngine updating LFO:", lfoConfig)
			globalLFO.frequency.rampTo(lfoConfig.frequency, 0.1)
			globalLFO.type = lfoConfig.type
		}
	}

	export async function ensureSynths() {
		await start()

		if (!globalReverb && !globalDelay) {
			if (effectChain === "parallel") {
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
		}
		if (!globalBitCrusher) {
			globalBitCrusher = new BitCrusher(bitCrusherConfig.bits)
			globalBitCrusher.wet.value = bitCrusherConfig.wet
			globalBitCrusher.toDestination()
		}
		if (!masterSaturation) {
			const makeSaturationCurve = (amount = 1.5) => {
				const samples = 1024
				const curve = new Float32Array(samples)
				for (let i = 0; i < samples; i++) {
					const x = (i * 2) / samples - 1
					const drive = 1 + amount * 3
					curve[i] = Math.tanh(x * drive) / Math.tanh(drive)
				}
				return curve
			}
			masterSaturation = new WaveShaper(makeSaturationCurve(1.5))
			masterSaturation.oversample = "4x"
		}
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
			globalReverb && globalReverb.disconnect()
			globalReverb && globalReverb.connect(masterCompressor)
			globalDelay && globalDelay.disconnect()
			globalDelay && globalDelay.connect(masterCompressor)
			globalBitCrusher && globalBitCrusher.disconnect()
			globalBitCrusher && globalBitCrusher.connect(masterCompressor)
		}
		if (!globalLFO) {
			globalLFO = new LFO({
				frequency: lfoConfig.frequency,
				type: lfoConfig.type,
				min: -1,
				max: 1,
			})
			globalLFO.start()
		}

		for (const config of synths) {
			if (!config.instance) {
				config.channel = new Channel().connect(masterCompressor)
				config.channel.pan.value = config.pan
				try {
					config.channel.solo = !!config.isSoloed
				} catch {}
				config.sendGain = new Gain().connect(globalReverb)
				config.sendGain.gain.value = config.reverbSend
				config.delayGain = new Gain().connect(globalDelay)
				config.delayGain.gain.value = config.delaySend

				const oscillatorType = config.shape || "sine"
				config.instance = new Synth({
					oscillator: { type: oscillatorType },
					envelope: config.envelope,
				})
				if (!config.filter) {
					if (config.filterCutoff && config.filterCutoff > 20) {
						config.filter = new Filter(config.filterCutoff, "lowpass")
						config.filter.Q.value = config.filterQ || 1
						config.instance.connect(config.filter)
					}
				}
				let sourceNode = config.filter || config.instance
				sourceNode.connect(config.channel)
				sourceNode.connect(config.sendGain)
				sourceNode.connect(config.delayGain)
				if (!config.bitCrusherGain) {
					config.bitCrusherGain = new Gain(config.bitCrusherSend).connect(
						globalBitCrusher
					)
				}
				sourceNode.connect(config.bitCrusherGain)

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
				}
				applyShapeComp(config)
			}
			if (config.sendGain) config.sendGain.gain.value = config.reverbSend
			if (config.delayGain) config.delayGain.gain.value = config.delaySend
			if (config.bitCrusherGain)
				config.bitCrusherGain.gain.value = config.bitCrusherSend || 0
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
					} else {
						config.lfoMod.mult.factor.value = depthHz
						config.lfoMod.add.addend.value = config.filterCutoff
					}
				} else if (config.lfoMod) {
					// Disconnect and dispose LFO modulation chain
					try {
						config.lfoMod.add.disconnect()
						config.lfoMod.mult.disconnect()
						globalLFO.disconnect(config.lfoMod.mult)
						config.lfoMod.mult.dispose()
						config.lfoMod.add.dispose()
					} catch (e) {
						console.warn("Error cleaning up LFO chain:", e)
					}
					config.lfoMod = null

					// Recreate the filter with fresh AudioParam to restore direct control
					if (config.filter && config.instance) {
						const targetFreq = Math.max(config.filterCutoff, 20)
						const oldFilter = config.filter

						// Create new filter
						config.filter = new Filter(targetFreq, "lowpass")
						config.filter.Q.value = config.filterQ

						// Reconnect audio chain
						try {
							config.instance.disconnect(oldFilter)
						} catch {}
						config.instance.connect(config.filter)
						config.filter.connect(config.channel)
						if (config.sendGain) config.filter.connect(config.sendGain)
						if (config.delayGain) config.filter.connect(config.delayGain)
						if (config.bitCrusherGain)
							config.filter.connect(config.bitCrusherGain)

						// Dispose old filter
						try {
							oldFilter.dispose()
						} catch {}
					}
				}
			}
		}

		// Notify parent of engine refs
		dispatch("engine-ready", {
			globalReverb,
			globalDelay,
			globalBitCrusher,
			masterCompressor,
			globalLFO,
		})

		// Update mixer channel references
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
		mixerSynthChannels = mixerSynthChannels
	}

	export function startStopLoop(drumComponentRef) {
		const transport = getTransport()
		if (!isPlaying) {
			ensureSynths().then(async () => {
				transport.bpm.value = bpm
				for (const config of synths) {
					startLoopForSynth(config)
				}
				if (drumComponentRef) {
					await drumComponentRef.startDrums()
				}
				transport.start()
				isPlaying = true
				dispatch("play-state", { isPlaying })
			})
		} else {
			transport.stop()
			for (const config of synths) {
				stopLoopForSynth(config)
			}
			if (drumComponentRef) {
				drumComponentRef.stopDrums()
			}
			isPlaying = false
			dispatch("play-state", { isPlaying })
		}
	}

	function startLoopForSynth(config) {
		config.loop = new Loop((time) => {
			playRandomNoteForSynth(config, time)
		}, config.stepInterval)
		config.loop.start(0)
	}
	function stopLoopForSynth(config) {
		if (config.loop) {
			config.loop.stop()
			config.loop.dispose()
			config.loop = null
		}
	}

	function playRandomNoteForSynth(config, timeOverride) {
		if (config.isMuted) return
		// Probability gate: skip triggering when random exceeds configured probability
		const prob = typeof config.probability === "number" ? config.probability : 1
		if (Math.random() > prob) return
		const pentaMinor = [0, 3, 5, 7, 10]
		const semis = pentaMinor[Math.floor(Math.random() * pentaMinor.length)]
		const octave =
			Math.floor(Math.random() * (config.maxOctave - config.minOctave + 1)) +
			config.minOctave
		const randomNote = Frequency(`${config.rootNote || "C"}${octave}`)
			.transpose(semis)
			.toNote()
		const velocity = config.velocityIsProbability ? prob : config.velocity
		config.instance.triggerAttackRelease(
			randomNote,
			config.noteLength,
			timeOverride,
			velocity
		)
	}

	export function handleSynthChange(event) {
		const config = event.detail
		if (config.channel && config.pan !== undefined) {
			config.channel.pan.value = config.pan
		}
		if (config.minOctave > config.maxOctave) {
			config.maxOctave = config.minOctave
		}
		if (config.maxOctave < config.minOctave) {
			config.minOctave = config.maxOctave
		}
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
		if (config.filterCutoff !== undefined && config.filterQ !== undefined) {
			if (config.filter) {
				if (!config.lfoMod) {
					config.filter.frequency.value = Math.max(config.filterCutoff, 20)
				}
				config.filter.Q.value = config.filterQ
			} else if (config.filterCutoff > 20 && config.instance) {
				config.filter = new Filter(config.filterCutoff, "lowpass")
				config.filter.Q.value = config.filterQ
				try {
					config.instance.disconnect()
				} catch {}
				config.instance.connect(config.filter)
				config.filter.connect(config.channel)
				if (config.sendGain) config.filter.connect(config.sendGain)
				if (config.delayGain) config.filter.connect(config.delayGain)
			}
		}
		if (config.bitCrusherGain)
			config.bitCrusherGain.gain.value = config.bitCrusherSend || 0

		if (config.filter && config.lfoCutoffDepth !== undefined) {
			if (config.lfoCutoffDepth > 0) {
				const depthHz = Math.max(5, config.filterCutoff * config.lfoCutoffDepth)
				if (!config.lfoMod && globalLFO) {
					const mult = new Multiply(depthHz)
					const add = new Add(config.filterCutoff)
					globalLFO.connect(mult)
					mult.connect(add)
					add.connect(config.filter.frequency)
					config.lfoMod = { mult, add }
				} else if (config.lfoMod) {
					config.lfoMod.mult.factor.value = depthHz
					config.lfoMod.add.addend.value = config.filterCutoff
				}
			} else if (config.lfoMod) {
				console.log(
					`Removing LFO modulation from ${config.name}, restoring cutoff to ${config.filterCutoff}`
				)
				// Disconnect and dispose LFO modulation chain
				try {
					config.lfoMod.add.disconnect()
					config.lfoMod.mult.disconnect()
					globalLFO.disconnect(config.lfoMod.mult)
					config.lfoMod.mult.dispose()
					config.lfoMod.add.dispose()
				} catch (e) {
					console.warn("Error cleaning up LFO chain:", e)
				}
				config.lfoMod = null

				// Recreate the filter with fresh AudioParam to restore direct control
				if (config.filter && config.instance) {
					const targetFreq = Math.max(config.filterCutoff, 20)
					const oldFilter = config.filter

					// Create new filter
					config.filter = new Filter(targetFreq, "lowpass")
					config.filter.Q.value = config.filterQ

					// Reconnect audio chain: instance -> new filter -> outputs
					try {
						config.instance.disconnect(oldFilter)
					} catch {}
					config.instance.connect(config.filter)
					config.filter.connect(config.channel)
					if (config.sendGain) config.filter.connect(config.sendGain)
					if (config.delayGain) config.filter.connect(config.delayGain)
					if (config.bitCrusherGain)
						config.filter.connect(config.bitCrusherGain)

					// Dispose old filter
					try {
						oldFilter.dispose()
					} catch {}

					console.log(`Recreated filter at ${targetFreq}Hz`)
				}
			}
		}

		if (config.filter) {
			config.filter.Q.value = config.filterQ
		}
		if (config.sendGain) {
			config.sendGain.gain.value = config.reverbSend
		}
		if (config.delayGain) {
			config.delayGain.gain.value = config.delaySend
		}
		if (config.bitCrusherGain) {
			config.bitCrusherGain.gain.value = config.bitCrusherSend || 0
		}
		// Rebuild tone loop if playing
		if (isPlaying && config.loop) {
			stopLoopForSynth(config)
			startLoopForSynth(config)
		}
	}

	$: if (isPlaying && bpm) {
		try {
			getTransport().bpm.value = bpm
			if (globalDelay && delayConfig.delayTime) {
				globalDelay.delayTime.value = delayConfig.delayTime
			}
		} catch (e) {}
	}

	// Config changes now handled via explicit function calls from parent
</script>

<!-- This component has no UI; it manages Tone graph and loops -->
