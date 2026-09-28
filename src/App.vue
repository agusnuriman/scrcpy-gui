<script setup>
import { ref, onMounted, computed } from "vue";
import { Command } from "@tauri-apps/plugin-shell";

const os = ref("linux");
const arch = ref("x86_64");
const isCheckingDeps = ref(true);
const dependenciesReady = ref(false);
const missingDeps = ref([]);
const isInstalling = ref(false);
const installLogs = ref("");
const usingFallback = ref(false);
const scrcpyVersion = ref("");
const isUpdating = ref(false);
const portableBin = ref(localStorage.getItem("scrcpy_portable_bin") || "");

const SCRCPY_REPO_API =
  "https://api.github.com/repos/Genymobile/scrcpy/releases/latest";
// Portable install root. Fixed per platform so the binary and its matching
// scrcpy-server always stay in the same directory.
const PORTABLE_DIR_UNIX = "$HOME/.local/share/scrcpy-gui/portable";

const activeTab = ref("connect");
const ipAddress = ref(localStorage.getItem("scrcpy_ip") || "192.168.1.");
const port = ref(localStorage.getItem("scrcpy_port") || "");
const statusLog = ref("[READY] Awaiting commands...");
const isConnected = ref(false);
const isLoading = ref(false);
const pairPort = ref("");
const pairCode = ref("");

const statusText = computed(() => {
  if (!dependenciesReady.value) return "ACTION REQUIRED";
  if (isConnected.value) return "CONNECTED";
  return "STANDBY";
});

const statusClass = computed(() => {
  if (!dependenciesReady.value) return "status-badge-warning";
  if (isConnected.value) return "status-badge-success";
  return "status-badge-standby";
});

const displayLog = computed(() => {
  return !dependenciesReady.value
    ? installLogs.value || "[INFO] System diagnostic ready..."
    : statusLog.value;
});

function logInstall(msg) {
  if (msg) {
    installLogs.value += msg.trim() + "\n";
  }
}

function detectOS() {
  const ua = navigator.userAgent.toLowerCase();
  if (ua.includes("win")) return "windows";
  if (ua.includes("mac")) return "macos";
  return "linux";
}

function createAdbCommand(args) {
  if (usingFallback.value && os.value === "windows") {
    const psArgs = [
      "-NoProfile",
      "-Command",
      `& "$env:LOCALAPPDATA\\scrcpy-gui\\bin\\adb.exe" ${args.join(" ")}`,
    ];
    return Command.create("run-powershell", psArgs);
  }
  return Command.create("run-adb", args);
}

// Shell scope only allowlists the literal name `scrcpy`, so an absolute path
// (the portable install) has to be launched through bash/powershell instead.
function shellQuote(value) {
  return `'${String(value).replace(/'/g, "'\\''")}'`;
}

function createScrcpyCommand(args) {
  if (portableBin.value && os.value !== "windows") {
    const command = [portableBin.value, ...args].map(shellQuote).join(" ");
    return Command.create("run-bash", ["-c", `exec ${command}`]);
  }
  if (usingFallback.value && os.value === "windows") {
    const psArgs = [
      "-NoProfile",
      "-Command",
      `& "$env:LOCALAPPDATA\\scrcpy-gui\\bin\\scrcpy.exe" ${args.join(" ")}`,
    ];
    return Command.create("run-powershell", psArgs);
  }
  return Command.create("run-scrcpy", args);
}

async function checkDependencies() {
  isCheckingDeps.value = true;
  missingDeps.value = [];
  os.value = detectOS();
  arch.value = await detectArchitecture();

  // Drop a portable install that no longer runs, so the system binary is used.
  if (portableBin.value && os.value !== "windows") {
    const probe = await Command.create("run-bash", [
      "-c",
      `exec ${shellQuote(portableBin.value)} --version`,
    ])
      .execute()
      .catch(() => null);
    if (!probe || probe.code !== 0) {
      logInstall(
        "[WARN] Portable scrcpy is unusable. Falling back to system binary.",
      );
      portableBin.value = "";
      localStorage.removeItem("scrcpy_portable_bin");
    }
  }

  logInstall("[INFO] Checking ADB installation...");

  try {
    const adbCheck = createAdbCommand(["--version"]);
    const adbOut = await adbCheck.execute();
    if (adbOut.code !== 0) {
      missingDeps.value.push("adb");
      logInstall("[FAIL] ADB not found.");
    } else {
      logInstall("[PASS] ADB found.");
    }
  } catch {
    missingDeps.value.push("adb");
    logInstall("[FAIL] ADB not found.");
  }

  logInstall("[INFO] Checking Scrcpy installation...");
  try {
    const scrcpyCheck = createScrcpyCommand(["--version"]);
    const scrcpyOut = await scrcpyCheck.execute();
    if (scrcpyOut.code !== 0) {
      scrcpyVersion.value = "";
      missingDeps.value.push("scrcpy");
      logInstall("[FAIL] Scrcpy not found.");
    } else {
      scrcpyVersion.value = parseScrcpyVersion(scrcpyOut.stdout);
      logInstall(
        scrcpyVersion.value
          ? `[PASS] Scrcpy ${scrcpyVersion.value} found.`
          : "[PASS] Scrcpy found.",
      );
    }
  } catch {
    scrcpyVersion.value = "";
    missingDeps.value.push("scrcpy");
    logInstall("[FAIL] Scrcpy not found.");
  }

  if (
    missingDeps.value.length > 0 &&
    os.value === "windows" &&
    !usingFallback.value
  ) {
    usingFallback.value = true;
    logInstall("[INFO] Checking Windows local fallback path...");
    await checkDependencies();
    return;
  }

  dependenciesReady.value = missingDeps.value.length === 0;
  isCheckingDeps.value = false;
  if (dependenciesReady.value) {
    statusLog.value =
      "[PASS] All dependencies verified.\n[READY] Awaiting commands...";
  }
}

