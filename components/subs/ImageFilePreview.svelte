<script lang="ts">
	import { onMount } from 'svelte';

	export let file: File;
	export let previewWidth: string = '200px';
	export let previewHeight: string = '200px';

	let image: HTMLImageElement;
	let title: string;
	let showImage = false;

	onMount(() => {
		if (file) {
			showImage = true;

			const reader = new FileReader();
			reader.addEventListener('load', function () {
				image.setAttribute('src', reader.result as string);
			});
			reader.readAsDataURL(file);

			return;
		}
		showImage = false;
	});

	$: title = file.name;
</script>

{#if showImage}
	<img
		bind:this={image}
		src=""
		alt={title}
		{title}
		style={`max-width: ${previewWidth}; max-height: ${previewHeight};`}
	/>
{:else}
	<span>Image Preview</span>
{/if}
