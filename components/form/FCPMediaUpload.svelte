<script lang="ts">
	import { media_url } from '$liwe3/utils/utils';
	import { media_get } from '$modules/mediamanager/actions';
	import type { Media } from '$modules/mediamanager/types';
	import { onMount } from 'svelte';
	import type { FormField } from '$liwe3/components/FormCreator.svelte';
	import Upload from '../Upload.svelte';
	import { type MediaManagerItem } from './FCPMediaImage.svelte';

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
	let upload: any;

	export const submit = async () => {
		if (!upload) return;

		const res = await upload.submit();

		if (res && res.length > 0) {
			media = res[0];
			onchange(name, res, field);
		}

		return true;
	};

	const _load_media = async () => {
		if (!value) return;

		const v = (value as MediaManagerItem).id ?? value;

		const res = await media_get(v);

		if (res?.error) return;

		media = res;
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
	<Upload bind:this={upload} showUploadButton={false} />
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
