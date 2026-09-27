<script setup>
import { ref, onMounted } from 'vue';
import { Command } from '@tauri-apps/plugin-shell';

const os = ref('linux');
const isCheckingDeps = ref(true);
const dependenciesReady = ref(false);
const missingDeps = ref([]);
const isInstalling = ref(false);
const installLogs = ref('');
const usingFallback = ref(false);

const activeTab = ref('connect');
const ipAddress = ref(localStorage.getItem('scrcpy_ip') || '192.168.1.');
const port = ref(localStorage.getItem('scrcpy_port') || '');
const statusLog = ref('Ready to connect.');
const isConnected = ref(false);
const isLoading = ref(false);
const pairPort = ref('');
const pairCode = ref('');

function logInstall(msg) {
  if (msg) {
    installLogs.value += msg.trim() + '\n';
  }
}

function detectOS() {
  const ua = navigator.userAgent.toLowerCase();
  if (ua.includes('win')) return 'windows';
  if (ua.includes('mac')) return 'macos';
  return 'linux';
}

function createAdbCommand(args) {
  if (usingFallback.value && os.value === 'windows') {
    const psArgs = ['-NoProfile', '-Command', `& "$env:LOCALAPPDATA\\scrcpy-gui\\bin\\adb.exe" ${args.join(' ')}`];
    return Command.create('run-powershell', psArgs);
  }
  return Command.create('run-adb', args);
}

function createScrcpyCommand(args) {
  if (usingFallback.value && os.value === 'windows') {
    const psArgs = ['-NoProfile', '-Command', `& "$env:LOCALAPPDATA\\scrcpy-gui\\bin\\scrcpy.exe" ${args.join(' ')}`];
    return Command.create('run-powershell', psArgs);
  }
  return Command.create('run-scrcpy', args);
}

async function checkDependencies() {
  isCheckingDeps.value = true;
  missingDeps.value = [];
  os.value = detectOS();

  try {
    const adbCheck = createAdbCommand(['--version']);
    const adbOut = await adbCheck.execute();
    if (adbOut.code !== 0) missingDeps.value.push('adb');
  } catch {
    missingDeps.value.push('adb');
  }

  try {
    const scrcpyCheck = createScrcpyCommand(['--version']);
    const scrcpyOut = await scrcpyCheck.execute();
    if (scrcpyOut.code !== 0) missingDeps.value.push('scrcpy');
  } catch {
    missingDeps.value.push('scrcpy');
  }

  if (missingDeps.value.length > 0 && os.value === 'windows' && !usingFallback.value) {
    usingFallback.value = true;
    await checkDependencies();
    return;
  }

  dependenciesReady.value = missingDeps.value.length === 0;
  isCheckingDeps.value = false;
}

async function installDependencies() {
  isInstalling.value = true;
  logInstall(`[INFO] Starting installation for ${os.value}...`);

  try {
    if (os.value === 'linux') {
      logInstall('[INFO] Requesting privileges to update packages...');
      const cmd1 = Command.create('run-pkexec', ['apt-get', 'update']);
      cmd1.on('stdout', logInstall);
      cmd1.on('stderr', logInstall);
      await cmd1.execute();

      logInstall('[INFO] Installing scrcpy and adb...');
      const cmd2 = Command.create('run-pkexec', ['apt-get', 'install', '-y', 'scrcpy', 'adb']);
      cmd2.on('stdout', logInstall);
      cmd2.on('stderr', logInstall);
      await cmd2.execute();
    } else if (os.value === 'macos') {
      logInstall('[INFO] Installing via Homebrew...');
      const cmd = Command.create('run-brew', ['install', 'scrcpy', 'android-platform-tools']);
      cmd.on('stdout', logInstall);
      cmd.on('stderr', logInstall);
      await cmd.execute();
    } else if (os.value === 'windows') {
      logInstall('[INFO] Downloading and extracting latest Scrcpy release...');
      const psScript = `
        $ProgressPreference = 'SilentlyContinue';
        $dir = "$env:LOCALAPPDATA\\scrcpy-gui\\bin";
        New-Item -ItemType Directory -Force -Path $dir | Out-Null;
        $api = Invoke-RestMethod -Uri "https://api.github.com/repos/Genymobile/scrcpy/releases/latest";
        $url = ($api.assets | Where-Object name -match "win64.*zip")[0].browser_download_url;
        $zip = "$env:TEMP\\scrcpy.zip";
        Write-Host "Downloading $url ...";
        Invoke-WebRequest -Uri $url -OutFile $zip;
        Write-Host "Extracting to $dir ...";
        Expand-Archive -Path $zip -DestinationPath $dir -Force;
        $sub = Get-ChildItem -Directory $dir | Where-Object Name -match "scrcpy";
        if ($sub) { Move-Item -Path "$($sub.FullName)\\*" -Destination $dir -Force; Remove-Item $sub.FullName -Force; }
        Write-Host "Installation complete.";
      `;
      const cmd = Command.create('run-powershell', ['-NoProfile', '-Command', psScript]);
      cmd.on('stdout', logInstall);
      cmd.on('stderr', logInstall);
      await cmd.execute();
      usingFallback.value = true;
    }

    logInstall('[SUCCESS] Installation process finished. Verifying...');
    await checkDependencies();
  } catch (err) {
    logInstall(`[ERROR] Installation failed: ${err}`);
  } finally {
    isInstalling.value = false;
  }
}

