Then upload the matching .bin file. When you want to update, change the version number and upload the new .bin. The C3 picks it up automatically within 6 hours (or immediately via USB OTACHECK command).


---




### Step 2: Compile Your Firmware In Arduino IDE

```
1. Open your ESP32-S3 firmware in Arduino IDE
2. Go to: Sketch → Export Compiled Binary
3. Arduino IDE compiles and saves the .bin file
4. Find it in your sketch folder:
   
   Windows: C:\Users\YOU\Documents\Arduino\YourSketch\build\esp32.esp32.esp32s3\YourSketch.ino.bin
   Mac:     ~/Documents/Arduino/YourSketch/build/esp32.esp32.esp32s3/YourSketch.ino.bin
   
5. Rename it to something meaningful:
   varuna_s3_v2.1.bin
```

### Step 3: Create A GitHub Release

```
1. Go to your repo: https://github.com/YOUR_USERNAME/varuna-firmware
2. Click "Releases" on the right sidebar
3. Click "Create a new release" (or "Draft a new release")

4. Fill in:
   Tag version:    v2.1  (or whatever version)
   Release title:  Firmware v2.1 — Continuous Monitoring Fix
   Description:    
       - Added 2-second continuous flood monitoring
       - Emergency transmit on mode escalation
       - Fixed sampling rate proportionality bug
   
5. Drag and drop your .bin file into the "Attach binaries" area
   OR click "Attach binaries by dropping them here or selecting them"
   
6. Click "Publish release"
```

### Step 4: Get The Download URL

```
After publishing, the .bin file gets a permanent URL:

https://github.com/YOUR_USERNAME/varuna-firmware/releases/download/v2.1/varuna_s3_v2.1.bin
       ──────────── ───────────────                       ───── ────────────────────
       your account  your repo                             tag   exact filename

This URL:
    ✓ Is publicly accessible
    ✓ Works with HTTPS GET
    ✓ Returns a 302 redirect to the actual file on GitHub's CDN
    ✓ The C3's HTTPClient follows the redirect and downloads the binary
    ✓ Never expires
```

### Step 5: Verify It Works

```
Test in your browser:
    Paste the URL → it should download the .bin file

Test with curl:
    curl -L -o test.bin https://github.com/YOUR_USERNAME/varuna-firmware/releases/download/v2.1/varuna_s3_v2.1.bin
    
    The -L flag follows redirects (same as setFollowRedirects on ESP32)
    
    Check file size:
    ls -la test.bin
    → Should match your original .bin file size exactly
```

---

## Part 2: GitHub API — How The Website Fetches The Release List

The website doesn't need the user to paste URLs manually. It can fetch the list of all releases automatically using GitHub's free public API.

### The API Call

```
GET https://api.github.com/repos/YOUR_USERNAME/varuna-firmware/releases

No authentication needed for public repos.
Rate limit: 60 requests per hour per IP (more than enough).

Returns JSON array of all releases, each containing:
{
    "tag_name": "v2.1",
    "name": "Firmware v2.1 — Continuous Monitoring Fix",
    "published_at": "2025-01-15T14:30:00Z",
    "assets": [
        {
            "name": "varuna_s3_v2.1.bin",
            "size": 280000,
            "browser_download_url": "https://github.com/.../download/v2.1/varuna_s3_v2.1.bin"
        }
    ]
}
```

### What The Website Does With This

```
1. OTA modal opens
2. Website calls GitHub API → gets list of releases
3. Populates a dropdown:
   
   ┌─────────────────────────────────────────────────┐
   │  Select Firmware Version                    ▼   │
   ├─────────────────────────────────────────────────┤
   │  v2.1 — varuna_s3_v2.1.bin (273 KB) Jan 15    │
   │  v2.0 — varuna_s3_v2.0.bin (268 KB) Jan 10    │
   │  v1.9 — varuna_s3_v1.9.bin (265 KB) Jan 05    │
   └─────────────────────────────────────────────────┘

4. User selects a version
5. User clicks "Flash to Device"
6. Website writes to Firebase:
   devices/VARUNA_001/commands/ota = {
       url: "https://github.com/.../download/v2.1/varuna_s3_v2.1.bin",
       size: 280000,
       pending: true
   }
7. C3 picks it up within 5 seconds
8. C3 downloads from GitHub URL
9. C3 flashes S3
10. C3 writes result to Firebase
11. Website shows success/failure
```

