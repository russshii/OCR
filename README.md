<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Modern OCR Web App</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Custom styles for the Inter font and overall look */
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f0f2f5; /* Light gray background */
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            margin: 0;
            padding: 20px;
            box-sizing: border-box;
        }
        .container {
            background-color: #ffffff;
            border-radius: 16px; /* More rounded corners */
            box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1); /* Softer shadow */
            padding: 30px;
            width: 100%;
            max-width: 900px; /* Increased max width for better table display */
            display: flex;
            flex-direction: column;
            gap: 25px;
        }
        .input-group {
            display: flex;
            flex-direction: column;
            gap: 15px;
        }
        .file-input-wrapper {
            border: 2px dashed #cbd5e1; /* Dashed border for file input */
            border-radius: 12px;
            padding: 20px;
            text-align: center;
            cursor: pointer;
            transition: border-color 0.3s ease;
        }
        .file-input-wrapper:hover {
            border-color: #93c5fd; /* Light blue on hover */
        }
        .file-input-wrapper input[type="file"] {
            display: none;
        }
        .file-input-wrapper label {
            display: block;
            cursor: pointer;
            color: #4a5568; /* Darker text */
            font-weight: 500;
        }
        .file-input-wrapper img {
            max-width: 100%;
            max-height: 200px;
            border-radius: 8px;
            margin-top: 15px;
            display: none; /* Hidden by default */
        }
        .button-primary {
            background-image: linear-gradient(to right, #6366f1, #8b5cf6); /* Gradient button */
            color: white;
            padding: 12px 25px;
            border-radius: 12px;
            font-weight: 600;
            transition: transform 0.2s ease, box-shadow 0.2s ease;
            box-shadow: 0 5px 15px rgba(99, 102, 241, 0.3);
        }
        .button-primary:hover {
            transform: translateY(-2px);
            box-shadow: 0 8px 20px rgba(99, 102, 241, 0.4);
        }
        .button-secondary {
            background-color: #e2e8f0; /* Light gray */
            color: #4a5568;
            padding: 12px 25px;
            border-radius: 12px;
            font-weight: 600;
            transition: background-color 0.2s ease, transform 0.2s ease;
        }
        .button-secondary:hover {
            background-color: #cbd5e1;
            transform: translateY(-1px);
        }
        .loading-spinner {
            border: 4px solid rgba(255, 255, 255, 0.3);
            border-top: 4px solid #ffffff;
            border-radius: 50%;
            width: 24px;
            height: 24px;
            animation: spin 1s linear infinite;
            display: inline-block;
            vertical-align: middle;
            margin-right: 8px;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        .result-area {
            background-color: #f7fafc; /* Lighter background for result */
            border: 1px solid #e2e8f0;
            border-radius: 12px;
            padding: 20px;
            min-height: 150px;
            color: #2d3748; /* Darker text */
            font-size: 0.95rem;
            line-height: 1.6;
            overflow-x: auto; /* Allow horizontal scroll for tables */
            max-height: 400px; /* Limit height of result area */
        }
        .result-area table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 10px;
            border-radius: 8px; /* Rounded corners for table */
            overflow: hidden; /* Ensures rounded corners apply to content */
        }
        .result-area th, .result-area td {
            border: 1px solid #e2e8f0;
            padding: 10px 15px; /* Increased padding */
            text-align: left;
        }
        .result-area th {
            background-color: #edf2f7;
            font-weight: 600;
            color: #4a5568;
            position: sticky; /* Make header sticky for scrolling */
            top: 0;
            z-index: 1;
        }
        .result-area tr:nth-child(even) {
            background-color: #f7fafc;
        }
        .result-area tr:hover {
            background-color: #e6f0ff; /* Light blue on row hover */
        }
        .message-box {
            background-color: #fff;
            border-radius: 8px;
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
            padding: 20px;
            max-width: 400px;
            text-align: center;
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            z-index: 1000;
            display: none; /* Hidden by default */
            animation: fadeIn 0.3s ease-out;
        }
        .message-box.show {
            display: block;
        }
        .message-box h3 {
            font-size: 1.25rem;
            font-weight: 600;
            margin-bottom: 10px;
            color: #333;
        }
        .message-box p {
            font-size: 1rem;
            color: #555;
            margin-bottom: 20px;
        }
        .message-box button {
            background-color: #6366f1;
            color: white;
            padding: 10px 20px;
            border-radius: 8px;
            font-weight: 500;
            cursor: pointer;
            transition: background-color 0.2s ease;
        }
        .message-box button:hover {
            background-color: #4f46e5;
        }
        @keyframes fadeIn {
            from { opacity: 0; transform: translate(-50%, -60%); }
            to { opacity: 1; transform: translate(-50%, -50%); }
        }

        /* Responsive adjustments */
        @media (max-width: 768px) {
            .container {
                padding: 20px;
                margin: 10px;
                max-width: 100%; /* Allow full width on smaller screens */
            }
            .button-primary, .button-secondary {
                padding: 10px 20px;
                font-size: 0.9rem;
            }
            .result-area {
                font-size: 0.85rem;
                padding: 15px;
            }
            .result-area th, .result-area td {
                padding: 8px 10px;
            }
        }
    </style>
</head>
<body>
    <div class="container">
        <h1 class="text-3xl font-bold text-center text-gray-800 mb-4">Image to Table Data Extractor</h1>

        <div class="input-group">
            <div class="file-input-wrapper" id="fileInputWrapper">
                <input type="file" id="imageInput" accept="image/*">
                <label for="imageInput" class="flex flex-col items-center justify-center">
                    <svg class="w-12 h-12 text-gray-400 mb-2" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12"></path>
                    </svg>
                    <span class="text-lg font-semibold">Drag & Drop your image here or Click to Upload</span>
                    <span class="text-sm text-gray-500 mt-1">Or paste an image from clipboard (Ctrl+V / Cmd+V)</span>
                </label>
                <img id="imagePreview" src="#" alt="Image Preview" class="mt-4 hidden">
            </div>
            <button id="ocrButton" class="button-primary w-full flex items-center justify-center" disabled>
                Perform OCR
            </button>
        </div>

        <div class="output-group">
            <h2 class="text-xl font-semibold text-gray-700 mb-3">Extracted Data:</h2>
            <div id="ocrResult" class="result-area">
                Upload an image or paste from clipboard and click 'Perform OCR' to see the extracted data in a table.
            </div>
            <div id="actionButtons" class="flex flex-wrap justify-center gap-4 mt-4 hidden">
                <button id="downloadCsvButton" class="button-secondary flex-1">Download CSV</button>
                <button id="copyTableButton" class="button-secondary flex-1">Copy Table Data</button>
            </div>
        </div>
    </div>

    <!-- Message Box for alerts -->
    <div id="messageBox" class="message-box">
        <h3 id="messageBoxTitle"></h3>
        <p id="messageBoxContent"></p>
        <button id="messageBoxClose">OK</button>
    </div>

    <script type="module">
        // Get references to DOM elements
        const imageInput = document.getElementById('imageInput');
        const imagePreview = document.getElementById('imagePreview');
        const ocrButton = document.getElementById('ocrButton');
        const ocrResult = document.getElementById('ocrResult');
        const fileInputWrapper = document.getElementById('fileInputWrapper');
        const messageBox = document.getElementById('messageBox');
        const messageBoxTitle = document.getElementById('messageBoxTitle');
        const messageBoxContent = document.getElementById('messageBoxContent');
        const messageBoxClose = document.getElementById('messageBoxClose');
        const actionButtons = document.getElementById('actionButtons');
        const downloadCsvButton = document.getElementById('downloadCsvButton');
        const copyTableButton = document.getElementById('copyTableButton');

        let base64Image = ''; // To store the base64 encoded image
        let extractedTableData = []; // To store the parsed data for CSV/Copy

        // Function to show custom message box
        function showMessageBox(title, message) {
            messageBoxTitle.textContent = title;
            messageBoxContent.textContent = message;
            messageBox.classList.add('show');
        }

        // Function to hide custom message box
        messageBoxClose.addEventListener('click', () => {
            messageBox.classList.remove('show');
        });

        /**
         * Processes an image file (from input or paste) and updates the UI.
         * @param {File} file The image file to process.
         */
        function processImageFile(file) {
            if (!file || !file.type.startsWith('image/')) {
                showMessageBox('Invalid File Type', 'Please upload or paste an image file (PNG, JPG, JPEG).');
                imagePreview.classList.add('hidden');
                ocrButton.disabled = true;
                ocrResult.innerHTML = 'Upload an image or paste from clipboard and click \'Perform OCR\' to see the extracted data.';
                actionButtons.classList.add('hidden');
                return;
            }

            const reader = new FileReader();
            reader.onload = (e) => {
                imagePreview.src = e.target.result;
                imagePreview.classList.remove('hidden');
                base64Image = e.target.result.split(',')[1]; // Extract base64 part
                ocrButton.disabled = false; // Enable OCR button once image is loaded
                ocrResult.innerHTML = 'Image loaded. Click \'Perform OCR\' to extract data.';
                actionButtons.classList.add('hidden'); // Hide action buttons until new data is extracted
            };
            reader.readAsDataURL(file); // Read file as Data URL (base64)
        }

        // Event listener for image input change (from file selection)
        imageInput.addEventListener('change', (event) => {
            const file = event.target.files[0];
            processImageFile(file);
        });

        // Drag and drop functionality
        fileInputWrapper.addEventListener('dragover', (event) => {
            event.preventDefault();
            fileInputWrapper.classList.add('border-blue-500');
        });

        fileInputWrapper.addEventListener('dragleave', (event) => {
            event.preventDefault();
            fileInputWrapper.classList.remove('border-blue-500');
        });

        fileInputWrapper.addEventListener('drop', (event) => {
            event.preventDefault();
            fileInputWrapper.classList.remove('border-blue-500');
            const files = event.dataTransfer.files;
            if (files.length > 0) {
                processImageFile(files[0]); // Process the first dropped file
            }
        });

        // Paste functionality (for images from clipboard)
        document.addEventListener('paste', (event) => {
            const items = event.clipboardData.items;
            for (let i = 0; i < items.length; i++) {
                if (items[i].type.indexOf('image') !== -1) {
                    const file = items[i].getAsFile();
                    if (file) {
                        processImageFile(file);
                        showMessageBox('Image Pasted', 'Image successfully pasted from clipboard.');
                        event.preventDefault(); // Prevent default paste behavior
                        return;
                    }
                }
            }
        });


        /**
         * Filters out duplicate rows based on a specified key (e.g., "Batch ID").
         * Keeps the first occurrence of each unique key.
         * @param {Array<Object>} data The array of objects to filter.
         * @param {string} key The key to check for duplicates.
         * @returns {Array<Object>} A new array with unique entries.
         */
        function filterUniqueByKey(data, key) {
            const seen = new Set();
            return data.filter(item => {
                const value = item[key];
                if (seen.has(value)) {
                    return false;
                }
                seen.add(value);
                return true;
            });
        }

        /**
         * Renders the extracted data into an HTML table.
         * @param {Array<Object>} data The array of objects to display.
         * @returns {string} The HTML string for the table.
         */
        function renderTable(data) {
            if (!data || data.length === 0) {
                return '<p>No structured data found in the image.</p>';
            }

            // Define the order of headers and exclude 'Description'
            const headers = ["Asset Name", "Batch ID", "Date", "Document Type", "Total Pages"];

            let tableHtml = '<table><thead><tr>';
            // Create table headers
            headers.forEach(header => {
                tableHtml += `<th>${header}</th>`;
            });
            tableHtml += '</tr></thead><tbody>';

            // Create table rows
            data.forEach(row => {
                tableHtml += '<tr>';
                headers.forEach(header => {
                    tableHtml += `<td>${row[header] || ''}</td>`; // Use || '' for undefined values
                });
                tableHtml += '</tr>';
            });

            tableHtml += '</tbody></table>';
            return tableHtml;
        }

        /**
         * Downloads the given data as a CSV file.
         * @param {Array<Object>} data The array of objects to convert to CSV.
         */
        function downloadCSV(data) {
            if (!data || data.length === 0) {
                showMessageBox('No Data', 'No data available to download.');
                return;
            }

            // Define the order of headers and exclude 'Description'
            const headers = ["Asset Name", "Batch ID", "Date", "Document Type", "Total Pages"];
            const csvRows = [];

            // Add headers to CSV
            csvRows.push(headers.map(header => `"${header}"`).join(','));

            // Add data rows to CSV
            data.forEach(row => {
                const values = headers.map(header => {
                    const value = row[header] || '';
                    // Escape double quotes by doubling them, then wrap in quotes
                    return `"${String(value).replace(/"/g, '""')}"`;
                });
                csvRows.push(values.join(','));
            });

            const csvString = csvRows.join('\n');
            const blob = new Blob([csvString], { type: 'text/csv;charset=utf-8;' });
            const link = document.createElement('a');
            link.href = URL.createObjectURL(blob);
            link.setAttribute('download', 'extracted_data.csv');
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
            showMessageBox('Download Complete', 'CSV file has been downloaded.');
        }

        /**
         * Copies the table data to the clipboard in a tab-separated format.
         * @param {Array<Object>} data The array of objects to copy.
         */
        function copyTableData(data) {
            if (!data || data.length === 0) {
                showMessageBox('No Data', 'No data available to copy.');
                return;
            }

            // Define the order of headers and exclude 'Description'
            const headers = ["Asset Name", "Batch ID", "Date", "Document Type", "Total Pages"];
            const tsvRows = [];

            // Add headers to TSV
            tsvRows.push(headers.join('\t'));

            // Add data rows to TSV
            data.forEach(row => {
                const values = headers.map(header => String(row[header] || ''));
                tsvRows.push(values.join('\t'));
            });

            const tsvString = tsvRows.join('\n');

            // Use document.execCommand('copy') for better compatibility in iframes
            const textarea = document.createElement('textarea');
            textarea.value = tsvString;
            textarea.style.position = 'fixed'; // Avoid scrolling to bottom
            textarea.style.opacity = '0'; // Hide it
            document.body.appendChild(textarea);
            textarea.select();
            try {
                const successful = document.execCommand('copy');
                if (successful) {
                    showMessageBox('Copied!', 'Table data copied to clipboard.');
                } else {
                    showMessageBox('Copy Failed', 'Failed to copy data. Please try again.');
                }
            } catch (err) {
                console.error('Copy command failed', err);
                showMessageBox('Copy Failed', 'Failed to copy data. Your browser might not support this feature or there was an error.');
            } finally {
                document.body.removeChild(textarea);
            }
        }


        // Event listener for OCR button click
        ocrButton.addEventListener('click', async () => {
            if (!base64Image) {
                showMessageBox('No Image', 'Please upload an image first.');
                return;
            }

            ocrButton.disabled = true;
            ocrButton.innerHTML = '<span class="loading-spinner"></span> Processing...'; // Show loading spinner
            ocrResult.innerHTML = 'Extracting data and formatting as table... This may take a moment.';
            actionButtons.classList.add('hidden'); // Hide action buttons during processing

            try {
                // Prepare the payload for the Gemini API
                // Updated prompt to specifically handle the new image format for Asset Name and Batch ID
                const prompt = `Extract the data from this image and format it as a JSON array of objects. Each object should represent a row in the table. For "Batch ID", extract the numeric part at the very beginning of the first column (e.g., "173368" from "173368_wng-1a7e676"). For "Asset Name", extract the short code like "wng" that appears after the first underscore and before the hyphen in the first column (e.g., "wng" from "173368_wng-1a7e676"). For "Document Type", use the value from the "Type" column. For "Date", use the value from the "Creation Time" column. For "Total Pages", extract the numerical value from the "Total Pages" column. Do NOT include a "Description" field, "Total Documents", "Stage", "Processing Start Time", or "Expires on". If any of the requested fields are not found, use an empty string for that field. Ensure the keys are exactly: "Asset Name", "Batch ID", "Date", "Document Type", "Total Pages".`;
                let chatHistory = [];
                chatHistory.push({ role: "user", parts: [{ text: prompt }] });

                const payload = {
                    contents: [
                        {
                            role: "user",
                            parts: [
                                { text: prompt },
                                {
                                    inlineData: {
                                        mimeType: imageInput.files[0] ? imageInput.files[0].type : 'image/png', // Use actual mime type or default
                                        data: base64Image
                                    }
                                }
                            ]
                        }
                    ],
                    generationConfig: {
                        responseMimeType: "application/json",
                        responseSchema: {
                            type: "ARRAY",
                            items: {
                                type: "OBJECT",
                                properties: {
                                    "Asset Name": { "type": "STRING" },
                                    "Batch ID": { "type": "STRING" },
                                    "Date": { "type": "STRING" },
                                    "Document Type": { "type": "STRING" },
                                    "Total Pages": { "type": "STRING" }
                                },
                                // Updated propertyOrdering to exclude "Description"
                                propertyOrdering: ["Asset Name", "Batch ID", "Date", "Document Type", "Total Pages"]
                            }
                        }
                    }
                };

                const apiKey = ""; // Canvas will provide this API key at runtime
                const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;

                const response = await fetch(apiUrl, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });

                const result = await response.json();

                if (result.candidates && result.candidates.length > 0 &&
                    result.candidates[0].content && result.candidates[0].content.parts &&
                    result.candidates[0].content.parts.length > 0) {
                    const jsonString = result.candidates[0].content.parts[0].text;
                    let parsedData = JSON.parse(jsonString);

                    // Filter out duplicate batch IDs
                    extractedTableData = filterUniqueByKey(parsedData, "Batch ID");

                    ocrResult.innerHTML = renderTable(extractedTableData);
                    actionButtons.classList.remove('hidden'); // Show action buttons
                } else {
                    console.error('Unexpected API response structure:', result);
                    ocrResult.innerHTML = 'Error: Could not extract structured data. Please try again or with a different image.';
                    showMessageBox('OCR Failed', 'Failed to extract data in table format. The image might be too complex or the table structure is unclear.');
                }
            } catch (error) {
                console.error('Error performing OCR:', error);
                ocrResult.innerHTML = 'An error occurred during OCR. Please try again.';
                showMessageBox('Error', 'An unexpected error occurred. Please check your network connection and try again.');
            } finally {
                ocrButton.disabled = false;
                ocrButton.innerHTML = 'Perform OCR'; // Restore button text
            }
        });

        // Event listeners for new action buttons
        downloadCsvButton.addEventListener('click', () => {
            downloadCSV(extractedTableData);
        });

        copyTableButton.addEventListener('click', () => {
            copyTableData(extractedTableData);
        });
    </script>
</body>
</html>
