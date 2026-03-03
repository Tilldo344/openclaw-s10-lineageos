# openclaw-s10-lineageos
Personal AI assistant running OpenClaw on a Samsung S10+ with LineageOS, featuring local Llama 3.2 and cloud Kimi K2.5 models, accessible via Telegram and webUI currently.

OpenClaw on Samsung S10+ with LineageOS – Personal AI Assistant
This repository documents the setup of OpenClaw on a Samsung Galaxy S10+ running LineageOS, transforming an old phone into a 24/7 home server for AI assistance.

The project utilizes both cloud LLMs (NVIDIA NIM with Kimi K2.5) and a local model (Llama 3.2 3B via Ollama). Interaction happens through a Telegram bot and a web-based dashboard accessible via SSH tunnel.

A key challenge overcome was installing and configuring the system on a device with a broken display, relying entirely on hardware keys and remote access.

🎯 Project Goals
Repurpose an old Samsung S10+ as a dedicated, low-power AI server

Integrate two model sources:

Cloud: Kimi K2.5 (via NVIDIA NIM API)

Local: Llama 3.2 3B (via Ollama, running on the phone itself)

Enable interaction through a Telegram bot

Provide secure remote access to the OpenClaw dashboard without an unstable VNC setup

🧰 Hardware & Software Overview
Component	Details
Device	Samsung Galaxy S10+ (Exynos), 8GB RAM
Display	Broken (lower portion unresponsive), touch non-functional
Operating System	LineageOS (Android 12+) – Minimal, Google-free
Terminal	Termux (from F-Droid)
Linux Environment	Ubuntu container via proot-distro (optional, for Ollama)
Local LLM	Ollama with model llama3.2:3b
Cloud LLM	NVIDIA NIM API with model moonshotai/kimi-k2.5 (API key required)
Orchestration	OpenClaw version 2026.2.26 (npm installation)
Remote Access	SSH (port 8022) + SSH tunnel for the web dashboard
Occasional GUI	X11 forwarding (using VcXsrv on Windows)
🚀 Installation Guide (Short Version)
This guide assumes basic familiarity with the command line and Android modding. Trivial steps (like opening a terminal) are omitted.

Phase 1: Preparing the Device – Installing LineageOS with a Broken Display
This phase was completed before setting up OpenClaw. The instructions are adapted from the official LineageOS wiki for beyond2lte (S10+ Exynos)  and modified for a device with a non-functional touchscreen.

1.1 Prerequisites
A Windows/Linux/macOS PC with heimdall installed. The LineageOS wiki provides custom-built versions .

The correct LineageOS recovery image (.img) and installation zip for the beyond2lte .

USB debugging enabled in the stock ROM before the screen failed (if possible). If not, the process relies solely on Download Mode and hardware keys.

1.2 Unlocking the Bootloader (Sightless Operation)
Enter Download Mode: With the device powered off, hold Volume Down + Bixby and connect the USB cable to the PC .

Navigate the Warning: The screen will show a warning. Since the touch screen is unresponsive, use the Volume keys to navigate and the Bixby button (or Power button, depending on prompt) to confirm. The exact mapping may require experimentation. The goal is to select the option to unlock the bootloader, which will wipe the device.

Initial Reboot: The device will reboot. It might prompt for a factory reset, which you can confirm using the hardware keys again.

Re-verify OEM Unlocking: After a minimal initial setup (navigating blindly), try to re-enable Developer Options and ensure "OEM Unlock" is still enabled. This step is difficult without a display; success means the bootloader is confirmed unlocked.

1.3 Flashing LineageOS Recovery
Boot into Download Mode Again: Repeat the key combination (Volume Down + Bixby + USB cable) .

Flash Recovery using Heimdall: On your PC, execute the following command in the directory containing recovery.img:

bash
heimdall flash --RECOVERY recovery.img --no-reboot
The --no-reboot flag is crucial .

Immediate Reboot to Recovery (Critical Step!): This is the most timing-sensitive part. After the flash completes, the phone will still show "Downloading...". You must immediately force a reboot into recovery before the system can overwrite the freshly flashed recovery.

Force reboot by holding Volume Down + Power for about 10 seconds until the screen goes black.

As soon as the screen is black, immediately switch to holding Volume Up + Bixby + Power (with the USB cable still connected to the PC) .