async function installDependencies() {
  isInstalling.value = true;
  installLogs.value = "";
  logInstall(
    `[INFO] Starting installation procedure for ${os.value.toUpperCase()}...`,
  );

  try {
    if (os.value === "linux") {
      logInstall("[INFO] Requesting root privileges to update packages...");
      const cmd1 = Command.create("run-pkexec", ["apt-get", "update"]);
      cmd1.on("stdout", logInstall);
      cmd1.on("stderr", logInstall);
      await cmd1.execute();

      logInstall("[INFO] Installing scrcpy and adb...");
      const cmd2 = Command.create("run-pkexec", [
        "apt-get",
        "install",
        "-y",
        "scrcpy",
        "adb",
      ]);
      cmd2.on("stdout", logInstall);
      cmd2.on("stderr", logInstall);
      await cmd2.execute();
    } else if (os.value === "macos") {
      logInstall("[INFO] Installing via Homebrew...");
      const cmd = Command.create("run-brew", [
        "install",
        "scrcpy",
        "android-platform-tools",
      ]);
      cmd.on("stdout", logInstall);
      cmd.on("stderr", logInstall);
      await cmd.execute();
    } else if (os.value === "windows") {
      logInstall("[INFO] Downloading and extracting latest Scrcpy release...");
      const psScript = `
        $ProgressPreference = 'SilentlyContinue';
        $dir = "$env:LOCALAPPDATA\\scrcpy-gui\\bin";
        New-Item -ItemType Directory -Force -Path $dir | Out-Null;
        $api = Invoke-RestMethod -Uri "https://api.github.com/repos/Genymobile/scrcpy/releases/latest";
        $url = ($api.assets | Where-Object name -match "win64.*zip")[0].browser_download_url;
        $zip = "$env:TEMP\\scrcpy.zip";
        Write-Host "[INFO] Downloading release...";
        Invoke-WebRequest -Uri $url -OutFile $zip;
        Write-Host "[INFO] Extracting payload to $dir ...";
        Expand-Archive -Path $zip -DestinationPath $dir -Force;
        $sub = Get-ChildItem -Directory $dir | Where-Object Name -match "scrcpy";
        if ($sub) { Move-Item -Path "$($sub.FullName)\\*" -Destination $dir -Force; Remove-Item $sub.FullName -Force; }
        Write-Host "[PASS] Installation complete.";
      `;
      const cmd = Command.create("run-powershell", [
        "-NoProfile",
        "-Command",
        psScript,
      ]);
      cmd.on("stdout", logInstall);
      cmd.on("stderr", logInstall);
      await cmd.execute();
      usingFallback.value = true;
    }

    logInstall("[INFO] Installation finished. Re-verifying environment...");
    await checkDependencies();
  } catch (err) {
    logInstall(`[ERR] Installation failed: ${err}`);
  } finally {
    isInstalling.value = false;
  }
}

async function detectArchitecture() {
  try {
    if (os.value === "windows") {
      const out = await Command.create("run-powershell", [
        "-NoProfile",
        "-Command",
        "$env:PROCESSOR_ARCHITECTURE",
      ]).execute();
      const raw = (out.stdout || "").trim().toLowerCase();
      if (raw === "arm64") return "arm64";
      if (raw === "x86") return "x86";
      return "x86_64";
    }
    const out = await Command.create("run-bash", ["-c", "uname -m"]).execute();
    const raw = (out.stdout || "").trim().toLowerCase();
    if (raw === "arm64" || raw === "aarch64") return "arm64";
    return raw || "x86_64";
  } catch {
    return "x86_64";
  }
}

function parseScrcpyVersion(text) {
  const match = /scrcpy\s+(\d+\.\d+(?:\.\d+)?)/i.exec(text || "");
  return match ? match[1] : "";
}

function versionParts(value) {
  return String(value || "0")
    .replace(/^v/, "")
    .split(".")
    .map((part) => parseInt(part, 10) || 0);
}

function isNewerVersion(candidate, current) {
  const next = versionParts(candidate);
  const now = versionParts(current);
  for (let i = 0; i < Math.max(next.length, now.length); i += 1) {
    const a = next[i] || 0;
    const b = now[i] || 0;
    if (a !== b) return a > b;
  }
  return false;
}

async function fetchLatestScrcpyRelease() {
  if (os.value === "windows") {
    const script =
      "$ProgressPreference = 'SilentlyContinue'; " +
      `(Invoke-WebRequest -UseBasicParsing -Uri "${SCRCPY_REPO_API}" ` +
      '-Headers @{ "Accept" = "application/vnd.github+json"; "User-Agent" = "scrcpy-gui" }).Content';
    const out = await Command.create("run-powershell", [
      "-NoProfile",
      "-Command",
      script,
    ]).execute();
    return JSON.parse(out.stdout);
  }
  const script = `curl -fsSL -H "Accept: application/vnd.github+json" -H "User-Agent: scrcpy-gui" ${shellQuote(SCRCPY_REPO_API)}`;
  const out = await Command.create("run-bash", ["-c", script]).execute();
  return JSON.parse(out.stdout);
}

