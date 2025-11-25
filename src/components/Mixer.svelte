<script>
	import ModuleContainer from "./ModuleContainer.svelte"
	import { getDestination } from "tone"

	export let synthChannels = []
	export let drumChannels = []

	let mainVolume = 0 // dB

	function handleMixerChange(channel) {
		if (!channel || !channel.channel) return
		// Update volume and pan
		if (channel.volume !== undefined) {
			channel.channel.volume.value = channel.volume
		}
		if (channel.pan !== undefined) {
			channel.channel.pan.value = channel.pan
		}
		if (channel.sendGain && channel.reverbSend !== undefined) {
			channel.sendGain.gain.value = channel.reverbSend
		}
		// Sync reverbSend back to config
		if (channel.configRef && channel.reverbSend !== undefined) {
			channel.configRef.reverbSend = channel.reverbSend
		}
		if (channel.drumRef && channel.reverbSend !== undefined) {
			channel.drumRef.reverbSend = channel.reverbSend
		}
		if (channel.isMuted !== undefined) {
			channel.channel.mute = channel.isMuted
		}
	}

	// Reactive updates for all channels
	$: synthChannels.forEach((ch) => handleMixerChange(ch))
	$: drumChannels.forEach((ch) => handleMixerChange(ch))

	// Update main output volume
	$: try {
		getDestination().volume.value = mainVolume
	} catch (e) {
		// Destination not ready yet
	}
</script>

<div class="mixer-grid">
	<!-- Drum Channels -->
	{#each drumChannels as channel}
		<ModuleContainer type="drum" isMuted={channel.isMuted}>
			<div class="mixer-channel">
				<h3>{channel.name}</h3>

				<label>
					Rev <br />
					{Math.round((channel.reverbSend || 0) * 100)}%
					<input
						type="range"
						min="0"
						max="1"
						step="0.01"
						bind:value={channel.reverbSend}
					/>
				</label>

				<label>
					Pan: <br />
					{channel.pan.toFixed(2)}
					<input
						type="range"
						min="-1"
						max="1"
						step="0.01"
						bind:value={channel.pan}
					/>
				</label>

				<label>
					{channel.volume.toFixed(1)}dB
					<input
						type="range"
						orient="vertical"
						min="-48"
						max="6"
						step="0.5"
						bind:value={channel.volume}
					/>
				</label>
				<button
					class="mute-button"
					class:active={channel.isMuted}
					on:click={() => (channel.isMuted = !channel.isMuted)}
				>
					M
				</button>
			</div>
		</ModuleContainer>
	{/each}
	<!-- Synth Channels -->
	{#each synthChannels as channel}
		<ModuleContainer isMuted={channel.isMuted}>
			<div class="mixer-channel" class:muted={channel.isMuted}>
				<h3>{channel.name}</h3>

				<!-- <label>
					<input type="checkbox" bind:checked={channel.isMuted} />
					Mute
				</label> -->
				<label>
					Rev <br />
					{Math.round((channel.reverbSend || 0) * 100)}%
					<input
						type="range"
						min="0"
						max="1"
						step="0.01"
						bind:value={channel.reverbSend}
					/>
				</label>
				<label>
					Pan <br />
					{channel.pan.toFixed(2)}
					<input
						type="range"
						min="-1"
						max="1"
						step="0.01"
						bind:value={channel.pan}
					/>
				</label>
				<label>
					{channel.volume.toFixed(1)}dB
					<input
						type="range"
						orient="vertical"
						min="-48"
						max="6"
						step="0.5"
						bind:value={channel.volume}
					/>
				</label>

				<button
					class="mute-button"
					class:active={channel.isMuted}
					on:click={() => (channel.isMuted = !channel.isMuted)}
				>
					M
				</button>
			</div>
		</ModuleContainer>
	{/each}
	<!-- main out -->
	<ModuleContainer type="main-out">
		<div class="mixer-channel main-out">
			<h3>MAIN</h3>

			<label>
				{mainVolume.toFixed(1)}dB
				<input
					type="range"
					orient="vertical"
					min="-48"
					max="6"
					step="0.5"
					bind:value={mainVolume}
				/>
			</label>
		</div>
	</ModuleContainer>
</div>

<style>
	.mixer-grid {
		flex: 1;
		display: flex;
		gap: var(--gap-sm);
		flex-wrap: wrap;
		margin: var(--gap-md) auto 0;
		justify-content: space-around;
		max-width: fit-content;
	}

	.mixer-channel {
		/* min-width: 120px; */
		width: 56px;
		display: flex;
		flex-direction: column;
		gap: var(--gap-md);
	}

	.mixer-channel.muted {
		opacity: 0.7;
	}

	/* .mixer-channel.main-out removed (empty ruleset) */

	.mixer-channel h3 {
		font-size: 1rem;
		text-align: center;
		padding-bottom: var(--gap-sm);
		border-bottom: 1px solid var(--border-secondary);
	}

	.mute-button {
		width: 100%;
		padding: 0.5rem;
		font-size: 1.5rem;
		background: var(--bg-secondary);
		border: 1px solid var(--border-secondary);
		cursor: pointer;
		transition: all 0.1s;
	}

	.mute-button:hover {
		border-color: var(--accent-synth);
	}

	.mute-button.active {
		background: var(--bg-primary);
		border-color: var(--border-muted);
		color: var(--border-muted);
		/* opacity: 0.6; */
	}

	.mixer-channel label {
		display: flex;
		flex-direction: column;
		gap: var(--gap-sm);
	}

	.mixer-channel input[type="range"] {
		width: 100%;
	}

	.mixer-channel input[type="range"][orient="vertical"] {
		writing-mode: bt-lr;
		-webkit-appearance: slider-vertical;
		appearance: slider-vertical;
		height: 100px;
		width: 20px;
		align-self: center;
		background: var(--bg-secondary);
		border: 1px solid var(--border-secondary);
	}

	.mixer-channel.main-out input[type="range"][orient="vertical"] {
		height: 250px;
	}

	.mixer-channel input[type="range"][orient="vertical"]::-webkit-slider-thumb {
		width: 28px;
		height: 10px;
	}

	.mixer-channel input[type="range"][orient="vertical"]::-moz-range-thumb {
		width: 28px;
		height: 10px;
	}
</style>
