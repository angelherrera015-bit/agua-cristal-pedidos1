<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AGUA CRISTAL ZONA 6 - Pedidos</title>
    <!-- Carga de Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Carga de íconos Lucide para React (usados aquí como iconos generales) -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        /* Estilos personalizados para la paleta azul agua */
        :root {
            --color-primary: #06b6d4; /* Tailwind cyan-500 */
            --color-secondary: #0891b2; /* Tailwind cyan-600 */
            --color-light: #f0fdfa; /* Tailwind cyan-50 */
            --color-dark: #0e7490; /* Tailwind cyan-700 */
        }
        .bg-water-primary { background-color: var(--color-primary); }
        .hover\:bg-water-secondary:hover { background-color: var(--color-secondary); }
        .text-water-primary { color: var(--color-primary); }
        .border-water-primary { border-color: var(--color-primary); }
        .shadow-water { box-shadow: 0 10px 15px -3px rgba(6, 182, 212, 0.5), 0 4px 6px -2px rgba(6, 182, 212, 0.05); }

        body { font-family: 'Inter', sans-serif; background-color: var(--color-light); }

        /* Estilo para los botones principales */
        .main-button {
            transition: all 0.3s ease;
            transform: scale(1);
        }
        .main-button:hover {
            transform: scale(1.03);
            box-shadow: 0 20px 25px -5px rgba(6, 182, 212, 0.4);
        }

        /* Estilo para el botón de WhatsApp */
        .whatsapp-button {
            transition: all 0.3s ease;
            box-shadow: 0 4px 6px -1px rgba(34, 197, 94, 0.1), 0 2px 4px -2px rgba(34, 197, 94, 0.06);
        }
        .whatsapp-button:hover {
            background-color: #16a34a; /* green-600 */
            transform: translateY(-1px);
        }
    </style>