---

## Part 3: How This Fits Into Your Existing Website

Looking at your website spec, the OTA functionality lives in two places:

### Place 1: The OTA Modal (already in your spec)

```
Your spec already defines:

#### OTA Modal (`.ota-modal-overlay`)
- Full-screen backdrop, centered modal (440px wide)
- Title: "Firmware OTA Update" with upload icon
- Description of the OTA process
- File input label (dashed border, accepts `.bin`)
- Upload button
- Progress bar
- Cancel + Close buttons
```

**This needs to be MODIFIED** from a file upload modal to a GitHub release picker modal. Here's what it should become:

```
OTA Modal — Updated Design:
┌──────────────────────────────────────────┐
│  ⚡ Firmware OTA Update            [✕]  │
│                                          │
│  The C3 will download the firmware from  │
│  GitHub and flash the S3 processor.      │
│  This takes 1-3 minutes.                 │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ Select Version                  ▼  │  │
│  ├────────────────────────────────────┤  │
│  │ v2.1 — 273 KB — Jan 15 2025      │  │
│  │ v2.0 — 268 KB — Jan 10 2025      │  │
│  │ v1.9 — 265 KB — Jan 05 2025      │  │
│  └────────────────────────────────────┘  │
│                                          │
│  Selected: v2.1 (varuna_s3_v2.1.bin)     │
│  Size: 273 KB                            │
│                                          │
│  ── OR upload custom .bin ──             │
│  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┐  │
│  │  Drop .bin file or click to browse │  │
│  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┘  │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │          ████████░░░░ 67%          │  │
│  └────────────────────────────────────┘  │
│  Buoy is downloading firmware...         │
│                                          │
│  [Cancel]              [Flash to Device] │
└──────────────────────────────────────────┘
```

The modal supports TWO input methods:
1. **Select from GitHub Releases** (dropdown, populated from API)
2. **Upload custom .bin** (drag and drop, for testing unreleased firmware)

For the custom upload path: since we can't use Firebase Storage, the website converts the .bin to base64 and writes it to Firebase RTDB. But this is a fallback — the primary path is GitHub Releases.

Actually, let me reconsider. For custom .bin uploads without Firebase Storage, we have a problem. The C3 downloads from a URL. If the user uploads a custom .bin, we need somewhere to host it temporarily.

**Simplest solution: don't support custom .bin upload through the website.** If a developer wants to flash custom firmware, they use the USB serial connection directly (Arduino IDE or esptool). The website OTA is specifically for deploying tested releases from GitHub. This keeps the system clean.

### Place 2: Console Panel OTA Button

```
Your spec says:
- Console toolbar has an "OTA Update" button (green)
- Clicking it opens the OTA modal

This just calls openOtaModal() — no change needed
```

---

## Part 4: Complete Data Flow Diagram

