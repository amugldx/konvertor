<script lang="ts">
	import { browser } from '$app/environment';
	import { FFmpeg } from '@ffmpeg/ffmpeg';
	import { confetti } from '@neoconfetti/svelte';
	import { onMount } from 'svelte';
	import { tweened } from 'svelte/motion';
	import { fade } from 'svelte/transition';

	type State = 'loading' | 'loaded' | 'convert.start' | 'convert.error' | 'convert.done';

	let state: State = 'loading';
	let error = '';
	let ffmpeg: FFmpeg;
	let progress = tweened(0);

	async function readFile(file: File): Promise<Uint8Array> {
		return new Promise((resolve) => {
			const fileReader = new FileReader();

			fileReader.onload = () => {
				const { result } = fileReader;
				if (result instanceof ArrayBuffer) {
					resolve(new Uint8Array(result));
				}
			};

			fileReader.onerror = () => {
				error = 'Could not read file';
			};

			fileReader.readAsArrayBuffer(file);
		});
	}

	async function convertVideo(video: File) {
		state = 'convert.start';
		const videoData = await readFile(video);

		await ffmpeg.writeFile('input.webm', videoData);
		await ffmpeg.exec(['-i', 'input.webm', 'output.mp4']);
		const data = await ffmpeg.readFile('output.mp4');
		state = 'convert.done';

		return data as Uint8Array;
	}

	function downloadVideo(data: Uint8Array) {
		if (browser) {
			const a = document.createElement('a');
			a.href = URL.createObjectURL(new Blob([data.buffer], { type: 'video/mp4' }));
			a.download = 'video.mp4';

			setTimeout(() => {
				a.click();
			}, 1000);
		}
	}

	async function handleDrop(event: DragEvent) {
		if (!event.dataTransfer) return;

		if (event.dataTransfer?.files.length > 1) {
			error = 'Upload one file at a time.';
		}

		if (event.dataTransfer.files[0].type === 'video/webm') {
			error = '';
			const [file] = event.dataTransfer.files;
			const data = await convertVideo(file);
			downloadVideo(data);
		} else {
			error = 'Only .webm file is supported';
		}
	}

	async function loadFFmpeg() {
		const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.4/dist/esm';

		ffmpeg = new FFmpeg();

		ffmpeg.on('progress', (event) => {
			$progress = event.progress * 100;
		});

		await ffmpeg.load({
			coreURL: `${baseURL}/ffmpeg-core.js`,
			wasmURL: `${baseURL}/ffmpeg-core.wasm`
		});

		state = 'loaded';
	}

	onMount(() => {
		loadFFmpeg();
	});
</script>

<h1 class="title">Webm to Mp4 Convertor</h1>
<p class="sub-heading">
	Made with &#129505 by
	<a href="https://github.com/amugldx" target="_blank">@amugldx</a>
</p>
<!-- svelte-ignore a11y-no-static-element-interactions -->
<div
	class="drop"
	on:drop|preventDefault={handleDrop}
	on:dragover|preventDefault={() => {}}
	data-state={state}
>
	{#if state === 'loading'}
		<p in:fade>Loading FFmpeg...</p>
	{/if}

	{#if state === 'loaded'}
		<p in:fade>Drag video here...</p>
	{/if}

	{#if state === 'convert.start'}
		<p in:fade>Converting video...</p>

		<div class="progress-bar">
			<div class="progress" style:--progress="{$progress}%">wait...</div>
		</div>
	{/if}

	{#if state === 'convert.done'}
		<div use:confetti />
		<p in:fade>Done... &#127881;</p>
	{/if}

	{#if error}
		<p class="error" in:fade>{error}</p>
	{/if}
</div>

<style>
	.title {
		text-align: center;
	}

	.sub-heading {
		text-align: center;
	}

	.drop {
		width: 600px;
		height: 400px;
		display: grid;
		place-content: center;
		margin-block-start: 2rem;
		border: 10px dashed hsl(220 10% 20%);
		border-radius: 8px;

		& p {
			font-size: 2rem;
			text-align: center;

			&.error {
				color: hsl(9 100% 64%);
			}
		}
	}

	.progress-bar {
		--progress-bar-clr: hsl(180 100% 50%);
		--progress-txt-clr: hsl(0 0% 0%);

		width: 300px;
		height: 40px;
		position: relative;
		font-weight: 700;
		background-color: hsl(200 10% 14%);
		border-radius: 8px;

		& .progress {
			width: var(--progress);
			height: 100%;
			position: absolute;
			left: 0%;
			display: grid;
			place-content: center;
			background: var(--progress-bar-clr);
			color: var(--progress-txt-clr);
			border-radius: 8px;
		}
	}
</style>
