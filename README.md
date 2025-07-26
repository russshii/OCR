import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, collection, onSnapshot, query, orderBy, serverTimestamp } from 'firebase/firestore';
import { Upload, FileText, XCircle, CheckCircle, Download, Loader2, Edit, Save, Trash2 } from 'lucide-react';

// Utility function to convert File to Base64
const fileToBase64 = (file) => {
    return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onload = () => resolve(reader.result.split(',')[1]);
        reader.onerror = (error) => reject(error);
    });
};

// Known batch types for validation/correction
const KNOWN_BATCH_TYPES = [
    'Non-Incident Statement',
    'MTS LFP Hybrid',
    'Daily Report',
    'Weekly Summary',
    'Monthly Report',
    'Quarterly Review',
    'Annual Statement',
    'Incident Report',
    'Customer Feedback',
    'Product Survey'
];

// Fuzzy matching for batch types (simple Levenshtein distance for demonstration)
const getEditDistance = (a, b) => {
    if (a.length === 0) return b.length;
    if (b.length === 0) return a.length;

    const matrix = [];

    // increment along the first column of each row
    for (let i = 0; i <= b.length; i++) {
        matrix[i] = [i];
    }

    // increment each column in the first row
    for (let j = 0; j <= a.length; j++) {
        matrix[0][j] = j;
    }

    // Fill in the rest of the matrix
    for (let i = 1; i <= b.length; i++) {
        for (let j = 1; j <= a.length; j++) {
            if (b.charAt(i - 1) === a.charAt(j - 1)) {
                matrix[i][j] = matrix[i - 1][j - 1];
            } else {
                matrix[i][j] = Math.min(matrix[i - 1][j - 1] + 1, // substitution
                    Math.min(matrix[i][j - 1] + 1, // insertion
                        matrix[i - 1][j] + 1)); // deletion
            }
        }
    }

    return matrix[b.length][a.length];
};

const fuzzyMatchBatchType = (input) => {
    let bestMatch = null;
    let minDistance = Infinity;

    const lowerInput = input.toLowerCase();

    for (const type of KNOWN_BATCH_TYPES) {
        const distance = getEditDistance(lowerInput, type.toLowerCase());
        if (distance < minDistance) {
            minDistance = distance;
            bestMatch = type;
        }
    }
    // Set a threshold for "fuzzy" match (e.g., if distance is too high, it's not a good match)
    // This threshold can be adjusted based on expected OCR error rates.
    if (minDistance <= Math.floor(input.length / 3) || minDistance <= 3) { // Example threshold
        return bestMatch;
    }
    return null; // No good fuzzy match found
};