```
DEVELOPER                 GITHUB                    WEBSITE                FIREBASE              C3                  S3
─────────                 ──────                    ───────                ────────              ──                  ──

1. Compiles firmware
   in Arduino IDE
   "Export Compiled Binary"
   Gets: varuna_s3_v2.1.bin
        │
2. Creates GitHub Release
   Tag: v2.1
   Attaches .bin file
        │
        └──────────────→ Stores file
                         URL ready:
                         github.com/.../
                         v2.1/varuna_s3_v2.1.bin
                              │
                              │
3.                            │     User clicks "OTA Update"
                              │     in console toolbar
                              │          │
                              │          ↓
                              │     Modal opens
                              │     Website calls GitHub API:
                              │     GET /repos/.../releases
                              │          │
                              │←─────────┘
                              │     Returns release list
                              │──────────→
                              │          │
                              │     Dropdown populated
                              │     User selects v2.1
                              │     Clicks "Flash to Device"
                              │          │
4.                            │          └──────────────→ RTDB SET:
                              │                           commands/ota = {
                              │                             url: "github.com/...",
                              │                             size: 280000,
                              │                             pending: true
                              │                           }
                              │                                │
5.                            │                                │←── C3 polls
                              │                                │    reads pending=true
                              │                                │
                              │                                │──→ PUT status:
                              │                                │    "downloading"
                              │          │                     │
                              │          │←────────────────────┘
                              │     Modal shows:               │
                              │     "Buoy downloading..."      │
                              │                                │
6.                            │←───────────────────────────────┘
                         C3 sends:                         
                         GET /releases/download/            
                         v2.1/varuna_s3_v2.1.bin           
                              │                                │
                              └───────────────────────────────→│
                                   (follows 302 redirect)      │
                                   Downloads to SD card        │
                                                               │
7.                                                             │──→ PUT status:
                                                               │    "flashing"
                              │          │                     │
                              │          │←────────────────────┘
                              │     Modal shows:               │
                              │     "Flashing S3..."           │
                              │     Progress: 60%              │
                              │                                │
8.                                                             │──→ BOOT LOW
                                                               │    RESET pulse
                                                               │    SLIP sync
                                                               │    FLASH blocks
                                                               │    FLASH end
                                                               │    BOOT HIGH
                                                               │    RESET pulse
                                                               │         │
9.                                                             │         │ S3 boots
                                                               │←─ PONG ─│ new firmware
                                                               │         │
10.                                                            │──→ PUT status:
                                                               │    "success"
                              │          │                     │
                              │          │←────────────────────┘
                              │     Modal shows:               
                              │     "✓ Update Successful"      
                              │     Progress: 100% green       
```

---

## Part 5: What Goes In Your Website HTML File

### The GitHub Configuration Constants

```javascript
// Add these near the top of your <script> section
const GITHUB_OWNER = 'YOUR_USERNAME';        // Your GitHub username
const GITHUB_REPO  = 'varuna-firmware';       // Your firmware repo name
const GITHUB_API   = `https://api.github.com/repos/${GITHUB_OWNER}/${GITHUB_REPO}/releases`;
```

### The OTA Modal HTML (replace your existing OTA modal)

```html
<div class="ota-modal-overlay" id="otaModalOverlay">
    <div class="ota-modal">
        <div class="ota-modal-header">
            <span>⚡ Firmware OTA Update</span>
            <button onclick="closeOtaModal()" aria-label="Close">✕</button>
        </div>
        
        <p class="ota-desc">
            Select a firmware version from GitHub Releases. 
            The C3 will download and flash the S3 automatically. 
            Takes 1-3 minutes. Do not interrupt.
        </p>
        
        <!-- GitHub Release Selector -->
        <div class="ota-release-selector">
            <label>Select Firmware Version</label>
            <select id="otaReleaseSelect" onchange="onOtaReleaseSelected()">
                <option value="">Loading releases...</option>
            </select>
        </div>
        
        <!-- Selected Release Info -->
        <div class="ota-release-info" id="otaReleaseInfo" style="display:none">
            <span id="otaReleaseName"></span>
            <span id="otaReleaseSize"></span>
        </div>
        
        <!-- Progress Bar -->
        <div class="ota-progress-container" id="otaProgressContainer" style="display:none">
            <div class="ota-progress-bar">
                <div class="ota-progress-fill" id="otaProgressFill"></div>
            </div>
            <div class="ota-progress-labels">
                <span id="otaProgressText">Waiting...</span>
                <span id="otaProgressPercent">0%</span>
            </div>
        </div>
        
        <!-- Status Message -->
        <div class="ota-status" id="otaStatus"></div>
        
        <!-- Buttons -->
        <div class="ota-buttons">
            <button class="ota-btn-cancel" onclick="closeOtaModal()">Cancel</button>
            <button class="ota-btn-flash" id="otaFlashBtn" onclick="startOtaFlash()" disabled>
                Select a version first
            </button>
        </div>
    </div>
