<script lang="ts">
	import JSZip from 'jszip';

	let selectedFiles: File[] = [];
	let width = 800;
	let height = 600;
	let quality = 100;

	function handleFileChange(event: Event) {
		const files = (event.target as HTMLInputElement).files;
		if (files) {
			selectedFiles = Array.from(files);
		}
	}

	async function resizeImagesAndDownload() {
		if (selectedFiles.length === 0) {
			alert('請選擇圖片檔案');
			return;
		}

		const zip = new JSZip();
		const promises = selectedFiles.map(async (file) => {
			if (file.name.endsWith('.zip')) {
				const zipContent = await JSZip.loadAsync(file);
				const filePromises = Object.keys(zipContent.files).map(async (filename) => {
					const zipFile = zipContent.files[filename];
					if (!zipFile.dir && isImage(filename)) {
						const imgBlob = await zipFile.async('blob');
						const img = await loadImage(new File([imgBlob], filename));
						const resizedBlob = await resizeImage(img, imgBlob.type);
						zip.file(filename, resizedBlob);
					}
				});
				await Promise.all(filePromises);
			} else if (isImage(file.name)) {
				const img = await loadImage(file);
				const resizedBlob = await resizeImage(img, file.type);
				zip.file(file.name, resizedBlob);
			}
		});

		await Promise.all(promises);

		const zipBlob = await zip.generateAsync({ type: 'blob' });
		downloadFiles('resized_images.zip', zipBlob);
	}

	function isImage(filename: string): boolean {
		return /\.(jpe?g|png|gif|bmp|webp)$/i.test(filename);
	}

	function loadImage(file: File): Promise<HTMLImageElement> {
		return new Promise((resolve, reject) => {
			const reader = new FileReader();
			reader.onload = (e) => {
				const img = new Image();
				img.onload = () => resolve(img);
				img.onerror = reject;
				img.src = e.target?.result as string;
			};
			reader.readAsDataURL(file);
		});
	}

	async function resizeImage(img: HTMLImageElement, mimeType: string): Promise<Blob> {
		const canvas = document.createElement('canvas');
		const ctx = canvas.getContext('2d')!;

		canvas.width = width;
		canvas.height = height;

		ctx.drawImage(img, 0, 0, width, height);

		return new Promise((resolve) => {
			canvas.toBlob(
				(blob) => {
					if (blob) resolve(blob);
				},
				mimeType,
				quality / 100
			);
		});
	}

	function downloadFiles(filename: string, blob: Blob) {
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = filename;
		document.body.appendChild(a);
		a.click();
		document.body.removeChild(a);
		URL.revokeObjectURL(url);
	}
</script>

<div>
	<h1>批量圖片調整</h1>
	<input type="file" multiple accept="image/*,.zip" on:change={handleFileChange} />
	<div>
		<label>
			寬度:
			<input type="number" bind:value={width} />
		</label>
		<label>
			高度:
			<input type="number" bind:value={height} />
		</label>
		<label>
			圖片品質:
			<input type="range" min="0" max="100" step="1" bind:value={quality} />
			<input type="number" min="0" max="100" step="1" bind:value={quality} />
		</label>
		<button on:click={resizeImagesAndDownload}>重新調整</button>
	</div>
</div>

<style>
	div {
		background: white;
		width: 100%;
		max-width: 600px;
	}

	h1 {
		margin-bottom: 20px;
		font-size: 24px;
		color: #2c3e50;
		text-align: center;
	}

	input[type='file'] {
		display: block;
		margin-bottom: 20px;
	}

	div > label {
		display: block;
		margin-bottom: 10px;
	}

	label {
		font-size: 14px;
		color: #34495e;
	}

	input[type='number'],
	input[type='range'] {
		margin-top: 5px;
		margin-bottom: 20px;
		width: 100%;
		box-sizing: border-box;
	}

	input[type='number'] {
		padding: 8px;
		border: 1px solid #ccc;
		border-radius: 4px;
	}

	input[type='range'] {
		margin-bottom: 10px;
	}

	button {
		background-color: #3498db;
		color: white;
		padding: 10px 15px;
		border: none;
		border-radius: 4px;
		cursor: pointer;
		font-size: 16px;
		transition: background-color 0.3s;
	}

	button:hover {
		background-color: #2980b9;
	}
</style>