const App = () => {
    const [selectedFiles, setSelectedFiles] = useState([]);
    const [extractedData, setExtractedData] = useState([]);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);
    const [editingRow, setEditingRow] = useState(null);
    const [editedData, setEditedData] = useState({});
    const [showModal, setShowModal] = useState(false);
    const [modalContent, setModalContent] = useState('');
    const [modalAction, setModalAction] = useState(null);

    // Firestore state
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [userExtractedData, setUserExtractedData] = useState([]);

    // Initialize Firebase
    useEffect(() => {
        try {
            const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
            const app = initializeApp(firebaseConfig);
            const firestore = getFirestore(app);
            const authentication = getAuth(app);
            setDb(firestore);
            setAuth(authentication);

            // Sign in anonymously or with custom token
            const signIn = async () => {
                try {
                    if (typeof __initial_auth_token !== 'undefined') {
                        await signInWithCustomToken(authentication, __initial_auth_token);
                    } else {
                        await signInAnonymously(authentication);
                    }
                } catch (e) {
                    console.error("Firebase Auth Error:", e);
                    setError("Failed to authenticate with Firebase.");
                }
            };
            signIn();

            // Listen for auth state changes
            const unsubscribeAuth = onAuthStateChanged(authentication, (user) => {
                if (user) {
                    setUserId(user.uid);
                } else {
                    setUserId(null);
                }
            });

            return () => unsubscribeAuth();
        } catch (e) {
            console.error("Firebase Initialization Error:", e);
            setError("Failed to initialize Firebase. Check __firebase_config.");
        }
    }, []);

    // Listen for real-time data from Firestore
    useEffect(() => {
        if (db && userId) {
            const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
            const userDocRef = collection(db, `artifacts/${appId}/users/${userId}/extracted_ocr_data`);
            const q = query(userDocRef, orderBy('timestamp', 'desc'));

            const unsubscribe = onSnapshot(q, (snapshot) => {
                const data = snapshot.docs.map(doc => ({
                    id: doc.id,
                    ...doc.data()
                }));
                setUserExtractedData(data);
            }, (err) => {
                console.error("Firestore data fetch error:", err);
                setError("Failed to fetch data from Firestore.");
            });

            return () => unsubscribe();
        }
    }, [db, userId]);


    const handleFileChange = (event) => {
        setError(null);
        const files = Array.from(event.target.files);
        setSelectedFiles(prev => [...prev, ...files]);
    };

    const handleRemoveFile = (indexToRemove) => {
        setSelectedFiles(prev => prev.filter((_, index) => index !== indexToRemove));
    };

    const processImageWithLLM = async (base64ImageData) => {
        const prompt = `You are an expert OCR post-processor. Analyze the provided image, which contains a table with batch details. Your task is to extract the following information for each row: 'Batch ID', 'Asset Name', 'Batch Type', 'Work Unit', and 'Pages of Single Batch'.
        For 'Batch ID', extract the full ID.
        For 'Asset Name', extract the part of the Batch ID between hyphens (e.g., 'btita' from '173974-btita-97c2'). If no hyphenated part, infer from common names or leave null.
        For 'Batch Type', identify the type from the third column.
        For 'Work Unit' and 'Pages of Single Batch', extract the integer values from the last two columns.
        Return the data as a JSON array of objects, where each object represents a row. If a field cannot be confidently extracted, use null.
        Example JSON structure:
        [
            {
                "Batch ID": "173974-btita-97c2",
                "Asset Name": "btita",
                "Batch Type": "Non-Incident Statement",
                "Work Unit": 1,
                "Pages of Single Batch": 1
            }
        ]
        `;

        const payload = {
            contents: [{
                role: "user",
                parts: [{
                    text: prompt
                }, {
                    inlineData: {
                        mimeType: "image/png", // Assuming PNG, adjust if needed
                        data: base64ImageData
                    }
                }]
            }],
            generationConfig: {
                responseMimeType: "application/json",
                responseSchema: {
                    type: "ARRAY",
                    items: {
                        type: "OBJECT",
                        properties: {
                            "Batch ID": {
                                "type": "STRING"
                            },
                            "Asset Name": {
                                "type": "STRING"
                            },
                            "Batch Type": {
                                "type": "STRING"
                            },
                            "Work Unit": {
                                "type": "INTEGER"
                            },
                            "Pages of Single Batch": {
                                "type": "INTEGER"
                            }
                        },
                        required: ["Batch ID", "Asset Name", "Batch Type", "Work Unit", "Pages of Single Batch"]
                    }
                }
            }
        };

        const apiKey = ""; // Canvas will provide this
        const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;

        try {
            const response = await fetch(apiUrl, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(payload)
            });

            if (!response.ok) {
                const errorData = await response.json();
                console.error("LLM API Error:", errorData);
                throw new Error(`LLM API request failed: ${errorData.error?.message || response.statusText}`);
            }

            const result = await response.json();
            if (result.candidates && result.candidates.length > 0 &&
                result.candidates[0].content && result.candidates[0].content.parts &&
                result.candidates[0].content.parts.length > 0) {
                const jsonString = result.candidates[0].content.parts[0].text;
                return JSON.parse(jsonString);
            } else {
                throw new Error("No content found in LLM response.");
            }
        } catch (err) {
            console.error("Error calling LLM API:", err);
            throw new Error(`Failed to process image with AI: ${err.message}`);
        }
    };

    const processExtractedData = (rawData) => {
        return rawData.map(row => {
            const processedRow = { ...row
            };

            // 1. Batch ID Validation (Basic Regex)
            const batchIdRegex = /^[0-9a-fA-F]{6}-[a-zA-Z0-9]+-[0-9a-fA-F]{4}$/;
            if (processedRow['Batch ID'] && !batchIdRegex.test(processedRow['Batch ID'])) {
                processedRow['Batch ID_error'] = true;
            }

            // 2. Asset Name Extraction/Correction
            if (processedRow['Batch ID']) {
                const parts = processedRow['Batch ID'].split('-');
                if (parts.length > 1) {
                    let extractedAsset = parts[1].toLowerCase();
                    // Simple dictionary-based correction for common OCR errors (e.g., britair vs btita)
                    if (extractedAsset.includes('britair')) {
                        extractedAsset = 'britair';
                    } else if (extractedAsset.includes('btita')) {
                        extractedAsset = 'btita';
                    }
                    processedRow['Asset Name'] = extractedAsset;
                } else {
                    processedRow['Asset Name'] = null; // Could not extract from Batch ID
                }
            }

            // 3. Batch Type Fuzzy Match
            if (processedRow['Batch Type']) {
                const correctedType = fuzzyMatchBatchType(processedRow['Batch Type']);
                if (correctedType) {
                    processedRow['Batch Type'] = correctedType;
                } else {
                    processedRow['Batch Type_error'] = true;
                }
            } else {
                processedRow['Batch Type_error'] = true; // Missing batch type
            }

            // 4. Work Unit & Page Count Validation/Correction
            ['Work Unit', 'Pages of Single Batch'].forEach(field => {
                let value = processedRow[field];
                if (typeof value === 'string') {
                    // Replace common OCR errors (e.g., 'l' with '1', 'o' with '0')
                    value = value.replace(/l/g, '1').replace(/o/g, '0').replace(/\s/g, '');
                }
                const parsedValue = parseInt(value, 10);
                if (isNaN(parsedValue)) {
                    processedRow[field] = null;
                    processedRow[`${field}_error`] = true;
                } else {
                    processedRow[field] = parsedValue;
                }
            });

            return processedRow;
        });
    };

    const handleProcessImages = async () => {
        if (selectedFiles.length === 0) {
            setError("Please select at least one image file.");
            return;
        }

        setLoading(true);
        setError(null);
        setExtractedData([]); // Clear previous results

        let allProcessedData = [];

        for (const file of selectedFiles) {
            try {
                const base64 = await fileToBase64(file);
                const rawLLMData = await processImageWithLLM(base64);
                const processed = processExtractedData(rawLLMData);
                allProcessedData = [...allProcessedData, ...processed];
            } catch (err) {
                console.error(`Error processing file ${file.name}:`, err);
                setError(`Failed to process ${file.name}: ${err.message}`);
                // Continue to next file even if one fails
            }
        }
        setExtractedData(allProcessedData);
        setLoading(false);

        // Save to Firestore
        if (db && userId && allProcessedData.length > 0) {
            const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
            const userDocRef = collection(db, `artifacts/${appId}/users/${userId}/extracted_ocr_data`);
            for (const dataRow of allProcessedData) {
                try {
                    await setDoc(doc(userDocRef), { // Use setDoc with a new doc ref to auto-generate ID
                        ...dataRow,
                        timestamp: serverTimestamp()
                    });
                } catch (e) {
                    console.error("Error saving data to Firestore:", e);
                    setError("Failed to save some data to Firestore.");
                }
            }
        }
    };

    const handleEditClick = (index) => {
        setEditingRow(index);
        setEditedData({ ...extractedData[index]
        });
    };

    const handleSaveEdit = () => {
        const updatedData = [...extractedData];
        updatedData[editingRow] = processExtractedData([editedData])[0]; // Re-process edited row for validation
        setExtractedData(updatedData);
        setEditingRow(null);

        // Update in Firestore
        if (db && userId && updatedData[editingRow] && updatedData[editingRow].id) {
            const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
            const docRef = doc(db, `artifacts/${appId}/users/${userId}/extracted_ocr_data`, updatedData[editingRow].id);
            setDoc(docRef, { ...updatedData[editingRow],
                timestamp: serverTimestamp()
            }, {
                merge: true
            }).catch(e => {
                console.error("Error updating document in Firestore:", e);
                setError("Failed to update data in Firestore.");
            });
        }
    };

    const handleCancelEdit = () => {
        setEditingRow(null);
        setEditedData({});
    };

    const handleInputChange = (e, field) => {
        setEditedData({
            ...editedData,
            [field]: e.target.value
        });
    };

    const handleExport = (format) => {
        if (extractedData.length === 0) {
            setModalContent("No data to export.");
            setModalAction(null);
            setShowModal(true);
            return;
        }

        // Correctly destructure properties with spaces by quoting them
        const dataToExport = extractedData.map((row) => {
            const newRow = { ...row
            };
            delete newRow['Batch ID_error'];
            delete newRow['Asset Name_error'];
            delete newRow['Batch Type_error'];
            delete newRow['Work Unit_error'];
            delete newRow['Pages of Single Batch_error'];
            return newRow;
        });

        if (format === 'json') {
            const jsonString = JSON.stringify(dataToExport, null, 2);
            const blob = new Blob([jsonString], {
                type: 'application/json'
            });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = 'ocr_data.json';
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            URL.revokeObjectURL(url);
        } else if (format === 'csv') {
            const headers = Object.keys(dataToExport[0] || {}).join(',');
            const rows = dataToExport.map(row =>
                Object.values(row).map(value => {
                    // Handle commas and quotes in CSV
                    if (typeof value === 'string' && value.includes(',')) {
                        return `"${value.replace(/"/g, '""')}"`;
                    }
                    return value;
                }).join(',')
            );
            const csvString = [headers, ...rows].join('\n');
            const blob = new Blob([csvString], {
                type: 'text/csv'
            });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = 'ocr_data.csv';
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            URL.revokeObjectURL(url);
        }
    };

    const showConfirmationModal = (message, action) => {
        setModalContent(message);
        setModalAction(() => action); // Store the function to be called
        setShowModal(true);
    };

    const handleModalConfirm = () => {
        if (modalAction) {
            modalAction();
        }
        setShowModal(false);
        setModalAction(null);
    };

    const handleModalCancel = () => {
        setShowModal(false);
        setModalAction(null);
    };


    return (
        <div className="min-h-screen bg-gray-100 p-4 font-inter text-gray-800 flex flex-col items-center">
            <style>{`
            @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
            body { font-family: 'Inter', sans-serif; }
            `}</style>

            <h1 className="text-4xl font-bold text-blue-700 mb-8 mt-4">Structured OCR App</h1>

            {userId && (
                <div className="bg-blue-100 text-blue-800 p-3 rounded-lg mb-6 shadow-md">
                    <p className="text-sm">Connected as User ID: <span className="font-mono text-blue-900 break-all">{userId}</span></p>
                </div>
            )}

            <div className="bg-white p-6 rounded-xl shadow-lg w-full max-w-4xl mb-8">
                <h2 className="text-2xl font-semibold text-gray-700 mb-4">Upload Documents</h2>

                <div className="border-2 border-dashed border-gray-300 rounded-lg p-6 text-center cursor-pointer hover:border-blue-500 transition-all duration-200"
                    onClick={() => document.getElementById('fileInput').click()}>
                    <Upload className="mx-auto h-12 w-12 text-gray-400 mb-3" />
                    <p className="text-gray-600 font-medium">Drag & drop images here, or click to browse</p>
                    <input type="file" id="fileInput" className="hidden" multiple accept="image/*" onChange={handleFileChange} />
                </div>

                {selectedFiles.length > 0 && (
                    <div className="mt-6">
                        <h3 className="text-lg font-medium text-gray-700 mb-3">Selected Files:</h3>
                        <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 gap-4">
                            {selectedFiles.map((file, index) => (
                                <div key={index} className="relative bg-gray-50 p-3 rounded-lg shadow-sm flex items-center justify-between">
                                    <div className="flex items-center">
                                        <FileText className="h-5 w-5 text-blue-500 mr-2" />
                                        <span className="text-sm text-gray-700 truncate">{file.name}</span>
                                    </div>
                                    <button onClick={() => handleRemoveFile(index)}
                                        className="ml-3 text-red-500 hover:text-red-700 p-1 rounded-full hover:bg-red-100 transition-colors">
                                        <XCircle className="h-5 w-5" />
                                    </button>
                                </div>
                            ))}
                        </div>
                        <button onClick={handleProcessImages} disabled={loading || selectedFiles.length === 0}
                            className="mt-6 w-full bg-blue-600 text-white py-3 px-6 rounded-lg font-semibold text-lg hover:bg-blue-700 transition-colors duration-300 flex items-center justify-center disabled:opacity-50 disabled:cursor-not-allowed shadow-md">
                            {loading ? (
                                <>
                                    <Loader2 className="animate-spin mr-3" /> Processing...
                                </>
                            ) : (
                                <>
                                    <CheckCircle className="mr-3" /> Extract Data
                                </>
                            )}
                        </button>
                    </div>
                )}
            </div>

            {error && (
                <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded-lg relative w-full max-w-4xl mb-8 shadow-md"
                    role="alert">
                    <strong className="font-bold">Error!</strong> <span className="block sm:inline">{error}</span>
                </div>
            )}

            {(extractedData.length > 0 || userExtractedData.length > 0) && (
                <div className="bg-white p-6 rounded-xl shadow-lg w-full max-w-4xl mb-8">
                    <h2 className="text-2xl font-semibold text-gray-700 mb-4">Extracted & Corrected Data</h2>

                    <div className="flex justify-end gap-2 mb-4">
                        <button onClick={() => handleExport('json')}
                            className="flex items-center px-4 py-2 bg-gray-200 text-gray-700 rounded-lg hover:bg-gray-300 transition-colors duration-200 text-sm font-medium shadow-sm">
                            <Download className="h-4 w-4 mr-2" /> Export JSON
                        </button>
                        <button onClick={() => handleExport('csv')}
                            className="flex items-center px-4 py-2 bg-gray-200 text-gray-700 rounded-lg hover:bg-gray-300 transition-colors duration-200 text-sm font-medium shadow-sm">
                            <Download className="h-4 w-4 mr-2" /> Export CSV
                        </button>
                    </div>

                    <div className="overflow-x-auto rounded-lg border border-gray-200">
                        <table className="min-w-full divide-y divide-gray-200">
                            <thead className="bg-gray-50">
                                <tr>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Batch ID</th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Asset Name</th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Batch Type</th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Work Unit</th>
                                    <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Pages of Single Batch</th>
                                    <th scope="col" className="relative px-6 py-3"><span className="sr-only">Edit</span></th>
                                </tr>
                            </thead>
                            <tbody className="bg-white divide-y divide-gray-200">
                                {userExtractedData.map((row, index) => (
                                    <tr key={row.id || index} className={index % 2 === 0 ? 'bg-white' : 'bg-gray-50'}>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">{editingRow === index ? (<input type="text" value={editedData['Batch ID'] || ''} onChange={(e) => handleInputChange(e, 'Batch ID')} className={`w-full p-2 border rounded-md ${editedData['Batch ID_error'] ? 'border-red-500' : 'border-gray-300'}`} />) : (<span className={row['Batch ID_error'] ? 'text-red-600 font-semibold' : ''}>{row['Batch ID'] || 'N/A'}</span>)}</td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">{editingRow === index ? (<input type="text" value={editedData['Asset Name'] || ''} onChange={(e) => handleInputChange(e, 'Asset Name')} className={`w-full p-2 border rounded-md ${editedData['Asset Name_error'] ? 'border-red-500' : 'border-gray-300'}`} />) : (<span className={row['Asset Name_error'] ? 'text-red-600 font-semibold' : ''}>{row['Asset Name'] || 'N/A'}</span>)}</td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">{editingRow === index ? (<input type="text" value={editedData['Batch Type'] || ''} onChange={(e) => handleInputChange(e, 'Batch Type')} className={`w-full p-2 border rounded-md ${editedData['Batch Type_error'] ? 'border-red-500' : 'border-gray-300'}`} />) : (<span className={row['Batch Type_error'] ? 'text-red-600 font-semibold' : ''}>{row['Batch Type'] || 'N/A'}</span>)}</td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">{editingRow === index ? (<input type="number" value={editedData['Work Unit'] || ''} onChange={(e) => handleInputChange(e, 'Work Unit')} className={`w-full p-2 border rounded-md ${editedData['Work Unit_error'] ? 'border-red-500' : 'border-gray-300'}`} />) : (<span className={row['Work Unit_error'] ? 'text-red-600 font-semibold' : ''}>{row['Work Unit'] !== null ? row['Work Unit'] : 'N/A'}</span>)}</td>
                                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-700">{editingRow === index ? (<input type="number" value={editedData['Pages of Single Batch'] || ''} onChange={(e) => handleInputChange(e, 'Pages of Single Batch')} className={`w-full p-2 border rounded-md ${editedData['Pages of Single Batch_error'] ? 'border-red-500' : 'border-gray-300'}`} />) : (<span className={row['Pages of Single Batch_error'] ? 'text-red-600 font-semibold' : ''}>{row['Pages of Single Batch'] !== null ? row['Pages of Single Batch'] : 'N/A'}</span>)}</td>
                                        <td className="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">{editingRow === index ? (<><button onClick={handleSaveEdit} className="text-green-600 hover:text-green-900 mr-2 p-1 rounded-full hover:bg-green-100 transition-colors"><Save className="h-5 w-5" /></button><button onClick={handleCancelEdit} className="text-gray-600 hover:text-gray-900 p-1 rounded-full hover:bg-gray-100 transition-colors"><XCircle className="h-5 w-5" /></button></>) : (<button onClick={() => handleEditClick(index)} className="text-blue-600 hover:text-blue-900 p-1 rounded-full hover:bg-blue-100 transition-colors"><Edit className="h-5 w-5" /></button>)}</td>
                                    </tr>
                                ))}
                            </tbody>
                        </table>
                    </div>
                </div>
            )}

            { /* Confirmation Modal */ } {showModal && (
                <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50">
                    <div className="bg-white p-6 rounded-lg shadow-xl max-w-sm w-full">
                        <h3 className="text-lg font-semibold mb-4">Confirmation</h3>
                        <p className="mb-6">{modalContent}</p>
                        <div className="flex justify-end gap-3">
                            <button onClick={handleModalCancel} className="px-4 py-2 bg-gray-200 text-gray-700 rounded-md hover:bg-gray-300 transition-colors">Cancel</button>
                            <button onClick={handleModalConfirm} className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition-colors">Confirm</button>
                        </div>
                    </div>
                </div>
            )}
        </div>
    );
};

export default App;
