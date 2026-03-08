<html>
<head>
    <title>QR Attendance Scanner</title>
    <script src="https://unpkg.com/html5-qrcode"></script>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body {
            background-color: #0a0b1e; /* Deep navy background */
            background: radial-gradient(circle at top right, #1a1b3a, #0a0b1e);
            min-height: 100vh;
        }
        .glass-card {
            background: rgba(255, 255, 255, 0.05);
            backdrop-filter: blur(12px);
            -webkit-backdrop-filter: blur(12px);
            border: 1px solid rgba(255, 255, 255, 0.1);
            border-radius: 20px;
            box-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.37);
        }
        #reader { 
            border-radius: 20px; 
            overflow: hidden; 
            border: 2px solid rgba(0, 255, 255, 0.2) !important; 
        }
        #reader__dashboard_section_csr button {
            background-color: #6f42c1 !important; /* Neon Purple accents */
            color: white !important;
            border: none !important;
            border-radius: 5px;
            padding: 8px 15px;
        }
        .info-label { color: #0dcaf0; font-weight: 600; font-size: 0.9rem; }
        .info-value { color: #ffffff; font-size: 1.1rem; }
    </style>
</head>
<body class="text-white">
    <div class="container-fluid py-5">
        <div class="row justify-content-center align-items-start">
            <div class="col-lg-5 text-center mb-4">
                <h2 class="mb-4 fw-bold" style="color: #0dcaf0;">Attendance Scanner</h2>
                <div id="reader" class="mx-auto shadow-lg" style="max-width: 500px;"></div>
                <div id="status" class="mt-4 p-3 rounded glass-card">Ready to Scan...</div>
            </div>

            <div class="col-lg-4">
                <div id="student-card" class="glass-card p-4 d-none">
                    <h4 class="text-center mb-4 fw-bold" style="letter-spacing: 1px;">STUDENT PROFILE</h4>
                    
                    <div id="photo-container" class="text-center mb-3">
                        </div>

                    <div class="row g-3">
                        <div class="col-12">
                            <div class="info-label">FULL NAME</div>
                            <div id="disp-name" class="info-value">---</div>
                        </div>
                        <div class="col-6">
                            <div class="info-label">LRN</div>
                            <div id="disp-lrn" class="info-value">---</div>
                        </div>
                        <div class="col-6">
                            <div class="info-label">LEVEL</div>
                            <div id="disp-level" class="info-value">---</div>
                        </div>
                        <div class="col-12">
                            <div class="info-label">ADVISER / SECTION</div>
                            <div id="disp-section" class="info-value">---</div>
                        </div>
                        <div class="col-12 mt-4 pt-3 border-top border-secondary">
                            <div class="d-flex justify-content-between align-items-center">
                                <span class="info-label">STATUS:</span>
                                <span id="disp-status" class="badge rounded-pill px-3 py-2">---</span>
                            </div>
                        </div>
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
                // 1. Parse QR Data
                const student = JSON.parse(decodedText);
                
                // 2. Map data to the UI
                document.getElementById('disp-name').innerText = `${student.firstName} ${student.lastName}`;
                document.getElementById('disp-lrn').innerText = student.lrn || "N/A";
                document.getElementById('disp-level').innerText = student.level || "N/A";
                document.getElementById('disp-section').innerText = student.adviser || student.section || "N/A";

                // Handle Photo display
                const photoContainer = document.getElementById('photo-container');
                photoContainer.innerHTML = ''; // Clear old photo
                if (student.picture) {
                    const img = document.createElement('img');
                    img.src = student.picture;
                    img.className = "rounded-circle border border-info p-1 shadow";
                    img.style = "width: 120px; height: 120px; object-fit: cover;";
                    photoContainer.appendChild(img);
                }

                // Show the card
                infoCard.classList.remove('d-none');

                // 3. Attendance Logic
                const now = new Date();
                const dateStr = now.toISOString().split('T')[0];
                const timeStr = now.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' });
                
                // Threshold: 6:31 AM
                const isLate = (now.getHours() > 6 || (now.getHours() === 6 && now.getMinutes() >= 31));
                const attendanceStatus = isLate ? "Late" : "Present";
                
                const statusBadge = document.getElementById('disp-status');
                statusBadge.innerText = attendanceStatus;
                statusBadge.className = isLate ? "badge rounded-pill px-3 py-2 bg-danger" : "badge rounded-pill px-3 py-2 bg-success";

                // 4. Firebase Integration
                statusDiv.style.color = "#0dcaf0";
                statusDiv.innerText = "Processing record...";

                await fetch(`${FB_URL}attendance/${dateStr}/${student.lrn}.json`, {
                    method: 'PUT',
                    body: JSON.stringify({
                        name: `${student.firstName} ${student.lastName}`,
                        lrn: student.lrn,
                        time: timeStr,
                        date: dateStr,
                        status: attendanceStatus
                    })
                });

                statusDiv.style.color = "#2ecc71";
                statusDiv.innerText = `✅ Attendance Logged: ${student.lastName}`;

            } catch (err) {
                console.error("Scan Error:", err);
                statusDiv.style.color = "#e74c3c";
                statusDiv.innerText = "❌ INVALID OR UNREGISTERED QR CODE";
                infoCard.classList.add('d-none');
            }
        }

        let scanner = new Html5QrcodeScanner("reader", { 
            fps: 20, 
            qrbox: { width: 250, height: 250 },
            aspectRatio: 1.0
        });
        scanner.render(onScanSuccess);
    </script>
</body>
</html>
