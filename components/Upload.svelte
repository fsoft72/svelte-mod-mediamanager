<script lang="ts">
	import { createEventDispatcher, onMount } from 'svelte';
	import FilePreview from '$modules/mediamanager/components/FilePreview.svelte';
	import ProgressBar from '$liwe3/components/ProgressBar.svelte';
	import { url_and_headers } from '$liwe3/utils/fetcher';
	import type { TreeItem } from '$liwe3/utils/tree';
	import SelectTree from '$liwe3/components/SelectTree.svelte';
	import { addToast } from '$liwe3/stores/ToastStore';
	import Button from '$liwe3/components/Button.svelte';
	import Select from 'svelte-select';
	import TagInput from '$liwe3/components/TagInput.svelte';

	export let previewWidth: string = '200px';
	export let previewHeight: string = '200px';
	export let id_folder: string = 'root';

	export let folders: TreeItem[] | null = null;
	export let tags: string[] | null = null;

	let files: File[] = [];
	let currentFileIndex = 0;
	let chunkSize = 1024 * 1024; // 1MB
	let uploadField: HTMLInputElement;

	let uploadProgress = 0;
	let uploadName = '';
	let isDragOver = false;
	let tags_selected: string[] = [];

	const dispatch = createEventDispatcher();

	async function uploadFiles() {
		let totFiles = files.length;

		while (currentFileIndex < files.length) {
			uploadName = files[currentFileIndex].name;
			uploadProgress = 0;

			dispatch('update', {
				name: uploadName,
				progress: uploadProgress,
				start: 0,
				size: files[currentFileIndex].size
			});

			await handleFileUpload();
			currentFileIndex++;
		}

		// remove files from list
		files = [];
		currentFileIndex = 0;
		uploadName = '';
		uploadProgress = 0;

		addToast({
			type: 'success',
			message: 'Files uploaded successfully'
		});

		dispatch('done', { files: totFiles });
	}

	async function handleFileUpload() {
		const file = files[currentFileIndex];
		const data = {
			filename: file.name,
			id_folder,
			tags: tags_selected,
			size: file.size.toString()
		};

		const { url, headers } = url_and_headers(`/api/media/upload/chunk/start`, true);

		headers['Content-Type'] = 'application/json';

		const response = await fetch(url, {
			method: 'POST',
			body: JSON.stringify(data),
			headers
		});

		const { id_upload } = await response.json();
		await sendChunk(id_upload);
	}

	async function sendChunk(id_upload: string) {
		const file = files[currentFileIndex];
		const fileSize = file.size;
		let start = 0;

		const { url, headers } = url_and_headers(`/api/media/upload/chunk/add`, true);

		headers['Content-Type'] = 'application/octet-stream';

		while (start < fileSize) {
			uploadProgress = Math.round((start / fileSize) * 100);

			dispatch('update', {
				name: uploadName,
				progress: uploadProgress,
				start: start,
				size: fileSize
			});

			const end = Math.min(start + chunkSize, file.size);
			const size = end - start;
			const chunk = file.slice(start, end);

			const response = await fetch(`${url}?id_upload=${id_upload}&start=${start}&size=${size}`, {
				method: 'POST',
				body: chunk,
				headers
			});

			const { bytes } = await response.json();
			start += bytes;
		}
		uploadProgress = 100;
	}

	const onDragOver = (e: any) => {
		e.preventDefault();
		isDragOver = true;
	};

	const onDragLeave = (e: any) => {
		e.preventDefault();
		isDragOver = false;
	};

	const _add_files = (_files: File[]) => {
		// add files to existing files
		// only add files that are not already in the list

		_files.forEach((_file: File) => {
			if (!files.find((file: File) => file.name === _file.name)) {
				files.push(_file);
			}
		});

		return files;
	};

	const onDrop = (e: any) => {
		e.preventDefault();
		isDragOver = false;
		files = _add_files(Array.from(e.dataTransfer.files));
	};

	const onFilesSelected = (e: any) => {
		// append new files to existing files
		files = _add_files(Array.from(e.target.files));
	};

	const onRemove = (e: any) => {
		const file = e.detail;
		const newFiles = files.filter((f) => f.name !== file.name);

		files = newFiles;
	};

	const onTreeFolderChange = (e: any) => {
		id_folder = e.detail.id;
	};
</script>

<!-- svelte-ignore a11y-interactive-supports-focus -->
<div
	class="upload-area"
	role="button"
	aria-label="Upload files"
	tabIndex="0"
	on:dragover={onDragOver}
	on:dragleave={onDragLeave}
	on:drop={onDrop}
	class:is-dragging-over={isDragOver}
>
	{#if folders || tags?.length}
		<div class="fields">
			{#if folders}
				<div class="folders-container">
					Upload Folder: <SelectTree
						name="id_folder"
						bind:value={id_folder}
						tree={folders}
						on:change={onTreeFolderChange}
					/>
				</div>
			{/if}
			{#if tags?.length}
				<div class="folders-container">
					Tags
					<TagInput {tags} bind:selected={tags_selected} />
				</div>
			{/if}
		</div>
	{/if}
	<div class="container">
		<p>
			Drag and drop files here or
			<!-- svelte-ignore a11y-interactive-supports-focus -->
			<!-- svelte-ignore a11y-invalid-attribute -->
			<a href="#" on:click={() => uploadField.click()}>browse</a>
		</p>
		<div class="preview">
			{#each files as file (file.name)}
				<div class="file">
					<FilePreview {file} on:remove={onRemove} {previewHeight} {previewWidth} />
				</div>
			{/each}
		</div>
		<div class="upload-field">
			<input bind:this={uploadField} type="file" multiple on:change={onFilesSelected} />
		</div>
		<Button disabled={files.length === 0} on:click={uploadFiles}>Upload</Button>
	</div>
	{#if uploadProgress > 0}
		<p>{uploadName} - {uploadProgress}%</p>
	{/if}
	<ProgressBar text={uploadName} percentage={uploadProgress} />
</div>

<style>
	.upload-area {
		border: 1px solid var(--liwe-darker-secondary-color);
		border-radius: 4px;
	}

	.container {
		padding: 4rem 4rem 0 4rem;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		margin-bottom: 1rem;
	}

	.upload-area.is-dragging-over {
		background-color: #eee;
	}

	.upload-field {
		display: none;
	}

	.preview {
		display: flex;
		flex-wrap: wrap;

		margin: 1rem 0;
	}

	.folders-container {
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 1em;

		padding: 0.5em 0;

		border-bottom: 1px solid var(--liwe-darker-secondary-color);
	}

	.fields {
		padding: 0.2em 1em;
	}
</style>