If successful, the LineageOS recovery menu will appear. If it boots to the (broken) system or a "System destroyed" message, the timing was off, and you need to repeat from step 1.3.1.

1.4 Sideloading LineageOS
Navigate Recovery with Volume Keys: In the recovery, use the Volume keys to move up/down and the Power button to select.

Wipe Data: Navigate to "Factory Reset" → "Format data / factory reset" and confirm. This is necessary to remove encryption.

Enable ADB Sideload: Go back to the main menu, select "Apply Update" → "Apply from ADB".

Sideload from PC: On your PC, run:

bash
adb -d sideload /path/to/lineage-*.zip
The process will show progress on the PC. It may stall at 47%, which is normal .

Reboot: Once complete, navigate back and select "Reboot system now". The first boot will take several minutes.

At this point, you have a functioning, Google-free LineageOS system, installed without ever using the touchscreen.

Phase 2: Setting Up the Environment (via SSH)
Now that LineageOS is running, all further interaction will happen remotely via SSH from your PC.

Find the Phone's IP: On your router's DHCP client list or using a network scanner, find the IP address assigned to your S10+.

Connect via SSH:

bash
ssh -p 8022 <username>@<phone-ip>
The default Termux username is typically u0_aXXX (check with whoami after first login). You will need to have previously started sshd in Termux and set a password.

Phase 3: OpenClaw Installation & Configuration (via SSH)
3.1 Termux Setup
bash
pkg update && pkg upgrade
pkg install openssh x11-repo xauth   # xauth for X11 forwarding, x11-repo must come first
passwd                               # Set a password for SSH
sshd                                 # Start the SSH server
3.2 Install OpenClaw
The easiest method is to use the Android-specific bootstrap script:

bash
curl -sL https://raw.githubusercontent.com/AidanPark/openclaw-android/main/bootstrap.sh | bash
If you prefer manual installation (or the script fails), you'll need to handle Android-specific paths:

bash
npm install -g openclaw
export TMPDIR=$PREFIX/tmp
mkdir -p $TMPDIR
# You may also need a hijack.js for os.networkInterfaces() – see "Troubleshooting"
3.3 Set Up Ollama (Local Model)
Ollama runs best inside a Linux container:

bash
proot-distro login ubuntu

# Inside the Ubuntu container
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.2:3b

# Start the server, binding to all interfaces so OpenClaw can reach it
export OLLAMA_HOST=0.0.0.0
ollama serve &
The model will be available at http://<phone-ip>:11434.

3.4 Configure NVIDIA NIM API (Cloud Model)
Obtain an API key from NVIDIA NIM.

Verify available models to ensure you have the correct name:

bash
curl -s https://integrate.api.nvidia.com/v1/models | grep -i kimi
The correct identifier for Kimi K2.5 is moonshotai/kimi-k2.5.

3.5 Create a Telegram Bot
Talk to @BotFather on Telegram to create a new bot and get its token.

3.6 Configure OpenClaw
Edit the configuration file at ~/.openclaw/openclaw.json. A working example (adjust IP address and add your API keys/tokens):