function savePreferences() {
  localStorage.setItem('scrcpy_ip', ipAddress.value);
  localStorage.setItem('scrcpy_port', port.value);
}

async function checkDeviceStatus() {
  isLoading.value = true;
  statusLog.value = 'Checking active devices...';
  try {
    const cmd = createAdbCommand(['devices']);
    const output = await cmd.execute();
    const result = output.stdout || output.stderr;
    statusLog.value = result;

    const target = `${ipAddress.value}:${port.value}`;
    if (result.includes(target) && result.includes('device')) {
      isConnected.value = true;
      statusLog.value += '\n[INFO] Device detected and ready.';
    } else {
      isConnected.value = false;
    }
  } catch (err) {
    statusLog.value = `[ERROR] Failed to read ADB status: ${err}`;
  } finally {
    isLoading.value = false;
  }
}

async function pairDevice() {
  if (!pairPort.value || !pairCode.value) {
    statusLog.value = '[WARNING] Please enter Pairing Port and 6-Digit Code.';
    return;
  }

  isLoading.value = true;
  statusLog.value = `Pairing with ${ipAddress.value}:${pairPort.value}...`;

  try {
    const target = `${ipAddress.value}:${pairPort.value}`;
    const cmd = createAdbCommand(['pair', target, pairCode.value]);
    const output = await cmd.execute();
    const result = output.stdout || output.stderr;
    statusLog.value = result;

    if (result.includes('Successfully paired')) {
      statusLog.value += '\n[SUCCESS] Device paired. Switch to Connect tab.';
      savePreferences();
      activeTab.value = 'connect';
    }
  } catch (err) {
    statusLog.value = `[ERROR] Pairing failed: ${err}`;
  } finally {
    isLoading.value = false;
  }
}

async function connectADB() {
  if (!port.value) {
    statusLog.value = '[WARNING] Please enter an active port first.';
    return;
  }

  savePreferences();
  isLoading.value = true;
  statusLog.value = `Connecting to ${ipAddress.value}:${port.value}...`;

  try {
    const target = `${ipAddress.value}:${port.value}`;
    const cmd = createAdbCommand(['connect', target]);
    const output = await cmd.execute();
    const result = output.stdout || output.stderr;
    statusLog.value = result;

    if (result.includes('connected to')) {
      isConnected.value = true;
    } else {
      isConnected.value = false;
    }
  } catch (err) {
    statusLog.value = `[ERROR] ADB execution failed: ${err}`;
    isConnected.value = false;
  } finally {
    isLoading.value = false;
  }
}

