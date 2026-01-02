<script>
	import favicon from '$lib/assets/favicon.svg';
	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />


</svelte:head>

<div class="app-background">
	<nav>
		<h1>LogTen Converter</h1>
	</nav>
	{@render children()}
</div>

<style>

	.app-background {
		min-height: 100%;
		width: 100vw;
		background-color: var(--color-background);
		color: var(--color-text);
		font-family: 'Roboto', sans-serif;
		display: flex;
		flex-direction: column;
		align-items: center;
		/* justify-content: space-around; */
		gap: 5rem;
		padding: 10rem 0 10rem 0;
	}

	nav {
		padding: 1rem 2rem;
		border-bottom: 1px solid var(--color-border);
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 1rem;
	}

	h1 {
		margin: 0;
		font-size: 1.5rem;
		font-weight: 500;
	}
</style>
