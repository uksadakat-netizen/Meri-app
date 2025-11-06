Meri app
<!DOCTYPE html>
<html lang="hi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ऐप स्टोर अपलोडर</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f7f7f7;
            color: #1f2937;
        }
        .container-wrapper {
            max-width: 768px;
            margin: auto;
            padding: 1rem;
        }
        .card {
            background-color: white;
            border-radius: 12px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }
        input[type="text"], textarea {
            border: 1px solid #d1d5db;
            padding: 10px;
            border-radius: 8px;
            width: 100%;
            box-sizing: border-box;
            transition: border-color 0.2s;
        }
        input[type="text"]:focus, textarea:focus {
            outline: none;
            border-color: #4f46e5;
            box-shadow: 0 0 0 1px #4f46e5;
        }
        .app-icon {
            width: 56px;
            height: 56px;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 24px;
            font-weight: 700;
            border-radius: 12px;
            color: white;
            background-color: #3b82f6; /* Blue */
        }
        /* Style for Monetized button */
        .btn-monetize {
            background-color: #7c3aed; /* Violet */
            color: white;
            padding: 10px 15px;
            border-radius: 8px;
            font-weight: 600;
            transition: background-color 0.2s;
            cursor: pointer;
            text-align: center;
            display: block;
            margin-top: 0.5rem;
        }
        .btn-monetize:hover {
            background-color: #6d28d9;
        }
        /* Style for Direct Download button */
        .btn-direct {
            background-color: #10b981; /* Emerald Green */
            color: white;
            padding: 10px 15px;
            border-radius: 8px;
            font-weight: 600;
            transition: background-color 0.2s;
            cursor: pointer;
            text-align: center;
            display: block;
            margin-top: 0.5rem;
        }
        .btn-direct:hover {
            background-color: #059669;
        }
        .app-card-monetized {
            border: 2px solid #a78bfa; /* Light Violet border for monetized apps */
        }
    </style>