</head>
<body class="min-h-screen flex flex-col antialiased">

    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, signOut } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setLogLevel, addDoc, onSnapshot, collection, query, where, getDocs, updateDoc, deleteDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global Firebase and App variables
        let app, db, auth;
        let userId = 'anon';
        let userRole = null; // 'ADMINISTRADOR' or 'USUARIO 1'
        let currentView = 'login';
        let dailySales = 0;
        let todayOrders = [];
        let isFirestoreAvailable = true; // Nuevo flag para el estado de la DB

        // --- PRECIOS ---
        const PRECIO_DOMICILIO = 10.00;
        const PRECIO_FISICA = 9.00;

        // --- ROLES y CREDENCIALES (Hardcoded para simulación) ---
        const CREDENTIALS = {
            'ADMINISTRADOR': 'Angel2006$',
            'USUARIO 1': 'Aguacristal06'
        };

        // =================================================================================
        // BLOQUE DE CONFIGURACIÓN DE FIREBASE
        // SI EJECUTAS ESTE ARCHIVO FUERA DEL ENTORNO DE TRABAJO (EJ: EN TU PROPIO SERVIDOR),
        // DEBES REEMPLAZAR LA VARIABLE FIREBASE_CONFIG CON TUS CREDENCIALES REALES.
        // =================================================================================
        const APP_ID = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const INITIAL_AUTH_TOKEN = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        
        // *** INSERTA AQUÍ TU OBJETO firebaseConfig (DEL PASO 3 ARRIBA) ***
        // Ejemplo de estructura: 
        // const FIREBASE_CONFIG = { apiKey: "...", authDomain: "...", projectId: "...", ... };
        // Si no pegas tu configuración, la aplicación buscará las credenciales del entorno.
        const FIREBASE_CONFIG = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        // =================================================================================


        const COLLECTION_PATH = `/artifacts/${APP_ID}/public/data/orders`;

        // Utility: Get date in YYYY-MM-DD format
        const getTodayDate = () => new Date().toISOString().slice(0, 10);

        // --- FIREBASE INITIALIZATION AND AUTH ---
        window.initFirebase = async () => {
            const loadingMessageElement = document.getElementById('loading-message');

            if (Object.keys(FIREBASE_CONFIG).length === 0) {
                console.error("Firebase Config not available. Entering demonstration mode.");
                isFirestoreAvailable = false;
                // Set dummy variables (null is fine, functions will check isFirestoreAvailable)
                app = null;
                db = null;
                auth = { currentUser: { uid: 'dummy-user' } }; // Mock auth object for login to proceed

                loadingMessageElement.innerHTML = '<span class="text-yellow-600 font-bold">⚠️ Advertencia:</span> Base de datos inactiva (Modo de Demostración). Solo funcionará la interfaz.';
                
                // Set up dummy data for the list view
                window.setupRealtimeListener(); 
                
                window.renderApp(); 
                return;
            }

            // --- REAL FIREBASE INIT ---
            try {
                setLogLevel('debug');
                app = initializeApp(FIREBASE_CONFIG);
                db = getFirestore(app);
                auth = getAuth(app);

                // Sign in with custom token or anonymously
                await new Promise(resolve => {
                    onAuthStateChanged(auth, async (user) => {
                        if (user) {
                            userId = user.uid;
                            console.log("Firebase Auth successful. User ID:", userId);
                        } else {
                            if (INITIAL_AUTH_TOKEN) {
                                await signInWithCustomToken(auth, INITIAL_AUTH_TOKEN);
                            } else {
                                await signInAnonymously(auth);
                            }
                        }
                        resolve();
                    });
                });

                // Start listening to orders
                window.setupRealtimeListener();

                // Initial UI rendering
                window.renderApp();

            } catch (error) {
                console.error("Error during Firebase initialization:", error);
                loadingMessageElement.textContent = `Error de Firebase: ${error.message}`;
                isFirestoreAvailable = false; // Ensure flag is set on error too
                window.renderApp('login'); // Render the login screen even on auth/init failure
            }
        };

        // --- STATE MANAGEMENT AND RENDERING ---
        window.renderApp = (newView = currentView) => {
            currentView = newView;
            const appContainer = document.getElementById('app-container');
            const isAdmin = userRole === 'ADMINISTRADOR';

            // Clear container
            appContainer.innerHTML = '';

            // Render based on the current view
            let content = '';

            if (!userRole) {
                content = window.renderLogin();
            } else if (currentView === 'home') {
                content = window.renderHome();
            } else if (currentView === 'form') {
                content = window.renderOrderForm();
            } else if (currentView === 'orders') {
                content = window.renderOrdersList(isAdmin);
            }

            appContainer.innerHTML = content;
            lucide.createIcons(); // Initialize Lucide icons
            window.attachEventListeners();

            // Display DB status notification if not available
            if (!isFirestoreAvailable && currentView !== 'login') {
                 const statusDiv = document.createElement('div');
                 statusDiv.className = 'fixed top-0 left-0 right-0 bg-yellow-400 text-yellow-900 p-2 text-center font-semibold text-sm z-50';
                 statusDiv.textContent = '⚠️ Modo de Demostración: La base de datos no está activa. Los pedidos no se guardarán.';
                 document.body.appendChild(statusDiv);
            }
        };

        window.renderLogin = () => {
            return `
                <div id="login-screen" class="p-8 max-w-sm mx-auto my-16 bg-white rounded-xl shadow-water border-4 border-water-primary">
                    <h1 class="text-3xl font-extrabold text-center mb-4 text-water-primary">AGUA CRISTAL ZONA 6</h1>
                    <p class="text-center text-sm text-gray-500 mb-6">Inicia Sesión para Continuar</p>
                    <div id="login-error" class="text-red-600 text-center mb-4 font-semibold hidden"></div>

                    <div class="space-y-4">
                        <div>
                            <label for="username" class="block text-sm font-medium text-gray-700">Usuario (ADMINISTRADOR o USUARIO 1)</label>
                            <input type="text" id="username" class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-water-primary focus:border-water-primary sm:text-sm" placeholder="ADMINISTRADOR o USUARIO 1" value="">
                        </div>
                        <div>
                            <label for="password" class="block text-sm font-medium text-gray-700">Contraseña</label>
                            <input type="password" id="password" class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-water-primary focus:border-water-primary sm:text-sm">
                        </div>
                        <button id="login-button" class="w-full flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-lg font-medium text-white bg-water-primary hover:bg-water-secondary focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-water-primary transition duration-150 ease-in-out">
                            Ingresar
                        </button>
                    </div>
                </div>
            `;
        };

        window.renderHome = () => {
            return `
                <header class="w-full bg-white shadow p-4 flex justify-between items-center sticky top-0 z-10">
                    <h1 class="text-xl font-bold text-water-dark">Bienvenido, ${userRole}</h1>
                    <div class="flex items-center space-x-4">
                        <button id="view-orders-button" class="flex items-center px-4 py-2 text-sm font-medium rounded-full bg-cyan-100 text-water-dark hover:bg-cyan-200 transition">
                            <i data-lucide="list-ordered" class="w-4 h-4 mr-2"></i> Ver Pedidos
                        </button>
                        <button id="logout-button" class="flex items-center px-4 py-2 text-sm font-medium rounded-full bg-red-100 text-red-700 hover:bg-red-200 transition">
                            <i data-lucide="log-out" class="w-4 h-4 mr-2"></i> Salir
                        </button>
                    </div>
                </header>
                <div class="flex-grow flex flex-col justify-center items-center p-6 space-y-8 max-w-4xl mx-auto">
                    <h2 class="text-4xl font-extrabold text-gray-800 text-center mb-6">Selecciona el Tipo de Pedido</h2>

                    <div class="grid md:grid-cols-2 gap-8 w-full">
                        <button id="order-domicilio" data-type="A DOMICILIO" class="main-button flex flex-col items-center justify-center p-10 bg-white rounded-2xl shadow-water text-water-dark border-4 border-water-primary">
                            <i data-lucide="truck" class="w-16 h-16 mb-4"></i>
                            <span class="text-2xl font-bold">A DOMICILIO</span>
                            <span class="text-sm mt-1 text-gray-500">Q.${PRECIO_DOMICILIO.toFixed(2)} por Garrafón</span>
                        </button>

                        <button id="order-fisica" data-type="ENTREGA FÍSICA" class="main-button flex flex-col items-center justify-center p-10 bg-white rounded-2xl shadow-water text-water-dark border-4 border-water-primary">
                            <i data-lucide="store" class="w-16 h-16 mb-4"></i>
                            <span class="text-2xl font-bold">ENTREGA FÍSICA</span>
                            <span class="text-sm mt-1 text-gray-500">Q.${PRECIO_FISICA.toFixed(2)} por Garrafón</span>
                        </button>
                    </div>
                </div>
            `;
        };

        window.renderOrderForm = (orderType = null) => {
            // Retrieve type from localStorage if needed (e.g., after a reload)
            const type = orderType || localStorage.getItem('currentOrderType');
            if (!type) {
                setTimeout(() => window.renderApp('home'), 0);
                return 'Cargando...';
            }

            const isDomicilio = type === 'A DOMICILIO';