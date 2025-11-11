<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Ultimate Time Manager: Clock, Timer, Stopwatch & Pomodoro</title>
    <!-- SEO METADATA ADDED HERE -->
    <meta name="description" content="An all-in-one web application featuring a real-time clock, high-precision stopwatch, countdown timer, set alarms, and a customizable Pomodoro technique tracker with persistence via Firestore.">
    <meta name="keywords" content="time manager, clock, timer, stopwatch, pomodoro, productivity, free app, time tracking">
    <meta name="author" content="Gemini AI">
    <!-- END SEO METADATA -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tone/14.8.49/Tone.min.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            transition: background-color 0.3s ease;
        }
        .content-section {
            display: none;
            animation: fadeIn 0.4s ease-in-out;
        }
        .content-section.active {
            display: block;
        }
        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(10px); }
            to { opacity: 1; transform: translateY(0); }
        }
        .tab-button.active {
            border-bottom: 3px solid #6366f1; /* Indigo-500 */
            color: #ffffff;
            font-weight: 600;
        }
        .display-time {
            font-variant-numeric: tabular-nums;
        }
    </style>
</head>
<body class="bg-gray-900 text-white min-h-screen p-4 md:p-8 flex flex-col items-center">

    <!-- Firebase SDK Imports (required for persistent data storage) -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, getDoc, onSnapshot, collection, query, where, updateDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        setLogLevel('debug'); // Enable debug logging

        // --- Global Variables (Mandatory for Canvas Environment) ---
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        
        let db;
        let auth;
        let userId = null;
        
        // Expose to global scope for use in the main script
        window.firebaseDependencies = {
            initializeApp, getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged,
            getFirestore, doc, setDoc, getDoc, onSnapshot, collection, query, where, updateDoc,
            appId, firebaseConfig, initialAuthToken
        };
    </script>
    
    <div id="app-root" class="w-full max-w-lg bg-gray-800 rounded-xl shadow-2xl p-6 md:p-8">

        <!-- Header and Real-Time Clock -->
        <h1 class="text-4xl font-extrabold text-center mb-2 text-indigo-400">Time Hub</h1>
        <p id="current-time-display" class="text-6xl md:text-7xl font-mono text-center mb-6 display-time">00:00:00</p>
        <p id="user-info" class="text-xs text-center text-gray-400 mb-6">[Loading user...]</p>

        <!-- Navigation Tabs -->
        <nav class="flex justify-between border-b border-gray-700 mb-8">
            <button class="tab-button py-3 px-2 flex-1 text-center text-gray-400 hover:text-white transition duration-200 rounded-t-lg" data-target="stopwatch">⏱️ Stopwatch</button>
            <button class="tab-button py-3 px-2 flex-1 text-center text-gray-400 hover:text-white transition duration-200 rounded-t-lg" data-target="timer">⏳ Timer</button>
            <button class="tab-button py-3 px-2 flex-1 text-center text-gray-400 hover:text-white transition duration-200 rounded-t-lg" data-target="alarm">🔔 Alarm</button>
            <button class="tab-button py-3 px-2 flex-1 text-center text-gray-400 hover:text-white transition duration-200 rounded-t-lg" data-target="pomodoro">🍅 Pomodoro</button>
        </nav>

        <!-- Status/Notification Message -->
        <div id="status-message" class="bg-indigo-900 bg-opacity-30 border border-indigo-700 text-indigo-200 p-3 rounded-lg mb-6 hidden"></div>

        <!-- 1. STOPWATCH SECTION -->
        <div id="stopwatch" class="content-section">
            <h2 id="stopwatch-display" class="text-5xl font-mono text-center mb-8 display-time">00:00:00.00</h2>
            <div class="flex justify-center space-x-4">
                <button id="stopwatch-start-pause" class="btn bg-indigo-600 hover:bg-indigo-700">Start</button>
                <button id="stopwatch-reset" class="btn bg-gray-600 hover:bg-gray-700">Reset</button>
                <button id="stopwatch-lap" class="btn bg-gray-600 hover:bg-gray-700 hidden">Lap</button>
            </div>
            <div class="mt-8">
                <h3 class="text-xl font-semibold mb-2 border-b border-gray-700 pb-1">Laps</h3>
                <ul id="laps-list" class="space-y-1 text-gray-400 font-mono text-sm">
                    <li class="text-center text-gray-500" id="no-laps">No laps recorded yet.</li>
                </ul>
            </div>
        </div>

        <!-- 2. TIMER SECTION -->
        <div id="timer" class="content-section">
            <h2 id="timer-display" class="text-5xl font-mono text-center mb-6 display-time">00:00:00</h2>
            <div class="flex justify-center space-x-4 mb-6">
                <input type="number" id="timer-hours" placeholder="H" min="0" max="99" value="0" class="input-time">
                <input type="number" id="timer-minutes" placeholder="M" min="0" max="59" value="1" class="input-time">
                <input type="number" id="timer-seconds" placeholder="S" min="0" max="59" value="0" class="input-time">
            </div>
            <div class="flex justify-center space-x-4">
                <button id="timer-start-pause" class="btn bg-indigo-600 hover:bg-indigo-700">Start Timer</button>
                <button id="timer-reset" class="btn bg-gray-600 hover:bg-gray-700" disabled>Reset</button>
            </div>
        </div>

        <!-- 3. ALARM SECTION -->
        <div id="alarm" class="content-section">
            <h2 id="alarm-status" class="text-3xl font-semibold text-center mb-6 text-green-400">Alarm Off</h2>
            <div class="flex justify-center space-x-4 mb-6">
                <input type="time" id="alarm-time-input" class="p-3 text-2xl rounded-lg bg-gray-700 text-white border border-gray-600 focus:ring-indigo-500 focus:border-indigo-500 appearance-none">
            </div>
            <div class="flex justify-center space-x-4">
                <button id="alarm-set" class="btn bg-indigo-600 hover:bg-indigo-700">Set Alarm</button>
                <button id="alarm-cancel" class="btn bg-red-600 hover:bg-red-700" disabled>Cancel Alarm</button>
            </div>
            <div id="alarm-list-container" class="mt-8">
                <h3 class="text-xl font-semibold mb-2 border-b border-gray-700 pb-1">Active Alarm</h3>
                <p id="active-alarm-display" class="text-center text-gray-400">No alarm set.</p>
            </div>
        </div>

        <!-- 4. POMODORO SECTION -->
        <div id="pomodoro" class="content-section">
            <div id="pomodoro-settings" class="mb-6 space-y-3 bg-gray-700 p-4 rounded-xl">
                <h3 class="text-lg font-bold text-indigo-400">Settings</h3>
                <div class="grid grid-cols-2 gap-4 text-sm">
                    <label>Work (min): <input type="number" id="pomodoro-work" value="25" min="1" class="input-time text-base"></label>
                    <label>Short Break (min): <input type="number" id="pomodoro-short" value="5" min="1" class="input-time text-base"></label>
                    <label>Long Break (min): <input type="number" id="pomodoro-long" value="15" min="1" class="input-time text-base"></label>
                    <label>Cycles: <input type="number" id="pomodoro-cycles" value="4" min="1" class="input-time text-base"></label>
                </div>
                <div class="flex justify-end space-x-2 pt-2">
                    <button id="pomodoro-save" class="btn-sm bg-indigo-700 hover:bg-indigo-800">Save</button>
                    <button id="pomodoro-load" class="btn-sm bg-gray-600 hover:bg-gray-700">Load Defaults</button>
                </div>
            </div>

            <h2 id="pomodoro-phase" class="text-2xl font-semibold text-center mb-4 text-pink-400">Ready to Work</h2>
            <h3 id="pomodoro-count" class="text-md font-medium text-center mb-6 text-gray-400">Cycle: 0/4</h3>

            <h2 id="pomodoro-display" class="text-6xl font-mono text-center mb-8 display-time">25:00</h2>

            <div class="flex justify-center space-x-4">
                <button id="pomodoro-start-pause" class="btn bg-indigo-600 hover:bg-indigo-700">Start</button>
                <button id="pomodoro-stop" class="btn bg-red-600 hover:bg-red-700" disabled>Stop</button>
                <button id="pomodoro-skip" class="btn bg-gray-600 hover:bg-gray-700" disabled>Skip</button>
            </div>
        </div>

        <!-- Error/Success Modal (Custom implementation instead of alert()) -->
        <div id="custom-modal" class="fixed inset-0 bg-black bg-opacity-75 items-center justify-center hidden z-50">
            <div class="bg-gray-800 p-6 rounded-lg shadow-xl max-w-sm w-full border-t-4 border-indigo-500">
                <h3 id="modal-title" class="text-xl font-bold mb-3 text-indigo-400">Notification</h3>
                <p id="modal-message" class="text-gray-200 mb-4">Message content.</p>
                <button id="modal-close-btn" class="w-full btn bg-indigo-600 hover:bg-indigo-700">Close</button>
            </div>
        </div>

    </div>

    <script>
        // Tailwind Button Styles
        const style = document.createElement('style');
        style.innerHTML = `
            .btn {
                padding: 0.75rem 1.5rem;
                border-radius: 0.5rem;
                font-weight: 600;
                transition: background-color 0.2s, transform 0.1s;
                box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            }
            .btn-sm {
                padding: 0.5rem 1rem;
                border-radius: 0.5rem;
                font-size: 0.875rem;
                font-weight: 500;
            }
            .input-time {
                width: 100%;
                padding: 0.5rem;
                border-radius: 0.5rem;
                text-align: center;
                background-color: #374151; /* gray-700 */
                color: #ffffff;
                border: 1px solid #4b5563; /* gray-600 */
                appearance: textfield; /* Hide number arrows */
            }
            .input-time::-webkit-inner-spin-button, 
            .input-time::-webkit-outer-spin-button {
                -webkit-appearance: none;
                margin: 0;
            }
        `;
        document.head.appendChild(style);

        // =========================================================================
        // TONE.JS SOUND UTILITY
        // =========================================================================
        let synth = null;
        const initAudio = () => {
            if (Tone.context.state !== 'running') {
                Tone.start();
            }
            if (!synth) {
                synth = new Tone.Synth().toDestination();
                // Ensure volume is reasonable
                synth.volume.value = -10; 
            }
        };

        const playAlertSound = (frequency = 440, duration = '8n') => {
            initAudio();
            synth.triggerAttackRelease(frequency, duration);
        };

        // =========================================================================
        // UTILITY FUNCTIONS
        // =========================================================================

        const $ = id => document.getElementById(id);
        const formatTime = (totalSeconds) => {
            const h = Math.floor(totalSeconds / 3600);
            const m = Math.floor((totalSeconds % 3600) / 60);
            const s = Math.floor(totalSeconds % 60);
            return [h, m, s].map(v => v.toString().padStart(2, '0')).join(':');
        };

        const showCustomModal = (title, message) => {
            $('modal-title').textContent = title;
            $('modal-message').textContent = message;
            $('custom-modal').classList.remove('hidden');
            $('custom-modal').classList.add('flex');
            // Stop any ongoing sound (like an alarm) when the modal is closed
            $('modal-close-btn').onclick = () => {
                $('custom-modal').classList.add('hidden');
                $('custom-modal').classList.remove('flex');
            };
        };

        const updateStatusMessage = (message, type = 'info') => {
            const statusEl = $('status-message');
            statusEl.textContent = message;
            statusEl.className = 'bg-opacity-30 border p-3 rounded-lg mb-6';
            
            if (type === 'success') {
                statusEl.classList.add('bg-green-900', 'border-green-700', 'text-green-200');
            } else if (type === 'error') {
                statusEl.classList.add('bg-red-900', 'border-red-700', 'text-red-200');
            } else { // info/default
                statusEl.classList.add('bg-indigo-900', 'border-indigo-700', 'text-indigo-200');
            }
            statusEl.classList.remove('hidden');
            setTimeout(() => statusEl.classList.add('hidden'), 5000);
        };

        // =========================================================================
        // FIREBASE & AUTH SETUP (Mandatory for persistent web apps)
        // =========================================================================

        let db, auth, userId, isAuthReady = false;
        
        window.onload = async () => {
            const deps = window.firebaseDependencies;
            const timeDisplay = $('current-time-display');
            const userInfoEl = $('user-info');
            
            if (!deps || !deps.firebaseConfig) {
                userInfoEl.textContent = "[Firestore Disabled: Missing Config]";
                isAuthReady = true;
            } else {
                try {
                    const app = deps.initializeApp(deps.firebaseConfig);
                    db = deps.getFirestore(app);
                    auth = deps.getAuth(app);
                    
                    if (deps.initialAuthToken) {
                        await deps.signInWithCustomToken(auth, deps.initialAuthToken);
                    } else {
                        await deps.signInAnonymously(auth);
                    }
                } catch (error) {
                    console.error("Firebase Initialization Error:", error);
                    userInfoEl.textContent = `[Auth Error: ${error.code}]`;
                    isAuthReady = true;
                }
            }

            deps.onAuthStateChanged(auth, (user) => {
                if (user) {
                    userId = user.uid;
                    userInfoEl.textContent = `User ID: ${userId}`;
                } else {
                    userId = crypto.randomUUID();
                    userInfoEl.textContent = `User ID: ${userId} (Anon)`;
                }
                isAuthReady = true;
                // Start persistent listeners and load settings once authenticated
                loadAlarmState();
                loadPomodoroSettings();
            });

            // Start Real-Time Clock
            setInterval(() => {
                const now = new Date();
                timeDisplay.textContent = now.toLocaleTimeString('en-US', { hour12: false, hour: '2-digit', minute: '2-digit', second: '2-digit' });
                // Check alarm every second
                checkAlarm();
            }, 1000);

            // Initialize all other functions
            initStopwatch();
            initTimer();
            initAlarm();
            initPomodoro();
            
            // Set initial tab
            switchTab('stopwatch');
        };

        // =========================================================================
        // TAB MANAGEMENT
        // =========================================================================

        const contentSections = document.querySelectorAll('.content-section');
        const tabButtons = document.querySelectorAll('.tab-button');

        const switchTab = (targetId) => {
            // Hide all content sections and remove 'active' from all buttons
            contentSections.forEach(section => section.classList.remove('active'));
            tabButtons.forEach(button => button.classList.remove('active'));

            // Show the target section and set the target button as active
            const targetSection = $(targetId);
            const targetButton = document.querySelector(`.tab-button[data-target="${targetId}"]`);

            if (targetSection && targetButton) {
                targetSection.classList.add('active');
                targetButton.classList.add('active');
            }
        };

        tabButtons.forEach(button => {
            button.addEventListener('click', () => {
                // Ensure audio is initialized on first user interaction
                initAudio();
                switchTab(button.dataset.target);
            });
        });

        // =========================================================================
        // 1. STOPWATCH LOGIC
        // =========================================================================

        let stopwatchInterval = null;
        let stopwatchTime = 0;
        let stopwatchRunning = false;
        let lapCounter = 0;

        const initStopwatch = () => {
            $('stopwatch-start-pause').addEventListener('click', toggleStopwatch);
            $('stopwatch-reset').addEventListener('click', resetStopwatch);
            $('stopwatch-lap').addEventListener('click', recordLap);
        };

        const updateStopwatchDisplay = () => {
            const cs = Math.floor((stopwatchTime % 1000) / 10); // Centiseconds
            const totalSeconds = Math.floor(stopwatchTime / 1000);
            const formattedTime = formatTime(totalSeconds);
            $('stopwatch-display').textContent = `${formattedTime}.${cs.toString().padStart(2, '0')}`;
        };

        const toggleStopwatch = () => {
            if (stopwatchRunning) {
                clearInterval(stopwatchInterval);
                $('stopwatch-start-pause').textContent = 'Start';
                $('stopwatch-reset').disabled = false;
                $('stopwatch-lap').classList.add('hidden');
            } else {
                $('stopwatch-start-pause').textContent = 'Pause';
                $('stopwatch-lap').classList.remove('hidden');
                $('stopwatch-reset').disabled = true;

                const startTime = Date.now() - stopwatchTime;
                stopwatchInterval = setInterval(() => {
                    stopwatchTime = Date.now() - startTime;
                    updateStopwatchDisplay();
                }, 10); // Update every 10ms for centisecond accuracy
            }
            stopwatchRunning = !stopwatchRunning;
        };

        const resetStopwatch = () => {
            if (stopwatchRunning) toggleStopwatch();
            stopwatchTime = 0;
            lapCounter = 0;
            updateStopwatchDisplay();
            $('laps-list').innerHTML = '<li class="text-center text-gray-500" id="no-laps">No laps recorded yet.</li>';
            $('stopwatch-lap').classList.add('hidden');
            $('stopwatch-reset').disabled = false;
        };
        
        const recordLap = () => {
            if (!stopwatchRunning) return;
            lapCounter++;
            const totalSeconds = Math.floor(stopwatchTime / 1000);
            const cs = Math.floor((stopwatchTime % 1000) / 10); 
            const formattedLapTime = `${formatTime(totalSeconds)}.${cs.toString().padStart(2, '0')}`;
            
            const lapItem = document.createElement('li');
            lapItem.className = 'flex justify-between p-1 bg-gray-700 rounded-md';
            lapItem.innerHTML = `
                <span>LAP ${lapCounter}:</span>
                <span class="text-indigo-300">${formattedLapTime}</span>
            `;

            const list = $('laps-list');
            if ($('no-laps')) $('no-laps').remove();
            list.prepend(lapItem); // Add new lap to the top
        };

        // =========================================================================
        // 2. TIMER LOGIC
        // =========================================================================

        let timerInterval = null;
        let timerSecondsLeft = 0;
        let timerRunning = false;

        const initTimer = () => {
            $('timer-start-pause').addEventListener('click', toggleTimer);
            $('timer-reset').addEventListener('click', resetTimer);
            updateTimerDisplay();
        };

        const updateTimerDisplay = () => {
            $('timer-display').textContent = formatTime(timerSecondsLeft);
        };

        const loadTimerSettings = () => {
            const h = parseInt($('timer-hours').value) || 0;
            const m = parseInt($('timer-minutes').value) || 0;
            const s = parseInt($('timer-seconds').value) || 0;
            timerSecondsLeft = (h * 3600) + (m * 60) + s;
            updateTimerDisplay();
        };

        const toggleTimer = () => {
            if (timerRunning) {
                clearInterval(timerInterval);
                $('timer-start-pause').textContent = 'Resume';
                $('timer-reset').disabled = false;
            } else {
                if (timerSecondsLeft <= 0) loadTimerSettings();
                if (timerSecondsLeft <= 0) {
                    showCustomModal('Error', 'Please set a time greater than zero.');
                    return;
                }

                $('timer-start-pause').textContent = 'Pause';
                $('timer-reset').disabled = true;
                disableTimeInputs(true);
                
                timerInterval = setInterval(tickTimer, 1000);
            }
            timerRunning = !timerRunning;
        };

        const tickTimer = () => {
            if (timerSecondsLeft > 0) {
                timerSecondsLeft--;
                updateTimerDisplay();
            } else {
                clearInterval(timerInterval);
                timerRunning = false;
                $('timer-start-pause').textContent = 'Start Timer';
                $('timer-reset').disabled = false;
                disableTimeInputs(false);
                playAlertSound(660, '1n');
                showCustomModal('⏰ Timer Done!', 'The countdown has finished.');
                updateTimerDisplay(); // Ensures it shows 00:00:00
            }
        };

        const resetTimer = () => {
            if (timerRunning) toggleTimer();
            loadTimerSettings();
            $('timer-reset').disabled = true;
            disableTimeInputs(false);
            $('timer-start-pause').textContent = 'Start Timer';
        };
        
        const disableTimeInputs = (disable) => {
            $('timer-hours').disabled = disable;
            $('timer-minutes').disabled = disable;
            $('timer-seconds').disabled = disable;
        };
        
        // Listeners for input changes to pre-load the display
        ['timer-hours', 'timer-minutes', 'timer-seconds'].forEach(id => {
            $(id).addEventListener('change', loadTimerSettings);
        });
        
        // =========================================================================
        // 3. ALARM LOGIC
        // =========================================================================

        let activeAlarmTime = null; // Stored as "HH:MM" string (24h)
        let alarmCheckInterval = null; // The interval running in the background

        const ALARM_DOC_PATH = (uid) => `artifacts/${auth.currentUser.uid}/clock-settings/alarm-config`;

        const initAlarm = () => {
            $('alarm-set').addEventListener('click', setAlarm);
            $('alarm-cancel').addEventListener('click', cancelAlarm);
        };
        
        const saveAlarmState = async (alarmTime) => {
             if (!isAuthReady || !auth.currentUser) return;
             try {
                 await setDoc(doc(db, ALARM_DOC_PATH()), { time: alarmTime, active: !!alarmTime, timestamp: new Date() }, { merge: true });
                 updateStatusMessage(alarmTime ? `Alarm saved for ${alarmTime}` : 'Alarm state cleared.', 'success');
             } catch (e) {
                 console.error("Error saving alarm state:", e);
                 updateStatusMessage("Could not save alarm state. Storage unavailable.", 'error');
             }
         };

        const loadAlarmState = () => {
            if (!isAuthReady || !auth.currentUser) return;

            // Use onSnapshot for real-time updates of the alarm state
            onSnapshot(doc(db, ALARM_DOC_PATH()), (docSnapshot) => {
                if (docSnapshot.exists()) {
                    const data = docSnapshot.data();
                    if (data.active && data.time) {
                        activeAlarmTime = data.time;
                        $('alarm-time-input').value = activeAlarmTime;
                        updateAlarmUI(true);
                    } else {
                        activeAlarmTime = null;
                        updateAlarmUI(false);
                    }
                }
            }, (error) => {
                console.error("Error loading alarm state:", error);
                updateStatusMessage("Failed to load alarm settings.", 'error');
            });
        };
        
        const updateAlarmUI = (isSet) => {
            if (isSet && activeAlarmTime) {
                $('alarm-status').textContent = 'Alarm Set: ' + activeAlarmTime;
                $('alarm-status').classList.remove('text-green-400');
                $('alarm-status').classList.add('text-yellow-400');
                $('alarm-cancel').disabled = false;
                $('alarm-set').textContent = 'Update Alarm';
                $('active-alarm-display').textContent = `Waking up at ${activeAlarmTime}.`;
            } else {
                $('alarm-status').textContent = 'Alarm Off';
                $('alarm-status').classList.remove('text-yellow-400');
                $('alarm-status').classList.add('text-green-400');
                $('alarm-cancel').disabled = true;
                $('alarm-set').textContent = 'Set Alarm';
                $('active-alarm-display').textContent = 'No alarm set.';
            }
        };

        const setAlarm = () => {
            const timeInput = $('alarm-time-input').value;
            if (!timeInput) {
                showCustomModal('Error', 'Please select a valid time.');
                return;
            }
            activeAlarmTime = timeInput;
            saveAlarmState(activeAlarmTime);
            updateAlarmUI(true);
            updateStatusMessage(`Alarm set for ${activeAlarmTime}.`, 'success');
            // Ensure the main clock loop will check the alarm
        };

        const cancelAlarm = () => {
            if (activeAlarmTime) {
                activeAlarmTime = null;
                saveAlarmState(null);
                updateAlarmUI(false);
                updateStatusMessage('Alarm cancelled.', 'info');
            }
        };

        const checkAlarm = () => {
            if (!activeAlarmTime) return;

            const now = new Date();
            const currentHour = now.getHours().toString().padStart(2, '0');
            const currentMinute = now.getMinutes().toString().padStart(2, '0');
            const currentTimeStr = `${currentHour}:${currentMinute}`;

            if (currentTimeStr === activeAlarmTime) {
                const nowSeconds = now.getSeconds();
                // Trigger the alarm only once when seconds is 0
                if (nowSeconds === 0) {
                    playAlertSound(880, '4n'); // Higher frequency for alarm
                    showCustomModal('🚨 ALARM WAKE UP! 🚨', `It is now ${activeAlarmTime}!`);
                    cancelAlarm(); // Auto-cancel after trigger
                }
            }
        };


        // =========================================================================
        // 4. POMODORO LOGIC
        // =========================================================================

        const POMODORO_DOC_PATH = (uid) => `artifacts/${auth.currentUser.uid}/clock-settings/pomodoro-config`;
        const PHASES = {
            WORK: 'Work',
            SHORT_BREAK: 'Short Break',
            LONG_BREAK: 'Long Break'
        };

        let pomodoroInterval = null;
        let pomodoroState = {
            phase: PHASES.WORK,
            running: false,
            secondsLeft: 0,
            pomodorosCompleted: 0
        };

        let pomodoroConfig = {
            workDuration: 25 * 60,
            shortBreakDuration: 5 * 60,
            longBreakDuration: 15 * 60,
            cyclesUntilLongBreak: 4
        };

        const initPomodoro = () => {
            $('pomodoro-start-pause').addEventListener('click', togglePomodoro);
            $('pomodoro-stop').addEventListener('click', resetPomodoro);
            $('pomodoro-skip').addEventListener('click', skipPhase);
            $('pomodoro-save').addEventListener('click', savePomodoroSettings);
            $('pomodoro-load').addEventListener('click', loadDefaultPomodoroSettings);
            
            // Set initial display
            setPomodoroDuration(pomodoroConfig.workDuration);
            updatePomodoroUI();
        };
        
        // --- Persistence Functions ---
        const savePomodoroSettings = async () => {
            const work = parseInt($('pomodoro-work').value) * 60 || 1500;
            const short = parseInt($('pomodoro-short').value) * 60 || 300;
            const long = parseInt($('pomodoro-long').value) * 60 || 900;
            const cycles = parseInt($('pomodoro-cycles').value) || 4;
            
            if (work <= 0 || short <= 0 || long <= 0 || cycles <= 0) {
                showCustomModal('Error', 'All Pomodoro settings must be positive numbers.');
                return;
            }

            pomodoroConfig = {
                workDuration: work,
                shortBreakDuration: short,
                longBreakDuration: long,
                cyclesUntilLongBreak: cycles
            };

            if (!isAuthReady || !auth.currentUser) {
                updateStatusMessage("Settings updated locally (not saved to cloud).", 'info');
                return;
            }

            try {
                await setDoc(doc(db, POMODORO_DOC_PATH()), pomodoroConfig, { merge: true });
                updateStatusMessage('Pomodoro settings saved successfully!', 'success');
            } catch (e) {
                console.error("Error saving pomodoro settings:", e);
                updateStatusMessage("Could not save settings. Storage unavailable.", 'error');
            }
        };

        const loadPomodoroSettings = () => {
             if (!isAuthReady || !auth.currentUser) return;
             
             onSnapshot(doc(db, POMODORO_DOC_PATH()), (docSnapshot) => {
                 if (docSnapshot.exists()) {
                     const data = docSnapshot.data();
                     pomodoroConfig = { ...pomodoroConfig, ...data };
                     // Update form inputs
                     $('pomodoro-work').value = pomodoroConfig.workDuration / 60;
                     $('pomodoro-short').value = pomodoroConfig.shortBreakDuration / 60;
                     $('pomodoro-long').value = pomodoroConfig.longBreakDuration / 60;
                     $('pomodoro-cycles').value = pomodoroConfig.cyclesUntilLongBreak;
                     
                     // Reset display based on new config if not running
                     if (!pomodoroState.running) {
                         setPomodoroDuration(pomodoroConfig.workDuration);
                     }
                     updateStatusMessage('Pomodoro settings loaded from cloud.', 'info');
                 }
             }, (error) => {
                 console.error("Error loading pomodoro settings:", error);
                 updateStatusMessage("Failed to load Pomodoro settings. Using defaults.", 'error');
             });
        };
        
        const loadDefaultPomodoroSettings = () => {
            pomodoroConfig = {
                workDuration: 25 * 60,
                shortBreakDuration: 5 * 60,
                longBreakDuration: 15 * 60,
                cyclesUntilLongBreak: 4
            };
            $('pomodoro-work').value = 25;
            $('pomodoro-short').value = 5;
            $('pomodoro-long').value = 15;
            $('pomodoro-cycles').value = 4;
            
            if (!pomodoroState.running) {
                setPomodoroDuration(pomodoroConfig.workDuration);
            }
            updateStatusMessage('Default settings loaded locally.', 'info');
        };

        // --- Core Timer Functions ---
        
        const setPomodoroDuration = (durationInSeconds) => {
            pomodoroState.secondsLeft = durationInSeconds;
            $('pomodoro-display').textContent = formatTime(durationInSeconds).slice(3); // Show M:S
        };

        const updatePomodoroUI = () => {
            const isWork = pomodoroState.phase === PHASES.WORK;
            $('pomodoro-phase').textContent = pomodoroState.phase;
            $('pomodoro-phase').classList.toggle('text-pink-400', isWork);
            $('pomodoro-phase').classList.toggle('text-green-400', !isWork);
            $('pomodoro-count').textContent = `Cycle: ${pomodoroState.pomodorosCompleted}/${pomodoroConfig.cyclesUntilLongBreak}`;

            // Button states
            $('pomodoro-start-pause').textContent = pomodoroState.running ? 'Pause' : 'Start';
            $('pomodoro-stop').disabled = !pomodoroState.running;
            $('pomodoro-skip').disabled = !pomodoroState.running;
        };

        const togglePomodoro = () => {
            if (pomodoroState.running) {
                clearInterval(pomodoroInterval);
            } else {
                if (pomodoroState.secondsLeft === 0) startNextPhase(); // Start fresh if timer is at 0
                pomodoroInterval = setInterval(tickPomodoro, 1000);
            }
            pomodoroState.running = !pomodoroState.running;
            updatePomodoroUI();
        };

        const tickPomodoro = () => {
            if (pomodoroState.secondsLeft > 0) {
                pomodoroState.secondsLeft--;
                $('pomodoro-display').textContent = formatTime(pomodoroState.secondsLeft).slice(3);
            } else {
                // Phase is finished!
                clearInterval(pomodoroInterval);
                pomodoroState.running = false;
                startNextPhase();
            }
        };

        const startNextPhase = () => {
            let nextPhase;
            let nextDuration;
            let message;

            if (pomodoroState.phase === PHASES.WORK) {
                pomodoroState.pomodorosCompleted++;
                
                // Check for Long Break
                if (pomodoroState.pomodorosCompleted % pomodoroConfig.cyclesUntilLongBreak === 0) {
                    nextPhase = PHASES.LONG_BREAK;
                    nextDuration = pomodoroConfig.longBreakDuration;
                    message = 'Time for a long, well-deserved break!';
                } else {
                    nextPhase = PHASES.SHORT_BREAK;
                    nextDuration = pomodoroConfig.shortBreakDuration;
                    message = 'Short break time!';
                }
            } else { // Short or Long Break finished, next is always Work
                nextPhase = PHASES.WORK;
                nextDuration = pomodoroConfig.workDuration;
                message = 'Back to focus!';
            }

            pomodoroState.phase = nextPhase;
            setPomodoroDuration(nextDuration);
            togglePomodoro(); // Auto-start the next phase
            playAlertSound(440, '4n');
            updatePomodoroUI();
            showCustomModal(`Phase Change: ${nextPhase}`, message);
        };
        
        const resetPomodoro = () => {
            if (pomodoroState.running) clearInterval(pomodoroInterval);
            pomodoroState.running = false;
            pomodoroState.phase = PHASES.WORK;
            pomodoroState.pomodorosCompleted = 0;
            setPomodoroDuration(pomodoroConfig.workDuration);
            updatePomodoroUI();
            updateStatusMessage('Pomodoro cycle reset.', 'info');
        };
        
        const skipPhase = () => {
            if (pomodoroState.running) {
                clearInterval(pomodoroInterval);
                pomodoroState.running = false;
                startNextPhase();
                togglePomodoro(); // Start the new phase immediately
            }
        };

    </script>
</body>
</html>
