<script lang="ts">
	import { run } from 'svelte/legacy';

	import { createEventDispatcher, onMount } from 'svelte';

	import Button from '$liwe3/components/Button.svelte';
	import Input from '$liwe3/components/Input.svelte';
	import Modal from '$liwe3/components/Modal.svelte';

	import {
		Icon,
		ArrowPath,
		ArrowUturnLeft,
		ListBullet,
		Folder,
		FolderOpen,
		EllipsisHorizontalCircle,
		CheckCircle,
		ChevronDown,
		ChevronRight,
		Photo,
		Stop,
		XCircle
	} from 'svelte-hero-icons';
	import Upload from '$modules/mediamanager/components/Upload.svelte';
	import {
		media_delete_items,
		media_folder_create,
		media_folder_delete,
		media_folders_tree,
		media_list
	} from '../actions';
	import type { Media, MediaTreeItem } from '../types';
	import { format_size, media_url } from '$liwe3/utils/utils';
	import { user_init } from '$modules/user/actions';
	import type { TreeItem } from '$liwe3/utils/tree';
	import SimpleTree from '$liwe3/components/SimpleTree.svelte';
	import FormCreator, { type FormField } from '$liwe3/components/FormCreator.svelte';
	import { addToast } from '$liwe3/stores/ToastStore.svelte';

	let search = $state('');

	let openFolders = $state(false);
	let openAssets = $state(true);
	let medias: Media[] = $state([]);
	let tree: TreeItem = $state({ id: 'empty', name: 'empty', children: [] });
	let id_folder: string = $state('root');
	let sidebarOpen = $state(false);
	let selectedCards: string[] = $state([]);

	let listOfFolders: string[] = []; // this is the list of subfolders to query media for
	let foldersMap: { [key: string]: TreeItem } = {};

	let subfolders: TreeItem[] = $state([]);

	let showUpload = $state(false);
	let showNewFolder = $state(false);

	const dispatch = createEventDispatcher();

	const newFolderFields: FormField[] = [
		{
			type: 'text',
			name: 'folder',
			label: 'Folder',
			required: true,
			autofocus: true
		}
	];

	// recursive function to build the tree
	const _build_tree = (dest: TreeItem, node: MediaTreeItem) => {
		foldersMap[node.id!] = dest;

		if (!node.subfolders) return;

		if (!dest.children) dest.children = [];

		for (const subfolder of node.subfolders) {
			if (!subfolder.id) continue;

			const new_subfolder: TreeItem = {
				id: subfolder.id,
				name: subfolder.name!,
				icon: Folder,
				id_parent: node.id,
				children: []
			};

			dest.children.push(new_subfolder);

			_build_tree(new_subfolder, subfolder);
		}
	};

	// This function starts from the current id_folder
	// and recursively goes down the tree to find the id of all the subfolders
	const _createListOfFolders = () => {
		const res: string[] = [];

		const _recursive = (node: TreeItem) => {
			res.push(node.id);

			if (!node.children) return;

			for (const child of node.children) {
				_recursive(child);
			}
		};

		_recursive(foldersMap[id_folder]);

		listOfFolders = res;
	};

	// recursive function to set the id_parent property
	const _set_id_parent = (node: TreeItem) => {
		if (!node.children) return;

		for (const child of node.children) {
			child.id_parent = node.id;

			if (!child.children || !child.children.length) continue;
			_set_id_parent(child);
		}
	};

	const _getIdParent = (id_fold: string) => {
		return foldersMap[id_fold]?.id_parent ?? '';
	};

	const getFolderName = (id_fold: string) => {
		return foldersMap[id_fold]?.name ?? '';
	};

	const media_list_update = async () => {
		const res = await media_list(listOfFolders);

		if (res.error) {
			console.log('=== MEDIA LIST ERROR: ', res.error);
			return;
		}

		medias = res;
	};

	const tree_update = async () => {
		const res = await media_folders_tree();

		if (res.error) {
			console.log('=== MEDIA FOLDERS TREE ERROR: ', res.error);
			return;
		}

		const root: TreeItem = {
			id: 'root',
			name: 'root',
			children: [],
			isOpen: true
		};

		const new_tree: TreeItem = {
			id: res.id,
			name: res.name,
			children: [],
			icon: Folder,
			isOpen: true
		};

		root.children?.push(new_tree);

		_build_tree(new_tree, res);
		_set_id_parent(root);

		tree = root;
	};

	const selectCard = (id: string) => {
		if (selectedCards.includes(id)) {
			selectedCards = selectedCards.filter((item) => item !== id);
		} else {
			selectedCards.push(id);
		}

		selectedCards = selectedCards;
	};

	const selectAll = () => {
		selectedCards = medias.map((media) => media.id ?? '');
	};

	const selectNone = () => {
		selectedCards = [];
	};

	const deleteMedias = async () => {
		const res = await media_delete_items(selectedCards);

		if (res.error) {
			console.log('=== MEDIA DELETE ERROR: ', res.error);
			return;
		}

		selectedCards = [];

		await media_list_update();
	};

	const uploadDone = async () => {
		showUpload = false;

		await media_list_update();
	};

	const folderSelected = (data: any) => {
		console.log('=== DEBUG: ', data);
		_updateView(data[0].slice(1, -1));
	};

	const _updateView = (id_fold: string) => {
		id_folder = id_fold;

		subfolders = foldersMap[id_folder].children ?? [];

		_createListOfFolders();

		media_list_update();
	};

	const createNewFolder = async (data: { folder: string }) => {
		const res = await media_folder_create(id_folder, data.folder);

		if (res.error) {
			addToast({
				type: 'error',
				message: res.error.message
			});

			return;
		}

		showNewFolder = false;

		addToast({
			type: 'success',
			message: 'Folder created'
		});

		await tree_update();
		_updateView(res.id);
	};

	const deleteFolder = async () => {
		const res = await media_folder_delete(id_folder);

		const id_parent = _getIdParent(id_folder);

		if (res.error) {
			addToast({
				type: 'error',
				message: res.error.message
			});

			return;
		}

		addToast({
			type: 'success',
			message: 'Folder deleted'
		});

		await tree_update();
		_updateView(id_parent);
	};

	const chooseMedia = (media: Media) => {
		dispatch('choose', media);
	};

	onMount(async () => {
		await user_init();

		await tree_update();
		await media_list_update();
	});