// Returns null when upstream publishes no usable build for this platform, or
// when the asset carries no sha256 digest. Unverified binaries are never run.
function pickScrcpyAsset(release) {
  const tag = String(release.tag_name || "");
  if (!/^v?\d+(\.\d+)*$/.test(tag)) return null;

  let wanted;
  if (os.value === "windows") {
    wanted =
      arch.value === "x86"
        ? `scrcpy-win32-${tag}.zip`
        : `scrcpy-win64-${tag}.zip`;
  } else if (os.value === "macos") {
    wanted = `scrcpy-macos-${arch.value === "arm64" ? "aarch64" : "x86_64"}-${tag}.tar.gz`;
  } else {
    wanted = `scrcpy-linux-${arch.value === "arm64" ? "aarch64" : "x86_64"}-${tag}.tar.gz`;
  }

  const asset = (release.assets || []).find((item) => item.name === wanted);
  if (!asset) return null;

  const digest = String(asset.digest || "");
  const sha256 = digest.startsWith("sha256:")
    ? digest.slice(7).toLowerCase()
    : "";
  if (!/^[0-9a-f]{64}$/.test(sha256)) return null;

  return { tag, name: asset.name, url: asset.browser_download_url, sha256 };
}

function portableUnixScript(target) {
  return [
    "set -eu",
    `BASE=${PORTABLE_DIR_UNIX}`,
    `DEST="$BASE/${target.tag}"`,
    'TMP="$(mktemp -d)"',
    `trap 'rm -rf "$TMP"' EXIT`,
    'mkdir -p "$BASE"',
    `echo "[INFO] Downloading ${target.name}..."`,
    `curl -fsSL --retry 2 --connect-timeout 20 -o "$TMP/payload" ${shellQuote(target.url)}`,
    "if command -v sha256sum >/dev/null 2>&1; then",
    `  GOT="$(sha256sum "$TMP/payload" | cut -d' ' -f1)"`,
    "else",
    `  GOT="$(shasum -a 256 "$TMP/payload" | cut -d' ' -f1)"`,
    "fi",
    `if [ "$GOT" != "${target.sha256}" ]; then`,
    `  echo "[FAIL] sha256 mismatch: expected ${target.sha256}, got $GOT"`,
    "  exit 1",
    "fi",
    'echo "[PASS] sha256 verified."',
    'echo "[INFO] Extracting payload..."',
    'rm -rf "$DEST"',
    'mkdir -p "$DEST"',
    'tar xzf "$TMP/payload" -C "$DEST" --strip-components=1',
    `find "$BASE" -mindepth 1 -maxdepth 1 -type d ! -name ${JSON.stringify(target.tag)} -exec rm -rf {} +`,
    'if [ ! -x "$DEST/scrcpy" ]; then',
    '  echo "[FAIL] scrcpy binary missing after extraction."',
    "  exit 1",
    "fi",
    'echo "[PATH] $DEST/scrcpy"',
  ].join("\n");
}

function portableWindowsScript(target) {
  return `
$ErrorActionPreference = 'Stop';
$ProgressPreference = 'SilentlyContinue';
$dir = "$env:LOCALAPPDATA\\scrcpy-gui\\bin";
$zip = "$env:TEMP\\scrcpy-${target.tag}.zip";
$tmp = "$env:TEMP\\scrcpy-unpack";
New-Item -ItemType Directory -Force -Path $dir | Out-Null;
Write-Host "[INFO] Downloading ${target.name}...";
Invoke-WebRequest -Uri "${target.url}" -OutFile $zip;
$got = (Get-FileHash $zip -Algorithm SHA256).Hash.ToLower();
if ($got -ne "${target.sha256}") {
  Write-Host "[FAIL] sha256 mismatch: expected ${target.sha256}, got $got";
  exit 1;
}
Write-Host "[PASS] sha256 verified.";
Write-Host "[INFO] Extracting payload...";
if (Test-Path $tmp) { Remove-Item $tmp -Recurse -Force; }
Expand-Archive -Path $zip -DestinationPath $tmp -Force;
$sub = Get-ChildItem -Directory $tmp | Select-Object -First 1;
if ($sub) { Copy-Item -Path "$($sub.FullName)\\*" -Destination $dir -Recurse -Force; }
Remove-Item $tmp -Recurse -Force;
Remove-Item $zip -Force;
Write-Host "[PATH] $dir\\scrcpy.exe";
`;
}

async function installPortableScrcpy(target) {
  const script =
    os.value === "windows"
      ? portableWindowsScript(target)
      : portableUnixScript(target);
  const cmd =
    os.value === "windows"
      ? Command.create("run-powershell", ["-NoProfile", "-Command", script])
      : Command.create("run-bash", ["-c", script]);

  cmd.on("stdout", (line) => {
    statusLog.value += line;
  });
  cmd.on("stderr", (line) => {
    statusLog.value += line;
  });

  const out = await cmd.execute();
  if (out.code !== 0) return false;

  const path = /\[PATH\]\s*(.+)/.exec(out.stdout || "");
  if (!path) return false;

  if (os.value === "windows") {
    usingFallback.value = true;
  } else {
    portableBin.value = path[1].trim();
  }
  localStorage.setItem("scrcpy_portable_bin", portableBin.value);
  return true;
}