async function disconnectADB() {
  savePreferences();
  isLoading.value = true;
  
  const target = `${ipAddress.value}:${port.value}`;
  statusLog.value = `Disconnecting from ${target}...`;

  try {
    const cmd = createAdbCommand(['disconnect', target]);
    const output = await cmd.execute();
    const result = output.stdout || output.stderr;
    statusLog.value = result;
    isConnected.value = false;
    statusLog.value += '\n[INFO] Device disconnected.';
  } catch (err) {
    statusLog.value = `[ERROR] Disconnect failed: ${err}`;
  } finally {
    isLoading.value = false;
  }
}

async function startScrcpy() {
  statusLog.value = 'Launching Scrcpy...';
  try {
    const cmd = createScrcpyCommand(['-b', '8M', '--max-fps', '60']);
    await cmd.spawn();
    statusLog.value = '[SUCCESS] Scrcpy is running.';
  } catch (err) {
    statusLog.value = `[ERROR] Failed to launch Scrcpy: ${err}`;
  }
}

onMounted(() => {
  checkDependencies().then(() => {
    if (dependenciesReady.value && port.value) {
      checkDeviceStatus();
    }
  });
});
</script>

<template>
  <div class="container">
    <div class="header-title">
      <svg class="svg-icon" viewBox="0 0 24 24" width="22" height="22" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
        <rect width="14" height="20" x="5" y="2" rx="2" ry="2"/>
        <path d="M12 18h.01"/>
      </svg>
      <h2>Scrcpy Wireless Manager</h2>
    </div>
    <p class="subtitle">Wireless Android device controller via ADB & Scrcpy</p>

    <!-- LOADING STATE -->
    <div v-if="isCheckingDeps" class="card setup-card">
      <p>Checking system requirements...</p>
    </div>

    <!-- SETUP WIZARD -->
    <div v-else-if="!dependenciesReady" class="card setup-card">
      <h3>Dependency Setup Required</h3>
      <div class="os-badge">Detected OS: <strong>{{ os.toUpperCase() }}</strong></div>
      
      <ul class="dep-list">
        <li>
          <span>ADB</span>
          <span :class="missingDeps.includes('adb') ? 'text-red' : 'text-green'">
            <svg v-if="missingDeps.includes('adb')" viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none"><circle cx="12" cy="12" r="10"/><line x1="12" y1="8" x2="12" y2="12"/><line x1="12" y1="16" x2="12.01" y2="16"/></svg>
            <svg v-else viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none"><path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><polyline points="22 4 12 14.01 9 11.01"/></svg>
            {{ missingDeps.includes('adb') ? 'Missing' : 'Found' }}
          </span>
        </li>
        <li>
          <span>Scrcpy</span>
          <span :class="missingDeps.includes('scrcpy') ? 'text-red' : 'text-green'">
            <svg v-if="missingDeps.includes('scrcpy')" viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none"><circle cx="12" cy="12" r="10"/><line x1="12" y1="8" x2="12" y2="12"/><line x1="12" y1="16" x2="12.01" y2="16"/></svg>
            <svg v-else viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none"><path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><polyline points="22 4 12 14.01 9 11.01"/></svg>
            {{ missingDeps.includes('scrcpy') ? 'Missing' : 'Found' }}
          </span>
        </li>
      </ul>

      <button class="btn btn-connect" :disabled="isInstalling" @click="installDependencies" style="margin-top: 16px;">
        <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
          <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/>
          <polyline points="7 10 12 15 17 10"/>
          <line x1="12" y1="15" x2="12" y2="3"/>
        </svg>
        {{ isInstalling ? 'Installing...' : 'Install & Setup Tools' }}
      </button>

      <div class="status-box" style="margin-top: 16px;" v-if="installLogs">
        <div class="status-header">
          <div class="status-title">
            <svg viewBox="0 0 24 24" width="14" height="14" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
              <circle cx="12" cy="12" r="3"/><path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"/>
            </svg>
            <span>Installation Log</span>
          </div>
        </div>
        <pre>{{ installLogs }}</pre>
      </div>
    </div>

    <!-- MAIN DASHBOARD -->
    <div v-else>
      <div class="tabs">
        <button :class="['tab-btn', { active: activeTab === 'connect' }]" @click="activeTab = 'connect'">
          <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
            <path d="M5 12.55a11 11 0 0 1 14.08 0"/>
            <path d="M1.42 9a16 16 0 0 1 21.16 0"/>
            <path d="M8.53 16.11a6 6 0 0 1 6.95 0"/>
            <line x1="12" y1="20" x2="12.01" y2="20"/>
          </svg>
          Connect (Daily)
        </button>
        <button :class="['tab-btn', { active: activeTab === 'pair' }]" @click="activeTab = 'pair'">
          <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
            <circle cx="7.5" cy="15.5" r="5.5"/>
            <path d="m21 2-9.6 9.6"/>
            <path d="m15.5 7.5 3 3L22 7l-3-3"/>
          </svg>
          Pair Device (1x)
        </button>
      </div>

      <!-- TAB 1: CONNECT -->
      <div v-if="activeTab === 'connect'" class="card">
        <div class="form-group">
          <label>Device IP Address</label>
          <input v-model="ipAddress" type="text" placeholder="192.168.1.xxx" @change="savePreferences" />
        </div>

        <div class="form-group">
          <label>Active Wireless Port</label>
          <input v-model="port" type="text" placeholder="Port from Wireless Debugging screen" @change="savePreferences" />
        </div>

        <div class="actions-grid">
          <button class="btn btn-connect" :disabled="isLoading" @click="connectADB">
            <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
              <path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"/>
              <path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"/>
            </svg>
            Connect
          </button>

          <button class="btn btn-disconnect" :disabled="isLoading" @click="disconnectADB">
            <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
              <line x1="18" y1="6" x2="6" y2="18"/>
              <line x1="6" y1="6" x2="18" y2="18"/>
            </svg>
            Disconnect
          </button>
        </div>

        <div class="actions-full">
          <button class="btn btn-launch" :disabled="!isConnected" @click="startScrcpy">
            <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
              <rect width="20" height="14" x="2" y="3" rx="2"/>
              <line x1="8" x2="16" y1="21" y2="21"/>
              <line x1="12" x2="12" y1="17" y2="21"/>
            </svg>
            Launch Screen
          </button>
        </div>
        
        <button class="btn btn-secondary" :disabled="isLoading" @click="checkDeviceStatus">
          <svg viewBox="0 0 24 24" width="14" height="14" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
            <path d="M21.5 2v6h-6M21.34 15.57a10 10 0 1 1-.57-8.38l5.67-5.67"/>
          </svg>
            Scan Status
        </button>
      </div>

      <!-- TAB 2: PAIRING -->
      <div v-else class="card">
        <div class="alert-info">
          <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="12" r="10"/>
            <line x1="12" y1="16" x2="12" y2="12"/>
            <line x1="12" y1="8" x2="12.01" y2="8"/>
          </svg>
          <span>Access <b>Pair device with pairing code</b> in Android settings.</span>
        </div>
        
        <div class="form-group">
          <label>Device IP Address</label>
          <input v-model="ipAddress" type="text" />
        </div>

        <div class="form-group">
          <label>Pairing Port</label>
          <input v-model="pairPort" type="text" placeholder="e.g. 33595" />
        </div>

        <div class="form-group">
          <label>6-Digit Pairing Code</label>
          <input v-model="pairCode" type="text" placeholder="e.g. 550483" />
        </div>

        <button class="btn btn-pair" :disabled="isLoading" @click="pairDevice">
          <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
            <path d="M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2"/>
            <circle cx="9" cy="7" r="4"/>
            <line x1="19" y1="8" x2="19" y2="14"/>
            <line x1="22" y1="11" x2="16" y2="11"/>
          </svg>
          {{ isLoading ? 'Processing...' : 'Pair Device' }}
        </button>
      </div>

      <!-- Panel Status -->
      <div class="status-box">
        <div class="status-header">
          <div class="status-title">
            <svg viewBox="0 0 24 24" width="14" height="14" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round">
              <polyline points="4 17 10 11 4 5"/>
              <line x1="12" y1="19" x2="20" y2="19"/>
            </svg>
            <span>Terminal Output</span>
          </div>
          <span :class="['badge', isConnected ? 'online' : 'offline']">
            {{ isConnected ? 'CONNECTED' : 'STANDBY' }}
          </span>
        </div>
        <pre>{{ statusLog }}</pre>
      </div>
    </div>
  </div>