</div>
```

### The JavaScript OTA Logic

```javascript
// ============================================================
// OTA UPDATE VIA GITHUB RELEASES
// ============================================================

let otaReleases = [];          // Cached release list
let otaSelectedRelease = null; // Currently selected release
let otaMonitorRef = null;      // Firebase listener for OTA status
let otaInProgress = false;

// ---- Open OTA Modal ----
function openOtaModal() {
    document.getElementById('otaModalOverlay').classList.add('open');
    fetchGitHubReleases();  // Load releases when modal opens
}

// ---- Close OTA Modal ----
function closeOtaModal() {
    document.getElementById('otaModalOverlay').classList.remove('open');
    if (otaMonitorRef) {
        otaMonitorRef.off();   // Stop listening
        otaMonitorRef = null;
    }
}

// ---- Fetch Releases From GitHub API ----
async function fetchGitHubReleases() {
    const select = document.getElementById('otaReleaseSelect');
    select.innerHTML = '<option value="">Loading releases...</option>';
    
    try {
        const response = await fetch(GITHUB_API);
        
        if (!response.ok) {
            throw new Error(`GitHub API returned ${response.status}`);
        }
        
        otaReleases = await response.json();
        
        if (otaReleases.length === 0) {
            select.innerHTML = '<option value="">No releases found</option>';
            return;
        }
        
        // Build dropdown options
        select.innerHTML = '<option value="">— Select firmware version —</option>';
        
        otaReleases.forEach((release, index) => {
            // Find the .bin asset in this release
            const binAsset = release.assets.find(a => a.name.endsWith('.bin'));
            
            if (binAsset) {
                const date = new Date(release.published_at).toLocaleDateString();
                const sizeKB = (binAsset.size / 1024).toFixed(0);
                const label = `${release.tag_name} — ${binAsset.name} (${sizeKB} KB) — ${date}`;
                
                const option = document.createElement('option');
                option.value = index;
                option.textContent = label;
                select.appendChild(option);
            }
        });
        
        // Log to console
        if (typeof varunaConsole !== 'undefined' && varunaConsole.initialized) {
            varunaConsole.addLine(
                `Loaded ${otaReleases.length} firmware releases from GitHub`, 
                'info'
            );
        }
        
    } catch (error) {
        select.innerHTML = `<option value="">Error: ${error.message}</option>`;
        console.error('GitHub API error:', error);
        
        if (typeof varunaConsole !== 'undefined' && varunaConsole.initialized) {
            varunaConsole.addLine(
                `GitHub API error: ${error.message}`, 
                'error'
            );
        }
    }
}

// ---- Release Selected From Dropdown ----
function onOtaReleaseSelected() {
    const select = document.getElementById('otaReleaseSelect');
    const index = parseInt(select.value);
    const flashBtn = document.getElementById('otaFlashBtn');
    const infoDiv = document.getElementById('otaReleaseInfo');
    
    if (isNaN(index) || index < 0 || index >= otaReleases.length) {
        otaSelectedRelease = null;
        flashBtn.disabled = true;
        flashBtn.textContent = 'Select a version first';
        infoDiv.style.display = 'none';
        return;
    }
    
    const release = otaReleases[index];
    const binAsset = release.assets.find(a => a.name.endsWith('.bin'));
    
    if (!binAsset) {
        otaSelectedRelease = null;
        flashBtn.disabled = true;
        flashBtn.textContent = 'No .bin file in this release';
        return;
    }
    
    otaSelectedRelease = {
        tagName:     release.tag_name,
        name:        release.name,
        fileName:    binAsset.name,
        downloadUrl: binAsset.browser_download_url,
        size:        binAsset.size,
        publishedAt: release.published_at
    };
    
    // Show info
    document.getElementById('otaReleaseName').textContent = 
        `${release.tag_name} — ${binAsset.name}`;
    document.getElementById('otaReleaseSize').textContent = 
        `${(binAsset.size / 1024).toFixed(1)} KB`;
    infoDiv.style.display = 'flex';
    
    // Enable flash button
    flashBtn.disabled = false;
    flashBtn.textContent = `Flash ${release.tag_name} to Device`;
}