async function updateViaPackageManager() {
  if (os.value === "linux") {
    logInstall("[INFO] Requesting root privileges for apt upgrade...");
    const refresh = Command.create("run-pkexec", ["apt-get", "update"]);
    refresh.on("stdout", logInstall);
    refresh.on("stderr", logInstall);
    await refresh.execute();

    const upgrade = Command.create("run-pkexec", [
      "apt-get",
      "install",
      "-y",
      "--only-upgrade",
      "scrcpy",
      "adb",
    ]);
    upgrade.on("stdout", logInstall);
    upgrade.on("stderr", logInstall);
    const out = await upgrade.execute();
    return out.code === 0;
  }

  if (os.value === "macos") {
    logInstall("[INFO] Upgrading via Homebrew...");
    const cmd = Command.create("run-brew", ["upgrade", "scrcpy"]);
    cmd.on("stdout", logInstall);
    cmd.on("stderr", logInstall);
    const out = await cmd.execute();
    return out.code === 0;
  }

  return false;
}

async function checkScrcpyUpdate() {
  if (isUpdating.value) return;

  isUpdating.value = true;
  statusLog.value = "[INFO] Querying upstream scrcpy release feed...";
  installLogs.value = "";

  try {
    const release = await fetchLatestScrcpyRelease();
    const latest = String(release.tag_name || "").replace(/^v/, "");
    statusLog.value += `\n[INFO] Installed: ${scrcpyVersion.value || "unknown"} | Upstream: ${latest}`;

    if (!isNewerVersion(latest, scrcpyVersion.value)) {
      statusLog.value += "\n[PASS] Already running the newest release.";
      return;
    }

    const target = pickScrcpyAsset(release);
    if (!target) {
      statusLog.value += `\n[WARN] Upstream has no verified build for ${os.value}/${arch.value}. Using the system package manager instead.`;
      installLogs.value = "";
      const ok = await updateViaPackageManager();
      statusLog.value += installLogs.value;
      statusLog.value += ok
        ? "\n[PASS] System package manager finished."
        : "\n[FAIL] System package manager reported an error.";
      return;
    }

    statusLog.value += `\n[INFO] Installing scrcpy ${target.tag} in portable mode...`;
    const ok = await installPortableScrcpy(target);

    if (!ok) {
      statusLog.value +=
        "\n[WARN] Portable install failed. Falling back to the system package manager.";
      installLogs.value = "";
      const fallback = await updateViaPackageManager();
      statusLog.value += installLogs.value;
      statusLog.value += fallback
        ? "\n[PASS] System package manager finished."
        : "\n[FAIL] System package manager reported an error.";
    } else {
      statusLog.value += `\n[PASS] scrcpy ${target.tag} is now active.`;
    }

    await checkDependencies();
  } catch (err) {
    statusLog.value += `\n[ERR] Update check failed: ${err}`;
  } finally {
    isUpdating.value = false;
  }
}

function savePreferences() {
  localStorage.setItem("scrcpy_ip", ipAddress.value);
  localStorage.setItem("scrcpy_port", port.value);
}

async function checkDeviceStatus() {
  isLoading.value = true;
  statusLog.value = "[INFO] Scanning active ADB connections...";
  try {
    const cmd = createAdbCommand(["devices"]);
    const output = await cmd.execute();
    const result = output.stdout || output.stderr;
    statusLog.value = result;

    const target = `${ipAddress.value}:${port.value}`;
    if (result.includes(target) && result.includes("device")) {
      isConnected.value = true;
      statusLog.value += `\n[PASS] Target ${target} verified online.`;
    } else {
      isConnected.value = false;
    }
  } catch (err) {
    statusLog.value = `[ERR] Failed to read ADB status: ${err}`;
  } finally {
    isLoading.value = false;
  }
}

async function pairDevice() {
  if (!pairPort.value || !pairCode.value) {
    statusLog.value = "[WARN] Pairing Port and 6-Digit Code are required.";
    return;
  }

  isLoading.value = true;
  statusLog.value = `[INFO] Initiating pairing protocol with ${ipAddress.value}:${pairPort.value}...`;

  try {
    const target = `${ipAddress.value}:${pairPort.value}`;
    const cmd = createAdbCommand(["pair", target, pairCode.value]);
    const output = await cmd.execute();
    const result = output.stdout || output.stderr;
    statusLog.value = result;

    if (result.includes("Successfully paired")) {
      statusLog.value +=
        "\n[PASS] Device pairing authorized. Redirecting to Connect tab.";
      savePreferences();
      activeTab.value = "connect";
    }
  } catch (err) {
    statusLog.value = `[ERR] Pairing protocol failed: ${err}`;
  } finally {
    isLoading.value = false;
  }
}

async function connectADB() {
  if (!port.value) {
    statusLog.value = "[WARN] Active target port required.";
    return;
  }

  savePreferences();
  isLoading.value = true;
  statusLog.value = `[INFO] Establishing ADB TCP connection to ${ipAddress.value}:${port.value}...`;

  try {
    const target = `${ipAddress.value}:${port.value}`;
    const cmd = createAdbCommand(["connect", target]);
    const output = await cmd.execute();
    const result = output.stdout || output.stderr;
    statusLog.value = result;

    if (result.includes("connected to")) {
      isConnected.value = true;
    } else {
      isConnected.value = false;
    }
  } catch (err) {
    statusLog.value = `[ERR] Execution failed: ${err}`;
    isConnected.value = false;
  } finally {
    isLoading.value = false;
  }
}