json
{
  "meta": { ... },  // Automatically generated, do not modify manually
  "models": {
    "mode": "merge",
    "providers": {
      "nvidia": {
        "baseUrl": "https://integrate.api.nvidia.com/v1",
        "apiKey": "YOUR_NVIDIA_API_KEY",
        "api": "openai-completions",
        "models": [
          {
            "id": "moonshotai/kimi-k2.5",
            "name": "kimi-k2.5",
            "contextWindow": 200000,
            "maxTokens": 1024
          }
        ]
      },
      "ollama": {
        "baseUrl": "http://192.168.0.189:11434/v1",   // Replace with your phone's actual IP
        "apiKey": "dummy",
        "api": "openai-completions",
        "models": [
          {
            "id": "llama3.2:3b",
            "name": "Llama 3.2 3B (local)",
            "contextWindow": 65536,
            "maxTokens": 65536
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "nvidia/moonshotai/kimi-k2.5"
      },
      "models": {
        "nvidia/moonshotai/kimi-k2.5": { "alias": "kimi-k2.5" }
      }
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "YOUR_TELEGRAM_BOT_TOKEN",
      "dmPolicy": "pairing",
      "groupPolicy": "allowlist"
    }
  },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "loopback",        // Listen only locally, we'll use an SSH tunnel for access
    "auth": {
      "mode": "token",
      "token": "__OPENCLAW_REDACTED__" // This will be auto-generated on first start
    }
  },
  "plugins": {
    "entries": { "telegram": { "enabled": true } }
  }
}
Important: After editing, always validate the JSON syntax. A missing comma is the most common error. Retrieve the auto-generated gateway token with: openclaw config get gateway.auth.token

3.7 Start the OpenClaw Gateway
bash
openclaw gateway start &
# For persistent operation (survives SSH session closure):
nohup openclaw gateway start > ~/openclaw.log 2>&1 &
3.8 Access the Dashboard via SSH Tunnel (from your PC)
bash
ssh -N -L 18789:127.0.0.1:18789 -p 8022 username@phone-ip
Then open http://localhost:18789 in your PC's browser and enter the gateway token.

3.9 Link Telegram Bot in the Dashboard
Navigate to Channels → Telegram in the dashboard.

Enter your bot token and save.

Create an agent (e.g., "Kimi") and assign it the primary model (nvidia/moonshotai/kimi-k2.5).

Link the agent to your Telegram channel or for private chats.

🐛 Troubleshooting & Known Issues
Problem	Cause	Solution
OpenClaw fails to start – error os.networkInterfaces()	Android/Termux restricts network interface access	Use the bootstrap script. Alternatively, create a hijack.js file overriding the function and load it via NODE_OPTIONS.
OpenClaw tries to write to /tmp/ (not writable)	Android doesn't have a standard /tmp	export TMPDIR=$PREFIX/tmp and create that directory.
Ollama not reachable by OpenClaw	Ollama binds only to 127.0.0.1 by default	Start Ollama with export OLLAMA_HOST=0.0.0.0.
NVIDIA NIM API returns 403/404 for Kimi K2.5	Incorrect model name or invalid/expired key	Query available models with curl; renew your API key. Test with moonshotai/kimi-k2-instruct as a workaround.
Gateway token not accepted in dashboard	Token not entered or mismatched	Retrieve the token via openclaw config get gateway.auth.token and copy it into the dashboard login prompt.
Telegram bot does not respond	Wrong token, or agent not linked to channel	Verify the token in openclaw.json; in the dashboard, ensure an agent is created and assigned to the Telegram channel.
VNC server unstable / crashes	Android Phantom Process Killer, stale lock files	Abandon VNC. Use SSH tunnel for the dashboard and X11 forwarding for occasional GUI apps .
pkg install xauth fails	x11-repo not activated	Run pkg install x11-repo -y first, then pkg install xauth -y.
SSH "Connection refused"	sshd not running	Start sshd manually. Add it to ~/.bashrc for autostart.
Very slow model responses	maxTokens too high, temperature suboptimal	Lower maxTokens to 1024 or 2048; set temperature to around 0.6 in the agent's settings in the dashboard.
🔧 Quick Command Reference
Purpose	Command
SSH into phone	ssh -p 8022 username@phone-ip
SSH tunnel for dashboard	ssh -N -L 18789:127.0.0.1:18789 -p 8022 username@phone-ip
X11 forwarding (VcXsrv on Windows)	ssh -X -p 8022 username@phone-ip then run firefox &
Start Ollama with external access	export OLLAMA_HOST=0.0.0.0 && ollama serve &
Restart OpenClaw gateway	pkill -f openclaw-gateway && openclaw gateway start &
Read OpenClaw logs	tail -f ~/openclaw.log
Get gateway token	openclaw config get gateway.auth.token
Find your phone's IP (on the phone)	ip addr show or ifconfig
💡 Optional: Free LLM API Alternatives & Hardware Upgrades
Free Cloud LLM APIs (with usage limits)
Provider	Recommended Models	Key Free Tier Limits
Cerebras	llama-3.3-70b	14,400 req/day, 1M tokens/day
Mistral	mistral-large-latest	1B tokens/month, 500k tokens/min
OpenRouter	nvidia/nemotron-3-nano:free	20 req/min, 50 req/day
Google Gemini	gemini-3-flash	250k tokens/min, 20 req/day
Groq	groq/compound	70k tokens/min, 250 req/day

