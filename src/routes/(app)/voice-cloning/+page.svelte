<script>
	import { toast } from 'svelte-sonner';
    import { fade } from 'svelte/transition';
    import { getContext } from 'svelte';
    import { onMount } from 'svelte';
    
    const i18n = getContext('i18n');

	let voiceName = '';
	/** @type {Array<File>} */
	let selectedFiles = [];
	let removeNoise = true;
	let isLoading = false;
	let result = null;
	let activeTab = 'upload'; // Default to upload tab
	
	// Recording state with proper initialization
	/** @type {MediaRecorder|null} */
	let mediaRecorder = null;
	/** @type {Array<Blob>} */
	let audioChunks = [];
	let isRecording = false;
	/** @type {Blob|null} */
	let audioBlob = null;
	/** @type {string|null} */
	let recordedAudio = null;
	/** @type {AudioContext|null} */
	let audioContext = null;
	/** @type {AnalyserNode|null} */
	let analyser = null;
	let dataArray = new Uint8Array();
	/** @type {CanvasRenderingContext2D|null} */
	let canvasContext = null;
	/** @type {number|null} */
	let animationFrame = null;
	
	// Audio device selection
	/** @type {Array<{deviceId: string, label: string}>} */
	let audioInputDevices = [];
	let selectedAudioDeviceId = 'default';
	let deviceLoadingError = false;
	
	onMount(() => {
		if (typeof window !== 'undefined') {
			const canvas = document.getElementById('visualizer');
			if (canvas) {
				canvasContext = canvas.getContext('2d');
			}
			
			// Get available audio input devices
			loadAudioDevices();
		}
		
		return () => {
			if (animationFrame) {
				cancelAnimationFrame(animationFrame);
			}
			
			if (audioContext) {
				audioContext.close();
			}
		};
	});
	
	// Function to load available audio input devices
	async function loadAudioDevices() {
		try {
			// First request permission to access devices
			await navigator.mediaDevices.getUserMedia({ audio: true })
				.then(stream => {
					// Stop the stream immediately after getting permission
					stream.getTracks().forEach(track => track.stop());
				});
			
			// Now get the list of devices
			const devices = await navigator.mediaDevices.enumerateDevices();
			audioInputDevices = devices
				.filter(device => device.kind === 'audioinput')
				.map(device => ({
					deviceId: device.deviceId,
					label: device.label || `Microphone (${device.deviceId.slice(0, 8)}...)`
				}));
			
			// If we have devices but no default selected yet, select the first one
			if (audioInputDevices.length > 0 && selectedAudioDeviceId === 'default') {
				selectedAudioDeviceId = audioInputDevices[0].deviceId;
			}
			
			deviceLoadingError = false;
		} catch (error) {
			console.error('Error accessing media devices:', error);
			deviceLoadingError = true;
			toast.error('Failed to load audio devices. Please check your browser permissions.');
		}
	}
	
	// Handle device selection change
	function handleDeviceChange(event) {
		selectedAudioDeviceId = event.target.value;
		
		// If recording is in progress, stop it when device changes
		if (isRecording) {
			stopRecording();
		}
	}
	
	// Function to start recording
	async function startRecording() {
		try {
			const stream = await navigator.mediaDevices.getUserMedia({ 
				audio: { 
					deviceId: { exact: selectedAudioDeviceId } 
				} 
			});
			
			// Set up audio context for visualization
			audioContext = new (window.AudioContext || window.webkitAudioContext)();
			analyser = audioContext.createAnalyser();
			const source = audioContext.createMediaStreamSource(stream);
			source.connect(analyser);
			analyser.fftSize = 256;
			
			const bufferLength = analyser.frequencyBinCount;
			dataArray = new Uint8Array(bufferLength);
			
			// Start visualization
			visualize();
			
			mediaRecorder = new MediaRecorder(stream);
			
			mediaRecorder.ondataavailable = (event) => {
				audioChunks.push(event.data);
			};
			
			mediaRecorder.onstop = () => {
				audioBlob = new Blob(audioChunks, { type: 'audio/wav' });
				const audioURL = URL.createObjectURL(audioBlob);
				if (recordedAudio) {
					URL.revokeObjectURL(recordedAudio);
				}
				recordedAudio = audioURL;
				
				// Cleanup streams
				mediaRecorder.stream.getTracks().forEach(track => track.stop());
				
				// Stop visualization
				if (animationFrame) {
					cancelAnimationFrame(animationFrame);
				}
			};
			
			audioChunks = [];
			mediaRecorder.start();
			isRecording = true;
		} catch (error) {
			toast.error(`Microphone access error: ${error.message}`);
		}
	}
	
	function stopRecording() {
		if (mediaRecorder && isRecording) {
			mediaRecorder.stop();
			isRecording = false;
		}
	}
	
	function visualize() {
		if (!analyser || !canvasContext) return;
		
		const canvas = canvasContext.canvas;
		const WIDTH = canvas.width;
		const HEIGHT = canvas.height;
		
		canvasContext.clearRect(0, 0, WIDTH, HEIGHT);
		
		function draw() {
			animationFrame = requestAnimationFrame(draw);
			
			analyser.getByteTimeDomainData(dataArray);
			
			canvasContext.fillStyle = 'rgb(30, 30, 30)';
			canvasContext.fillRect(0, 0, WIDTH, HEIGHT);
			
			canvasContext.lineWidth = 2;
			canvasContext.strokeStyle = 'rgb(0, 200, 200)';
			canvasContext.beginPath();
			
			const sliceWidth = WIDTH / dataArray.length;
			let x = 0;
			
			for (let i = 0; i < dataArray.length; i++) {
				const v = dataArray[i] / 128.0;
				const y = v * HEIGHT / 2;
				
				if (i === 0) {
					canvasContext.moveTo(x, y);
				} else {
					canvasContext.lineTo(x, y);
				}
				
				x += sliceWidth;
			}
			
			canvasContext.lineTo(WIDTH, HEIGHT / 2);
			canvasContext.stroke();
		}
		
		draw();
	}
	
	// Submit recording
	async function submitRecording() {
		if (!voiceName) {
			toast.error('Please provide a name for your voice clone');
			return;
		}
		
		if (!audioBlob) {
			toast.error('Please record audio first');
			return;
		}
		
		isLoading = true;
		result = null;
		
		try {
			const formData = new FormData();
			formData.append('name', voiceName);
			formData.append('remove_background_noise', removeNoise.toString());
			
			// Create a File object from the Blob
			const audioFile = new File([audioBlob], `recording-${Date.now()}.wav`, { type: 'audio/wav' });
			formData.append('files[]', audioFile);
			
			const response = await fetch('/api/audio/clone-voice', {
				method: 'POST',
				body: formData
			});
			
			if (!response.ok) {
				const errorData = await response.json();
				throw new Error(errorData.detail || 'Failed to clone voice');
			}
			
			result = await response.json();
			toast.success('Voice cloned successfully!');
		} catch (error) {
			toast.error(error.message || 'Failed to clone voice');
		} finally {
			isLoading = false;
		}
	}
	
	async function handleSubmit() {
		if (!voiceName) {
			toast.error('Please provide a name for your voice clone');
			return;
		}
		
		if (selectedFiles.length < 1) {
			toast.error('Please upload at least one audio file');
			return;
		}
		
		isLoading = true;
		result = null;
		
		try {
			const formData = new FormData();
			formData.append('name', voiceName);
			formData.append('remove_background_noise', removeNoise.toString());
			
			selectedFiles.forEach(file => {
				formData.append('files[]', file);
			});
			
			const response = await fetch('/api/audio/clone-voice', {
				method: 'POST',
				body: formData
			});
			
			if (!response.ok) {
				const errorData = await response.json();
				throw new Error(errorData.detail || 'Failed to clone voice');
			}
			
			result = await response.json();
			toast.success('Voice cloned successfully!');
		} catch (error) {
			toast.error(error.message || 'Failed to clone voice');
		} finally {
			isLoading = false;
		}
	}
	
	function handleFileSelect(event) {
		const files = Array.from(event.target.files);
		const audioFiles = files.filter(file => 
			file.type.startsWith('audio/') || 
			file.name.endsWith('.mp3') || 
			file.name.endsWith('.wav')
		);
		
		if (audioFiles.length !== files.length) {
			toast.warning('Some files were skipped because they are not audio files');
		}
		
		selectedFiles = [...selectedFiles, ...audioFiles];
	}
	
	function removeFile(index) {
		selectedFiles = selectedFiles.filter((_, i) => i !== index);
	}
	
	// Text for the user to read in Portuguese
	const portugueseEssay = [
		"O autoconhecimento é a base para o desenvolvimento pessoal e profissional. Conhecer a si mesmo significa entender suas emoções, motivações, pontos fortes e fracos. Esse processo permite que enfrentemos desafios com mais segurança e façamos escolhas alinhadas com nossos valores e objetivos.",
		"A inteligência emocional complementa o autoconhecimento e nos permite navegar pelo complexo universo das emoções. Ela envolve reconhecer não apenas nossas próprias emoções, mas também as dos outros, desenvolvendo empatia e aprimorando nossos relacionamentos interpessoais.",
		"Quando desenvolvemos autoconhecimento, ganhamos clareza sobre nossos propósitos. Entendemos o que nos motiva e o que nos afasta de nossos objetivos. Esse entendimento nos ajuda a estabelecer metas realistas e significativas, além de nos dar a determinação necessária para alcançá-las.",
		"A prática da autorreflexão é essencial para o autoconhecimento. Reservar um tempo para examinar nossos pensamentos, ações e reações nos permite identificar padrões de comportamento que podem ser benéficos ou prejudiciais. Com essa consciência, podemos trabalhar ativamente para fortalecer hábitos positivos e transformar os negativos.",
		"O autoconhecimento também nos ajuda a lidar melhor com o estresse e as adversidades. Ao entender como reagimos em diferentes situações, podemos desenvolver estratégias eficazes de enfrentamento. Isso não apenas melhora nossa saúde mental, mas também fortalece nossa resiliência diante dos desafios da vida.",
		"A inteligência emocional nos permite reconhecer quando estamos sendo influenciados por emoções intensas, permitindo que façamos pausas antes de tomar decisões importantes. Essa consciência evita que sejamos dominados por reações impulsivas, que muitas vezes levam a arrependimentos.",
		"No ambiente de trabalho, pessoas com alto grau de autoconhecimento e inteligência emocional tendem a ser líderes mais eficazes. Elas comunicam-se melhor, inspiram confiança e criam ambientes onde os outros se sentem valorizados e compreendidos. Isso leva a equipes mais produtivas e engajadas.",
		"O autoconhecimento também é fundamental para o estabelecimento de limites saudáveis em nossas relações. Quando sabemos quem somos e o que precisamos, podemos expressar isso de forma clara e respeitosa aos outros, criando relacionamentos mais autênticos e equilibrados.",
		"A jornada do autoconhecimento é contínua e exige honestidade e coragem. É preciso estar disposto a encarar aspectos de si mesmo que podem ser desconfortáveis ou dolorosos. No entanto, essa coragem traz consigo a liberdade de viver de forma mais consciente e autêntica.",
		"Por fim, o autoconhecimento e a inteligência emocional nos conectam mais profundamente com nossa humanidade. Ao compreendermos nossas próprias lutas e vulnerabilidades, desenvolvemos maior compaixão por nós mesmos e pelos outros, contribuindo para um mundo mais empático e acolhedor."
	];
