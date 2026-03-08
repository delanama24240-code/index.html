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
        const FB_URL = "https://attendance-monitoring-84aeb-default-rtdb.firebaseio.com/";

        async function onScanSuccess(decodedText) {
            const statusDiv = document.getElementById('status');
            const infoCard = document.getElementById('student-card');
            
            try {
                // 1. Get the LRN from the tiny QR code
                const qrData = JSON.parse(decodedText);
                const scannedLrn = qrData.lrn;

                if (!scannedLrn) throw new Error("No LRN found");

                statusDiv.innerText = "🔍 Fetching Profile...";

                // 2. Fetch the FULL DATA (photo, level, etc.) from Firebase
                const response = await fetch(`${FB_URL}students/${scannedLrn}.json`);
                const student = await response.json();

                if (!student) {
                    statusDiv.innerText = "❌ STUDENT NOT FOUND";
                    return;
                }

                // 3. UI Update using fetched data
                document.getElementById('disp-name').innerText = `${student.firstName} ${student.lastName}`.trim();
                document.getElementById('disp-lrn').innerText = student.lrn;
                document.getElementById('disp-level-section').innerText = `${student.level || "N/A"} - ${student.section || "N/A"}`;
                document.getElementById('disp-adviser').innerText = student.adviser || "Not Assigned";
                document.getElementById('student-photo-display').src = student.picture || "";

                infoCard.classList.remove('d-none');

                // 4. Attendance Logic
                const now = new Date();
                const dateStr = now.toISOString().split('T')[0];
                const timeStr = now.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
                
                const isLate = (now.getHours() > 6 || (now.getHours() === 6 && now.getMinutes() >= 31));
                const attendanceStatus = isLate ? "Late" : "Present";
                
                const statusBadge = document.getElementById('disp-status');
                statusBadge.innerText = attendanceStatus;
                statusBadge.className = isLate ? "badge bg-danger" : "badge bg-success";

                // 5. Save Attendance Log
                await fetch(`${FB_URL}attendance/${dateStr}/${scannedLrn}.json`, {
                    method: 'PUT',
                    body: JSON.stringify({
                        name: `${student.firstName} ${student.lastName}`,
                        lrn: scannedLrn,
                        time: timeStr,
                        status: attendanceStatus,
                        level: student.level || "N/A",
                        section: student.section || "N/A"
                    })
                });

                statusDiv.innerText = `✅ Verified: ${student.lastName}`;

            } catch (err) {
                console.error(err);
                statusDiv.innerText = "❌ ERROR: INVALID QR CODE";
            }
        }

        let scanner = new Html5QrcodeScanner("reader", { fps: 10, qrbox: 250 });
        scanner.render(onScanSuccess);
    </script>
</body>
</html>
