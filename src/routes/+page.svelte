<script lang="ts">
	import { tweened } from 'svelte/motion';
	import { fade } from 'svelte/transition';
	import { FFmpeg } from '@ffmpeg/ffmpeg';
	import { onMount } from 'svelte';
	import { confetti } from '@neoconfetti/svelte';

	type State = 'loading' | 'loaded' | 'convert.start' | 'convert.done' | 'convert.error';
	let state: State = 'loading';
	let error = '';
	let ffmpeg: FFmpeg;
	let progress = tweened(0);

	async function download(data: Uint8Array) {
		const a = document.createElement('a');
		const blob = new Blob([data.buffer], { type: 'video/mp4' });
		const url = URL.createObjectURL(blob);
		a.href = url;
		a.download = 'output.mp4';
		setTimeout(() => {
			a.click();
		}, 1000);
		// URL.revokeObjectURL(url);
	}

	async function readFile(file: File): Promise<Uint8Array> {
		return new Promise((resolve) => {
			const reader = new FileReader();
			reader.onload = () => {
				const { result } = reader;
				if (result instanceof ArrayBuffer) {
					resolve(new Uint8Array(result));
				}
			};

			reader.onerror = () => {
				error = 'Could not read file';
			};

			reader.readAsArrayBuffer(file);
		});
	}
	async function ConvetVideo(video: File) {
		state = 'convert.start';
		const videoData = await readFile(video);
		await ffmpeg.writeFile('input.webm', videoData);
		await ffmpeg.exec(['-i', 'input.webm', 'output.mp4']);
		const data = await ffmpeg.readFile('output.mp4');
		state = 'convert.done';
		return data as Uint8Array;
	}
	async function handleFileDrop(ev: DragEvent) {
		if (!ev.dataTransfer) return;
		if (ev.dataTransfer.files.length > 1) {
			error = 'One file at a time';
		}

		if (ev.dataTransfer.files[0].type === 'video/webm') {
			error = '';
			console.log('error :', error);
			const [file] = ev.dataTransfer.files;
			const data = await ConvetVideo(file);
			download(data);
			console.log(file);
		} else {
			error = 'Only webM is supported';
		}
	}

	async function FFmpegLoader() {
		console.log('Loading FFmpeg...');
		const baseURL = ' https://unpkg.com/@ffmpeg/core@0.12.4/dist/esm';
		ffmpeg = new FFmpeg();
		ffmpeg.on('progress', (ev: any) => {
			$progress = ev.progress * 100;
		});

		await ffmpeg.load({
			coreURL: `${baseURL}/ffmpeg-core.js`,
			wasmURL: `${baseURL}/ffmpeg-core.wasm`
		});
		console.log('FFmpeg loaded successfully!');
		state = 'loaded';
	}

	onMount(() => {
		FFmpegLoader();
	});
	// $: state;
</script>

<svelte:head>
	<title>Home</title>
	<meta name="description" content="Svelte demo app" />
</svelte:head>

<main>
	<h1 class="text-5xl text-center">WEBM to MP4</h1>
	<!-- svelte-ignore a11y-no-static-element-interactions -->
	<div
		on:drop|preventDefault={handleFileDrop}
		on:dragover|preventDefault={() => {}}
		data-state={state}
		class="w-[600px] h-[600px] grid place-content-center mt-8 border-dashed border-4 border-black rounded-lg"
	>
		{#if state === 'loading'}
			<p in:fade class="ml-1 text-lg text-center">Loading...</p>
		{/if}
		{#if state === 'loaded'}
			<p in:fade class="ml-1 text-lg text-center">Drag your video here</p>
		{/if}
		{#if state === 'convert.start'}
			<p in:fade class="ml-1 text-lg text-center">Conversion in progress...</p>
			<div class="progress-bar">
				<div class="progress" style:--progress={$progress}>
					{$progress.toFixed(0)}%
				</div>
			</div>
		{/if}
		{#if state === 'convert.done'}
			<div use:confetti />
			<p in:fade class="ml-1 text-lg text-center">Done 🎊🎉</p>
		{/if}
		{#if state === error}
			<p in:fade class="ml-1 text-lg text-center text-red-600">{error}</p>
		{/if}
		<!-- <div class=""></div> -->
	</div>
</main>

<style lang="postcss">
	.progress-bar {
		--progress-bar-clr: hsl(180, 58%, 50%);
		--progress-bar-txt-clr: hsl(0, 0%, 0%);

		width: 300px;
		height: 40px;
		position: relative;
		font-weight: 700;
		border-radius: 8px;
		background-color: hsl(200, 10%, 14%);

		& .progress {
			width: var(--progress);
			height: 100%;
			position: absolute;
			left: 0;
			display: grid;
			place-content: center;
			background-color: var(--progress-bar-clr);
			border-radius: 8px;
		}
	}

	/* :has([data-state='convert.done']) {
		--txt-clr: hsl(0, 0%, 0%);
		--bg-clr: hsl(60, 100%, 50%);
	}

	:has([data-state='convert.error']) {
		--txt-clr: hsl(0, 0%, 0%);
		--bg-clr: hsl(10, 100%, 50%);
	}

	:has([data-state='convert.start']) {
		--txt-clr: hsl(0, 0%, 0%);
		--bg-clr: hsl(162, 75%, 51%);
	} */
</style>