async function disconnectADB() {
  savePreferences();
  isLoading.value = true;

  const target = `${ipAddress.value}:${port.value}`;
  statusLog.value = `[INFO] Terminating connection to ${target}...`;

  try {
    const cmd = createAdbCommand(["disconnect", target]);
    const output = await cmd.execute();
    const result = output.stdout || output.stderr;
    statusLog.value = result;
    isConnected.value = false;
    statusLog.value += "\n[PASS] Connection terminated safely.";
  } catch (err) {
    statusLog.value = `[ERR] Termination failed: ${err}`;
  } finally {
    isLoading.value = false;
  }
}

async function startScrcpy() {
  statusLog.value = "[INFO] Booting Scrcpy runtime engine...";
  try {
    const cmd = createScrcpyCommand([
      "-m",
      "1024",
      "-b",
      "4M",
      "--no-audio",
      "--max-fps",
      "60",
    ]);
    await cmd.spawn();
    statusLog.value = "[PASS] Scrcpy engine deployed successfully.";
  } catch (err) {
    statusLog.value = `[ERR] Deployment fault: ${err}`;
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
  <div class="app-root">
    <!-- Window Chrome (Title Bar) -->
    <div class="titlebar" data-tauri-drag-region>
      <div class="window-controls">
        <div class="control-btn close"></div>
        <div class="control-btn minimize"></div>
        <div class="control-btn maximize"></div>
      </div>
      <div class="window-title" data-tauri-drag-region>scrcpy-gui</div>
      <div class="window-platform">{{ os }} / {{ arch }}</div>
    </div>

    <div class="container">
      <!-- App Header -->
      <div class="app-header">
        <div class="header-left">
          <div class="app-icon">
            <svg
              viewBox="0 0 24 24"
              width="20"
              height="20"
              stroke="currentColor"
              stroke-width="2"
              fill="none"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <rect width="14" height="20" x="5" y="2" rx="2" ry="2" />
              <path d="M12 18h.01" />
            </svg>
          </div>
          <div>
            <h2 class="app-title">Scrcpy Wireless Manager</h2>
            <p class="app-subtitle">
              Wireless Android device controller via ADB & Scrcpy
            </p>
          </div>
        </div>
        <div class="header-right">
          <div class="status-badge" :class="statusClass">
            <span class="pulse-dot"></span>
            {{ statusText }}
          </div>
        </div>
      </div>

      <!-- DEPENDENCY SETUP WIZARD -->
      <div v-if="isCheckingDeps" class="panel">
        <div class="section-header">SYSTEM DEPENDENCIES CHECK</div>
        <p class="text-secondary text-sm mt-2">
          Scanning runtime environment...
        </p>
      </div>

      <div v-else-if="!dependenciesReady" class="panel">
        <div class="section-header">ACTION REQUIRED</div>
        <div class="dep-list mt-2">
          <div class="dep-item">
            <div class="dep-info">
              <span class="dep-name">ADB binary path</span>
              <span class="dep-desc">Android Debug Bridge protocol</span>
            </div>
            <div
              class="dep-status"
              :class="
                missingDeps.includes('adb') ? 'status-missing' : 'status-found'
              "
            >
              <svg
                v-if="missingDeps.includes('adb')"
                viewBox="0 0 24 24"
                width="14"
                height="14"
                stroke="currentColor"
                stroke-width="2"
                fill="none"
              >
                <line x1="18" y1="6" x2="6" y2="18" />
                <line x1="6" y1="6" x2="18" y2="18" />
              </svg>
              <svg
                v-else
                viewBox="0 0 24 24"
                width="14"
                height="14"
                stroke="currentColor"
                stroke-width="2"
                fill="none"
              >
                <polyline points="20 6 9 17 4 12" />
              </svg>
              {{ missingDeps.includes("adb") ? "Missing" : "Found" }}
            </div>
          </div>
          <div class="dep-item">
            <div class="dep-info">
              <span class="dep-name">Scrcpy engine</span>
              <span class="dep-desc">Screen copy runtime utility</span>
            </div>
            <div
              class="dep-status"
              :class="
                missingDeps.includes('scrcpy')
                  ? 'status-missing'
                  : 'status-found'
              "
            >
              <svg
                v-if="missingDeps.includes('scrcpy')"
                viewBox="0 0 24 24"
                width="14"
                height="14"
                stroke="currentColor"
                stroke-width="2"
                fill="none"
              >
                <line x1="18" y1="6" x2="6" y2="18" />
                <line x1="6" y1="6" x2="18" y2="18" />
              </svg>
              <svg
                v-else
                viewBox="0 0 24 24"
                width="14"
                height="14"
                stroke="currentColor"
                stroke-width="2"
                fill="none"
              >
                <polyline points="20 6 9 17 4 12" />
              </svg>
              {{ missingDeps.includes("scrcpy") ? "Missing" : "Found" }}
            </div>
          </div>
        </div>

        <button
          class="btn btn-primary mt-4"
          :disabled="isInstalling"
          @click="installDependencies"
        >
          <svg
            viewBox="0 0 24 24"
            width="16"
            height="16"
            stroke="currentColor"
            stroke-width="2"
            fill="none"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
            <polyline points="7 10 12 15 17 10" />
            <line x1="12" y1="15" x2="12" y2="3" />
          </svg>
          {{
            isInstalling
              ? "Executing Protocol..."
              : "Install & Setup Scrcpy Engine"
          }}
        </button>
        <button class="btn btn-neutral mt-2">Locate Manually...</button>
      </div>

      <!-- MAIN DASHBOARD -->
      <div v-else class="panel">
        <div class="tab-switcher">
          <button
            class="tab-item"
            :class="{ active: activeTab === 'connect' }"
            @click="activeTab = 'connect'"
          >
            Connect (Daily)
          </button>
          <button
            class="tab-item"
            :class="{ active: activeTab === 'pair' }"
            @click="activeTab = 'pair'"
          >
            Pair Device (1x)
          </button>
        </div>

        <!-- Connect Tab -->
        <div v-if="activeTab === 'connect'">
          <div class="form-group">
            <label class="section-header">DEVICE IP ADDRESS</label>
            <div class="input-wrapper">
              <svg
                class="input-icon"
                viewBox="0 0 24 24"
                width="16"
                height="16"
                stroke="currentColor"
                stroke-width="2"
                fill="none"
                stroke-linecap="round"
                stroke-linejoin="round"
              >
                <circle cx="12" cy="12" r="10" />
                <line x1="2" y1="12" x2="22" y2="12" />
                <path
                  d="M12 2a15.3 15.3 0 0 1 4 10 15.3 15.3 0 0 1-4 10 15.3 15.3 0 0 1-4-10 15.3 15.3 0 0 1 4-10z"
                />
              </svg>
              <input
                v-model="ipAddress"
                type="text"
                placeholder="192.168.1.129"
                @change="savePreferences"
              />
            </div>
          </div>
          <div class="form-group">
            <label class="section-header">ACTIVE WIRELESS PORT</label>
            <div class="input-wrapper">
              <svg
                class="input-icon"
                viewBox="0 0 24 24"
                width="16"
                height="16"
                stroke="currentColor"
                stroke-width="2"
                fill="none"
                stroke-linecap="round"
                stroke-linejoin="round"
              >
                <polygon points="13 2 3 14 12 14 11 22 21 10 12 10 13 2" />
              </svg>
              <input
                v-model="port"
                type="text"
                placeholder="41809"
                @change="savePreferences"
              />
            </div>
          </div>

          <div class="grid-2 mt-4">
            <button
              class="btn btn-primary"
              :disabled="isLoading"
              @click="connectADB"
            >
              <svg
                viewBox="0 0 24 24"
                width="16"
                height="16"
                stroke="currentColor"
                stroke-width="2"
                fill="none"
                stroke-linecap="round"
                stroke-linejoin="round"
              >
                <path
                  d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"
                />
                <path
                  d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"
                />
              </svg>
              Connect
            </button>
            <button
              class="btn btn-danger"
              :disabled="isLoading"
              @click="disconnectADB"
            >
              <svg
                viewBox="0 0 24 24"
                width="16"
                height="16"
                stroke="currentColor"
                stroke-width="2"
                fill="none"
                stroke-linecap="round"
                stroke-linejoin="round"
              >
                <line x1="18" y1="6" x2="6" y2="18" />
                <line x1="6" y1="6" x2="18" y2="18" />
              </svg>
              Disconnect
            </button>
          </div>
          <button
            class="btn btn-success mt-2"
            :disabled="!isConnected"
            @click="startScrcpy"
          >
            <svg
              viewBox="0 0 24 24"
              width="16"
              height="16"
              stroke="currentColor"
              stroke-width="2"
              fill="none"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <rect width="20" height="14" x="2" y="3" rx="2" />
              <line x1="8" x2="16" y1="21" y2="21" />
              <line x1="12" x2="12" y1="17" y2="21" />
            </svg>
            Launch Screen
          </button>
          <button
            class="btn btn-neutral mt-2"
            :disabled="isLoading"
            @click="checkDeviceStatus"
          >
            <svg
              viewBox="0 0 24 24"
              width="14"
              height="14"
              stroke="currentColor"
              stroke-width="2"
              fill="none"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <path
                d="M21.5 2v6h-6M21.34 15.57a10 10 0 1 1-.57-8.38l5.67-5.67"
              />
            </svg>
            Scan Status
          </button>
        </div>

        <!-- Pair Tab -->
        <div v-else>
          <div class="form-group">
            <label class="section-header">DEVICE IP ADDRESS</label>
            <div class="input-wrapper">
              <input v-model="ipAddress" type="text" />
            </div>
          </div>
          <div class="form-group">
            <label class="section-header">PAIRING PORT</label>
            <div class="input-wrapper">
              <input v-model="pairPort" type="text" placeholder="e.g. 33595" />
            </div>
          </div>
          <div class="form-group">
            <label class="section-header">6-DIGIT PAIRING CODE</label>
            <div class="input-wrapper">
              <input v-model="pairCode" type="text" placeholder="e.g. 550483" />
            </div>
          </div>
          <button
            class="btn btn-primary mt-4"
            :disabled="isLoading"
            @click="pairDevice"
          >
            <svg
              viewBox="0 0 24 24"
              width="16"
              height="16"
              stroke="currentColor"
              stroke-width="2"
              fill="none"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <path d="M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2" />
              <circle cx="9" cy="7" r="4" />
              <line x1="19" y1="8" x2="19" y2="14" />
              <line x1="22" y1="11" x2="16" y2="11" />
            </svg>
            Pair Device
          </button>
        </div>

        <!-- Engine Updates -->
        <div class="engine-row mt-4">
          <div class="engine-info">
            <span class="section-header">SCRCPY ENGINE</span>
            <span class="engine-value">
              {{ scrcpyVersion ? "v" + scrcpyVersion : "version unknown" }}
            </span>
          </div>
          <button
            class="btn btn-neutral btn-inline"
            :disabled="isUpdating"
            @click="checkScrcpyUpdate"
          >
            <svg
              viewBox="0 0 24 24"
              width="14"
              height="14"
              stroke="currentColor"
              stroke-width="2"
              fill="none"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
              <polyline points="7 10 12 15 17 10" />
              <line x1="12" y1="15" x2="12" y2="3" />
            </svg>
            {{ isUpdating ? "Checking..." : "Check for updates" }}
          </button>
        </div>
      </div>

      <!-- TERMINAL CONSOLE -->
      <div class="terminal-console mt-4">
        <div class="terminal-header">
          <span>>_ Terminal Output</span>
        </div>
        <div class="terminal-body">
          <pre>{{ displayLog }}<span class="cursor"></span></pre>
        </div>
      </div>

      <!-- FOOTER -->
      <div class="app-footer mt-4">
        <div>Tauri v2 • Native IPC Ready</div>
        <div>Arch: {{ arch }} • scrcpy {{ scrcpyVersion || "unknown" }}</div>
      </div>
    </div>
  </div>
</template>

<style>
/* CSS Reset & Variables */
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

:root {
  --bg-canvas: #0b0b0d;
  --bg-surface: #18181b;
  --bg-surface-high: #232328;
  --bg-terminal: #0a0a0c;
  --border-subtle: #27272a;
  --border-focus: #3b82f6;
  --text-primary: #f4f4f5;
  --text-secondary: #a1a1aa;
  --text-muted: #71717a;
  --text-mono: #e4e4e7;
  --accent-primary: #2563eb;
  --accent-primary-hover: #1d4ed8;
  --accent-success: #059669;
  --accent-success-hover: #047857;
  --accent-danger: #dc2626;
  --accent-danger-hover: #b91c1c;
  --accent-warning: #d97706;
  --btn-neutral: #1e1e24;
  --btn-neutral-hover: #27272a;
  --font-sans: "Geist Sans", system-ui, -apple-system, sans-serif;
  --font-mono: "JetBrains Mono", "Geist Mono", ui-monospace, monospace;
}

body {
  background: var(--bg-canvas);
  color: var(--text-primary);
  font-family: var(--font-sans);
  user-select: none;
  font-size: 13px;
  line-height: 18px;
}

.app-root {
  display: flex;
  flex-direction: column;
  height: 100vh;
}

/* Utilities */
.mt-2 {
  margin-top: 8px;
}
.mt-4 {
  margin-top: 16px;
}
.text-sm {
  font-size: 11px;
}
.text-secondary {
  color: var(--text-secondary);
}

/* Window Chrome */
.titlebar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: #121214;
  border-bottom: 1px solid var(--border-subtle);
  -webkit-app-region: drag;
}
.window-controls {
  display: flex;
  gap: 6px;
}
.control-btn {
  width: 12px;
  height: 12px;
  border-radius: 50%;
}
.close {
  background: #ef4444;
}
.minimize {
  background: #f59e0b;
}
.maximize {
  background: #10b981;
}
.window-title {
  font-family: var(--font-mono);
  font-size: 12px;
  color: var(--text-secondary);
  font-weight: 500;
  letter-spacing: 0.05em;
}
.window-platform {
  background: rgba(39, 39, 42, 0.8);
  border: 1px solid rgba(63, 63, 70, 0.6);
  color: #d4d4d8;
  border-radius: 9999px;
  padding: 2px 10px;
  font-size: 11px;
}

/* Layout */
.container {
  padding: 16px;
  max-width: 520px;
  margin: 0 auto;
  width: 100%;
}

/* Header */
.app-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 24px;
}
.header-left {
  display: flex;
  gap: 12px;
}
.app-icon {
  width: 36px;
  height: 36px;
  border-radius: 8px;
  background: rgba(37, 99, 235, 0.1);
  border: 1px solid rgba(59, 130, 246, 0.2);
  color: #60a5fa;
  display: flex;
  align-items: center;
  justify-content: center;
}
.app-title {
  font-size: 16px;
  font-weight: 600;
  line-height: 22px;
}
.app-subtitle {
  font-size: 11px;
  color: var(--text-secondary);
  margin-top: 2px;
}

