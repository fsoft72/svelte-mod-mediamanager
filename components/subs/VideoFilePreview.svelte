<script lang="ts">
	import { onMount } from 'svelte';

	export let file: File;
	export let previewWidth: string = '200px';
	export let previewHeight: string = '200px';

	let showVideo = false;
	let video: HTMLVideoElement;

	onMount(() => {
		showVideo = true;
		const reader = new FileReader();
		reader.addEventListener('load', function () {
			video.setAttribute('src', reader.result as string);
		});
		reader.readAsDataURL(file);
	});
</script>

{#if showVideo}
	<!-- svelte-ignore a11y-media-has-caption -->
	<video
		bind:this={video}
		controls
		style={`max-width: ${previewWidth}; max-height: ${previewHeight};`}
	>
		<source src="" type={file.type} />
	</video>
{:else}
	<span>Video Preview</span>
{/if}
