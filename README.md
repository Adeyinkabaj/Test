<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Web Network Monitor</title>
    <!-- Load Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Configure Tailwind for Inter font and custom colors -->
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                    },
                    colors: {
                        'primary': '#4f46e5',
                        'secondary': '#6366f1',
                        'success': '#10b981',
                        'danger': '#ef4444',
                        'card': '#f9fafb',
                    }
                }
            }
        }
    </script>
    <style>
        /* Custom scrollbar for the log panel */
        #log-output {
            max-height: 500px;
            overflow-y: auto;
        }
        /* Style for the real-time status table */
        .status-table th {
            background-color: #6366f1;
            color: white;
            padding: 12px;
        }
    </style>
</head>
<body class="bg-gray-100 font-sans min-h-screen p-4 md:p-8">

    <div class="max-w-4xl mx-auto">
        <!-- Header -->
        <header class="bg-primary text-white p-6 rounded-xl shadow-lg mb-8">
            <h1 class="text-3xl font-bold">🌐 Web Network Monitor</h1>
            <p class="text-indigo-200 mt-1">Real-time status check for Hosts/IPs without the command line.</p>
        </header>

        <!-- Input and Control Panel -->
        <div class="bg-white p-6 rounded-xl shadow-md mb-8">
            <h2 class="text-xl font-semibold text-primary mb-4">Configuration</h2>
            <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
                <!-- Host List Input -->
                <div class="md:col-span-2">
                    <label for="host-list" class="block text-sm font-medium text-gray-700 mb-2">
                        Hosts/IPs to Monitor (One per line)
                    </label>
                    <textarea id="host-list" rows="5" class="w-full p-3 border border-gray-300 rounded-lg focus:ring-secondary focus:border-secondary transition duration-150" placeholder="e.g., 192.168.1.1&#10;google.com&#10;server-app.local">129.18.100.19