/* Status Badge */
.status-badge {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 4px 10px;
  border-radius: 9999px;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.02em;
  border: 1px solid transparent;
}
.pulse-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  animation: pulse 2s infinite;
}
@keyframes pulse {
  0% {
    opacity: 1;
  }
  50% {
    opacity: 0.4;
  }
  100% {
    opacity: 1;
  }
}
.status-badge-standby {
  background: rgba(245, 158, 11, 0.1);
  border-color: rgba(245, 158, 11, 0.2);
  color: #fbbf24;
}
.status-badge-standby .pulse-dot {
  background: #fbbf24;
}
.status-badge-warning {
  background: rgba(245, 158, 11, 0.15);
  border-color: rgba(245, 158, 11, 0.3);
  color: #f59e0b;
}
.status-badge-warning .pulse-dot {
  background: #f59e0b;
}
.status-badge-success {
  background: rgba(16, 185, 129, 0.1);
  border-color: rgba(16, 185, 129, 0.2);
  color: #34d399;
}
.status-badge-success .pulse-dot {
  background: #34d399;
}

/* Panel / Cards */
.panel {
  background: var(--bg-surface);
  border: 1px solid var(--border-subtle);
  border-radius: 12px;
  padding: 16px;
}
.section-header {
  font-size: 11px;
  font-weight: 700;
  color: var(--text-secondary);
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 8px;
}

