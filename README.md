<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hair Salon Finder</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6;
        }
        .container {
            max-width: 900px;
        }
        .card {
            background-color: white;
            border-radius: 12px;
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1-px rgba(0, 0, 0, 0.06);
        }
        .button {
            transition: all 0.3s ease;
        }
        .button:hover {
            transform: translateY(-2px);
            box-shadow: 4px 6px 15px rgba(0, 0, 0, 0.1);
        }
        .loader {
            border: 4px solid #f3f3f3;
            border-top: 4px solid #3b82f6;
            border-radius: 50%;
            width: 30px;
            height: 30px;
            animation: spin 1s linear infinite;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        .animate-fade-in {
            animation: fadeIn 0.5s ease-in-out;
        }
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(10px); }
            to { opacity: 1; transform: translateY(0); }
        }
    </style>
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInWithCustomToken, signInAnonymously } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, addDoc, onSnapshot, getDocs, deleteDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        
        setLogLevel('Debug');
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        
        // This is a placeholder API key. You must replace it with your own key
        // and configure API restrictions in your Google Cloud Console for it to work.
        const GOOGLE_PLACES_API_KEY = 'AIzaSyAyh7Ik2dnRJsgwvBtiKX-rYCOyLOEjWsM';

        let db;
        let auth;
        let userId;
        let isAuthReady = false;

        const resultsContainer = document.getElementById('results-container');
        const searchInput = document.getElementById('search-input');
        const searchButton = document.getElementById('search-button');
        const exportButton = document.getElementById('export-button');
        const clearButton = document.getElementById('clear-button');
        const loadingSpinner = document.getElementById('loading-spinner');
        const statusMessage = document.getElementById('status-message');
        const userIdDisplay = document.getElementById('user-id-display');


        /**
         * A helper function to manage the status message and loading spinner UI.
         * @param {string} message The message to display.
         * @param {boolean} isLoading Whether to show the loading spinner.
         */
        const updateStatus = (message, isLoading) => {
            statusMessage.textContent = message;
            if (isLoading) {
                loadingSpinner.classList.remove('hidden');
            } else {
                loadingSpinner.classList.add('hidden');
            }
        };

        // Initialize Firebase and handle authentication
        const initializeFirebase = async () => {
            try {
                const app = initializeApp(firebaseConfig);
                auth = getAuth(app);
                db = getFirestore(app);

                const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : '';
                
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }

                userId = auth.currentUser?.uid;
                userIdDisplay.textContent = `User ID: ${userId || 'N/A'}`;
                isAuthReady = true;
                
                // Set up the real-time listener for the user's salons
                setupRealtimeListener();

            } catch (error) {
                console.error("Firebase initialization failed:", error);
                updateStatus('Failed to initialize the app. Please check the console for details.', false);
            }
        };

        const setupRealtimeListener = () => {
            if (!isAuthReady || !userId) return;
            const mySalonsRef = collection(db, `artifacts/${appId}/users/${userId}/my_salons`);
            onSnapshot(mySalonsRef, (querySnapshot) => {
                const salons = [];
                querySnapshot.forEach((doc) => {
                    salons.push({ id: doc.id, ...doc.data() });
                });
                displayResults(salons);
                if (salons.length > 0) {
                    exportButton.classList.remove('hidden');
                    clearButton.classList.remove('hidden');
                } else {
                    exportButton.classList.add('hidden');
                    clearButton.classList.add('hidden');
                }
            }, (error) => {
                console.error("Error listening to Firestore:", error);
                updateStatus('An error occurred while fetching your list.', false);
            });
        };
        
        /**
         * Searches for salons using the Google Places API and saves them to Firestore.
         * @param {string} city The city to search for.
         * @returns {Promise<number>} The number of salons added, or -1 on error.
         */
        const searchSalons = async (city) => {
            if (!isAuthReady || !userId) {
                return 0;
            }

            try {
                const query = `hair salon in ${city}`;
                const url = `https://maps.googleapis.com/maps/api/place/textsearch/json?query=${encodeURIComponent(query)}&key=${GOOGLE_PLACES_API_KEY}`;
                
                const response = await fetch(url);
                if (!response.ok) {
                    throw new Error('Network response was not ok');
                }

                const data = await response.json();
                const searchResults = data.results || [];

                const mySalonsRef = collection(db, `artifacts/${appId}/users/${userId}/my_salons`);
                
                // Add new results to the database
                for (const salon of searchResults) {
                    const salonData = {
                        name: salon.name || 'N/A',
                        address: salon.formatted_address || 'N/A',
                        phone: salon.formatted_phone_number || 'N/A',
                        website: salon.website || 'N/A'
                    };
                    await addDoc(mySalonsRef, salonData);
                }
                return searchResults.length;

            } catch (error) {
                console.error("Error during search and save:", error);
                if (error instanceof TypeError && error.message === 'Failed to fetch') {
                    updateStatus(`Search failed: This app cannot make live API calls from this temporary environment due to CORS security restrictions. However, this code will work as expected on your own website.`, false);
                } else {
                    updateStatus("An unexpected error occurred during the search.", false);
                }
                return -1; // Indicate a failure
            }
        };

        const displayResults = (results) => {
            resultsContainer.innerHTML = '';
            if (results.length === 0) {
                resultsContainer.innerHTML = '<p class="text-gray-500 text-center">Your salon list is empty. Start a search to add salons.</p>';
                return;
            }
            results.forEach((salon) => {
                const salonCard = document.createElement('div');
                salonCard.className = 'card p-6 mb-4 animate-fade-in flex flex-col md:flex-row justify-between items-start md:items-center space-y-4 md:space-y-0';
                salonCard.innerHTML = `
                    <div class="flex-1">
                        <h3 class="text-xl font-semibold text-gray-800">${salon.name}</h3>
                        <p class="text-gray-600 mt-2">${salon.address}</p>
                        <p class="text-gray-600">Phone: ${salon.phone}</p>
                        <a href="${salon.website}" target="_blank" class="text-blue-600 hover:underline mt-1 block">${salon.website}</a>
                    </div>
                `;
                resultsContainer.appendChild(salonCard);
            });
        };

        // Function to clear all documents from the user's list
        const clearSalons = async () => {
            if (!isAuthReady || !userId) {
                updateStatus('Authentication not ready. Cannot clear list.', false);
                return;
            }
            try {
                const mySalonsRef = collection(db, `artifacts/${appId}/users/${userId}/my_salons`);
                const querySnapshot = await getDocs(mySalonsRef);
                
                if (querySnapshot.empty) {
                    updateStatus("Your list is already clear.", false);
                    return;
                }
                
                const deletePromises = [];
                querySnapshot.forEach((doc) => {
                    deletePromises.push(deleteDoc(doc.ref));
                });
                await Promise.all(deletePromises);
                updateStatus("Your list has been cleared.", false);
            } catch (error) {
                console.error("Error clearing salons from Firestore:", error);
                updateStatus('An error occurred while clearing the list.', false);
            }
        };

        const exportSalons = () => {
            const results = Array.from(resultsContainer.querySelectorAll('.card')).map(card => {
                const name = card.querySelector('h3').textContent.trim();
                const address = card.querySelector('p:nth-of-type(1)').textContent.trim();
                const phone = card.querySelector('p:nth-of-type(2)').textContent.replace('Phone: ', '').trim();
                const website = card.querySelector('a').href;
                return { name, address, phone, website };
            });

            if (results.length === 0) {
                updateStatus('No results to export.', false);
                return;
            }

            const header = ['"Salon Name"', '"Address"', '"Phone"', '"Website"'];
            const csvRows = [
                header.join(','),
                ...results.map(row => 
                    `"${row.name.replace(/"/g, '""')}",` +
                    `"${row.address.replace(/"/g, '""')}",` +
                    `"${row.phone.replace(/"/g, '""')}",` +
                    `"${row.website.replace(/"/g, '""')}"`
                )
            ];

            const csvString = csvRows.join('\n');
            const blob = new Blob([csvString], { type: 'text/csv;charset=utf-8;' });
            const link = document.createElement('a');
            const url = URL.createObjectURL(blob);
            link.setAttribute('href', url);
            link.setAttribute('download', 'hair_salons.csv');
            link.style.display = 'none';
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);

            updateStatus('Results exported to hair_salons.csv!', false);
        };

        searchButton.addEventListener('click', async () => {
            const cityInput = searchInput.value.trim();
            if (!cityInput) {
                updateStatus('Please enter a city name.', false);
                return;
            }

            updateStatus('Searching...', true);
            
            try {
                const salonsAdded = await searchSalons(cityInput);

                let message = '';
                if (salonsAdded > 0) {
                    message = `Search complete! Found ${salonsAdded} salons in ${cityInput}.`;
                } else {
                    message = `No salons were found for "${cityInput}". Please check the city name.`;
                }
                
                updateStatus(message, false);

            } catch (error) {
                console.error("Search failed:", error);
                updateStatus("An unexpected error occurred during the search.", false);
            }
        });

        exportButton.addEventListener('click', exportSalons);
        clearButton.addEventListener('click', clearSalons);

        // Initial setup
        initializeFirebase();
    </script>