</script>

<div class="container">
	{#if sidebarOpen}
		<div class="sidebar">
			<!-- svelte-ignore a11y_click_events_have_key_events -->
			<div class="title">
				Folders

				<!-- svelte-ignore a11y_click_events_have_key_events -->
				<!-- svelte-ignore a11y_no_static_element_interactions -->
				<div class="btn" onclick={() => (sidebarOpen = false)}>
					<Icon src={XCircle} size="24" />
				</div>
			</div>
			<SimpleTree
				items={tree.children ?? []}
				level={0}
				multipleSelection={false}
				fontSize="20"
				onselect={folderSelected}
			/>
		</div>
	{/if}
	<div class="main">
		<div class="toolbar">
			<Button onclick={() => (sidebarOpen = true)}><Icon src={FolderOpen} size="24" /></Button>
			<Input bind:value={search} placeholder="Search..." />

			<Button color="primary" size="md" variant="outline" onclick={() => (showUpload = true)}>
				Upload
			</Button>
		</div>
		<div class="actions-toolbar">
			<!-- svelte-ignore a11y_click_events_have_key_events -->
			<!-- svelte-ignore a11y_no_static_element_interactions -->
			<div class="btn" onclick={media_list_update}>
				<Icon src={ArrowPath} size="24" />
			</div>
			<div class="row">
				<div class="order-by">
					<select>
						<option value="name">Name</option>
						<option value="date">Date</option>
					</select>
				</div>
				<div class="btn">
					<Icon src={ListBullet} size="24" />
				</div>
			</div>
		</div>

		<!-- folders section -->
		<div class="folders">
			<div class="section-title">
				<!-- svelte-ignore a11y_click_events_have_key_events -->
				<!-- svelte-ignore a11y_no_static_element_interactions -->
				<div class="row" onclick={() => (openFolders = !openFolders)}>
					<div class="btn">
						{#if openFolders}
							<Icon src={ChevronDown} size="24" />
						{:else}
							<Icon src={ChevronRight} size="24" />
						{/if}
					</div>
					Folders
				</div>
				<div>
					{getFolderName(id_folder)}
				</div>
				<div class="select-items">
					{#if subfolders.length <= 0}
						<Button size="sm" mode="danger" onclick={() => deleteFolder()}>Delete</Button>
					{/if}
					{#if id_folder != 'root'}
						<Button size="sm" onclick={() => (showNewFolder = true)}>New Folder</Button>
					{/if}
				</div>
			</div>
			{#if openFolders}
				<div class="folders-data">
					{#if id_folder !== 'root' && id_folder !== 'default-root'}
						<!-- svelte-ignore a11y_click_events_have_key_events -->
						<!-- svelte-ignore a11y_no_static_element_interactions -->
						<div
							class="folder"
							style="width: 40px"
							onclick={() => _updateView(_getIdParent(id_folder))}
						>
							<Icon src={ArrowUturnLeft} size="24" />
						</div>
					{:else}
						<div class="folder" style="width: 40px; cursor: default">
							<Icon src={Stop} size="24" />
						</div>
					{/if}

					{#each subfolders as subfolder (subfolder.id)}
						<!-- svelte-ignore a11y_click_events_have_key_events -->
						<!-- svelte-ignore a11y_no_static_element_interactions -->
						<div class="folder" onclick={() => _updateView(subfolder.id)}>
							<div class="btn">
								<Icon src={Folder} size="24" />
							</div>
							{subfolder.name}
						</div>
					{/each}
				</div>
			{/if}
		</div>

		<!-- assets -->
		<div class="folders">
			<div class="section-title">
				<!-- svelte-ignore a11y_click_events_have_key_events -->
				<!-- svelte-ignore a11y_no_static_element_interactions -->
				<div class="row" onclick={() => (openAssets = !openAssets)}>
					<div class="btn">
						{#if openAssets}
							<Icon src={ChevronDown} size="24" />
						{:else}
							<Icon src={ChevronRight} size="24" />
						{/if}
					</div>
					Assets
				</div>
				<div class="select-items">
					Selected: {selectedCards.length} / {medias.length}
					<Button size="sm" variant="outline" mode="primary" onclick={selectAll}>Select All</Button>
					<Button size="sm" variant="outline" mode="secondary" onclick={selectNone}>None</Button>
					<Button
						size="sm"
						variant="outline"
						mode="danger"
						onclick={() => deleteMedias()}
						disabled={selectedCards.length === 0}
					>
						Delete
					</Button>
				</div>
			</div>
			{#if openAssets}
				<div class="assets-data">
					{#each medias as media (media.id)}
						<div class="card">
							<div
								class="image"
								style={`background-image: url(${media_url(
									media.id ?? '',
									media.filename ?? '',
									true
								)})`}
							>
								<!-- svelte-ignore a11y_click_events_have_key_events -->
								<!-- svelte-ignore a11y_no_static_element_interactions -->
								<div class="card-selector" onclick={() => selectCard(media.id ?? '')}>
									<Icon
										src={media.id && selectedCards.includes(media.id)
											? CheckCircle
											: EllipsisHorizontalCircle}
										size="24"
										style="color: var(--liwe3-light-color)"
									/>
								</div>

								<div class="card-action">
									<Button size="xs" variant="solid" onclick={() => chooseMedia(media)}>
										Choose
									</Button>
								</div>
							</div>
							<div class="title">{media.name}</div>
							<div class="fold">
								<Icon src={Folder} size="16" />
								{getFolderName(media.id_folder ?? '')}
							</div>
							<div class="tags">
								<div class="tag">Tag 1</div>
								<div class="tag">Tag 2</div>
								<div class="tag">Tag 3</div>
							</div>
							<div class="format">
								<div class="row">
									<div class="btn">
										<Icon src={Photo} size="24" />
									</div>
									{(media.filename ?? '').split('.').pop()?.toUpperCase()}
								</div>

								<div class="format-data">Size: {format_size(media.size ?? 0)}</div>
							</div>
						</div>
					{/each}
				</div>
			{/if}
		</div>
	</div>
</div>

{#if showUpload}
	<Modal
		size="lg"
		title="Upload Files"
		oncancel={() => (showUpload = false)}
		closeOnOutsideClick={false}
		closeOnEsc={false}
	>
		<Upload bind:id_folder folders={tree.children} ondone={uploadDone} />
	</Modal>
{/if}

{#if showNewFolder}
	<Modal
		size="md"
		title="New Folder"
		oncancel={() => (showNewFolder = false)}
		closeOnOutsideClick={true}
		closeOnEsc={true}
	>
		<FormCreator fields={newFolderFields} onsubmit={(e) => createNewFolder(e.detail)} />
	</Modal>
{/if}

<style>
	.container {
		display: flex;
		position: relative;

		border: 1px solid var(--liwe3-border-color);
	}

	.sidebar {
		width: 300px;
		position: absolute;
		top: 0;
		left: 0;
		background-color: var(--liwe3-paper);
		height: 100vh;
		z-index: 1002;

		box-shadow: 5px 0 5px 0 rgba(0, 0, 0, 0.3);
	}

	.sidebar .title {
		display: flex;
		justify-content: space-between;
		align-items: center;

		padding: 0.2em 0.5em 0.2em 0.5em;
		border-bottom: 1px solid var(--liwe3-darker-secondary-color);
		margin-top: 0.2em;
		margin-bottom: 0.4em;
	}

	.main {
		width: 100%;
	}

	.toolbar {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 1em;
	}

	.actions-toolbar {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 0.5em;

		margin-top: 5px;
		background-color: var(--liwe3-paper);
		border: 1px solid var(--liwe3-darker-secondary-color);
	}

	.row {
		display: flex;
		align-items: center;
		gap: 0.5em;
	}

	.section-title {
		display: flex;
		align-items: center;
		justify-content: space-between;
		user-select: none;

		margin-top: 1em;
	}

	.folders-data,
	.assets-data {
		display: flex;
		flex-wrap: wrap;
		gap: 1em;

		margin-top: 0.5em;

		padding-left: 1em;
	}

	.assets-data {
		height: calc(100vh - 340px);

		overflow-y: scroll;
	}

	.folder {
		display: flex;
		align-items: center;
		justify-content: flex-start;
		width: 260px;
		padding: 0.5em;

		border: 1px solid var(--liwe3-darker-secondary-color);

		box-shadow: 0 5px 5px 0 rgba(0, 0, 0, 0.6);

		cursor: pointer;
		user-select: none;
	}

	.card {
		border: 1px solid var(--liwe3-darker-secondary-color);
		border-radius: 5px;

		width: 300px;
		height: 330px;

		box-shadow: 0 5px 5px 0 rgba(0, 0, 0, 0.6);

		user-select: none;
	}

	.card .image {
		position: relative;
		border-radius: 5px 5px 0 0;
		background-color: blue;
		height: 180px;

		background-size: cover;
		background-position: top;
	}

	.card .title {
		width: 100%;
		overflow: hidden;
		text-overflow: ellipsis;
		font-size: 0.9em;
		white-space: nowrap;
	}

	.card-selector {
		position: absolute;
		top: 5px;
		left: 5px;

		cursor: pointer;
	}

	.card-action {
		position: absolute;
		top: 5px;
		right: 5px;
	}

	.fold,
	.title {
		padding: 0.5em;
	}

	.fold {
		font-size: 0.8em;
	}

	.tags {
		display: flex;
		flex-wrap: wrap;
		gap: 0.5em;
		font-size: 0.7em;
		border-top: 1px solid var(--liwe3-darker-secondary-color);
		padding: 0.5em;
	}

	.format {
		display: flex;
		justify-content: space-between;
		align-items: center;

		border-top: 1px solid var(--liwe3-darker-secondary-color);

		font-size: 0.8em;
		padding: 0.5em;
	}

	.btn {
		cursor: pointer;
	}

	.select-items {
		display: flex;
		align-items: center;
		gap: 0.5em;
		font-size: 0.8em;
		padding-right: 0.5em;
	}
</style>
