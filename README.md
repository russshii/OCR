<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>OCR Table Extractor</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Inter Font -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f0f2f5;
        }
        /* Custom styling for file input to make it look better */
        .custom-file-input::-webkit-file-upload-button {
            visibility: hidden;
        }
        .custom-file-input::before {
            content: 'Select File';
            display: inline-block;
            background: linear-gradient(to right, #4F46E5, #6366F1);
            color: white;
            border: none;
            border-radius: 0.5rem;
            padding: 0.75rem 1.5rem;
            cursor: pointer;
            font-weight: 600;
            transition: background-color 0.3s ease;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }
        .custom-file-input:hover::before {
            background: linear-gradient(to right, #6366F1, #4F46E5);
        }
        .custom-file-input:active::before {
            background: linear-gradient(to right, #4338CA, #4F46E5);
            transform: translateY(1px);
        }
        .custom-file-input {
            border: 2px dashed #9CA3AF;
            border-radius: 0.5rem;
            padding: 1.5rem;
            text-align: center;
            cursor: pointer;
            transition: border-color 0.3s ease;
        }
        .custom-file-input:hover {
            border-color: #6366F1;
        }
        textarea {
            resize: vertical; /* Allow vertical resizing */
        }
    </style>
</head>
<body>
    <div id="root"></div>

    <!-- React CDN -->
    <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>

    <!-- Babel Standalone for JSX transformation in browser -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

    <!-- Tesseract.js CDN -->
    <script src='https://unpkg.com/tesseract.js@5.0.0/dist/tesseract.min.js'></script>

    <!-- Your React App Component -->
    <!-- Use type="text/babel" to allow Babel to transpile JSX -->
    <script type="text/babel">
        // Document Type Shortening Rules
        const documentTypeMapping = {
            "Life Limited Parts Status": "LLP",
            "Folio 12": "FOLIO12",
            "Movement Traceability Sheet": "MTS",
            "Airbus Aircraft Inspection Report": "AIR",
            "Boeing Aircraft Readiness Log": "ARL",
            "Boeing Aircraft Readiness Log Cover Page": "ARL COVER PAGE",
            "Engine Data Submittal": "EDS",
            "MTS LLP Hybrid": "MTS HYBRID",
            "Disk Sheet": "DISK SHEET",
            "Non-Incident Statement": "NIS",
            "Airworthiness Certificate": "AWD",
            "Industry Item List": "P&W Industry Item List",
        };

        // Function to extract specific table data based on column headers
        const extractSpecificTableData = (data) => {
            const lines = data.lines;
            if (!lines || lines.length === 0) {
                return [];
            }

            let headerLineIndex = -1;
            const headerKeywords = {
                'batch id': null,
                'name': null, // Alternative for Batch ID
                'total pages': null,
                'document type': null
            };

            // 1. Find the header line and approximate column positions
            for (let i = 0; i < lines.length; i++) {
                const lineText = lines[i].text.toLowerCase();
                // Check if the line contains key indicators for a header
                if (
                    (lineText.includes('batch id') || lineText.includes('name')) &&
                    lineText.includes('total pages') &&
                    lineText.includes('document type')
                ) {
                    headerLineIndex = i;
                    // Refine headerKeywords positions based on actual word positions in the detected header line
                    lines[i].words.forEach(word => {
                        const lowerWord = word.text.toLowerCase();
                        if (lowerWord.includes('batch') && lowerWord.includes('id')) headerKeywords['batch id'] = word.bbox.x0;
                        if (lowerWord.includes('name')) headerKeywords['name'] = word.bbox.x0;
                        if (lowerWord.includes('total') && lowerWord.includes('pages')) headerKeywords['total pages'] = word.bbox.x0;
                        if (lowerWord.includes('document') && lowerWord.includes('type')) headerKeywords['document type'] = word.bbox.x0;
                    });
                    break; // Found the header
                }
            }

            if (headerLineIndex === -1) {
                console.warn("Could not find a clear header line with 'Batch ID/Name', 'Total Pages', and 'Document Type'. Attempting general line-by-line parsing for best effort.");
                // Fallback: If no clear header, try to extract based on general line structure
                // This is a very basic fallback and might not yield accurate results for tables.
                return lines.map(line => {
                    const cells = line.text.split(/\s{2,}/); // Split by 2 or more spaces for basic column separation
                    let batchId = cells[0] || '';
                    let totalPages = cells[1] || '';
                    let documentType = cells[2] || '';

                    // Apply shortening if any cell looks like a document type
                    const matchedKey = Object.keys(documentTypeMapping).find(key => documentType.includes(key));
                    const shortDocumentType = matchedKey ? documentTypeMapping[matchedKey] : documentType;

                    return {
                        batchId: batchId,
                        shortDocumentType: shortDocumentType,
                        totalPages: totalPages
                    };
                }).filter(row => row.batchId || row.shortDocumentType || row.totalPages); // Filter out completely empty rows
            }

            const results = [];
            const columnXCoords = {
                batchId: headerKeywords['batch id'] || headerKeywords['name'], // Prioritize 'batch id' then 'name'
                totalPages: headerKeywords['total pages'],
                documentType: headerKeywords['document type']
            };

            // Filter out null coordinates and sort by x-position to define column order
            const sortedColumnKeys = Object.keys(columnXCoords)
                .filter(key => columnXCoords[key] !== null)
                .sort((a, b) => columnXCoords[a] - columnXCoords[b]);

            // 2. Iterate through lines *after* the header to extract data
            for (let i = headerLineIndex + 1; i < lines.length; i++) {
                const line = lines[i];
                const rowData = {
                    batchId: '',
                    shortDocumentType: '',
                    totalPages: ''
                };

                // Group words by their approximate column
                const cellsContent = {}; // Stores words grouped by their assigned column key
                line.words.forEach(word => {
                    let assignedColumnKey = null;
                    let minDistance = Infinity;

                    // Find the closest column header's x-coordinate for the word's x0
                    for (const colKey of sortedColumnKeys) {
                        const colX = columnXCoords[colKey];
                        if (colX !== null) {
                            const distance = Math.abs(word.bbox.x0 - colX);
                            // A tolerance is crucial for misalignment
                            if (distance < minDistance && distance < 70) { // Increased tolerance for potentially wider columns/misalignment
                                minDistance = distance;
                                assignedColumnKey = colKey;
                            }
                        }
                    }

                    if (assignedColumnKey) {
                        if (!cellsContent[assignedColumnKey]) {
                            cellsContent[assignedColumnKey] = [];
                        }
                        cellsContent[assignedColumnKey].push(word);
                    }
                });

                // Reconstruct cell text from grouped words and apply rules
                if (cellsContent.batchId) {
                    rowData.batchId = cellsContent.batchId.map(w => w.text).join(' ').trim();
                } else if (cellsContent.name) { // Use 'name' if 'batchId' wasn't explicitly found but 'name' column was
                    rowData.batchId = cellsContent.name.map(w => w.text).join(' ').trim();
                }


                if (cellsContent.totalPages) {
                    rowData.totalPages = cellsContent.totalPages.map(w => w.text).join(' ').trim();
                }

                if (cellsContent.documentType) {
                    let docTypeRaw = cellsContent.documentType.map(w => w.text).join(' ').trim();
                    // Apply shortening rules
                    const matchedKey = Object.keys(documentTypeMapping).find(key => docTypeRaw.includes(key));
                    rowData.shortDocumentType = matchedKey ? documentTypeMapping[matchedKey] : docTypeRaw;
                }

                // Only add row if it contains at least some data in the target columns
                if (rowData.batchId || rowData.shortDocumentType || rowData.totalPages) {
                    results.push(rowData);
                }
            }

            return results;
        };


        // Main App Component
        const App = () => {
            const [ocrText, setOcrText] = React.useState(''); // Still capture raw text internally for debugging if needed
            const [tableData, setTableData] = React.useState([]); // This will store the parsed specific table data
            const [loading, setLoading] = React.useState(false);
            const [message, setMessage] = React.useState('');
            const [showMessageBox, setShowMessageBox] = React.useState(false);
            const fileInputRef = React.useRef(null);
            const workerRef = React.useRef(null); // Ref to store the Tesseract worker instance

            const showMessage = (msg) => {
                setMessage(msg);
                setShowMessageBox(true);
            };

            const closeMessageBox = () => {
                setShowMessageBox(false);
                setMessage('');
            };

            // Initialize Tesseract worker once on component mount
            React.useEffect(() => {
                const initializeTesseractWorker = async () => {
                    if (!workerRef.current && window.Tesseract) {
                        try {
                            const worker = await window.Tesseract.createWorker('eng', 1, {
                                // Explicitly set workerPath to ensure it loads from unpkg
                                workerPath: 'https://unpkg.com/tesseract.js@5.0.0/dist/worker.min.js',
                                // langPath is often inferred if workerPath is correct,
                                // but if issues persist, explicitly set it:
                                // langPath: 'https://unpkg.com/tesseract.js@5.0.0/lang-data/',
                                logger: m => { /* console.log(m); */ }
                            });
                            workerRef.current = worker;
                            console.log('Tesseract worker initialized successfully.');
                        } catch (error) {
                            console.error('Failed to initialize Tesseract worker:', error);
                            showMessage('Failed to initialize OCR engine. Please try again or check your network connection.');
                        }
                    }
                };

                initializeTesseractWorker();

                // Cleanup worker on component unmount
                return () => {
                    if (workerRef.current) {
                        console.log('Terminating Tesseract worker.');
                        workerRef.current.terminate();
                        workerRef.current = null;
                    }
                };
            }, []); // Empty dependency array ensures this runs once on mount and cleans up on unmount

            // Handle file upload and OCR processing
            const handleFileChange = async (event) => {
                const file = event.target.files[0];
                if (!file) {
                    return;
                }

                setLoading(true);
                setOcrText(''); // Clear raw OCR text
                setTableData([]);
                setShowMessageBox(false);

                // Check if worker is ready
                if (!workerRef.current) {
                    showMessage('OCR engine is still loading or failed to initialize. Please wait a moment or refresh the page.');
                    setLoading(false);
                    return;
                }

                try {
                    // Use the pre-initialized worker for recognition
                    const { data } = await workerRef.current.recognize(file);

                    setOcrText(data.text); // Still capture raw text internally for debugging if needed, but not display
                    const extractedSpecificTable = extractSpecificTableData(data);
                    setTableData(extractedSpecificTable);

                    if (extractedSpecificTable.length === 0) {
                        showMessage('No specific table data (Batch ID, Total Pages, Document Type) could be extracted. Please ensure the image/PDF contains clear tabular data with these headers.');
                    }

                } catch (error) {
                    console.error('OCR Recognition Error:', error);
                    showMessage('An error occurred during OCR processing. Please try again.');
                    setOcrText('Error: Could not process the file.');
                    setTableData([]);
                } finally {
                    setLoading(false);
                }
            };

            // Function to download table data as CSV
            const downloadCsv = () => {
                if (tableData.length === 0) {
                    showMessage('No data to download.');
                    return;
                }

                // Create CSV header
                const csvHeader = ["Batch ID", "Short Document Type", "Total Pages"].join(',');
                // Map data to CSV rows
                const csvRows = tableData.map(row =>
                    `"${row.batchId.replace(/"/g, '""')}",` +
                    `"${row.shortDocumentType.replace(/"/g, '""')}",` +
                    `"${row.totalPages.replace(/"/g, '""')}"`
                ).join('\n');

                const csvContent = csvHeader + '\n' + csvRows;

                const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
                const link = document.createElement('a');
                if (link.download !== undefined) {
                    const url = URL.createObjectURL(blob);
                    link.setAttribute('href', url);
                    link.setAttribute('download', 'extracted_table_data.csv');
                    link.style.visibility = 'hidden';
                    document.body.appendChild(link);
                    link.click();
                    document.body.removeChild(link);
                } else {
                    showMessage('Your browser does not support downloading files directly. Please copy the text and paste it into a file.');
                }
            };

            return (
                <div className="min-h-screen bg-gray-100 flex items-center justify-center p-4">
                    <div className="bg-white p-8 rounded-xl shadow-lg w-full max-w-4xl border border-gray-200">
                        <h1 className="text-4xl font-extrabold text-center text-gray-800 mb-8">
                            <span className="text-indigo-600">Table</span> OCR Extractor
                        </h1>

                        {/* File Upload Section */}
                        <div className="mb-8">
                            <label htmlFor="fileInput" className="block text-lg font-medium text-gray-700 mb-3">
                                Upload an Image or PDF (containing tables):
                            </label>
                            <input
                                type="file"
                                id="fileInput"
                                ref={fileInputRef}
                                accept="image/*, application/pdf"
                                onChange={handleFileChange}
                                className="custom-file-input w-full"
                            />
                            <p className="text-sm text-gray-500 mt-2 text-center">Supported formats: JPG, PNG, GIF, BMP, PDF</p>
                            <p className="text-sm text-gray-500 mt-1 text-center font-semibold text-indigo-700">
                                Optimized for tables with 'Batch ID/Name', 'Total Pages', and 'Document Type' columns.
                            </p>
                        </div>

                        {/* Loading Indicator */}
                        {loading && (
                            <div className="flex items-center justify-center space-x-3 py-6">
                                <div className="animate-spin rounded-full h-10 w-10 border-b-2 border-indigo-500"></div>
                                <p className="text-xl text-indigo-600 font-medium">Processing table data...</p>
                            </div>
                        )}

                        {/* Extracted Table Display */}
                        {!loading && tableData.length > 0 && (
                            <div className="mb-8 p-4 border border-gray-300 rounded-lg bg-gray-50 overflow-auto max-h-96">
                                <h2 className="text-2xl font-semibold text-gray-700 mb-4">Extracted Table:</h2>
                                <table className="min-w-full divide-y divide-gray-200 border border-gray-200 rounded-lg">
                                    <thead className="bg-gray-200">
                                        <tr>
                                            <th className="px-6 py-3 text-left text-xs font-medium text-gray-600 uppercase tracking-wider">Batch ID</th>
                                            <th className="px-6 py-3 text-left text-xs font-medium text-gray-600 uppercase tracking-wider">Short Document Type</th>
                                            <th className="px-6 py-3 text-left text-xs font-medium text-gray-600 uppercase tracking-wider">Total Pages</th>
                                        </tr>
                                    </thead>
                                    <tbody className="bg-white divide-y divide-gray-200">
                                        {tableData.map((row, rowIndex) => (
                                            <tr key={rowIndex} className="hover:bg-gray-50">
                                                <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-800">{row.batchId}</td>
                                                <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-800">{row.shortDocumentType}</td>
                                                <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-800">{row.totalPages}</td>
                                            </tr>
                                        ))}
                                    </tbody>
                                </table>
                            </div>
                        )}

                        {/* Download Button */}
                        {!loading && tableData.length > 0 && (
                            <div className="flex justify-center mb-6">
                                <button
                                    onClick={downloadCsv}
                                    className="bg-green-600 hover:bg-green-700 text-white font-semibold py-2 px-6 rounded-lg shadow-md transition duration-300 ease-in-out transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-green-500 focus:ring-offset-2"
                                >
                                    Download as CSV
                                </button>
                            </div>
                        )}

                        {/* Message Box for Alerts */}
                        {showMessageBox && (
                            <div className="fixed inset-0 bg-gray-800 bg-opacity-75 flex items-center justify-center z-50">
                                <div className="bg-white p-6 rounded-lg shadow-xl max-w-sm w-full text-center">
                                    <p className="text-lg font-semibold text-gray-800 mb-4">{message}</p>
                                    <button
                                        onClick={closeMessageBox}
                                        className="bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-2 px-4 rounded-lg"
                                    >
                                        OK
                                    </button>
                                </div>
                            </div>
                        )}
                    </div>
                </div>
            );
        };

        // Render the App component into the root div
        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(React.createElement(App));
    </script>
</body>
</html>