</head>
<body>
    <!-- Firebase SDK Imports -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, addDoc, setDoc, updateDoc, deleteDoc, onSnapshot, collection, query, orderBy, getDocs, limit, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";

        // Global Firebase Variables (Provided by Canvas Environment)
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        
        // Initialize Firebase
        let app, db, auth;
        let userId = '';
        let isAuthReady = false;
        
        // Set Firebase Log Level for debugging
        setLogLevel('debug');

        try {
            app = initializeApp(firebaseConfig);
            db = getFirestore(app);
            auth = getAuth(app);
        } catch (e) {
            console.error("Firebase Initialization Error:", e);
            document.getElementById('loading-message').textContent = 'Firebase शुरू करने में त्रुटि।';
        }

        const MAX_RETRIES = 5;
        const BASE_DELAY = 1000;

        // Utility for Exponential Backoff Retry
        async function withRetry(fn, ...args) {
            for (let i = 0; i < MAX_RETRIES; i++) {
                try {
                    return await fn(...args);
                } catch (error) {
                    if (error.code === 'unavailable' || error.code === 'resource-exhausted') {
                        const delay = BASE_DELAY * Math.pow(2, i);
                        console.warn(`Firestore operation failed, retrying in ${delay}ms:`, error);
                        if (i === MAX_RETRIES - 1) throw error;
                        await new Promise(resolve => setTimeout(resolve, delay));
                    } else {
                        throw error;
                    }
                }
            }
        }

        // Authentication Setup
        onAuthStateChanged(auth, async (user) => {
            if (!user) {
                try {
                    if (initialAuthToken) {
                        await signInWithCustomToken(auth, initialAuthToken);
                    } else {
                        await signInAnonymously(auth);
                    }
                    // User will be available in the next onAuthStateChanged call
                } catch (error) {
                    console.error("Authentication Error:", error);
                    document.getElementById('loading-message').textContent = 'प्रमाणीकरण में त्रुटि।';
                }
            } else {
                userId = user.uid;
                isAuthReady = true;
                console.log("User Authenticated:", userId);
                document.getElementById('loading-message').textContent = 'ऐप लिस्टिंग लोड हो रही है...';
                loadApps();
                setupFormListener(); // Setup form listener only after auth is ready
            }
        });

        // Firestore Collection Path
        function getCollectionRef() {
            // Using a public collection for all users to see the uploaded apps
            return collection(db, `artifacts/${appId}/public/data/apps`);
        }

        // --- Data Submission Logic ---
        async function handleSubmit(event) {
            event.preventDefault();
            if (!isAuthReady) {
                document.getElementById('message-box').textContent = 'प्रमाणीकरण की प्रतीक्षा कर रहा है...';
                return;
            }

            const form = event.target;
            const appName = form.appName.value.trim();
            const developerName = form.developerName.value.trim();
            const directLink = form.directLink.value.trim();
            const monetizationLink = form.monetizationLink.value.trim();
            const description = form.description.value.trim();
            
            const messageBox = document.getElementById('message-box');
            messageBox.className = 'text-sm font-medium p-3 rounded-lg mt-4';

            if (!appName || !directLink || !developerName) {
                messageBox.textContent = 'ऐप का नाम, डेवलपर का नाम और डायरेक्ट लिंक भरना ज़रूरी है।';
                messageBox.classList.add('bg-red-100', 'text-red-700');
                return;
            }

            const appData = {
                appName,
                developerName,
                directLink,
                monetizationLink: monetizationLink || null, // Allow null if empty
                description,
                uploaderId: userId, // Store the Uploader's ID
                timestamp: serverTimestamp() // Add server timestamp
            };

            try {
                await withRetry(addDoc, getCollectionRef(), appData);
                messageBox.textContent = 'ऐप सफलतापूर्वक अपलोड हो गई है! GitHub पर कमाई शुरू करने के लिए लिंक शेयर करें।';
                messageBox.classList.add('bg-green-100', 'text-green-700');
                form.reset();
            } catch (e) {
                console.error("Error adding document: ", e);
                messageBox.textContent = 'अपलोड करने में त्रुटि आई। कृपया कंसोल देखें।';
                messageBox.classList.add('bg-red-100', 'text-red-700');
            }
        }

        function setupFormListener() {
            const form = document.getElementById('app-upload-form');
            if (form) {
                form.addEventListener('submit', handleSubmit);
            }
        }

        // --- Data Loading and Rendering Logic ---
        function createInitial(name) {
            return name.charAt(0).toUpperCase();
        }

        function renderApp(doc) {
            const data = doc.data();
            const appId = doc.id;
            const isMonetized = !!data.monetizationLink;

            const cardClasses = isMonetized 
                ? 'card p-4 mb-4 app-card-monetized' 
                : 'card p-4 mb-4 bg-gray-50';

            const monetizedTag = isMonetized 
                ? `<span class="text-xs font-semibold text-white bg-purple-500 rounded-full px-2 py-0.5 ml-2">Monetized</span>` 
                : '';

            const appList = document.getElementById('app-list');
            const card = document.createElement('div');
            card.className = cardClasses;
            card.id = `app-${appId}`;

            card.innerHTML = `
                <div class="flex items-start mb-3">
                    <div class="app-icon mr-4">${createInitial(data.appName)}</div>
                    <div class="flex-grow">
                        <h2 class="text-xl font-bold text-gray-800 flex items-center">${data.appName} ${monetizedTag}</h2>
                        <p class="text-sm text-gray-500">डेवलपर: ${data.developerName}</p>
                    </div>
                </div>
                <p class="text-gray-600 mb-4">${data.description || 'कोई विवरण नहीं दिया गया।'}</p>
                <div class="grid grid-cols-1 md:grid-cols-2 gap-3">
                    ${isMonetized ? `
                        <a href="${data.monetizationLink}" target="_blank" rel="noopener noreferrer" class="btn-monetize">
                            विज्ञापन लिंक से देखें (कमाई)
                        </a>
                    ` : ''}
                    <a href="${data.directLink}" target="_blank" rel="noopener noreferrer" class="${isMonetized ? 'btn-direct' : 'btn-direct w-full'}">
                        ${isMonetized ? 'डायरेक्ट डाउनलोड' : 'डाउनलोड करें'}
                    </a>
                </div>
                <div class="mt-3 pt-2 border-t border-gray-100">
                    <p class="text-xs text-gray-400">अपलोडर ID: ${data.uploaderId}</p>
                </div>
            `;

            // Prepend new app to the list
            appList.prepend(card); 
        }

        function loadApps() {
            if (!isAuthReady) return;
            const appList = document.getElementById('app-list');
            // Clear existing content to prevent duplicates on refresh/re-render
            appList.innerHTML = ''; 

            const appsQuery = query(getCollectionRef(), limit(50)); // Fetch up to 50 apps

            // Real-time listener for app updates
            onSnapshot(appsQuery, (snapshot) => {
                let initialLoad = appList.children.length === 0;

                snapshot.docChanges().forEach((change) => {
                    const docId = change.doc.id;
                    const existingElement = document.getElementById(`app-${docId}`);

                    if (change.type === "added") {
                        // For newly added apps or initial load
                        if (!existingElement) {
                            renderApp(change.doc);
                        }
                    } else if (change.type === "modified") {
                        // If the app data is modified, simply re-render it
                        if (existingElement) {
                            existingElement.remove();
                            renderApp(change.doc);
                        }
                    } else if (change.type === "removed") {
                        // If the app is deleted
                        if (existingElement) {
                            existingElement.remove();
                        }
                    }
                });

                if (initialLoad && snapshot.empty) {
                     document.getElementById('loading-message').textContent = 'अभी कोई ऐप लिस्टिंग नहीं है। आप नीचे से पहली ऐप अपलोड करें!';
                } else if (!snapshot.empty) {
                     document.getElementById('loading-message').style.display = 'none';
                }
            }, (error) => {
                console.error("Error listening to apps collection: ", error);
                document.getElementById('loading-message').textContent = 'ऐप लिस्ट लोड करने में त्रुटि।';
            });
        }
        
        // This is necessary to kick off the auth process and subsequent data load
        // It runs once the script module is loaded.
    </script>

    <div class="container-wrapper">
        <header class="text-center py-6">
            <h1 class="text-3xl font-extrabold text-gray-900">ऐप स्टोर अपलोडर और व्यूअर</h1>
            <p class="text-md text-gray-600 mt-2">अपनी ऐप लिस्ट करें और मोनेटाइजेशन लिंक जोड़कर कमाई शुरू करें।</p>
            <p id="loading-message" class="text-md font-semibold text-indigo-600 mt-4">प्रमाणीकरण और डेटाबेस से जुड़ रहा है...</p>
        </header>

        <!-- APP LISTING FORM -->
        <div class="card p-6 mb-8">
            <h2 class="text-xl font-bold mb-4 text-indigo-600">नई ऐप अपलोड करें</h2>
            <form id="app-upload-form">
                <div class="space-y-4">
                    <div>
                        <label for="appName" class="block text-sm font-medium text-gray-700 mb-1">ऐप का नाम *</label>
                        <input type="text" id="appName" name="appName" required placeholder="जैसे: मेरा कैलकुलेटर प्रो">
                    </div>
                    <div>
                        <label for="developerName" class="block text-sm font-medium text-gray-700 mb-1">डेवलपर का नाम *</label>
                        <input type="text" id="developerName" name="developerName" required placeholder="आपका नाम / कंपनी का नाम">
                    </div>
                    <div>
                        <label for="description" class="block text-sm font-medium text-gray-700 mb-1">विवरण (Description)</label>
                        <textarea id="description" name="description" rows="3" placeholder="ऐप के बारे में 2-3 लाइनें लिखें..."></textarea>
                    </div>
                    <div>
                        <label for="directLink" class="block text-sm font-medium text-gray-700 mb-1">ऐप डायरेक्ट लिंक (Play Store या APK URL) *</label>
                        <input type="text" id="directLink" name="directLink" required placeholder="https://play.google.com/store/apps/details?id=...">
                    </div>
                    <div class="p-3 bg-yellow-50 rounded-lg border border-yellow-200">
                        <label for="monetizationLink" class="block text-sm font-bold text-gray-800 mb-1">मोनेटाइजेशन/विज्ञापन लिंक (कमाई के लिए) - वैकल्पिक</label>
                        <p class="text-xs text-gray-600 mb-2">यहाँ **Ad-based URL Shortener** से छोटा किया गया लिंक डालें।</p>
                        <input type="text" id="monetizationLink" name="monetizationLink" placeholder="https://shorte.st/YourMonetizedLink">
                    </div>
                </div>
                <button type="submit" class="w-full mt-6 bg-indigo-600 text-white p-3 rounded-lg font-semibold hover:bg-indigo-700 transition duration-150">
                    ऐप लिस्ट करें
                </button>
                <div id="message-box" class="text-sm font-medium p-3 rounded-lg mt-4 hidden"></div>
            </form>
        </div>

        <!-- UPLOADED APP LIST -->
        <h2 class="text-2xl font-bold text-gray-900 mb-4 border-b pb-2">अपलोड की गई ऐप्स (सभी यूज़र्स द्वारा)</h2>
        <div id="app-list">
            <!-- Apps will be rendered here by JavaScript -->
        </div>
        
        <footer class="text-center py-4 mt-8 border-t border-gray-200">
            <p class="text-sm text-gray-500">यूज़र ID: <span id="user-id-display">लोड हो रहा है...</span></p>
        </footer>
    </div>
    
    <script>
        // Display User ID once available (optional, for debugging/identification)
        const userIdDisplay = document.getElementById('user-id-display');
        const checkUserId = setInterval(() => {
            if (window.userId) {
                userIdDisplay.textContent = window.userId;
                clearInterval(checkUserId);
            }
        }, 500);
    </script>
</body>
</html>

