
<html>
<h1>SORASCAN</h1>
<head>
    <title>MAHS Attendance Scanner</title>
    <script src="https://unpkg.com/html5-qrcode"></script>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body {
            background-color: #0a0b1e;
            background: radial-gradient(circle at top right, #1a1b3a, #0a0b1e);
            min-height: 100vh;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        .glass-card {
            background: rgba(255, 255, 255, 0.05);
            backdrop-filter: blur(12px);
            border: 1px solid rgba(255, 255, 255, 0.1);
            border-radius: 20px;
        }
        #reader { 
            border-radius: 20px; 
            overflow: hidden; 
            border: 2px solid rgba(0, 255, 255, 0.3) !important; 
        }
        /* ID Card Styling */
        .info-label { color: #0dcaf0; font-weight: 700; font-size: 0.75rem; text-transform: uppercase; letter-spacing: 1px; }
        .info-value { color: #ffffff; font-size: 1.1rem; margin-bottom: 12px; font-weight: 500; }
        #student-photo-display {
            width: 140px; height: 140px; object-fit: cover;
            border: 4px solid #3a8dff; padding: 3px; background: #1a1b3a;
            box-shadow: 0 0 20px rgba(58, 141, 255, 0.3);
        }
        .status-msg { font-size: 1.2rem; font-weight: bold; min-height: 60px; display: flex; align-items: center; justify-content: center; }
        
        /* Scanner UI Cleanup */
        #reader__dashboard_section_csr button {
            background: #3a8dff !important;
            border: none !important;
            color: white !important;
            padding: 8px 20px !important;
            border-radius: 10px !important;
            text-transform: uppercase;
            font-size: 12px;
            font-weight: bold;
        }
    </style>
</head>
<body class="text-white">
    <div class="container py-5">
        <div class="row justify-content-center g-4">
            <div class="col-lg-5 text-center">
                <h2 class="mb-4 fw-bold" style="color: #0dcaf0; letter-spacing: 2px;">MAHS SCANNER</h2>
                <div id="reader" class="mx-auto shadow-lg" style="max-width: 500px;"></div>
                <div id="status" class="mt-4 p-3 rounded glass-card status-msg">Ready to Scan Student QR...</div>
            </div>

            <div class="col-lg-4">
                <div id="student-card" class="glass-card p-4 d-none shadow-lg">
                    <div class="text-center mb-4">
                        <img id="student-photo-display" src="" class="rounded-circle" alt="Student">
                    </div>
                    
                    <div class="info-label">Full Name</div>
                    <div id="disp-name" class="info-value">---</div>

                    <div class="row">
                        <div class="col-6">
                            <div class="info-label">LRN / ID</div>
                            <div id="disp-lrn" class="info-value">---</div>
                        </div>
                        <div class="col-6">
                            <div class="info-label">Grade & Section</div>
                            <div id="disp-level" class="info-value">---</div>
                        </div>
                    </div>

                    <div class="info-label">Adviser</div>
                    <div id="disp-adviser" class="info-value">---</div>

                    <div class="mt-3 pt-3 border-top border-secondary d-flex justify-content-between align-items-center">
                        <div>
                            <div class="info-label">Time In</div>
                            <div id="disp-time" class="text-white fw-bold">--:--</div>
                        </div>
                        <div class="text-end">
                            <span id="disp-status" class="badge rounded-pill px-4 py-2" style="font-size: 0.9rem;">---</span>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script>
        // Update this with your actual Firebase URL if different
        const FB_URL = "https://attendance-monitoring-84aeb-default-rtdb.firebaseio.com/";
        let isProcessing = false;

        async function onScanSuccess(decodedText) {
            if (isProcessing) return; 
            
            const statusDiv = document.getElementById('status');
            const infoCard = document.getElementById('student-card');
            
            try {
                isProcessing = true;
                statusDiv.innerHTML = "🔍 VERIFYING REGISTRATION...";

                // 1. Parse QR Data (from your generator: {fn, ln, gr, id})
                let qrData;
                try {
                    qrData = JSON.parse(decodedText);
                } catch (e) {
                    showError("⚠️ INVALID QR FORMAT");
                    return;
                }

                const scannedLrn = qrData.id || qrData.lrn; // Supports both keys
                if (!scannedLrn) {
                    showError("⚠️ NOT A STUDENT QR");
                    return;
                }

                // 2. Security Check: Verify against Firebase Registered Students
                const response = await fetch(`${FB_URL}students/${scannedLrn}.json`);
                const student = await response.json();

                if (!student) {
                    showError("❌ ACCESS DENIED: UNREGISTERED");
                    infoCard.classList.add('d-none');
                    return;
                }

                // 3. Logic for Attendance Time (Late threshold 6:31 AM)
                const now = new Date();
                const dateStr = now.toISOString().split('T')[0];
                const timeStr = now.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', hour12: true });
                
                // Set Late if 6:31 AM or later
                const isLate = (now.getHours() > 6 || (now.getHours() === 6 && now.getMinutes() >= 31));
                const attendanceStatus = isLate ? "LATE" : "PRESENT";

                // 4. Update ID Card UI
                displayIDCard(student, timeStr, attendanceStatus, isLate);
                
                // 5. Save to Attendance Log
                await fetch(`${FB_URL}attendance/${dateStr}/${scannedLrn}.json`, {
                    method: 'PUT',
                    body: JSON.stringify({
                        name: `${student.firstName} ${student.lastName}`,
                        lrn: scannedLrn,
                        time: timeStr,
                        status: attendanceStatus,
                        grade: student.grade || "N/A"
                    })
                });

                statusDiv.innerHTML = `<span style="color: #2ecc71;">✅ ACCESS GRANTED: ${student.lastName}</span>`;
                
            } catch (err) {
                console.error(err);
                showError("❌ DATABASE CONNECTION ERROR");
            } finally {
                // Pause for 3 seconds to show info before resetting
                setTimeout(() => { isProcessing = false; }, 3500);
            }
        }

        function displayIDCard(student, time, status, isLate) {
            const infoCard = document.getElementById('student-card');
            infoCard.classList.remove('d-none');
            
            document.getElementById('disp-name').innerText = `${student.firstName} ${student.middleName || ''} ${student.lastName}`;
            document.getElementById('disp-lrn').innerText = student.lrn;
            document.getElementById('disp-level').innerText = student.grade || "N/A";
            document.getElementById('disp-adviser').innerText = student.adviser || "---";
            document.getElementById('disp-time').innerText = time;
            
            const photoImg = document.getElementById('student-photo-display');
            photoImg.src = student.picture || "https://via.placeholder.com/140?text=NO+PHOTO";

            const statusBadge = document.getElementById('disp-status');
            statusBadge.innerText = status;
            statusBadge.className = isLate ? "badge bg-danger shadow-sm" : "badge bg-success shadow-sm";
        }

        function showError(msg) {
            const statusDiv = document.getElementById('status');
            statusDiv.innerHTML = `<span class="text-danger">${msg}</span>`;
            document.getElementById('student-card').classList.add('d-none');
            isProcessing = false;
        }

        // Initialize Scanner
        let scanner = new Html5QrcodeScanner("reader", { 
            fps: 15, 
            qrbox: { width: 250, height: 250 },
            aspectRatio: 1.0
        });
        scanner.render(onScanSuccess);
    </script>
</body>
</html>