192.168.1.1
google.com
github.com
local-server
</textarea>
                </div>
                <!-- Controls -->
                <div class="md:col-span-1 space-y-4 pt-8 md:pt-0 flex flex-col justify-end">
                    <button id="start-monitor" class="w-full bg-success hover:bg-green-600 text-white font-bold py-3 px-4 rounded-lg shadow-md transition duration-150 ease-in-out disabled:opacity-50" onclick="startMonitoring()">
                        ▶️ Start Monitoring
                    </button>
                    <button id="stop-monitor" class="w-full bg-danger hover:bg-red-600 text-white font-bold py-3 px-4 rounded-lg shadow-md transition duration-150 ease-in-out disabled:opacity-50" disabled onclick="stopMonitoring()">
                        🛑 Stop Monitoring
                    </button>
                    <label class="flex items-center space-x-2">
                        <input type="number" id="interval" min="1" value="5" class="w-20 p-2 border border-gray-300 rounded-lg text-center" />
                        <span class="text-gray-600 text-sm">Refresh Interval (seconds)</span>
                    </label>
                </div>
            </div>
        </div>

        <!-- Real-time Status Table -->
        <div class="bg-white p-6 rounded-xl shadow-md mb-8">
            <h2 class="text-xl font-semibold text-primary mb-4">Live Status Board</h2>
            <div class="overflow-x-auto">
                <table class="min-w-full divide-y divide-gray-200 rounded-lg overflow-hidden status-table">
                    <thead>
                        <tr>
                            <th class="px-6 py-3 text-left text-xs font-medium uppercase tracking-wider">Host/IP</th>
                            <th class="px-6 py-3 text-center text-xs font-medium uppercase tracking-wider">Status</th>
                            <th class="px-6 py-3 text-center text-xs font-medium uppercase tracking-wider">Last Check</th>
                        </tr>
                    </thead>
                    <tbody id="status-body" class="bg-white divide-y divide-gray-200">
                        <!-- Status rows will be inserted here -->
                    </tbody>
                </table>
            </div>
        </div>

        <!-- Log Output -->
        <div class="bg-card p-4 rounded-xl shadow-inner border border-gray-300">
            <h2 class="text-lg font-semibold text-gray-800 mb-2">Monitor Log</h2>
            <div id="log-output" class="text-sm font-mono p-2 bg-gray-50 rounded-lg text-gray-700">
                Awaiting monitor start...
            </div>
        </div>

    </div> <!-- End max-w-4xl container -->

    <script>
        let monitoringInterval = null;
        let logElement = document.getElementById('log-output');
        const statusBody = document.getElementById('status-body');

        // Utility to safely clear and log messages
        function log(message, isClear = false) {
            const timestamp = new Date().toLocaleTimeString();
            const logEntry = document.createElement('div');
            logEntry.innerHTML = `<span class="text-gray-500">[${timestamp}]</span> ${message}`;

            if (isClear) {
                logElement.innerHTML = '';
            }
            logElement.appendChild(logEntry);
            // Auto-scroll to the bottom
            logElement.scrollTop = logElement.scrollHeight;
        }

        // The core pinging function using an image request (a common, simple browser method)
        function checkHost(host) {
            return new Promise(resolve => {
                // A common technique: Try to load a tiny resource (like an image) from the host on a common port (80 or 443).
                // Note: The browser's Same-Origin Policy will often block the loading, but the failure/success of the request
                // still provides a basic connectivity clue. True ICMP ping is not possible from client-side JS.
                const timeout = 3000; // 3 seconds timeout

                // We try an HTTPS connection as it's the standard.
                const img = new Image();
                let timer = null;

                const cleanup = (status, time, errorMsg = '') => {
                    clearTimeout(timer);
                    img.onload = img.onerror = null; // Prevent memory leaks/duplicate calls
                    resolve({ host, status, time, errorMsg });
                };

                const start = performance.now();

                // Set a timeout to resolve 'Offline' if the image load takes too long
                timer = setTimeout(() => {
                    cleanup('Offline', performance.now() - start, 'Timed out');
                }, timeout);

                img.onload = () => {
                    cleanup('Online', performance.now() - start);
                };

                img.onerror = () => {
                    // onerror might fire for connection refused, DNS fail, or CORS issues.
                    // We treat a quick failure as 'Offline' in this context.
                    cleanup('Offline', performance.now() - start, 'Connection error/Refused');
                };

                // Use the host/IP directly. This relies on the browser trying to resolve and connect.
                // We append a random query parameter to prevent caching.
                img.src = `https://${host}/favicon.ico?_r=${Math.random()}`; 
                
                // Fallback attempt for HTTP (less secure, but sometimes necessary for local devices)
                // If the HTTPS check fails, we could potentially retry with HTTP, but for simplicity, 
                // we rely on the primary attempt.
            });
        }

        // Function to process all hosts
        async function runCheck() {
            const hostListText = document.getElementById('host-list').value;
            const hosts = hostListText.split('\n').map(h => h.trim()).filter(h => h.length > 0);
            
            if (hosts.length === 0) {
                log("No hosts defined. Stopping check.", true);
                stopMonitoring();
                return;
            }

            log(`Starting check for ${hosts.length} hosts...`);

            const promises = hosts.map(host => checkHost(host));
            const results = await Promise.allSettled(promises);

            const now = new Date().toLocaleTimeString();

            results.forEach(result => {
                if (result.status === 'fulfilled') {
                    const { host, status, time, errorMsg } = result.value;
                    updateStatusTable(host, status, time, now, errorMsg);
                    log(`Checked ${host}: <span class="font-bold text-${status === 'Online' ? 'success' : 'danger'}-700">${status}</span> in ${time.toFixed(2)}ms`);
                } else {
                    // Handle promise rejection (shouldn't happen with the current checkHost implementation)
                    updateStatusTable(host, 'Error', 0, now, 'Promise Rejected');
                    log(`Checked ${host}: <span class="font-bold text-danger-700">Error</span> (Promise Rejected)`);
                }
            });
            log(`Check cycle completed at ${now}.`, false);
        }

        function updateStatusTable(host, status, time, timestamp, errorMsg) {
            let row = document.getElementById(`row-${host}`);
            const isOnline = status === 'Online';
            const statusColor = isOnline ? 'bg-success text-white' : 'bg-danger text-white';
            const icon = isOnline ? '✅' : '❌';
            const detailText = isOnline ? `${time.toFixed(0)} ms` : errorMsg;
            const rowClass = isOnline ? 'hover:bg-green-50' : 'hover:bg-red-50';

            if (!row) {
                row = statusBody.insertRow();
                row.id = `row-${host}`;
                row.className = `transition duration-150 ease-in-out ${rowClass}`;
                row.innerHTML = `
                    <td class="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">${host}</td>
                    <td id="status-cell-${host}" class="px-6 py-4 whitespace-nowrap text-center">
                        <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full ${statusColor}">${icon} ${status}</span>
                        <div class="text-xs text-gray-500 mt-1">${detailText}</div>
                    </td>
                    <td id="time-cell-${host}" class="px-6 py-4 whitespace-nowrap text-sm text-gray-500 text-center">${timestamp}</td>
                `;
                // If a new row is added, we don't clear the statusBody inside runCheck anymore, 
                // so we need to ensure the new row is sorted (or just let the user see it append).
                // Given the current structure, let's keep the row append simple.
            } else {
                // Update existing row
                const statusCell = document.getElementById(`status-cell-${host}`);
                const timeCell = document.getElementById(`time-cell-${host}`);

                statusCell.innerHTML = `
                    <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full ${statusColor}">${icon} ${status}</span>
                    <div class="text-xs text-gray-500 mt-1">${detailText}</div>
                `;
                timeCell.textContent = timestamp;
                row.className = `transition duration-150 ease-in-out ${rowClass}`; // Update hover effect
            }
        }

        // Start/Stop Logic
        function startMonitoring() {
            const intervalSeconds = parseInt(document.getElementById('interval').value);
            if (isNaN(intervalSeconds) || intervalSeconds < 1) {
                log('Invalid interval. Please enter a number greater than 0.');
                return;
            }

            document.getElementById('start-monitor').disabled = true;
            document.getElementById('stop-monitor').disabled = false;
            document.getElementById('host-list').disabled = true;

            log(`Monitoring started. Refreshing every ${intervalSeconds} seconds.`, true);
            
            // Run immediately and then set interval
            runCheck();
            monitoringInterval = setInterval(runCheck, intervalSeconds * 1000);
        }

        function stopMonitoring() {
            if (monitoringInterval) {
                clearInterval(monitoringInterval);
                monitoringInterval = null;
                log('Monitoring stopped.');
            }

            document.getElementById('start-monitor').disabled = false;
            document.getElementById('stop-monitor').disabled = true;
            document.getElementById('host-list').disabled = false;
        }
    </script>
</body>
</html>

