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
        // Define the App component directly within this script
        const App = () => {
            const [ocrText, setOcrText] = React.useState('');
            const [tableData, setTableData] = React.useState([]);
            const [loading, setLoading] = React.useState(false);
            const [message, setMessage] = React.useState('');
            const [showMessageBox, setShowMessageBox] = React.useState(false);
            const fileInputRef = React.useRef(null);

            const showMessage = (msg) => {
                setMessage(msg);
                setShowMessageBox(true);
            };

            const closeMessageBox = () => {
                setShowMessageBox(false);
                setMessage('');
            };

            const parseTableFromOCR = (data) => {
                const words = data.words;
                if (!words || words.length === 0) {
                    return [];
                }

                words.sort((a, b) => {
                    if (Math.abs(a.bbox.y0 - b.bbox.y0) < 10) {
                        return a.bbox.x0 - b.bbox.x0;
                    }
                    return a.bbox.y0 - b.bbox.y0;
                });

                const rows = [];
                let currentRow = [];
                let currentY = -1;
                const yTolerance = 15;

                words.forEach(word => {
                    if (currentY === -1 || Math.abs(word.bbox.y0 - currentY) > yTolerance) {
                        if (currentRow.length > 0) {
                            rows.push(currentRow);
                        }
                        currentRow = [word];
                        currentY = word.bbox.y0;
                    } else {
                        currentRow.push(word);
                    }
                });
                if (currentRow.length > 0) {
                    rows.push(currentRow);
                }

                const columnXPositions = new Set();
                rows.forEach(row => {
                    let lastX1 = 0;
                    row.forEach(word => {
                        if (word.bbox.x0 - lastX1 > 30) {
                            columnXPositions.add(word.bbox.x0);
                        }
                        lastX1 = word.bbox.x1;
                    });
                });

                const sortedColumnXPositions = Array.from(columnXPositions).sort((a, b) => a - b);

                if (sortedColumnXPositions.length === 0 && rows.length > 0) {
                    return rows.map(row => [row.map(word => word.text).join(' ')]);
                }

                const table = [];
                rows.forEach(row => {
                    const rowCells = Array(sortedColumnXPositions.length + 1).fill('');
                    let currentColumnIndex = 0;

                    row.forEach(word => {
                        let assignedColumn = -1;
                        for (let i = 0; i < sortedColumnXPositions.length; i++) {
                            if (word.bbox.x0 >= sortedColumnXPositions[i] - 10) {
                                assignedColumn = i + 1;
                            }
                        }

                        if (assignedColumn === -1) {
                            assignedColumn = 0;
                        }

                        if (rowCells[assignedColumn]) {
                            rowCells[assignedColumn] += ' ' + word.text;
                        } else {
                            rowCells[assignedColumn] = word.text;
                        }
                    });
                    table.push(rowCells.map(cell => cell.trim()));
                });

                const cleanedTable = table.filter(row => row.some(cell => cell.length > 0));
                return cleanedTable;
            };

            const handleFileChange = async (event) => {
                const file = event.target.files[0];
                if (!file) {
                    return;
                }

                setLoading(true);
                setOcrText('');
                setTableData([]);
                setShowMessageBox(false);

                if (typeof window.Tesseract === 'undefined') {
                    showMessage('Tesseract.js is not loaded. Please ensure the Tesseract.js CDN script is included in your HTML.');
                    setLoading(false);
                    return;
                }

                try {
                    const { data } = await window.Tesseract.recognize(
                        file,
                        'eng',
                        { logger: m => {} } // Simplified logger
                    );

                    setOcrText(data.text);
                    const parsedTable = parseTableFromOCR(data);
                    setTableData(parsedTable);

                    if (parsedTable.length === 0) {
                        showMessage('No table data could be extracted. Please ensure the image/PDF contains clear tabular data.');
                    }

                } catch (error) {
                    console.error('OCR Error:', error);
                    showMessage('An error occurred during OCR processing. Please try again.');
                    setOcrText('Error: Could not process the file.');
                    setTableData([]);
                } finally {
                    setLoading(false);
                }
            };

            const downloadCsv = () => {
                if (tableData.length === 0) {
                    showMessage('No data to download.');
                    return;
                }

                const csvContent = tableData.map(row =>
                    row.map(cell => `"${cell.replace(/"/g, '""')}"`).join(',')
                ).join('\n');

                const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
                const link = document.createElement('a');
                if (link.download !== undefined) {
                    const url = URL.createObjectURL(blob);
                    link.setAttribute('href', url);
                    link.setAttribute('download', 'extracted_table.csv');
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
                        </div>

                        {loading && (
                            <div className="flex items-center justify-center space-x-3 py-6">
                                <div className="animate-spin rounded-full h-10 w-10 border-b-2 border-indigo-500"></div>
                                <p className="text-xl text-indigo-600 font-medium">Processing table data...</p>
                            </div>
                        )}

                        {!loading && tableData.length > 0 && (
                            <div className="mb-8 p-4 border border-gray-300 rounded-lg bg-gray-50 overflow-auto max-h-96">
                                <h2 className="text-2xl font-semibold text-gray-700 mb-4">Extracted Table:</h2>
                                <table className="min-w-full divide-y divide-gray-200 border border-gray-200 rounded-lg">
                                    <thead className="bg-gray-200">
                                        <tr>
                                            {tableData[0] && tableData[0].map((header, index) => (
                                                <th key={index} className="px-6 py-3 text-left text-xs font-medium text-gray-600 uppercase tracking-wider">
                                                    {header}
                                                </th>
                                            ))}
                                        </tr>
                                    </thead>
                                    <tbody className="bg-white divide-y divide-gray-200">
                                        {tableData.slice(1).map((row, rowIndex) => (
                                            <tr key={rowIndex} className="hover:bg-gray-50">
                                                {row.map((cell, cellIndex) => (
                                                    <td key={cellIndex} className="px-6 py-4 whitespace-nowrap text-sm text-gray-800">
                                                        {cell}
                                                    </td>
                                                ))}
                                            </tr>
                                        ))}
                                    </tbody>
                                </table>
                            </div>
                        )}

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

                        {!loading && ocrText && (
                            <div className="mb-6">
                                <label htmlFor="ocrRawOutput" className="block text-lg font-medium text-gray-700 mb-3">
                                    Raw OCR Text (for reference):
                                </label>
                                <textarea
                                    id="ocrRawOutput"
                                    rows="8"
                                    className="w-full p-4 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-transparent outline-none bg-gray-50 text-gray-800"
                                    value={ocrText}
                                    readOnly
                                    placeholder="Raw OCR text will appear here..."
                                ></textarea>
                            </div>
                        )}

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
