<script>
	import { run } from "svelte/legacy";
import {
		getCards,
		gibberish,
		keepOnlyCapsAndNumbers,
		recognize,
	} from "../utils/backend";
	import Xmark from "./icons/Xmark.svelte";
	import { cameraState } from "./state.svelte";

	let videoContainer = $state();
	let highlight = $state();
	let video = $state();

	export async function startCamera() {
		try {
			const stream = await navigator.mediaDevices.getUserMedia({
				video: {
					width: { ideal: 999999 }, // Request a width of 1920 pixels (Full HD)
					height: { ideal: 999999 }, // Request a height of 1080 pixels (Full HD)
					facingMode: { ideal: "environment" },
				},
			});
			video.srcObject = stream;
			cameraState.isActive = true;
		} catch (error) {
			console.error("Error accessing the camera: ", error);
			alert("Could not access the camera.");
		}
	}

	export function stopCamera() {
		cameraState.isActive = false;
		const stream = video.srcObject;
		if (stream) {
			const tracks = stream.getTracks();
			// @ts-ignore
			tracks.forEach((track) => track.stop()); // Stops each track in the stream
			video.srcObject = null; // Clears the video source
		}
	}

	function adaptiveThreshold(grayscale, width, height, blockSize, c) {
		const halfBlockSize = Math.floor(blockSize / 2);
		const result = new Uint8Array(grayscale.length);

		// Integral image for fast local sum computation
		const integral = new Uint32Array((width + 1) * (height + 1));

		// Compute integral image
		for (let y = 0; y < height; y++) {
			for (let x = 0; x < width; x++) {
				const index = y * width + x;
				const gray = grayscale[index];
				const integralIndex = (y + 1) * (width + 1) + (x + 1);
				integral[integralIndex] =
					gray +
					integral[integralIndex - 1] +
					integral[integralIndex - (width + 1)] -
					integral[integralIndex - (width + 2)];
			}
		}

		// Perform thresholding using the integral image
		for (let y = 0; y < height; y++) {
			for (let x = 0; x < width; x++) {
				const x1 = Math.max(0, x - halfBlockSize);
				const x2 = Math.min(width - 1, x + halfBlockSize);
				const y1 = Math.max(0, y - halfBlockSize);
				const y2 = Math.min(height - 1, y + halfBlockSize);

				const sum =
					integral[(y2 + 1) * (width + 1) + (x2 + 1)] -
					integral[y1 * (width + 1) + (x2 + 1)] -
					integral[(y2 + 1) * (width + 1) + x1] +
					integral[y1 * (width + 1) + x1];

				const area = (x2 - x1 + 1) * (y2 - y1 + 1);
				const mean = sum / area;

				const index = y * width + x;
				result[index] = grayscale[index] <= mean - c ? 0 : 255; // Black or white
			}
		}

		return result;
	}

	function preprocessImage(canvas) {
		// Load image into canvas
		const width = canvas.width;
		const height = canvas.height;
		const context = canvas.getContext("2d");
		const imageData = context.getImageData(0, 0, width, height);
		const data = imageData.data;

		// Convert to grayscale
		for (let i = 0; i < data.length; i += 4) {
			const gray =
				0.299 * data[i] + 0.587 * data[i + 1] + 0.114 * data[i + 2];
			data[i] = data[i + 1] = data[i + 2] = gray; // Set R, G, B to gray
		}
		context.putImageData(imageData, 0, 0);
		// Apply Adaptive Thresholding
		const blockSize = 15; // Size of the local region for threshold calculation
		const c = 10; // Constant to subtract from the mean to fine-tune thresholding
		const grayscale = new Uint8Array(width * height);

		// Extract grayscale values into a 1D array
		for (let y = 0; y < height; y++) {
			for (let x = 0; x < width; x++) {
				const index = (y * width + x) * 4;
				grayscale[y * width + x] = data[index];
			}
		}

		// Apply adaptive threshold
		const thresholded = adaptiveThreshold(
			grayscale,
			width,
			height,
			blockSize,
			c,
		);

		// Map back to image data
		for (let y = 0; y < height; y++) {
			for (let x = 0; x < width; x++) {
				const index = (y * width + x) * 4;
				const value = thresholded[y * width + x];
				data[index] = data[index + 1] = data[index + 2] = value; // Black or white
			}
		}
		context.putImageData(imageData, 0, 0);

		return canvas;
	}

	function gettHighlightedArea() {
		const canvas = document.createElement('canvas')
		const context = canvas.getContext("2d");

		//! Highlight
		const videoRect = video.getBoundingClientRect();
		const highlightRect = highlight.getBoundingClientRect();

		// Calculate the relative position and size of the highlight on the video
		const scaleX = video.videoWidth / videoRect.width;
		const scaleY = video.videoHeight / videoRect.height;

		const highlightX = (highlightRect.left -videoRect.left) * scaleX;
		const highlightY = (highlightRect.top - videoRect.top) * scaleY;
		const highlightWidth = highlightRect.width * scaleX ;
		const highlightHeight = highlightRect.height * scaleY;

		// Set the canvas dimensions to match the highlight section
		canvas.width = highlightWidth 
		canvas.height = highlightHeight;  

		// the image is slithtly wider than the box but im too tired 
		// of it to try and understand why. Better wider than smaller

		// Draw the highlighted section of the video onto the canvas
		context.drawImage(
			video,
			highlightX,
			highlightY,
			highlightWidth,
			highlightHeight, // Source from video
			0,
			0,
			highlightWidth,
			highlightHeight, // Draw onto canvas
		);

		return canvas
	}



	export async function capture() {
		cameraState.running = true;
		cameraState.error = undefined;

		try{
			
			console.time("Cropping")
			const canvas = gettHighlightedArea()
			const image = canvas.toDataURL("image/png");
			console.timeEnd("Cropping")
				

			console.time("Preprocessing")
			preprocessImage(canvas)
			console.timeEnd("Preprocessing")


			console.time("OCR")
			const text = await recognize(image);	
			console.timeEnd("OCR")

	
			console.time("API Call")
			const lines = text
			.split("\n")
			.map((s) => gibberish(keepOnlyCapsAndNumbers(s).trim()))
			.filter((s) => s);
			
			const options = (
				await Promise.all(
					lines.map(async (l) => await getCards(l, cameraState.filter)),
				)).flat();
			console.timeEnd("API Call")
				
			cameraState.currentListOfOptions = options;

			if (!options || options.length == 0) {
					cameraState.error = "No results"
			}
		} catch (error) {
			cameraState.error = error
		} finally {
			cameraState.running = false 
		}
	}
