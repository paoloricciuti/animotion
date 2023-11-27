<script lang="ts">
	type State = 'ready' | 'ready.countdown' | 'recording'
	type Optional<T> = T | undefined

	export let width: Optional<number> = undefined
	export let height: Optional<number> = undefined
	export let audio = true
	export let frameRate = 60

	let state: State = 'ready'
	let recorder: MediaRecorder
	let stream: MediaStream
	let videoChunks = [] as Blob[]
	let timerInterval: NodeJS.Timeout
	let timer = 3

	async function startTimer() {
		state = 'ready.countdown'

		return new Promise((resolve) => {
			timerInterval = setInterval(() => {
				timer--
				if (timer === 0) {
					clearInterval(timerInterval)
					resolve(timer)
				}
			}, 1000)
		})
	}

	function stopRecording() {
		state = 'ready'
		stream.getTracks().forEach((track) => track.stop())
		recorder && recorder.stop()
		clearInterval(timerInterval)
		timer = 3
	}

	async function startRecording() {
		if (!stream) return

		state = 'recording'
		recorder = new MediaRecorder(stream, { mimeType: 'video/webm;codecs=vp9' })
		recorder.start()

		recorder.addEventListener('dataavailable', (event) => {
			videoChunks.push(event.data)
		})

		recorder.addEventListener('stop', () => {
			downloadRecording(videoChunks)
			videoChunks = []
		})
	}

	async function prepareRecording() {
		try {
			stream = await navigator.mediaDevices.getDisplayMedia({
				video: { width, height, frameRate },
				audio,
				// @ts-ignore
				preferCurrentTab: true,
			})
			stream.addEventListener('inactive', stopRecording)
			await startTimer()
			startRecording()
		} catch (error) {
			console.error(`Error: ${error}`)
		}
	}

	function downloadRecording(videoChunks: Blob[]) {
		const blob = new Blob(videoChunks, { type: 'video/webm;codecs=vp9' })
		const a = document.createElement('a')
		a.href = URL.createObjectURL(blob)
		a.download = 'video.webm'
		a.click()
	}

	function handleKeydown(event: KeyboardEvent) {
		switch (event.key) {
			case 'R':
				if (state === 'ready') {
					prepareRecording()
				}
				break
			case 'S':
				if (state !== 'ready') {
					stopRecording()
				}
				break
		}
	}
</script>

<svelte:window on:keydown={handleKeydown} />

{#if state.includes('ready')}
	<div class="recorder" data-state={state}>
		<button on:click={prepareRecording} class="record">
			<div class="circle">
				{#if state === 'ready.countdown'}
					{timer}
				{/if}
			</div>
		</button>

		<div class="info">
			{#if state === 'ready'}
				<p class="shortcut">Shift + R</p>
				<p class="description">To start recording</p>
			{/if}

			{#if state === 'ready.countdown'}
				<p class="shortcut">Shift + S</p>
				<p class="description">To stop recording</p>
			{/if}
		</div>
	</div>
{/if}

<style>
	.recorder {
		--txt-clr: hsl(0 0%, 98%);
		--circle-bg-clr: hsl(0 100% 60%);
		--circle-border-clr: hsl(0 0% 98%);

		position: absolute;
		left: 50%;
		bottom: 40px;
		color: var(--txt-clr);
		text-align: center;
		z-index: 10;
	}

	[data-state='ready.countdown'] {
		& .circle {
			--txt-clr: hsl(0 0% 10%);
			--circle-bg-clr: hsl(0 0% 98%);
		}
	}

	[data-state='ready.canceled'] {
		& .circle {
			--txt-clr: hsl(0 0% 10%);
			--circle-bg-clr: green;
		}
	}

	.record {
		width: 80px;
		height: 80px;
		padding: 4px;
		border: 4px solid var(--circle-border-clr);
		border-radius: 50%;
	}

	.circle {
		width: 100%;
		height: 100%;
		display: grid;
		place-content: center;
		font-family: var(--r-main-font);
		font-size: 2rem;
		font-weight: 700;
		color: var(--txt-clr);
		background: var(--circle-bg-clr);
		border-radius: 50%;
		transition: background-color 0.6s;

		&:hover {
			--circle-bg-clr: hsl(0 100% 64%);
		}
	}

	.info {
		margin-block-start: 1rem;

		& .shortcut {
			font-weight: 700;
		}

		& .description {
			margin-block-start: 4px;
			opacity: 0.6;
		}
	}
</style>
