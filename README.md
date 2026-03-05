<!DOCTYPE html>
<html>
<head>
    <title>QR Attendance Scanner</title>
    <script src="https://unpkg.com/html5-qrcode"></script>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        .glass-card {
            background: rgba(255, 255, 255, 0.1);
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255, 255, 255, 0.2);
            border-radius: 15px;
        }
        #reader { border-radius: 20px; overflow: hidden; border: none !important; }
    </style>
</head>
<body class="bg-dark text-white">
    <div class="container-fluid py-5">
        <div class="row justify-content-center">
            <div class="col-md-6 text-center">
                <h2 class="mb-4">Attendance Scanner</h2>
                <div id="reader" class="mx-auto shadow-lg" style="max-width: 500px;"></div>
                <div id="status" class="mt-4 p-3 rounded bg-secondary">Ready to Scan...</div>
            </div>

            <div class="col-md-4">
                <div id="student-card" class="glass-card p-4 mt-5 d-none">
                    <h4 class="text-info border-bottom pb-2">Student Information</h4>
                    <div class="mt-3">
                        <p><strong>Name:</strong> <span id="disp-name">---</span></p>
                        <p><strong>LRN:</strong> <span id="disp-lrn">---</span></p>
                        <p><strong>Level:</strong> <span id="disp-level">---</span></p>
                        <p><strong>Section:</strong> <span id="disp-section">---</span></p>
                        <hr>
                        <p><strong>Status:</strong> <span id="disp-status" class="badge bg-success">---</span></p>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script>
        const FB_URL = "https://attendance-monitoring-84aeb-default-rtdb.firebaseio.com/";

        async function onScanSuccess(decodedText) {
            const statusDiv = document.getElementById('status');
            const infoCard = document.getElementById('student-card');
            
            statusDiv.className = "mt-4 p-3 rounded bg-warning text-dark";
            statusDiv.innerText = "Verifying LRN: " + decodedText + "...";

            try {
                // 1. CHECK IF STUDENT IS REGISTERED
                // Assuming your students are stored under /students/[lrn]
                const response = await fetch(`${FB_URL}students/${decodedText}.json`);
                const studentData = await response.json();

                if (!studentData) {
                    // NOT REGISTERED
                    statusDiv.className = "mt-4 p-3 rounded bg-danger text-white";
                    statusDiv.innerText = "❌ UNREGISTERED QR CODE";
                    infoCard.classList.add('d-none');
                    return;
                }

                // 2. IF REGISTERED, SHOW INFO ON THE SIDE
                document.getElementById('disp-name').innerText = studentData.name || "N/A";
                document.getElementById('disp-lrn').innerText = decodedText;
                document.getElementById('disp-level').innerText = studentData.level || "N/A";
                document.getElementById('disp-section').innerText = studentData.section || "N/A";
                
                infoCard.classList.remove('d-none');

                // 3. CALCULATE ATTENDANCE STATUS
                const now = new Date();
                const dateStr = now.toISOString().split('T')[0];
                const timeStr = now.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' });
                
                // Logic: Late after 6:30 AM (06:31 onwards)
                const isLate = (now.getHours() > 6 || (now.getHours() === 6 && now.getMinutes() >= 31));
                const attendanceStatus = isLate ? "Late" : "Present";
                
                document.getElementById('disp-status').innerText = attendanceStatus;
                document.getElementById('disp-status').className = isLate ? "badge bg-danger" : "badge bg-success";

                // 4. SAVE TO ATTENDANCE LOG
                await fetch(`${FB_URL}attendance/${dateStr}/${decodedText}.json`, {
                    method: 'PUT',
                    body: JSON.stringify({
                        name: studentData.name,
                        lrn: decodedText,
                        time: timeStr,
                        date: dateStr,
                        status: attendanceStatus
                    })
                });

                statusDiv.className = "mt-4 p-3 rounded bg-success text-white";
                statusDiv.innerText = `✅ Attendance Recorded for ${studentData.name}`;

            } catch (err) {
                console.error(err);
                statusDiv.className = "mt-4 p-3 rounded bg-danger text-white";
                statusDiv.innerText = "System Error: " + err.message;
            }
        }

        let scanner = new Html5QrcodeScanner("reader", { 
            fps: 10, 
            qrbox: { width: 250, height: 250 },
            rememberLastUsedCamera: true
        });
        scanner.render(onScanSuccess);
    </script>
</body>
</html>