</script>

<div
	bind:this={videoContainer}
	class={cameraState.isActive
		? "h-full grow flex flex-col relative p-4"
		: "hidden "}
>
	<div class="relative grow flex flex-col">
		<video
			bind:this={video}
			autoplay
			playsinline
			class="w-auto object-cover grow h-full relative"
		>
		</video>
		<div
			bind:this={highlight}
			class="absolute -translate-x-1/2 w-[264px] h-[64px] border-2 border-teal-800 left-1/2 top-1/4 rounded-md z-10"
		></div>
	</div>

	<button
		class="absolute z-10 top-8 right-8 bg-red-400 p-1 rounded-md"
		onclick={stopCamera}
	>
		<Xmark />
	</button>
	<button
		disabled={cameraState.running}
		onclick={capture}
		type="button"
		class="z-10 absolute bottom-12 justify-center w-4/5 -translate-x-1/2 left-1/2 inline-flex items-center rounded-md bg-indigo-600 px-8 py-3 text-sm font-semibold text-white shadow-sm hover:bg-indigo-500 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-indigo-600"
	>
	{#if cameraState.running}
		
	<svg class="animate-spin -ml-1 mr-3 h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
		<circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
		<path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
	  </svg>
	  {:else}
	  Scan
	  {/if}
	</button>
</div>
