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
		onupdate,
		ondone,
		oncompleted
	}: Props = $props();

	let files: File[] = $state([]);
	let currentFileIndex = 0;
	let chunkSize = 1024 * 1024; // 1MB
	let uploadField: HTMLInputElement | undefined = $state();

	let uploadProgress = $state(0);
	let uploadName = $state('');
	let isDragOver = $state(false);
	let tags_selected: string[] = $state([]);

	async function uploadFiles() {
		console.log('=== UPLOAD FILES: ', files.length);
		let totFiles = files.length;

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

		ondone && ondone(totFiles);
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
			return;
		}

		await sendChunk(id_upload);
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
			<!-- svelte-ignore a11y_interactive_supports_focus -->
			<!-- svelte-ignore a11y_invalid_attribute -->
			<a href="#" onclick={() => uploadField?.click()}>browse</a>
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
		<Button disabled={files.length === 0} onclick={uploadFiles}>Upload</Button>
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
