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
            <p class="text-xs text-indigo-300 mt-2">Note: This monitor checks for **web connectivity (HTTP/S response)**, not a pure ICMP ping, due to browser security restrictions.</p>
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
                    <textarea id="host-list" rows="5" class="w-full p-3 border border-gray-300 rounded-lg focus:ring-secondary focus:border-secondary transition duration-150" placeholder="e.g., mainstreetcapltd.com&#10;google.com&#10;server-app.local">192.168.100.30
google.com
mainstreetcapltd.com
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

        // The core pinging function, now using a promise chain to try HTTPS then HTTP
        function checkHost(host) {
            // Function to run the actual check using an Image object
            const runImageCheck = (scheme, h) => {
                return new Promise(resolve => {
                    const timeout = 3000; // 3 seconds timeout
                    const img = new Image();
                    let timer = null;

                    const cleanup = (status, time, errorMsg = '') => {
                        clearTimeout(timer);
                        img.onload = img.onerror = null; 
                        resolve({ status, time, errorMsg, scheme });
                    };

                    const start = performance.now();

                    timer = setTimeout(() => {
                        cleanup('Timeout', performance.now() - start, 'Timed out');
                    }, timeout);

                    img.onload = () => {
                        cleanup('Online', performance.now() - start);
                    };

                    img.onerror = () => {
                        cleanup('Offline', performance.now() - start, 'Connection error/Refused');
                    };

                    // Try to load a tiny resource to check for connectivity
                    img.src = `${scheme}://${h}/favicon.ico?_r=${Math.random()}`; 
                });
            };

            // Main execution flow: try HTTPS, then try HTTP if HTTPS fails
            const hostOnly = host.includes(':') ? host.split(':')[0] : host;

            return runImageCheck('https', hostOnly)
                .then(httpsResult => {
                    if (httpsResult.status === 'Online') {
                        return { host: host, ...httpsResult };
                    }
                    // HTTPS failed, try HTTP
                    return runImageCheck('http', hostOnly)
                        .then(httpResult => {
                            // Return whichever result is best
                            const status = httpResult.status === 'Online' ? 'Online' : 'Offline';
                            const errorMsg = httpResult.status === 'Online' ? '' : `Failed HTTPS & HTTP: ${httpResult.errorMsg}`;
                            return { host: host, status, time: httpResult.time, errorMsg, scheme: httpResult.scheme };
                        });
                })
                .catch(err => {
                    return { host: host, status: 'Error', time: 0, errorMsg: `Fatal JS Error: ${err.message}`, scheme: 'N/A' };
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
            const results = await Promise.all(promises);

            const now = new Date().toLocaleTimeString();

            results.forEach(result => {
                const { host, status, time, errorMsg, scheme } = result;
                updateStatusTable(host, status, time, now, errorMsg, scheme);
                
                const detail = status === 'Online' ? `via ${scheme.toUpperCase()} in ${time.toFixed(2)}ms` : errorMsg;
                log(`Checked ${host}: <span class="font-bold text-${status === 'Online' ? 'success' : 'danger'}-700">${status}</span> (${detail})`);
            });
            log(`Check cycle completed at ${now}.`, false);
        }

        function updateStatusTable(host, status, time, timestamp, errorMsg, scheme) {
            let row = document.getElementById(`row-${host}`);
            const isOnline = status === 'Online';
            const statusColor = isOnline ? 'bg-success text-white' : 'bg-danger text-white';
            const icon = isOnline ? '✅' : '❌';
            const detailText = isOnline ? `${time.toFixed(0)} ms (${scheme.toUpperCase()})` : errorMsg;
            const rowClass = isOnline ? 'hover:bg-green-50' : 'hover:bg-red-50';

            if (!row) {
                row = statusBody.insertRow();
                row.id = `row-${host}`;
                row.className = `transition duration-150 ease-in-out ${rowClass}`;
                // Insert placeholders for cells and update content below
            } else {
                row.className = `transition duration-150 ease-in-out ${rowClass}`;
            }

            row.innerHTML = `
                <td class="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">${host}</td>
                <td id="status-cell-${host}" class="px-6 py-4 whitespace-nowrap text-center">
                    <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full ${statusColor}">${icon} ${status}</span>
                    <div class="text-xs text-gray-500 mt-1">${detailText}</div>
                </td>
                <td id="time-cell-${host}" class="px-6 py-4 whitespace-nowrap text-sm text-gray-500 text-center">${timestamp}</td>
            `;
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