// ---- Start OTA Flash ----
async function startOtaFlash() {
    if (!otaSelectedRelease || otaInProgress) return;
    
    otaInProgress = true;
    const flashBtn = document.getElementById('otaFlashBtn');
    flashBtn.disabled = true;
    flashBtn.textContent = 'Sending to buoy...';
    
    // Show progress
    document.getElementById('otaProgressContainer').style.display = 'block';
    updateOtaProgress('Sending OTA command to buoy...', 5);
    setOtaStatus('Sending command to Firebase...', 'info');
    
    // Log to console
    if (typeof varunaConsole !== 'undefined' && varunaConsole.initialized) {
        varunaConsole.addLine('═══ OTA UPDATE STARTING ═══', 'system');
        varunaConsole.addLine(
            `Version: ${otaSelectedRelease.tagName} (${otaSelectedRelease.fileName})`, 
            'info'
        );
        varunaConsole.addLine(
            `URL: ${otaSelectedRelease.downloadUrl}`, 
            'info'
        );
    }
    
    try {
        // Write OTA command to Firebase
        // Use your existing Firebase database reference
        const otaRef = firebase.database().ref('devices/VARUNA_001/commands/ota');
        
        await otaRef.set({
            url:         otaSelectedRelease.downloadUrl,
            size:        otaSelectedRelease.size,
            fileName:    otaSelectedRelease.fileName,
            tagName:     otaSelectedRelease.tagName,
            pending:     true,
            status:      'pending',
            requestedAt: firebase.database.ServerValue.TIMESTAMP
        });
        
        setOtaStatus('Command sent — waiting for buoy to respond...', 'info');
        updateOtaProgress('Waiting for buoy...', 10);
        
        // Start monitoring the OTA status from C3
        startOtaMonitor();
        
    } catch (error) {
        setOtaStatus(`Error: ${error.message}`, 'error');
        updateOtaProgress('Failed', 0, 'error');
        flashBtn.disabled = false;
        flashBtn.textContent = `Flash ${otaSelectedRelease.tagName} to Device`;
        otaInProgress = false;
        
        if (typeof varunaConsole !== 'undefined' && varunaConsole.initialized) {
            varunaConsole.addLine(`OTA Error: ${error.message}`, 'error');
        }
    }
}

// ---- Monitor OTA Progress From C3 Via Firebase ----
function startOtaMonitor() {
    if (otaMonitorRef) otaMonitorRef.off();
    
    otaMonitorRef = firebase.database().ref('devices/VARUNA_001/commands/ota');
    
    otaMonitorRef.on('value', (snapshot) => {
        const data = snapshot.val();
        if (!data) return;
        
        const status = data.status || 'unknown';
        const flashBtn = document.getElementById('otaFlashBtn');
        
        switch (status) {
            case 'pending':
                updateOtaProgress('Waiting for buoy to start...', 10);
                setOtaStatus('Command pending — C3 will pick up within 5 seconds', 'info');
                break;
                
            case 'downloading':
                updateOtaProgress('Buoy is downloading firmware from GitHub...', 30);
                setOtaStatus('C3 is downloading firmware binary...', 'info');
                logConsole('C3 downloading firmware from GitHub...', 'info');
                break;
                
            case 'flashing':
                updateOtaProgress('Flashing S3 processor...', 65, 'flash');
                setOtaStatus('C3 is flashing the S3 via UART bootloader...', 'warning');
                logConsole('Flashing S3 via SLIP bootloader protocol...', 'warning');
                break;
                
            case 'success':
                updateOtaProgress('Update complete!', 100, 'success');
                setOtaStatus('✓ Firmware update successful! S3 is running new firmware.', 'success');
                flashBtn.textContent = '✓ Update Successful';
                logConsole('═══ OTA UPDATE SUCCESSFUL ═══', 'status');
                finishOta();
                break;
                
            case 'failed':
                const reason = data.message || 'Unknown error';
                updateOtaProgress('Update failed', 100, 'error');
                setOtaStatus(`✗ Update failed: ${reason}`, 'error');
                flashBtn.textContent = 'Update Failed — Try Again';
                flashBtn.disabled = false;
                logConsole(`═══ OTA FAILED: ${reason} ═══`, 'error');
                finishOta();
                break;
        }
    });
}

