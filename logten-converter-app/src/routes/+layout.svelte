<script>
	import { base } from '$app/paths';
	import favicon from '$lib/assets/favicon.svg';
	let { children } = $props();

	
</script>

<svelte:head>
	<link rel="stylesheet" href="{base}/global.css" />
	<link rel="icon" href={favicon} />
</svelte:head>

<div class="app-background">
	<nav>
		<h1>LogTen Converter</h1>
	</nav>
	{@render children()}
</div>

