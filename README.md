<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Online PDF Editor</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/pdf-lib@1.17.0/dist/pdf-lib.min.js"></script>
    <script src="https://unpkg.com/@pdf-lib/fontkit@0.0.4/dist/fontkit.umd.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.10.377/pdf.min.js"></script>
</head>
<body class="bg-gray-100 flex flex-col items-center justify-center min-h-screen">
    <div class="bg-white p-6 rounded-lg shadow-lg w-full max-w-4xl">
        <h1 class="text-2xl font-bold mb-4 text-center">Online PDF Editor</h1>
        <div class="mb-4">
            <input type="file" id="pdfInput" accept="application/pdf" class="block w-full text-sm text-gray-500
                file:mr-4 file:py-2 file:px-4 file:rounded file:border-0
                file:text-sm file:font-semibold file:bg-blue-50 file:text-blue-700
                hover:file:bg-blue-100" />
        </div>
        <div class="flex space-x-4 mb-4">
            <button id="addTextBtn" class="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600" disabled>Add Text</button>
            <button id="drawBtn" class="bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600" disabled>Draw</button>
            <button id="saveBtn" class="bg-purple-500 text-white px-4 py-2 rounded hover:bg-purple-600" disabled>Save PDF</button>
        </div>
        <div class="flex space-x-4 mb-4 hidden" id="textInputDiv">
            <input type="text" id="textContent" placeholder="Enter text" class="border p-2 rounded">
            <input type="number" id="textX" placeholder="X position" min="0" class="border p-2 rounded w-24">
            <input type="number" id="textY" placeholder="Y position" min="0" class="border p-2 rounded w-24">
            <button id="applyTextBtn" class="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600">Apply Text</button>
        </div>
        <canvas id="pdfCanvas" class="border max-w-full"></canvas>
    </div>

    <script>
        let pdfDoc, pdfBytes, currentPage, canvas, ctx, isDrawing = false;
        const pdfInput = document.getElementById('pdfInput');
        const addTextBtn = document.getElementById('addTextBtn');
        const drawBtn = document.getElementById('drawBtn');
        const saveBtn = document.getElementById('saveBtn');
        const textInputDiv = document.getElementById('textInputDiv');
        const applyTextBtn = document.getElementById('applyTextBtn');
        canvas = document.getElementById('pdfCanvas');
        ctx = canvas.getContext('2d');

        pdfInput.addEventListener('change', async (e) => {
            const file = e.target.files[0];
            if (file && file.type === 'application/pdf') {
                const arrayBuffer = await file.arrayBuffer();
                pdfDoc = await PDFLib.PDFDocument.load(arrayBuffer);
                pdfBytes = arrayBuffer;
                currentPage = pdfDoc.getPage(0);
                await renderPage();
                addTextBtn.disabled = false;
                drawBtn.disabled = false;
                saveBtn.disabled = false;
            }
        });

        async function renderPage() {
            const pdf = await pdfjsLib.getDocument({ data: pdfBytes }).promise;
            const page = await pdf.getPage(1);
            const viewport = page.getViewport({ scale: 1 });
            canvas.width = viewport.width;
            canvas.height = viewport.height;
            await page.render({
                canvasContext: ctx,
                viewport: viewport
            }).promise;
        }

        addTextBtn.addEventListener('click', () => {
            textInputDiv.classList.toggle('hidden');
            isDrawing = false;
        });

        applyTextBtn.addEventListener('click', async () => {
            const text = document.getElementById('textContent').value;
            const x = parseInt(document.getElementById('textX').value) || 50;
            const y = parseInt(document.getElementById('textY').value) || 50;
            if (text) {
                const font = await pdfDoc.embedFont(PDFLib.StandardFonts.Helvetica);
                currentPage.drawText(text, {
                    x,
                    y: currentPage.getHeight() - y,
                    font,
                    size: 12,
                    color: PDFLib.rgb(0, 0, 0)
                });
                pdfBytes = await pdfDoc.save();
                await renderPage();
                textInputDiv.classList.add('hidden');
            }
        });

        drawBtn.addEventListener('click', () => {
            isDrawing = !isDrawing;
            drawBtn.textContent = isDrawing ? 'Stop Drawing' : 'Draw';
            textInputDiv.classList.add('hidden');
        });

        canvas.addEventListener('mousedown', (e) => {
            if (!isDrawing) return;
            const { x, y } = getCanvasCoords(e);
            ctx.beginPath();
            ctx.moveTo(x, y);
            ctx.strokeStyle = 'red';
            ctx.lineWidth = 2;
            canvas.addEventListener('mousemove', draw);
        });

        canvas.addEventListener('mouseup', () => {
            if (isDrawing) {
                canvas.removeEventListener('mousemove', draw);
                saveDrawingToPDF();
            }
        });

        function draw(e) {
            const { x, y } = getCanvasCoords(e);
            ctx.lineTo(x, y);
            ctx.stroke();
        }

        function getCanvasCoords(e) {
            const rect = canvas.getBoundingClientRect();
            return {
                x: e.clientX - rect.left,
                y: e.clientY - rect.top
            };
        }

        async function saveDrawingToPDF() {
            const imgData = canvas.toDataURL('image/png');
            const img = await pdfDoc.embedPng(imgData);
            const { width, height } = currentPage.getSize();
            currentPage.drawImage(img, {
                x: 0,
                y: 0,
                width,
                height
            });
            pdfBytes = await pdfDoc.save();
            await renderPage();
        }

        saveBtn.addEventListener('click', async () => {
            const pdfBytes = await pdfDoc.save();
            const blob = new Blob([pdfBytes], { type: 'application/pdf' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = 'edited_pdf.pdf';
            a.click();
            URL.revokeObjectURL(url);
        });
    </script>
</body>
</html>
