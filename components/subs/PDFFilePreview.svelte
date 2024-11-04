<script lang="ts">
	interface Props {
		file: File;
		previewWidth?: string;
		previewHeight?: string;
	}

	let { file, previewWidth = '200px', previewHeight = '200px' }: Props = $props();

	let pdfUrl = $state('');

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
