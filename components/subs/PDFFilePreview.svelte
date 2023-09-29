<script lang="ts">
	export let file: File;
	export let previewWidth: string = '200px';
	export let previewHeight: string = '200px';

	let pdfUrl = '';

	const reader = new FileReader();
	reader.onload = () => {
		pdfUrl = reader.result as string;
	};

	if (file) {
		reader.readAsDataURL(file);
	}
</script>

{#if file && pdfUrl}
	<embed src={pdfUrl} type="application/pdf" width={previewWidth} height={previewHeight} />
{:else}
	<span>PDF Preview</span>
{/if}
