<script lang="ts">
	import { onMount } from 'svelte';

	interface Props {
		file: File;
		previewWidth?: string;
		previewHeight?: string;
	}

	let { file, previewWidth = '200px', previewHeight = '200px' }: Props = $props();

	let image: HTMLImageElement | undefined = $state();
	let title: string = $derived(file.name);
	let showImage = $state(false);

	onMount(() => {
		if (file) {
			showImage = true;

			const reader = new FileReader();
			reader.addEventListener('load', function () {
				image!.setAttribute('src', reader.result as string);
			});
			reader.readAsDataURL(file);

			return;
		}
		showImage = false;
	});
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
