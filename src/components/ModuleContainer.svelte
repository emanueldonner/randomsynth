<script>
	import { effect } from "astro:schema"

	export let type = "synth" // default type is synth
	export let isMuted = false
	export let isSoloed = false
	export let active = false // glow while active
	export let glowIntensity = 0 // 0-1 based on velocity
	export let glowDuration = 0.1 // seconds for transition

	// Compute dynamic glow based on intensity and type
	$: glowColor = type === "drum" ? "rgba(45, 231, 245," : "rgba(120, 255, 158,"
	$: boxShadow = active
		? `0 0 5px 2px ${glowColor} ${0.3 + glowIntensity * 0.6})`
		: "none"
	$: transitionDuration = `${glowDuration}s`
</script>

<div
	class="container"
	class:drum={type === "drum"}
	class:main-out={type === "main-out"}
	class:effect={type === "effect"}
	class:muted={isMuted}
	class:soloed={isSoloed}
	style="box-shadow: {boxShadow}; transition: box-shadow {transitionDuration} ease-out, border-color 0.15s;"
>
	<slot />
</div>

<style>
	.container {
		border: 1px solid var(--accent-synth);
		background: var(--bg-tertiary);
		padding: 1rem;
		margin-bottom: 1rem;
		border-radius: 8px;
		display: flex;
		flex-direction: column;
		/* justify-content: center; */
		align-items: center;
		gap: 0.75rem;
		position: relative;
		flex: 1;
	}
	.container.drum {
		border-color: var(--accent-drum);
	}
	.container.main-out {
		border-color: var(--accent-main);
	}
	.container.effect {
		border-color: var(--accent-effect);
		max-width: 300px;
	}
	.container.muted {
		border-color: var(--border-secondary);
		opacity: 0.7;
	}
	.container.soloed {
		border-color: var(--border-soloed);
	}
</style>
