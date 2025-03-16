<script>
	import { toast } from 'svelte-sonner';
    import { fade } from 'svelte/transition';
    import { getContext } from 'svelte';
    
    const i18n = getContext('i18n');

	let voiceName = '';
	let selectedFiles = [];
	let removeNoise = true;
	let isLoading = false;
	let result = null;
	
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
</script>

<div class="container mx-auto px-4 py-8">
	<div class="max-w-3xl mx-auto">
		<h1 class="text-3xl font-bold mb-8">{$i18n.t('Voice Cloning with TU•IPSE')}</h1>
		
		<div class="bg-popover rounded-lg p-6 mb-8 shadow-sm">
			<p class="text-sm text-muted-foreground mb-4">
				{$i18n.t('Upload MP3 or WAV files to clone your voice using TU•IPSE AI. You need at least one audio file, but we recommend uploading 5-10 samples for better results.')}
			</p>
		</div>
		
		<form on:submit|preventDefault={handleSubmit} class="space-y-6">
			<!-- Voice Name -->
			<div>
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
		
		<!-- Result Display -->
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