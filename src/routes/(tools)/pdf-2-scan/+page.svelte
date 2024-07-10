<script>
	import { Label, Button, Range, Toggle, ButtonGroup, Alert, Spinner } from 'flowbite-svelte';
	import { DownloadOutline, InfoCircleSolid } from 'flowbite-svelte-icons';
    import { PDFDocument } from 'pdf-lib';
    import * as pdfjsLib from 'pdfjs-dist/build/pdf';
    import 'pdfjs-dist/legacy/build/pdf.worker.mjs';

    let fileInput = null;
    let contrast = 7;
    let noiseIntensity = 7;
    let blurIntensity = 7;
    let pageTilt = 7;
    let applyGray = false;
    let selectedPreset = 'default';
    let isProcessing = false;
    let error ='';

    function clearError() {
        error = '';
    }

    function selectDefaultPreset() {
        selectedPreset = 'default';
    }

    function selectHighPreset() {
        selectedPreset = 'high';
    }

    function selectLowPreset() {
        selectedPreset = 'low';
    }

    const presets = {
        default: { contrast: 7, noiseIntensity: 7, blurIntensity: 7, pageTilt: 7 },
        high: { contrast: 10, noiseIntensity: 10, blurIntensity: 10, pageTilt: 10 },
        low: { contrast: 4, noiseIntensity: 4, blurIntensity: 4, pageTilt: 4 }
    };

    function applyPreset(preset) {
        const values = presets[preset];
        contrast = values.contrast;
        noiseIntensity = values.noiseIntensity;
        blurIntensity = values.blurIntensity;
        pageTilt = values.pageTilt;
    }

    $: applyPreset(selectedPreset);

    async function processPDF() {
        if (!fileInput || !fileInput.files.length) {
            error='Please upload a PDF file.';
        }
        else {
            isProcessing = true;
            const file = fileInput.files[0];
            const arrayBuffer = await file.arrayBuffer();
            const pdfBytes = await convertToScannedPDF(arrayBuffer);
            download(pdfBytes, `scanned.pdf`, 'application/pdf');
            isProcessing = false;
        }    
    }

    async function renderPage(page, scale) {
        const viewport = page.getViewport({ scale });
        const canvas = document.createElement('canvas');
        const context = canvas.getContext('2d');
        canvas.width = viewport.width;
        canvas.height = viewport.height;

        await page.render({
            canvasContext: context,
            viewport: viewport
        }).promise;

        return { canvas, context, viewport };
    }

    function applyGrayscale(context, width, height) {
        const imageData = context.getImageData(0, 0, width, height);
        for (let i = 0; i < imageData.data.length; i += 4) {
            const avg = (imageData.data[i] + imageData.data[i + 1] + imageData.data[i + 2]) / 3;
            imageData.data[i] = avg;
            imageData.data[i + 1] = avg;
            imageData.data[i + 2] = avg;
        }
        context.putImageData(imageData, 0, 0);
    }

    function addNoise(context, width, height, intensity) {
        const imageData = context.getImageData(0, 0, width, height);
        for (let i = 0; i < imageData.data.length; i += 4) {
            const noise = (Math.random() - 0.5) * intensity * 10;
            imageData.data[i] += noise;
            imageData.data[i + 1] += noise;
            imageData.data[i + 2] += noise;
        }
        context.putImageData(imageData, 0, 0);
    }

    function addBlur(context, width, height, intensity) {
        if (intensity === 0) return;
        const canvasCopy = document.createElement('canvas');
        const ctxCopy = canvasCopy.getContext('2d');
        canvasCopy.width = width;
        canvasCopy.height = height;
        ctxCopy.filter = `blur(${intensity/8}px)`;
        ctxCopy.drawImage(context.canvas, 0, 0, width, height);
        context.drawImage(canvasCopy, 0, 0);
    }

    function addContrast(context, width, height, contrast) {
        const imageData = context.getImageData(0, 0, width, height);
        const factor = (259 * (contrast * 10 + 255)) / (255 * (259 - contrast * 10));
        for (let i = 0; i < imageData.data.length; i += 4) {
            imageData.data[i] = factor * (imageData.data[i] - 128) + 128;
            imageData.data[i + 1] = factor * (imageData.data[i + 1] - 128) + 128;
            imageData.data[i + 2] = factor * (imageData.data[i + 2] - 128) + 128;
        }
        context.putImageData(imageData, 0, 0);
    }

    async function convertToScannedPDF(inputArrayBuffer) {
        const loadingTask = pdfjsLib.getDocument({ data: inputArrayBuffer });
        const pdf = await loadingTask.promise;
        const pdfDoc = await PDFDocument.create();
        const scale = 2.0;

        for (let pageNum = 1; pageNum <= pdf.numPages; pageNum++) {
            const page = await pdf.getPage(pageNum);
            const { canvas, context, viewport } = await renderPage(page, scale);

            if (applyGray) {
                applyGrayscale(context, canvas.width, canvas.height);
            }
            if (contrast > 0) {
                addContrast(context, canvas.width, canvas.height, contrast);
            }
            if (noiseIntensity > 0) {
                addNoise(context, canvas.width, canvas.height, noiseIntensity);
            }
            if (blurIntensity > 0) {
                addBlur(context, canvas.width, canvas.height, blurIntensity);
            }

            context.setTransform(
                1,
                pageTilt/1000,
                -pageTilt/1000,
                1,
                (Math.random() - 0.5) * 2,
                (Math.random() - 0.5) * 2
            );
            context.drawImage(canvas, 0, 0);

            const scannedImage = await pdfDoc.embedPng(canvas.toDataURL('image/png', 1.0));
            const pdfPage = pdfDoc.addPage([viewport.width / scale, viewport.height / scale]);
            pdfPage.drawImage(scannedImage, {
                x: 0,
                y: 0,
                width: viewport.width / scale,
                height: viewport.height / scale,
            });
        }

        return await pdfDoc.save();
    }

    function download(data, fileName, mimeType) {
        const blob = new Blob([data], { type: mimeType });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = fileName;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
    }
