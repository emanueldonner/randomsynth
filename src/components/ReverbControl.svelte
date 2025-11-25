<script>
	import ModuleContainer from "./ModuleContainer.svelte"
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
		<h3>Global Reverb</h3>
		<div class="control-group">
			<label>
				Decay: {reverbConfig.decay.toFixed(2)} s
				<input
					type="range"
					min="0.1"
					max="10"
					step="0.1"
					bind:value={reverbConfig.decay}
					on:input={handleChange}
				/>
			</label>
		</div>
		<div class="control-group">
			<label>
				Pre-Delay: {(reverbConfig.preDelay * 1000).toFixed(0)} ms
				<input
					type="range"
					min="0"
					max="0.1"
					step="0.005"
					bind:value={reverbConfig.preDelay}
					on:input={handleChange}
				/>
			</label>
		</div>
	</div>
</ModuleContainer>
