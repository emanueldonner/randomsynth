<script>
	import ModuleContainer from "../ModuleContainer.svelte"
	import { createEventDispatcher } from "svelte"

	export let reverbConfig
	const dispatch = createEventDispatcher()

	function handleChange() {
		console.log("reverb config changed in ReverbControl:", reverbConfig)
		dispatch("change")
	}
</script>

<ModuleContainer type="effect">
	<div class="reverb-controls">
		<h3>Reverb</h3>
		<div class="control-group">
			<span class="control-label">
				Decay: {reverbConfig.decay.toFixed(2)} s
			</span>
			<input
				type="range"
				min="0.1"
				max="14"
				step="0.1"
				bind:value={reverbConfig.decay}
				on:input={handleChange}
			/>
		</div>
		<div class="control-group">
			<span class="control-label">
				Pre-Delay: {(reverbConfig.preDelay * 1000).toFixed(0)} ms
			</span>
			<input
				type="range"
				min="0"
				max="0.1"
				step="0.005"
				bind:value={reverbConfig.preDelay}
				on:input={handleChange}
			/>
		</div>
	</div>
</ModuleContainer>

<style>
	.reverb-controls {
		display: flex;
		flex-direction: column;
		gap: var(--gap-md);
		align-items: center;
	}

	.control-group {
		display: flex;
		flex-direction: column;
		gap: var(--gap-sm);
		width: 100%;
	}
</style>