function finishOta() {
    otaInProgress = false;
    if (otaMonitorRef) {
        otaMonitorRef.off();
        otaMonitorRef = null;
    }
    
    // Re-enable button after 3 seconds
    setTimeout(() => {
        const flashBtn = document.getElementById('otaFlashBtn');
        if (otaSelectedRelease) {
            flashBtn.disabled = false;
            flashBtn.textContent = `Flash ${otaSelectedRelease.tagName} to Device`;
        }
    }, 3000);
}

// ---- Helper: Update Progress Bar ----
function updateOtaProgress(text, percent, style) {
    document.getElementById('otaProgressText').textContent = text;
    document.getElementById('otaProgressPercent').textContent = `${percent}%`;
    
    const fill = document.getElementById('otaProgressFill');
    fill.style.width = `${percent}%`;
    
    // Remove old style classes
    fill.classList.remove('flash', 'success', 'error');
    if (style) fill.classList.add(style);
}

// ---- Helper: Set Status Message ----
function setOtaStatus(text, type) {
    const el = document.getElementById('otaStatus');
    el.textContent = text;
    el.className = `ota-status ota-status-${type}`;
}

// ---- Helper: Log To Console If Available ----
function logConsole(text, type) {
    if (typeof varunaConsole !== 'undefined' && varunaConsole.initialized) {
        varunaConsole.addLine(text, type);
    }
}
```

### The CSS For The OTA Modal (add to your existing styles)

```css
/* OTA Release Selector */
.ota-release-selector {
    margin-bottom: 16px;
}

.ota-release-selector label {
    display: block;
    color: var(--text3);
    font-size: 11px;
    text-transform: uppercase;
    letter-spacing: 1px;
    margin-bottom: 6px;
}

.ota-release-selector select {
    width: 100%;
    padding: 10px 14px;
    background: var(--bg0);
    border: 1px solid var(--border);
    border-radius: var(--radius-xs);
    color: var(--text);
    font-family: var(--mono);
    font-size: 13px;
    outline: none;
    cursor: pointer;
}

.ota-release-selector select:focus {
    border-color: var(--accent);
}

.ota-release-info {
    display: flex;
    justify-content: space-between;
    padding: 8px 12px;
    background: rgba(59, 130, 246, 0.08);
    border: 1px solid rgba(59, 130, 246, 0.2);
    border-radius: var(--radius-xs);
    margin-bottom: 16px;
    font-size: 13px;
}

.ota-release-info span:first-child {
    color: var(--accent);
    font-weight: 500;
}

.ota-release-info span:last-child {
    color: var(--text3);
}

/* OTA Progress */
.ota-progress-container {
    margin-bottom: 12px;
}

.ota-progress-bar {
    width: 100%;
    height: 6px;
    background: var(--bg0);
    border-radius: 3px;
    overflow: hidden;
    margin-bottom: 6px;
}

.ota-progress-fill {
    height: 100%;
    background: var(--accent);
    border-radius: 3px;
    transition: width 0.5s ease;
    width: 0%;
}

.ota-progress-fill.flash { background: var(--amber); }
.ota-progress-fill.success { background: var(--green); }
.ota-progress-fill.error { background: var(--red); }