</template>

<style>
* { box-sizing: border-box; margin: 0; padding: 0; }
body {
  background: #18181b;
  color: #f4f4f5;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  user-select: none;
}
.container { padding: 24px; max-width: 480px; margin: 0 auto; }
.header-title { display: flex; align-items: center; gap: 10px; color: #3b82f6; }
h2 { font-size: 1.2rem; color: #fafafa; }
.subtitle { font-size: 0.85rem; color: #a1a1aa; margin: 4px 0 16px 32px; }

.tabs { display: flex; gap: 8px; margin-bottom: 12px; }
.tab-btn {
  flex: 1;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 9px;
  background: #27272a;
  border: 1px solid #3f3f46;
  color: #a1a1aa;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.8rem;
  font-weight: 500;
  transition: all 0.2s;
}
.tab-btn.active {
  background: #2563eb;
  border-color: #2563eb;
  color: #ffffff;
}

.card {
  background: #27272a;
  padding: 16px;
  border-radius: 10px;
  border: 1px solid #3f3f46;
  margin-bottom: 16px;
}
.setup-card {
  text-align: center;
}
.setup-card h3 {
  margin-bottom: 12px;
  color: #fafafa;
}
.os-badge {
  display: inline-block;
  background: #3f3f46;
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 0.8rem;
  margin-bottom: 16px;
  color: #e4e4e7;
}
.dep-list {
  list-style: none;
  text-align: left;
  margin-bottom: 16px;
  background: #18181b;
  padding: 12px;
  border-radius: 6px;
  border: 1px solid #3f3f46;
}
.dep-list li {
  display: flex;
  justify-content: space-between;
  padding: 6px 0;
  font-size: 0.85rem;
}
.dep-list li:first-child { border-bottom: 1px solid #27272a; }
.text-green { color: #34d399; display: flex; align-items: center; gap: 4px; }
.text-red { color: #f87171; display: flex; align-items: center; gap: 4px; }

.alert-info {
  display: flex;
  align-items: center;
  gap: 8px;
  background: #1e293b;
  border: 1px solid #334155;
  color: #93c5fd;
  padding: 10px;
  border-radius: 6px;
  font-size: 0.8rem;
  margin-bottom: 14px;
}
.form-group { margin-bottom: 12px; }
label { display: block; font-size: 0.8rem; margin-bottom: 6px; color: #d4d4d8; }
input {
  width: 100%;
  padding: 9px 12px;
  background: #18181b;
  border: 1px solid #3f3f46;
  border-radius: 6px;
  color: #fff;
  font-size: 0.9rem;
}
input:focus { outline: none; border-color: #3b82f6; }

.actions-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
  margin-top: 16px;
  margin-bottom: 10px;
}
.actions-full {
  margin-bottom: 10px;
}
.btn {
  width: 100%;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 10px 14px;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.85rem;
  cursor: pointer;
  transition: all 0.2s;
}
.btn-connect { background: #2563eb; color: #ffffff; }
.btn-disconnect { background: #dc2626; color: #ffffff; }
.btn-launch { background: #059669; color: #ffffff; }
.btn-secondary {
  background: #3f3f46;
  color: #e4e4e7;
  font-size: 0.8rem;
  padding: 8px;
  width: 100%;
}
.btn-pair { background: #7c3aed; color: #ffffff; margin-top: 6px; }
.btn:disabled { opacity: 0.45; cursor: not-allowed; }

.status-box {
  background: #121214;
  border: 1px solid #27272a;
  border-radius: 8px;
  padding: 12px;
  text-align: left;
}
.status-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px; }
.status-title { display: flex; align-items: center; gap: 6px; font-size: 0.75rem; color: #71717a; }
.badge { padding: 2px 8px; border-radius: 4px; font-size: 0.7rem; font-weight: 600; }
.badge.online { background: #064e3b; color: #34d399; }
.badge.offline { background: #27272a; color: #71717a; }
pre {
  font-family: monospace;
  font-size: 0.8rem;
  color: #e4e4e7;
  white-space: pre-wrap;
  word-break: break-all;
}
</style>
