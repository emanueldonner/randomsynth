<script>
	import ModuleContainer from "../ModuleContainer.svelte"
	import { createEventDispatcher } from "svelte"
	import Icon from "@iconify/svelte"
	export let delayConfig
	const dispatch = createEventDispatcher()

	const DELAY_TIME = ["2n", "4n", "8n", "8n.", "16n", "16n."]
	const NOTE_VALUE_TO_SYMBOL = {
		"1n": "mdi:music-note-whole", // whole note (circle)
		"2n": "mdi:music-note-half", // half note (half circle)
		"4n": "mdi:music-note-quarter", // quarter note
		"8n": "mdi:music-note-eighth", // eighth note
		"8n.": "mdi:music-note-eighth-dotted", // dotted eighth note
		"16n": "mdi:music-note-sixteenth", // sixteenth note
		"16n.": "mdi:music-note-sixteenth-dotted", // dotted sixteenth note
	}
</script>

<ModuleContainer type="effect">
	<div class="delay-controls">
		<h3>Delay</h3>
		<div class="control-group">
			<!-- Delay time represented as radio buttons with note values (e.g., "8n", "4n", "2n") -->
			<div class="control-group">
				<span class="control-label">Delay Time:</span>
				<div class="button-group">
					{#each DELAY_TIME as delaytime}
						<button
							class="icon-button"
							class:active={delayConfig.delayTime === delaytime}
							on:click={() => {
								delayConfig.delayTime = delaytime
								dispatch("change")
							}}
						>
							<Icon icon={NOTE_VALUE_TO_SYMBOL[delaytime]} />
						</button>
					{/each}
				</div>
			</div>
			<!-- <span class="control-label">
				Time: {delayConfig.delayTime}
			</span> -->
		</div>
		<div class="control-group">
			<span class="control-label">
				Feedback: {(delayConfig.feedback * 100).toFixed(0)} %
			</span>
			<input
				type="range"
				min="0"
				max="0.95"
				step="0.01"
				bind:value={delayConfig.feedback}
				on:input={() => dispatch("change")}
			/>
		</div>
	</div>
</ModuleContainer>

<style>
	.delay-controls {
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
	.button-group {
		display: flex;
		justify-content: center;
		gap: var(--gap-sm);
	}
	.icon-button {
		background: var(--bg-secondary);
		border: 1px solid var(--border-secondary);
		padding: 0.5rem;
		border-radius: 4px;
		cursor: pointer;
	}
	.icon-button.active {
		background: var(--accent-effect);
		border-color: var(--accent-effect);
		color: var(--bg-primary);
	}
</style>