.ota-progress-labels {
    display: flex;
    justify-content: space-between;
    font-size: 11px;
    color: var(--text3);
}

/* OTA Status */
.ota-status {
    padding: 8px 12px;
    border-radius: var(--radius-xs);
    font-size: 12px;
    margin-bottom: 16px;
    min-height: 20px;
}

.ota-status-info { color: var(--accent); background: rgba(59,130,246,0.06); }
.ota-status-warning { color: var(--amber); background: rgba(245,158,11,0.06); }
.ota-status-success { color: var(--green); background: rgba(16,185,129,0.06); }
.ota-status-error { color: var(--red); background: rgba(239,68,68,0.06); }

/* OTA Buttons */
.ota-buttons {
    display: flex;
    justify-content: flex-end;
    gap: 12px;
}

.ota-btn-cancel {
    padding: 8px 20px;
    background: transparent;
    border: 1px solid var(--border);
    border-radius: var(--radius-xs);
    color: var(--text3);
    cursor: pointer;
    font-size: 13px;
}

.ota-btn-flash {
    padding: 8px 24px;
    background: var(--green);
    border: none;
    border-radius: var(--radius-xs);
    color: #000;
    font-weight: 600;
    cursor: pointer;
    font-size: 13px;
    transition: var(--transition-fast);
}

.ota-btn-flash:hover:not(:disabled) {
    background: var(--green2);
}

.ota-btn-flash:disabled {
    background: var(--bg4);
    color: var(--text4);
    cursor: not-allowed;
}
```

---

## Part 6: Firebase RTDB Structure For OTA

```
Your Firebase RTDB (already at varuna-flood):

devices/
  VARUNA_001/
    commands/
      ota/                          ← Website WRITES here
        url: "https://github.com/.../v2.1/varuna_s3_v2.1.bin"
        size: 280000
        fileName: "varuna_s3_v2.1.bin"
        tagName: "v2.1"
        pending: true               ← C3 sets to false after reading
        status: "downloading"       ← C3 updates: pending → downloading → flashing → success/failed
        message: "Success"          ← C3 writes on completion
        requestedAt: 1737000000000  ← Server timestamp
```

The C3 code you already have handles this exact structure. The only C3 change needed (from the earlier prompt) is adding `WiFiClientSecure` with `setInsecure()` and `setFollowRedirects()` so it can follow GitHub's 302 redirect.

---

## Part 7: Connecting The Button In Your Console Toolbar

Your spec says the console toolbar has an "OTA Update" button. Just wire it:

```html
<!-- In your console toolbar -->
<button class="console-toolbar-btn ota-btn" onclick="openOtaModal()">
    ⚡ OTA Update
</button>
```

When clicked, `openOtaModal()` runs, which:
1. Shows the modal
2. Calls `fetchGitHubReleases()` 
3. Populates the dropdown
4. User picks a version and clicks Flash

---

## Summary: What You Need To Do

```
STEP   ACTION                                   WHERE
─────  ───────────────────────────────────────   ──────────────
1      Create public GitHub repo                 github.com
       "varuna-firmware"

2      Compile firmware in Arduino IDE           Your computer
       Sketch → Export Compiled Binary

3      Create GitHub Release                     github.com
       Tag: v2.1
       Attach the .bin file

4      Add GITHUB_OWNER and GITHUB_REPO          Your website .html
       constants to your JavaScript

5      Replace OTA modal HTML                    Your website .html
       with the GitHub release picker

6      Add OTA JavaScript functions              Your website .html
       (fetchGitHubReleases, startOtaFlash, 
       startOtaMonitor, etc.)

7      Add OTA CSS styles                        Your website .html

8      Update C3 firmware                        ESP32-C3 code
       (WiFiClientSecure + setFollowRedirects
       from the earlier prompt)

9      Test end to end                           Everything
```

**Steps 1-3 are one-time setup. Steps 4-7 go in your single .html file. Step 8 is the C3 firmware update from the earlier prompt. That's it.**
