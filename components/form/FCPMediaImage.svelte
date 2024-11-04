<script lang="ts">
	import { createEventDispatcher } from 'svelte';

	import Modal from '$liwe3/components/Modal.svelte';
	import { media_url } from '$liwe3/utils/utils';
	import { media_get } from '$modules/mediamanager/actions';
	import type { Media } from '$modules/mediamanager/types';
	import { onMount } from 'svelte';
	import MediaManager from '../MediaManager.svelte';
	import Button from '$liwe3/components/Button.svelte';
	import ImageContained from '$liwe3/components/ImageContained.svelte';

	interface Props {
		name?: string;
		value?: string;
	}

	let { name = '', value = $bindable('') }: Props = $props();

	let media: Media | undefined = $state();
	let imgURL: string = $state('');

	let showMediaManager: boolean = $state(false);

	const dispatch = createEventDispatcher();

	const _load_media = async () => {
		if (!value) return;

		const res = await media_get(value);

		if (res.error) return;

		media = res;
	};

	const onMediaChoose = (e: CustomEvent) => {
		media = e.detail;
		value = media.id ?? '';
		showMediaManager = false;

		dispatch('change', value);
	};

	onMount(async () => {
		_load_media();
	});

	$effect(() => {
		if (media && media.id && media.filename) {
			imgURL = media_url(media.id, media.filename);
		} else {
			imgURL = '';
		}
	});
</script>

<input type="hidden" {name} bind:value />
<div class="container">
	<Button
		variant="link"
		mode="success"
		on:click={(e) => {
			e.preventDefault();
			showMediaManager = true;
		}}
	>
		{#if imgURL}
			<ImageContained url={imgURL} size="contain" />
		{:else}
			Select image
		{/if}
	</Button>
</div>

{#if showMediaManager}
	<Modal title="Media Manager" size="lg" on:cancel={() => (showMediaManager = false)}>
		<MediaManager on:choose={onMediaChoose} />
	</Modal>
{/if}

<style>
	.container {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		height: 100%;
		width: 100%;

		border: 1px solid var(--liwe3-border-color);
		border-radius: var(--liwe3-border-radius);
		padding: 4px;
	}
</style>
