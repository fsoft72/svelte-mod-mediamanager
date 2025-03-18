<script lang="ts">
	import FilePreview from '$modules/mediamanager/components/FilePreview.svelte';
	import ProgressBar from '$liwe3/components/ProgressBar.svelte';
	import { url_and_headers } from '$liwe3/utils/fetcher';
	import type { TreeItem } from '$liwe3/utils/tree';
	import SelectTree from '$liwe3/components/SelectTree.svelte';
	import { addToast } from '$liwe3/stores/ToastStore.svelte';
	import Button from '$liwe3/components/Button.svelte';
	import TagInput from '$liwe3/components/TagInput.svelte';
	import md5 from '$liwe3/utils/md5';

	interface UpdateEvent {
		name: string;
		progress: number;
		start: number;
		size: number;
	}

	interface Props {
		previewWidth?: string;
		previewHeight?: string;
		id_folder?: string;
		folders?: TreeItem[] | null;
		tags?: string[] | null;
		anonymous?: boolean;

		showUploadButton?: boolean;
		maxFiles?: number;

		onupdate?: (evt: UpdateEvent) => void;
		ondone?: (files: number) => void;
		oncompleted?: (name: string, id_upload: string) => void;
	}

	let {
		previewWidth = '200px',
		previewHeight = '200px',
		id_folder = $bindable('root'),
		folders = null,
		tags = null,
		anonymous = false,
		maxFiles = 0,
		showUploadButton = true, // Added new property for controlling the upload button visibility
		onupdate,
		ondone,
		oncompleted
	}: Props = $props();

	let files: File[] = $state([]);
	let currentFileIndex = 0;
	let chunkSize = 1024 * 1024; // 1MB
	let uploadField: HTMLInputElement | undefined = $state();

	type UploadedFile = {
		id: string;
		filename: string;
		type: string;
	};

	let uploadProgress = $state(0);
	let uploadName = $state('');
	let isDragOver = $state(false);
	let tags_selected: string[] = $state([]);
	let uploadedFiles: UploadedFile[] = $state([]); // Updated to use the UploadedFile type

	export async function submit() {
		if (files.length === 0) {
			throw new Error('No files to upload');
		}

		let totFiles = files.length;
		uploadedFiles = []; // reset uploaded file IDs

		while (currentFileIndex < files.length) {
			uploadName = files[currentFileIndex].name;
			uploadProgress = 0;

			onupdate &&
				onupdate({
					name: uploadName,
					progress: uploadProgress,
					start: 0,
					size: files[currentFileIndex].size
				});

			const uploadResult = await handleFileUpload();
			if (uploadResult) uploadedFiles.push(uploadResult);
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

		ondone && ondone(totFiles);

		return uploadedFiles; // return the list of uploaded file IDs
	}

	async function handleFileUpload() {
		const file = files[currentFileIndex];
		const data: Record<string, any> = {
			filename: file.name,
			id_folder,
			tags: tags_selected,
			size: file.size.toString()
		};

		// if the anonymous flag is set, we need to generate a hash of the file
		// and we call it anonymous so we can check it later from the server
		if (anonymous) data['anonymous'] = md5(`${data.filename}${data.size}${data.id_folder}`);

		const { url, headers } = url_and_headers(`/api/media/upload/chunk/start`);

		headers['Content-Type'] = 'application/json';

		const response = await fetch(url, {
			method: 'POST',
			body: JSON.stringify(data),
			headers
		});

		const { id_upload, error } = await response.json();

		if (error) {
			addToast({
				type: 'error',
				message: error
			});
			return null;
		}

		await sendChunk(id_upload);
		return {
			id: id_upload,
			filename: file.name,
			type: file.type
		};
	}

	async function sendChunk(id_upload: string) {
		const file = files[currentFileIndex];
		const fileSize = file.size;
		let start = 0;

		const { url, headers } = url_and_headers(`/api/media/upload/chunk/add`);

		headers['Content-Type'] = 'application/octet-stream';

		while (start < fileSize) {
			uploadProgress = Math.round((start / fileSize) * 100);

			onupdate &&
				onupdate({
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
		oncompleted && oncompleted(file.name, id_upload);
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
		// First check if we're already at or above the max limit
		if (maxFiles > 0 && files.length >= maxFiles) {
			addToast({
				type: 'error',
				message: `Maximum of ${maxFiles} file${maxFiles === 1 ? '' : 's'} allowed`
			});
			return files;
		}

		// Calculate how many more files we can add
		const remainingSlots = maxFiles > 0 ? maxFiles - files.length : _files.length + 100;

		// Clone the current files array
		const updatedFiles = [...files];
		let addedCount = 0;

		// Only process files up to the remaining slots
		for (let i = 0; i < _files.length && (maxFiles === 0 || addedCount < remainingSlots); i++) {
			const _file = _files[i];
			// Only add if file not already in the list
			if (!updatedFiles.find((file: File) => file.name === _file.name)) {
				updatedFiles.push(_file);
				addedCount++;
			}
		}

		// Show message if files were skipped due to limit
		if (maxFiles > 0 && _files.length > remainingSlots) {
			addToast({
				type: 'info',
				message: `Only added ${addedCount} file(s). Maximum limit of ${maxFiles} file${maxFiles === 1 ? '' : 's'} reached.`
			});
		}

		return updatedFiles;
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

	const onRemove = (file: File) => {
		const newFiles = files.filter((f) => f.name !== file.name);

		files = newFiles;
	};

	const onTreeFolderChange = (new_id_folder: string) => {
		id_folder = new_id_folder;
	};
</script>

<!-- svelte-ignore a11y_interactive_supports_focus -->
<div
	class="upload-area"
	role="button"
	aria-label="Upload files"
	tabIndex="0"
	ondragover={onDragOver}
	ondragleave={onDragLeave}
	ondrop={onDrop}
	class:is-dragging-over={isDragOver}
>
	{#if folders || tags?.length}
		<div class="fields">
			{#if folders}
				<div class="folders-container">
					Upload Folder: <SelectTree
						name="id_folder"
						bind:value={id_folder}
						tree={{ children: folders }}
						onchange={onTreeFolderChange}
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
			<button onclick={() => uploadField?.click()} class="btn btn-link">browse</button>
		</p>
		<div class="preview">
			{#each files as file (file.name)}
				<div class="file">
					<FilePreview {file} onremove={onRemove} {previewHeight} {previewWidth} />
				</div>
			{/each}
		</div>
		<div class="upload-field">
			<input bind:this={uploadField} type="file" multiple onchange={onFilesSelected} />
		</div>
		{#if showUploadButton}
			<Button disabled={files.length === 0} onclick={submit}>Upload</Button>
		{/if}
	</div>
	{#if uploadProgress > 0}
		<p>{uploadName} - {uploadProgress}%</p>
	{/if}
	<ProgressBar text={uploadName} percentage={uploadProgress} />
</div>

<style>
	.upload-area {
		border: 1px solid var(--liwe3-darker-secondary-color);
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

		border-bottom: 1px solid var(--liwe3-darker-secondary-color);
	}

	.fields {
		padding: 0.2em 1em;
	}
</style>