</head>
<body class="bg-gray-100 flex items-center justify-center min-h-screen p-4">

    <div class="container mx-auto p-8 card">
        <h1 class="text-3xl md:text-4xl font-bold text-center text-gray-800 mb-6">Hair Salon Finder</h1>
        <p class="text-center text-gray-600 mb-8">
            Enter a U.S. city to find local hair salons and build your marketing list.
        </p>

        <div class="flex flex-col sm:flex-row items-center space-y-4 sm:space-y-0 sm:space-x-4 mb-6">
            <input type="text" id="search-input" placeholder="e.g., Los Angeles, CA" class="flex-1 p-3 border-2 border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500">
            <button id="search-button" class="button px-6 py-3 bg-blue-600 text-white font-semibold rounded-lg hover:bg-blue-700">
                Search
            </button>
        </div>

        <div id="status-message" class="text-center text-gray-700 mt-6 mb-6">Enter a city and click Search.</div>
        
        <div class="flex justify-center mb-6">
            <div id="loading-spinner" class="loader hidden"></div>
        </div>

        <div id="results-container" class="space-y-4">
            <!-- Search results will be displayed here -->
        </div>

        <div class="text-center mt-6 flex justify-center space-x-4">
            <button id="export-button" class="button px-6 py-3 bg-green-600 text-white font-semibold rounded-lg hover:bg-green-700 hidden">
                Export List
            </button>
            <button id="clear-button" class="button px-6 py-3 bg-gray-600 text-white font-semibold rounded-lg hover:bg-gray-700 hidden">
                Clear List
            </button>
        </div>
        
        <p id="user-id-display" class="mt-8 text-center text-sm text-gray-500"></p>
    </div>

</body>
</html>