/* Segmented Tabs */
.tab-switcher {
  background: var(--bg-surface-high);
  padding: 4px;
  border-radius: 8px;
  border: 1px solid var(--border-subtle);
  display: flex;
  gap: 4px;
  margin-bottom: 16px;
}
.tab-item {
  flex: 1;
  background: transparent;
  border: none;
  color: var(--text-secondary);
  padding: 6px 12px;
  border-radius: 6px;
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}
.tab-item:hover {
  background: rgba(39, 39, 42, 0.4);
  color: #e4e4e7;
}
.tab-item.active {
  background: var(--accent-primary);
  color: #ffffff;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

/* Dependency List */
.dep-list {
  background: var(--bg-canvas);
  border: 1px solid var(--border-subtle);
  border-radius: 8px;
}
.dep-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 12px;
}
.dep-item:first-child {
  border-bottom: 1px solid var(--border-subtle);
}
.dep-info {
  display: flex;
  flex-direction: column;
}
.dep-name {
  font-weight: 500;
  color: var(--text-primary);
}
.dep-desc {
  font-size: 11px;
  color: var(--text-muted);
}
.dep-status {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  font-weight: 500;
}
.status-missing {
  color: #f87171;
}
.status-found {
  color: #34d399;
}

/* Forms */
.form-group {
  margin-bottom: 12px;
}
.input-wrapper {
  position: relative;
  display: flex;
  align-items: center;
}
.input-icon {
  position: absolute;
  left: 12px;
  color: var(--text-muted);
}
input {
  width: 100%;
  background: var(--bg-surface-high);
  border: 1px solid var(--border-subtle);
  border-radius: 8px;
  padding: 8px 12px 8px 36px;
  color: var(--text-primary);
  font-family: var(--font-mono);
  font-size: 13px;
  transition: all 0.2s;
}
input:focus {
  outline: none;
  border-color: var(--border-focus);
  box-shadow: 0 0 0 1px var(--border-focus);
}

