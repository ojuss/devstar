<script>
	import { Fileupload, Range, Checkbox, Toggle } from 'flowbite-svelte';
	import { Input, ButtonGroup, Label, Button, CloseButton, Card, Alert } from 'flowbite-svelte';
	import { Table, TableBody, TableBodyCell, TableBodyRow, TableHead, TableHeadCell } from 'flowbite-svelte';
	import { InfoCircleSolid, DownloadOutline } from 'flowbite-svelte-icons';

    import { PDFDocument } from 'pdf-lib';
    import * as pdfjsLib from 'pdfjs-dist/build/pdf';
    import 'pdfjs-dist/legacy/build/pdf.worker.mjs';

    let fileInput = null;
    let contrast = 20;
    let noiseIntensity = 30;
    let blurIntensity = 1;
    let applyGray = true;

    async function processPDF() {
        if (!fileInput || !fileInput.files.length) {
            alert('Please select a PDF file.');
            return;
        }
        const file = fileInput.files[0];
        const arrayBuffer = await file.arrayBuffer();
        const pdfBytes = await convertToScannedPDF(arrayBuffer);
        download(pdfBytes, `scanned.pdf`, 'application/pdf');
    }

    function applyGrayscale(context, width, height) {
       // Code
    }

    function addNoise(context, width, height, intensity) {
        // Code
    }

    function addBlur(context, width, height, intensity) {
       // Code
    }

    function addContrast(context, width, height, contrast) {
        // Code
    }

    async function convertToScannedPDF(inputArrayBuffer) {
		// Code
    }

    function download(data, fileName, mimeType) {
        // Code
    }
</script>

<div class="card gap-16 items-center mx-auto max-w-screen-xl overflow-hidden rounded-lg">
	<div class="p-8">
		<Label class="space-y-2 mb-8">
			<span>Upload PDF file</span>
            <input bind:this={fileInput} type="file" accept=".pdf" class="block w-full text-sm text-gray-900 border border-gray-300 rounded-lg cursor-pointer bg-gray-50 dark:text-gray-400 focus:outline-none dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400">		
        </Label>

        <ul class="mb-8 items-center w-full rounded-lg border border-gray-200 sm:flex dark:bg-gray-800 dark:border-gray-600 divide-x rtl:divide-x-reverse divide-gray-200 dark:divide-gray-600">
            <li class="w-full">
                <Label class="p-4 pt-3">Contrast: {contrast}
                    <Range min="0" max="50" bind:value={contrast} />
                </Label>            
            </li>
            <li class="w-full">
                <Label class="p-4 pt-3">Noise Intensity: {noiseIntensity}
                    <Range min="0" max="50" bind:value={noiseIntensity} />	
                </Label>
            </li>
            <li class="w-full">
                <Label class="p-4 pt-3">Blur Intensity: {blurIntensity}
                    <Range min="0" max="5" step="0.1" bind:value={blurIntensity} />
                </Label>	
            </li>
            <li class="min-w-max">
                <Toggle class="px-4 py-5" bind:checked={applyGray}>Apply Grayscale</Toggle>
            </li>    
        </ul>
        
        <Button on:click={processPDF}>
            <DownloadOutline/>
			    <span class="pl-2">Download PDF</span>
        </Button> 
	</div>
</div>