</script>

<div class="container mx-auto px-4 py-8">
	<div class="max-w-3xl mx-auto">
		<h1 class="text-3xl font-bold mb-8">{$i18n.t('Voice Cloning with TU•IPSE')}</h1>
		
		<div class="bg-popover rounded-lg p-6 mb-8 shadow-sm">
			<p class="text-sm text-muted-foreground mb-4">
				{$i18n.t('Clone your voice using TU•IPSE AI. You can either upload audio files or record your voice directly in the browser.')}
			</p>
		</div>
		
		<!-- Tabs -->
		<div class="border-b border-input mb-6">
			<div class="flex -mb-px">
				<button 
					class={`py-2 px-4 font-medium ${activeTab === 'upload' ? 'border-b-2 border-primary text-primary' : 'text-muted-foreground'}`} 
					on:click={() => activeTab = 'upload'}
				>
					{$i18n.t('Upload Recordings')}
				</button>
				<button 
					class={`py-2 px-4 font-medium ${activeTab === 'record' ? 'border-b-2 border-primary text-primary' : 'text-muted-foreground'}`} 
					on:click={() => activeTab = 'record'}
				>
					{$i18n.t('Record Now')}
				</button>
			</div>
		</div>
		
		<!-- Voice Name (common for both tabs) -->
		<div class="mb-6">
			<label for="voice-name" class="block text-sm font-medium mb-2">
				{$i18n.t('Voice Name')}
			</label>
			<input
				id="voice-name"
				type="text"
				bind:value={voiceName}
				required
				class="w-full p-2.5 rounded-md border border-input bg-background text-sm"
				placeholder={$i18n.t('My Cool Voice')}
			/>
		</div>
		
		<!-- Upload Tab Content -->
		{#if activeTab === 'upload'}
			<form on:submit|preventDefault={handleSubmit} class="space-y-6">
				<!-- File Upload -->
				<div>
					<label for="audio-files" class="block text-sm font-medium mb-2">
						{$i18n.t('Audio Files')}
					</label>
					
					<div class="border border-dashed border-input rounded-lg p-8 text-center mb-4">
						<input 
							type="file" 
							id="audio-files" 
							multiple 
							accept="audio/*,.mp3,.wav" 
							on:change={handleFileSelect}
							class="hidden" 
						/>
						<label for="audio-files" class="cursor-pointer">
							<div class="flex flex-col items-center justify-center gap-2">
								<svg xmlns="http://www.w3.org/2000/svg" class="h-10 w-10 text-muted-foreground" fill="none" viewBox="0 0 24 24" stroke="currentColor">
									<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12" />
								</svg>
								<span class="text-sm font-medium">{$i18n.t('Click to upload audio files')}</span>
								<span class="text-xs text-muted-foreground">{$i18n.t('MP3 or WAV, 1-30 seconds per clip')}</span>
							</div>
						</label>
					</div>
					
					<!-- Selected Files List -->
					{#if selectedFiles.length > 0}
						<div class="space-y-3 mb-4">
							<h3 class="text-sm font-medium">{$i18n.t('Selected Files')} ({selectedFiles.length})</h3>
							<ul class="space-y-2">
								{#each selectedFiles as file, index}
									<li class="flex items-center justify-between bg-background/50 p-2 rounded-md">
										<div class="flex items-center">
											<span class="text-sm">{file.name}</span>
											<span class="text-xs text-muted-foreground ml-2">
												{(file.size / 1024 / 1024).toFixed(2)} MB
											</span>
										</div>
										<button 
											type="button" 
											on:click={() => removeFile(index)}
											class="text-destructive hover:text-destructive/80" 
											aria-label={$i18n.t('Remove file')}
										>
											<svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
												<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
											</svg>
										</button>
									</li>
								{/each}
							</ul>
						</div>
					{/if}
				</div>
				
				<!-- Noise Removal Option -->
				<div class="flex items-center">
					<input
						id="remove-noise"
						type="checkbox"
						bind:checked={removeNoise}
						class="h-4 w-4 rounded border-input bg-background"
					/>
					<label for="remove-noise" class="ml-2 block text-sm">
						{$i18n.t('Remove background noise')}
					</label>
				</div>
				
				<!-- Submit Button -->
				<div>
					<button
						type="submit"
						disabled={isLoading}
						class="w-full px-4 py-2.5 rounded-md bg-primary font-medium text-white hover:bg-primary/90 disabled:opacity-50 disabled:cursor-not-allowed"
					>
						{#if isLoading}
							<span class="flex items-center justify-center">
								<svg class="animate-spin -ml-1 mr-2 h-4 w-4 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
									<circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
									<path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
								</svg>
								{$i18n.t('Processing...')}
							</span>
						{:else}
							{$i18n.t('Clone Voice')}
						{/if}
					</button>
				</div>
			</form>
		{:else}
			<!-- Record Now Tab Content -->
			<div class="space-y-6">
				<!-- Reading Text -->
				<div class="bg-background/50 rounded-md p-4 max-h-60 overflow-y-auto mb-4 border border-input">
					<h3 class="text-lg font-medium mb-3">{$i18n.t('Read the text below (Portuguese)')}</h3>
					{#each portugueseEssay as paragraph, i}
						<p class="mb-3 text-sm">
							<strong>{i+1}.</strong> {paragraph}
						</p>
					{/each}
				</div>
				
				<!-- Microphone Device Selection -->
				<div class="mb-4">
					<label for="audio-device" class="block text-sm font-medium mb-2">
						{$i18n.t('Microphone')}
					</label>
					
					{#if deviceLoadingError}
						<div class="text-sm text-destructive mb-2">
							{$i18n.t('Failed to load audio devices. Please ensure you have given microphone permission.')}
							<button 
								type="button" 
								on:click={loadAudioDevices}
								class="ml-2 text-primary underline"
							>
								{$i18n.t('Retry')}
							</button>
						</div>
					{/if}
					
					<div class="relative">
						<select
							id="audio-device"
							bind:value={selectedAudioDeviceId}
							on:change={handleDeviceChange}
							class="w-full p-2.5 rounded-md border border-input bg-background text-sm appearance-none"
							disabled={isRecording || audioInputDevices.length === 0}
						>
							{#if audioInputDevices.length === 0}
								<option value="default">{$i18n.t('No microphones available')}</option>
							{:else}
								{#each audioInputDevices as device}
									<option value={device.deviceId}>{device.label}</option>
								{/each}
							{/if}
						</select>
						<div class="pointer-events-none absolute inset-y-0 right-0 flex items-center px-2 text-muted-foreground">
							<svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
							</svg>
						</div>
					</div>
					<div class="text-xs text-muted-foreground mt-1">
						{$i18n.t('Select your microphone device. Changing the device will stop any ongoing recording.')}
					</div>
				</div>
				
				<!-- Visualization Canvas -->
				<div class="mb-4">
					<canvas id="visualizer" width="640" height="100" class="w-full bg-background/20 rounded-md"></canvas>
				</div>
				
				<!-- Record Button -->
				<div class="flex justify-center mb-6">
					{#if !isRecording}
						<button 
							type="button"
							on:click={startRecording}
							class="flex items-center justify-center w-16 h-16 rounded-full bg-primary hover:bg-primary/90 text-white"
							disabled={isLoading || audioInputDevices.length === 0}
						>
							<svg xmlns="http://www.w3.org/2000/svg" class="h-8 w-8" fill="none" viewBox="0 0 24 24" stroke="currentColor">
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 11a7 7 0 01-7 7m0 0a7 7 0 01-7-7m7 7v4m0 0H8m4 0h4m-4-8a3 3 0 01-3-3V5a3 3 0 116 0v6a3 3 0 01-3 3z" />
							</svg>
						</button>
					{:else}
						<button 
							type="button"
							on:click={stopRecording}
							class="flex items-center justify-center w-16 h-16 rounded-full bg-destructive hover:bg-destructive/90 text-white"
						>
							<svg xmlns="http://www.w3.org/2000/svg" class="h-8 w-8" fill="none" viewBox="0 0 24 24" stroke="currentColor">
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
								<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 10a1 1 0 011-1h4a1 1 0 011 1v4a1 1 0 01-1 1h-4a1 1 0 01-1-1v-4z" />
							</svg>
						</button>
					{/if}
				</div>
				
				<!-- Audio Playback if recorded -->
				{#if recordedAudio}
					<div class="mb-6">
						<h3 class="text-sm font-medium mb-2">{$i18n.t('Preview Recording')}</h3>
						<audio controls src={recordedAudio} class="w-full"></audio>
					</div>
				{/if}
				
				<!-- Noise Removal Option -->
				<div class="flex items-center mb-6">
					<input
						id="remove-noise-record"
						type="checkbox"
						bind:checked={removeNoise}
						class="h-4 w-4 rounded border-input bg-background"
					/>
					<label for="remove-noise-record" class="ml-2 block text-sm">
						{$i18n.t('Remove background noise')}
					</label>
				</div>
				
				<!-- Submit Button -->
				<div>
					<button
						type="button"
						on:click={submitRecording}
						disabled={isLoading || !recordedAudio}
						class="w-full px-4 py-2.5 rounded-md bg-primary font-medium text-white hover:bg-primary/90 disabled:opacity-50 disabled:cursor-not-allowed"
					>
						{#if isLoading}
							<span class="flex items-center justify-center">
								<svg class="animate-spin -ml-1 mr-2 h-4 w-4 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
									<circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
									<path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
								</svg>
								{$i18n.t('Processing...')}
							</span>
						{:else}
							{$i18n.t('Submit Recording')}
						{/if}
					</button>
				</div>
			</div>
		{/if}
		
		<!-- Result Display (common for both tabs) -->
		{#if result}
			<div transition:fade class="mt-8 bg-success/10 border border-success rounded-lg p-6">
				<h3 class="text-lg font-medium text-success mb-2">{$i18n.t('Voice Cloned Successfully!')}</h3>
				<div class="text-sm space-y-2">
					<p><strong>{$i18n.t('Voice ID')}:</strong> {result.voice_id}</p>
					<p><strong>{$i18n.t('Name')}:</strong> {result.name}</p>
					<p class="text-muted-foreground">
						{$i18n.t('Your cloned voice is now available in the TU•IPSE voices library. The voice has been saved to your settings and can be used for text-to-speech conversion.')}
					</p>
				</div>
			</div>
		{/if}
	</div>
</div> 