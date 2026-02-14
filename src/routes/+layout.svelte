<script lang="ts">
    import "$lib/dailytrojan-lib/gameStyles.css"
	import "$lib/dailytrojan-lib/icons/tabler-icons.css"
	import "$lib/dailytrojan-lib/fonts/fonts.css"
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

{@render children()}
