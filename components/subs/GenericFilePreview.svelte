<script lang="ts">
	export let file: File;
	export let previewWidth: string = '200px';
	export let previewHeight: string = '200px';
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
