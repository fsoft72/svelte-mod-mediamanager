<script lang="ts">
	interface Props {
		file: File;
		previewWidth?: string;
		previewHeight?: string;
	}

	let { file, previewWidth = '200px', previewHeight = '200px' }: Props = $props();
</script>

<div class="content" style={`max-width: ${previewWidth}; max-height: ${previewHeight};`}>
	{file.name}
	{file.size}
</div>

<style>
	.content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
	}
</style>
