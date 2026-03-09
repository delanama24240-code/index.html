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
    
    try {
        // 1. UNPACK THE DATA FROM THE QR
        // The QR now contains a JSON string with: fname, lname, lrn, grade, pic
        const student = JSON.parse(decodedText);
        
        // 2. DISPLAY DATA IMMEDIATELY IN THE ID FORM
        document.getElementById('disp-name').innerText = `${student.firstName} ${student.lastName}`;
        document.getElementById('disp-lrn').innerText = student.lrn;
        document.getElementById('disp-level').innerText = student.grade;
        document.getElementById('disp-section').innerText = student.adviser || "N/A";

        // Update/Add the Photo to the ID form
        let photoImg = document.getElementById('student-photo-display');
        if (!photoImg) {
            photoImg = document.createElement('img');
            photoImg.id = 'student-photo-display';
            photoImg.className = "rounded-circle border border-info mb-3";
            photoImg.style = "width: 100px; height: 100px; object-fit: cover; display: block; margin: 0 auto;";
            infoCard.querySelector('h4').after(photoImg);
        }
        photoImg.src = student.picture || "";

        // Show the card
        infoCard.classList.remove('d-none');

        // 3. CALCULATE ATTENDANCE LOGIC
        const now = new Date();
        const dateStr = now.toISOString().split('T')[0];
        const timeStr = now.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' });
        
        // Late threshold: 6:31 AM
        const isLate = (now.getHours() > 6 || (now.getHours() === 6 && now.getMinutes() >= 31));
        const attendanceStatus = isLate ? "Late" : "Present";
        
        document.getElementById('disp-status').innerText = attendanceStatus;
        document.getElementById('disp-status').className = isLate ? "badge bg-danger" : "badge bg-success";

        // 4. SAVE TO FIREBASE
        // We use student.lrn from the QR data
        statusDiv.className = "mt-4 p-3 rounded bg-primary text-white";
        statusDiv.innerText = "Saving attendance...";

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

        statusDiv.className = "mt-4 p-3 rounded bg-success text-white";
        statusDiv.innerText = `✅ Recorded: ${student.lastName}`;

    } catch (err) {
        // If JSON.parse fails, it means the QR is not a valid student QR
        console.error("Scan Error:", err);
        statusDiv.className = "mt-4 p-3 rounded bg-danger text-white";
        statusDiv.innerText = "❌ UNREGISTERED QR CODE";
        infoCard.classList.add('d-none');
    }
}

let scanner = new Html5QrcodeScanner("reader", { 
    fps: 15, 
    qrbox: { width: 250, height: 250 }
});
scanner.render(onScanSuccess);
</script>