</script>

<div class="card gap-16 items-center mx-auto max-w-screen-xl overflow-hidden rounded-lg">
    <div class="p-8">             

        <Label class="space-y-2 mb-8">
            <span>Upload PDF</span>
            <input bind:this={fileInput} on:change={clearError} type="file" accept=".pdf" class="block w-full text-sm text-gray-900 border border-gray-300 rounded-lg cursor-pointer bg-gray-50 dark:text-gray-400 focus:outline-none dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400">       
        </Label>

        {#if error.length > 0}
			<Alert border color="red" class="mb-8">
				<InfoCircleSolid slot="icon" class="w-5 h-5"/>
				{error}
			</Alert>
        {/if} 

        <ul class="mb-8 items-center w-full rounded-lg border border-gray-200 sm:flex dark:bg-gray-800 dark:border-gray-600 divide-x rtl:divide-x-reverse divide-gray-200 dark:divide-gray-600">
            <li class="w-full">
                <Label class="p-4 pt-3">Contrast: {contrast}
                    <Range min="0" max="10" bind:value={contrast} />
                </Label>            
            </li>
            <li class="w-full">
                <Label class="p-4 pt-3">Noise Intensity: {noiseIntensity}
                    <Range min="0" max="10" bind:value={noiseIntensity} />   
                </Label>
            </li>
            <li class="w-full">
                <Label class="p-4 pt-3">Blur Intensity: {blurIntensity}
                    <Range min="0" max="10" bind:value={blurIntensity} />
                </Label>    
            </li>
            <li class="w-full">
                <Label class="p-4 pt-3">Page Tilt: {pageTilt}
                    <Range min="0" max="10" bind:value={pageTilt} />
                </Label>    
            </li>
            <li class="min-w-max">
                <Toggle class="px-4 py-5" bind:checked={applyGray}>Apply Grayscale</Toggle>
            </li>    
        </ul>
        
        <div class="grid grid-cols-2">
            <div>
                <span class="text-center items-center align-middle text-sm font-medium text-gray-900 dark:text-gray-100">
                    Select Preset:
                </span>
                <ButtonGroup class="ml-2">  
                    <Button outline color="dark" class="w-28" on:click={selectLowPreset}>Low</Button>
                    <Button outline color="dark" class="w-28" on:click={selectDefaultPreset}>Default</Button>
                    <Button outline color="dark" class="w-28" on:click={selectHighPreset}>High</Button> 
                </ButtonGroup>
            </div>
            <div class="text-right">
                {#if fileInput === null}
                    <Button color="blue" class="w-64" disabled>
                        <DownloadOutline/>
                        <span class="pl-2">Download Scanned PDF</span>
                    </Button> 
                {:else}
                    <Button color="blue" class="w-64" on:click={processPDF} disabled={isProcessing}>
                        {#if isProcessing}
                            <Spinner class="me-3" size="4" color="white" />Processing ...
                        {:else}
                            <DownloadOutline/>
                            <span class="pl-2">Download Scanned PDF</span>
                        {/if}
                    </Button>
                {/if}
            </div>
        </div>
    </div>
</div>
