<html>
<head>
    <title>QR Attendance Scanner</title>
    <script src="https://unpkg.com/html5-qrcode"></script>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body {
            background-color: #0a0b1e;
            background: radial-gradient(circle at top right, #1a1b3a, #0a0b1e);
            min-height: 100vh;
        }
        .glass-card {
            background: rgba(255, 255, 255, 0.05);
            backdrop-filter: blur(12px);
            border: 1px solid rgba(255, 255, 255, 0.1);
            border-radius: 20px;
        }
        #reader { border-radius: 20px; overflow: hidden; border: 2px solid rgba(0, 255, 255, 0.2) !important; }
        .info-label { color: #0dcaf0; font-weight: 600; font-size: 0.8rem; text-transform: uppercase; }
        .info-value { color: #ffffff; font-size: 1.1rem; margin-bottom: 10px; }
        #student-photo-display {
            width: 120px; height: 120px; object-fit: cover;
            border: 3px solid #6f42c1; padding: 3px;
        }
        /* Add a pulse effect when searching */
        .searching { opacity: 0.5; pointer-events: none; }
    </style>
</head>
<body class="text-white">
    <div class="container-fluid py-5">
        <div class="row justify-content-center g-4">
            <div class="col-lg-5 text-center">
                <h2 class="mb-4 fw-bold" style="color: #0dcaf0;">MAHS Scanner</h2>
                <div id="reader" class="mx-auto shadow-lg" style="max-width: 500px;"></div>
                <div id="status" class="mt-4 p-3 rounded glass-card">Ready to Scan...</div>
            </div>

            <div class="col-lg-4">
                <div id="student-card" class="glass-card p-4 d-none">
                    <div class="text-center mb-3">
                        <img id="student-photo-display" src="" class="rounded-circle shadow" alt="Student">
                    </div>
                    
                    <div class="info-label">Student Name</div>
                    <div id="disp-name" class="info-value">---</div>

                    <div class="row">
                        <div class="col-6">
                            <div class="info-label">LRN</div>
                            <div id="disp-lrn" class="info-value">---</div>
                        </div>
                        <div class="col-6">
                            <div class="info-label">Level & Section</div>
                            <div id="disp-level-section" class="info-value">---</div>
                        </div>
                    </div>

                    <div class="info-label">Adviser</div>
                    <div id="disp-adviser" class="info-value">---</div>

                    <div class="mt-3 pt-3 border-top border-secondary d-flex justify-content-between align-items-center">
                        <span class="info-label">Status</span>
                        <span id="disp-status" class="badge rounded-pill px-3 py-2">---</span>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script>
        // Use your specific Firebase URL
        const FB_URL = "https://attendance-monitoring-84aeb-default-rtdb.firebaseio.com/";

        async function onScanSuccess(decodedText) {
            const statusDiv = document.getElementById('status');
            const infoCard = document.getElementById('student-card');
            
            try {
                // 1. Validate if the QR is a JSON object (System Format)
                let qrData;
                try {
                    qrData = JSON.parse(decodedText);
                } catch (e) {
                    statusDiv.innerHTML = "<span class='text-warning'>⚠️ INVALID SYSTEM QR</span>";
                    infoCard.classList.add('d-none');
                    return;
                }

                const scannedLrn = qrData.lrn;
                if (!scannedLrn) {
                    statusDiv.innerHTML = "<span class='text-warning'>⚠️ WRONG QR FORMAT</span>";
                    return;
                }

                statusDiv.innerText = "🔍 Verifying Student...";
                infoCard.classList.add('searching');

                // 2. Fetch the FULL DATA from Firebase using the LRN
                const response = await fetch(`${FB_URL}students/${scannedLrn}.json`);
                const student = await response.json();

                // 3. Security Check: Does student exist in our DB?
                if (!student) {
                    statusDiv.innerHTML = "<span class='text-danger'>❌ UNAUTHORIZED / NOT FOUND</span>";
                    infoCard.classList.add('d-none');
                    return;
                }

                // 4. Update UI with verified data
                document.getElementById('disp-name').innerText = `${student.firstName} ${student.lastName}`.trim();
                document.getElementById('disp-lrn').innerText = student.lrn;
                
                // Use 'level' as registered in your AddStudent payload
                const levelSection = student.level || student.grade || "N/A";
                document.getElementById('disp-level-section').innerText = levelSection;
                document.getElementById('disp-adviser').innerText = student.adviser || "Not Assigned";
                
                // Handle the Base64 picture
                document.getElementById('student-photo-display').src = student.picture || "";

                infoCard.classList.remove('d-none', 'searching');

                // 5. Attendance Logic (6:31 AM Late Threshold)
                const now = new Date();
                const dateStr = now.toISOString().split('T')[0];
                const timeStr = now.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', hour12: true });
                
                const isLate = (now.getHours() > 6 || (now.getHours() === 6 && now.getMinutes() >= 31));
                const attendanceStatus = isLate ? "Late" : "Present";
                
                const statusBadge = document.getElementById('disp-status');
                statusBadge.innerText = attendanceStatus;
                statusBadge.className = isLate ? "badge bg-danger" : "badge bg-success shadow-sm";

                // 6. Log the Attendance to Firebase
                await fetch(`${FB_URL}attendance/${dateStr}/${scannedLrn}.json`, {
                    method: 'PUT',
                    body: JSON.stringify({
                        name: `${student.firstName} ${student.lastName}`,
                        lrn: scannedLrn,
                        time: timeStr,
                        status: attendanceStatus,
                        level: levelSection,
                        timestamp: ServerValue.TIMESTAMP // Optional: if you use FB SDK
                    })
                });

                statusDiv.innerHTML = `<span class="text-success">✅ Welcome, ${student.lastName}!</span>`;

            } catch (err) {
                console.error("Scan Error:", err);
                statusDiv.innerText = "❌ Scanner Error. Please try again.";
            }
        }

        // Initialize Scanner
        let scanner = new Html5QrcodeScanner("reader", { 
            fps: 15, // Faster frame rate for smoother scanning
            qrbox: { width: 250, height: 250 },
            aspectRatio: 1.0 
        });
        scanner.render(onScanSuccess);
    </script>
</body>
</html>
