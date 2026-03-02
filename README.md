<!DOCTYPE html>
<html>
<head>
    <title>QR Attendance Scanner</title>
    <script src="https://unpkg.com/html5-qrcode"></script>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body class="bg-dark text-white text-center">
    <div class="container py-5">
        <h2 class="mb-4">Attendance Scanner</h2>
        <div id="reader" class="mx-auto shadow-lg" style="max-width: 500px; border-radius: 20px; overflow: hidden;"></div>
        <div id="status" class="mt-4 p-3 rounded bg-secondary">Ready to Scan...</div>
    </div>

    <script>
        // MUST MATCH YOUR FIREBASE URL
        const FB_URL = "https://attendance-monitoring-84aeb-default-rtdb.firebaseio.com/";

        function onScanSuccess(decodedText) {
            const statusDiv = document.getElementById('status');
            statusDiv.className = "mt-4 p-3 rounded bg-primary text-white";
            statusDiv.innerText = "Saving LRN: " + decodedText + "...";
            
            const now = new Date();
            const dateStr = now.toISOString().split('T')[0]; // e.g., 2026-03-03
            const timeStr = now.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' });
            const status = (now.getHours() >= 8) ? "Late" : "On Time";

            // SAVE DIRECTLY TO FIREBASE
            fetch(`${FB_URL}attendance/${dateStr}/${decodedText}.json`, {
                method: 'PUT',
                body: JSON.stringify({
                    lrn: decodedText,
                    time: timeStr,
                    date: dateStr,
                    status: status
                })
            })
            .then(() => {
                statusDiv.className = "mt-4 p-3 rounded bg-success text-white";
                statusDiv.innerText = `✅ Recorded: ${decodedText} at ${timeStr}`;
                // Reset after 3 seconds for next scan
                setTimeout(() => { 
                    statusDiv.className = "mt-4 p-3 rounded bg-secondary";
                    statusDiv.innerText = "Ready to Scan"; 
                }, 3000);
            })
            .catch(err => {
                statusDiv.className = "mt-4 p-3 rounded bg-danger text-white";
                statusDiv.innerText = "Error: " + err;
            });
        }

        let scanner = new Html5QrcodeScanner("reader", { fps: 15, qrbox: 250 });
        scanner.render(onScanSuccess);
    </script>
</body>
</html>
