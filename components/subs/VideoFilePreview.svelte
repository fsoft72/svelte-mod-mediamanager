<script lang="ts">
	import { onMount } from 'svelte';

	interface Props {
		file: File;
		previewWidth?: string;
		previewHeight?: string;
	}

	let { file, previewWidth = '200px', previewHeight = '200px' }: Props = $props();

	let showVideo = $state(false);
	let video: HTMLVideoElement | undefined = $state();

	onMount(() => {
		showVideo = true;
		const reader = new FileReader();
		reader.addEventListener('load', function () {
			video!.setAttribute('src', reader.result as string);
		});
		reader.readAsDataURL(file);
	});
</script>

{#if showVideo}
	<!-- svelte-ignore a11y_media_has_caption -->
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
