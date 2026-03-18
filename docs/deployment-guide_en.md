# WallWhisper Deployment Guide

WallWhisper supports three deployment methods. Choose based on your devices and needs.

## Prerequisites

### 1. Obtain API Keys

| Service | Purpose | Where to Get |
|---------|---------|--------------|
| DeepSeek API | AI content generation | https://platform.deepseek.com |
| Tencent Cloud TTS | Text-to-speech | https://console.cloud.tencent.com/tts |
| EZVIZ Open Platform | Camera alerts | https://open.ys7.com |
| OpenClaw (optional) | Emily persona + memory | https://openclaw.com |

### 2. Prepare Your EZVIZ Camera

1. Add the camera in the EZVIZ Cloud Video app
2. Enable "Activity Detection Alert" (detect people or animal activity)
3. Note down the device serial number (on the device label)
4. Note down the device verification code (if you want to use the camera speaker feature)
5. Find the camera's LAN IP (check your router admin page)

## Method 1: Local Run (Development/Testing)

Best for development and testing on PC/Mac, using local speakers for playback.

### Requirements
- Python 3.9+
- Microphone/Speaker (optional, for local playback)
- Same LAN as the camera

### Steps

```bash
# 1. Clone the project
git clone https://github.com/treychen-369/WallWhisper.git
cd WallWhisper

# 2. Create virtual environment (recommended)
python -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate   # Windows

# 3. Install dependencies
pip install -r requirements.txt

# 4. Create config file
cp config.example.yaml config.yaml

# 5. Edit config.yaml, at minimum fill in:
#    - ai.api_key (DeepSeek)
#    - tts.app_id / tts.secret_id / tts.secret_key (Tencent Cloud)
#    - ezviz.app_key / ezviz.app_secret / ezviz.device_serial (EZVIZ)

# 6. Test step by step
python run.py test_tts                    # Test TTS
python run.py test_tts "Hello! Can you say cat?"  # Test custom text
python run.py test_ezviz                   # Test EZVIZ connection
python run.py test_speaker                 # Test camera speaker
python run.py test_full                    # End-to-end test

# 7. Start WallWhisper
python run.py emily
```

### FAQ

**Q: PyAudio installation fails?**
```bash
# Windows
pip install pipwin
pipwin install pyaudio

# Mac
brew install portaudio
pip install pyaudio

# Linux
sudo apt-get install portaudio19-dev
pip install pyaudio
```

**Q: No microphone/speaker?**

Enable `camera_speaker` mode to push audio directly to the camera speaker:
```yaml
camera_speaker:
  enabled: true
  cam_ip: "YOUR_CAMERA_IP"
  cam_password: "YOUR_CAMERA_PASSWORD"
```

## Method 2: Docker Deployment (Recommended)

Best for long-term running on a server, NAS, or router.

### Requirements
- Docker 20+
- docker-compose (optional)
- Same LAN as the camera

### Steps

```bash
# 1. Clone the project
git clone https://github.com/treychen-369/WallWhisper.git
cd WallWhisper

# 2. Prepare config file
cp config.example.yaml config.docker.yaml
# Edit config.docker.yaml, note:
#   - camera_speaker.ffmpeg_path should be "ffmpeg" (included in Docker image)
#   - camera_speaker.enabled should be true (no local speaker in Docker)

# 3. Build and start locally
docker-compose up -d --build

# 4. View logs
docker-compose logs -f wallwhisper

# 5. Stop
docker-compose down
```

### Using Pre-built Images

If you've built an image using CNB.cool or another CI system:

```bash
IMAGE=your-registry/wallwhisper:latest docker-compose up -d
```

### Resource Limits

`docker-compose.yml` includes resource limits suitable for routers:

| Parameter | Value | Description |
|-----------|-------|-------------|
| `mem_limit` | 128MB | Hard memory limit |
| `mem_reservation` | 64MB | Soft memory limit |
| `cpus` | 0.5 | CPU limit |
| `pids_limit` | 32 | Process count limit |
| `oom_score_adj` | 500 | OOM priority (killed first to protect host) |

> If your device has plenty of memory (e.g., NAS/server), feel free to increase these limits.

## Method 3: Xiaomi Router Deployment (For Geeks 🔧)

Run WallWhisper directly on a Docker-capable Xiaomi router (e.g., 7000/BE series).

### Prerequisites

1. Router has Docker installed (refer to community tutorials)
2. USB storage device available (Docker images and data stored on USB)
3. Router has internet access (to pull images and call APIs)

### Steps

```bash
# 1. Create WallWhisper directory on the router
ssh router
mkdir -p /opt/emily
mkdir -p /opt/emily/ezviz_token
mkdir -p /opt/emily/logs

# 2. Upload config file (from local PC)
# Use sync_router.py or manual scp
scp config.docker.yaml router:/opt/emily/

# 3. Upload deploy script
scp deploy.sh router:/opt/emily/

# 4. Set environment variables (adjust to your actual paths)
export EMILY_IMAGE="your-registry/wallwhisper:latest"
export EMILY_DIR="/opt/emily"

# 5. Execute deployment
ssh router sh /opt/emily/deploy.sh

# 6. View logs
ssh router docker logs --tail 30 wallwhisper
```

### deploy.sh Safety Features

The deploy script includes a 7-step safety process:

1. **Pre-check** — Check memory (<100MB refuses to deploy) and config files
2. **Snapshot** — Save current image ID and config backup
3. **Pull** — Download new image
4. **Stop** — Stop old container
5. **Start** — Start new container (with resource limits)
6. **Health Check** — Container status + log check + memory check
7. **Record** — Save successful deployment status

```bash
# Common commands
sh deploy.sh              # Normal deployment
sh deploy.sh --force      # Force re-deployment
sh deploy.sh --rollback   # Rollback to previous version
```

### Using sync_router.py for Management

Run `sync_router.py` on your local PC to safely manage WallWhisper on the router:

```bash
# Check router Emily status
python sync_router.py status

# Sync config file (with validation + backup + atomic replacement)
python sync_router.py config

# View config diff
python sync_router.py config --diff

# Sync and restart
python sync_router.py config --restart
```

## Verify Deployment

Regardless of deployment method, verify with:

1. **Check logs**: Confirm WallWhisper started successfully and is polling for alerts
2. **Walk past the camera**: Trigger human detection, wait for Emily to speak English
3. **Check resources**: Confirm memory and CPU usage are normal

```bash
# Docker logs
docker logs --tail 30 wallwhisper

# Resource monitoring
docker stats wallwhisper
```
