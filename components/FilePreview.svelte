<script lang="ts">
	import { createEventDispatcher } from 'svelte';

	import Button from '$liwe3/components/Button.svelte';
	import ImageFilePreview from './subs/ImageFilePreview.svelte';
	import PDFFilePreview from './subs/PDFFilePreview.svelte';
	import VideoFilePreview from './subs/VideoFilePreview.svelte';
	import GenericFilePreview from './subs/GenericFilePreview.svelte';
	import { format_size } from '$liwe3/utils/utils';

	interface Props {
		file: File;
		previewWidth?: string;
		previewHeight?: string;

		onremove?: (file: File) => void;
	}

	let { file, onremove, previewWidth = '200px', previewHeight = '200px' }: Props = $props();

	let fileType: string = $derived(file.type.split('/')[0]);

	const dispatch = createEventDispatcher();

	const onRemove = () => {
		onremove && onremove(file);
	};
</script>

<div class="container">
	<div class="remove">
		<Button mode="danger" size="xs" onclick={onRemove}>X</Button>
	</div>
	<div class="content">
		{#if fileType === 'image'}
			<ImageFilePreview {file} {previewWidth} {previewHeight} />
		{:else if fileType === 'video'}
			<VideoFilePreview {file} {previewWidth} {previewHeight} />
		{:else if fileType === 'audio'}
			<VideoFilePreview {file} {previewWidth} {previewHeight} />
		{:else if fileType === 'application'}
			<PDFFilePreview {file} {previewWidth} {previewHeight} />
		{:else}
			<GenericFilePreview {file} {previewWidth} {previewHeight} />
		{/if}
	</div>
	<div class="footer">
		<div>{file.name}</div>
		<div>{format_size(file.size)}</div>
	</div>
</div>

<style>
	.container {
		position: relative;
		border: 1px solid var(--liwe3-darker-secondary-color);
		border-radius: 0.2rem;
		margin: 0.5rem;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: space-between;

		box-shadow: 0 3px 5px rgba(0, 0, 0, 0.5);

		height: 200px;
	}

	.content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		height: 100%;
		padding: 0.5rem;
	}

	.remove {
		position: absolute;
		top: 0;
		right: 0;
	}

	.footer {
		display: flex;
		padding: 0.2rem;
		font-size: 60%;

		flex-direction: row;
		justify-content: space-between;

		height: 20px;
		bottom: 0;
		left: 0;

		width: 100%;

		background-color: var(--liwe3-darker-secondary-color);
	}
</style>
