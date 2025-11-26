<script>
	import { createEventDispatcher } from "svelte"
	import ModuleContainer from "../ModuleContainer.svelte"

	export let bitCrusherConfig

	const dispatch = createEventDispatcher()

	$: {
		console.log(
			"bitCrusher config changed in BitCrusherControl:",
			bitCrusherConfig
		)
		dispatch("change")
	}
</script>

<ModuleContainer type="effect">
	<h3>BitCrusher</h3>
	<div class="bitcrusher-settings">
		<div class="control-row">
			<label>
				Bits: {bitCrusherConfig.bits}
				<input
					type="range"
					min="1"
					max="16"
					step="1"
					bind:value={bitCrusherConfig.bits}
				/>
			</label>
		</div>
		<div class="control-row">
			<label>
				Wet: {(bitCrusherConfig.wet * 100).toFixed(0)}%
				<input
					type="range"
					min="0"
					max="1"
					step="0.01"
					bind:value={bitCrusherConfig.wet}
				/>
			</label>
		</div>
	</div>
</ModuleContainer>