/* Buttons */
.grid-2 {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
}
.btn {
  width: 100%;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 8px 16px;
  border-radius: 8px;
  font-size: 13px;
  font-weight: 500;
  border: none;
  cursor: pointer;
  transition: all 0.15s ease-in-out;
}
.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
.btn-primary {
  background: var(--accent-primary);
  color: #ffffff;
}
.btn-primary:hover:not(:disabled) {
  background: var(--accent-primary-hover);
}
.btn-danger {
  background: rgba(220, 38, 38, 0.1);
  color: #f87171;
  border: 1px solid rgba(220, 38, 38, 0.3);
}
.btn-danger:hover:not(:disabled) {
  background: var(--accent-danger);
  color: #ffffff;
}
.btn-success {
  background: var(--accent-success);
  color: #ffffff;
}
.btn-success:hover:not(:disabled) {
  background: var(--accent-success-hover);
  box-shadow: 0 0 12px -2px rgba(16, 185, 129, 0.25);
}
.btn-neutral {
  background: var(--btn-neutral);
  color: #d4d4d8;
  border: 1px solid rgba(63, 63, 70, 0.5);
}
.btn-neutral:hover:not(:disabled) {
  background: var(--btn-neutral-hover);
}

/* Engine update row */
.engine-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  border-top: 1px solid var(--border-subtle);
  padding-top: 12px;
}
.engine-info {
  display: flex;
  flex-direction: column;
}
.engine-info .section-header {
  margin-bottom: 2px;
}
.engine-value {
  font-family: var(--font-mono);
  font-size: 12px;
  color: var(--text-primary);
}
.btn-inline {
  width: auto;
  flex-shrink: 0;
}

/* Terminal Console */
.terminal-console {
  background: var(--bg-terminal);
  border: 1px solid rgba(39, 39, 42, 0.8);
  border-radius: 8px;
  overflow: hidden;
}
.terminal-header {
  padding: 6px 12px;
  background: rgba(24, 24, 27, 0.5);
  border-bottom: 1px solid rgba(39, 39, 42, 0.8);
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-family: var(--font-mono);
  font-size: 11px;
  color: var(--text-muted);
}
.terminal-body {
  padding: 12px;
  min-height: 80px;
  max-height: 160px;
  overflow-y: auto;
}
.terminal-body pre {
  font-family: var(--font-mono);
  font-size: 11px;
  line-height: 16px;
  color: var(--text-mono);
  white-space: pre-wrap;
  word-break: break-all;
}
.cursor {
  display: inline-block;
  width: 8px;
  height: 12px;
  background: #34d399;
  margin-left: 4px;
  animation: blink 1s step-end infinite;
  vertical-align: text-bottom;
}
@keyframes blink {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0;
  }
}

/* Footer */
.app-footer {
  display: flex;
  justify-content: space-between;
  font-family: var(--font-mono);
  font-size: 11px;
  color: var(--text-muted);
  padding: 0 4px;
}
</style>
