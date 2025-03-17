<script module>
	export type MediaManagerItem = {
		id: string;
		filename: string;
		mimetype: string;
	};
</script>

<script lang="ts">
	import Modal from '$liwe3/components/Modal.svelte';
	import { media_url } from '$liwe3/utils/utils';
	import { media_get } from '$modules/mediamanager/actions';
	import type { Media } from '$modules/mediamanager/types';
	import { onMount } from 'svelte';
	import MediaManager from '../MediaManager.svelte';
	import Button from '$liwe3/components/Button.svelte';
	import ImageContained from '$liwe3/components/ImageContained.svelte';
	import type { FormField } from '$liwe3/components/FormCreator.svelte';
	import Upload from '../Upload.svelte';

	interface Props {
		field: FormField;
		name?: string;
		value?: string | MediaManagerItem;

		// dependency injection
		_v: (field: FormField) => any;

		// events
		onchange: (name: string, value: any, field: FormField) => void;
	}

	let { field, name = '', value = $bindable(''), onchange }: Props = $props();

	let media: Media | undefined = $state();
	let imgURL: string = $state('');

	let showMediaManager: boolean = $state(false);

	const _load_media = async () => {
		if (!value) return;

		const v = (value as MediaManagerItem).id ?? value;

		const res = await media_get(v);

		if (res.error) return;

		media = res;
	};

	const onMediaChoose = (media: Media) => {
		if (!media) return;

		value = media.id ?? '';

		showMediaManager = false;

		const v = { id: media.id, filename: media.filename, type: media.mimetype };

		onchange && onchange(name, v, field);
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
<div class="fcp-upload-cont">
	<Upload showUploadButton={false} />
</div>

<style>
	.fcp-upload-cont {
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
