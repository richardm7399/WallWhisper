# WallWhisper System Architecture

## Overall Architecture

WallWhisper uses an **event-driven + pipeline** architecture. Core flow:

```
Camera human detection → Alert polling → Scene matching → AI content generation → TTS synthesis → Audio playback
```

## Module Breakdown

### 1. Perception Layer: `ezviz_monitor.py`

**Role**: Poll EZVIZ Cloud API for human detection alerts

```
EZVIZ Camera → EZVIZ Cloud (open.ys7.com) → ezviz_monitor.py polling
```

- Uses EZVIZ Open Platform REST API, polling alert list every 5 seconds
- Automatic AccessToken lifecycle management (auto-refresh on expiry)
- Token persisted to local file — no re-authentication needed after restart
- Supports filtering by alert type (human detection, motion detection, etc.)

### 2. Decision Layer: `emily_v2.py` — TriggerTracker + Scene Matching

**Role**: Determine interaction mode based on trigger frequency and time of day

```
Alert event → TriggerTracker (cooldown check) → Time period matching → Mode (pass_by/interact/scheduled)
```

- **TriggerTracker**: Maintains trigger history, implements cooldown and debounce
  - `pass_by_cooldown`: Greeting cooldown (default 120s)
  - `interact_cooldown`: Interactive teaching cooldown (default 180s)
  - `dual_window`: Dual-lens joint detection window (wide-angle + PTZ triggered simultaneously → interact)
- **Time Period Matching**: Matches current time to scenes configured in `config.yaml` under `time_scenes`
- **Night Silence**: No passive triggers during `quiet_hours`

### 3. Generation Layer: DeepSeek API / OpenClaw Emily API

**Role**: Generate personalized English content

**Mode A — Direct DeepSeek**:
```
emily_v2.py → OpenAI SDK → DeepSeek API → English content
```
- Uses OpenAI SDK compatible interface
- System prompt includes teaching strategies and scene constraints

**Mode B — OpenClaw Emily API (Recommended)**:
```
emily_v2.py → HTTP POST → emily-api.py → openclaw CLI → Emily Agent → English content
```
- emily-api.py deployed on OpenClaw server as HTTP gateway
- OpenClaw Emily Agent has full personality (SOUL.md), family info (USER.md), teaching memory (MEMORY.md)
- Long-term memory lets Emily "remember" previously taught words, avoiding repetition

### 4. Voice Layer: `tts_stream.py`

**Role**: Text-to-speech (streaming)

```
English text → Tencent Cloud WebSocket TTS → PCM audio stream → Playback/Push
```

- WebSocket long connection with streaming response
- Dual output modes:
  - `speak()` — Play while receiving (PyAudio local playback)
  - `synthesize()` — Receive complete PCM data and return (for camera speaker)
- Separate synthesis for English and Chinese segments, using voice_type=101009

### 5. Playback Layer: `camera_speaker.py`

**Role**: Push audio to camera speaker via RTSP Backchannel

```
PCM data → FFmpeg (PCM→AAC transcoding) → RTP packaging → RTSP Backchannel → Camera speaker
```

**RTSP Backchannel Technical Details**:
1. DESCRIBE request includes `Require: www.onvif.org/ver20/backchannel` header
2. SDP response contains an additional sendonly audio track (trackID=4)
3. SETUP trackID=4 using TCP interleaved mode with `mode=record`
4. Only supports AAC encoding (PT=104, 16kHz)
5. RTP frames sent through the same TCP connection

### 6. Conversation Layer: `conversation.py` + `asr_engine.py` + `camera_mic.py`

**Role**: Multi-turn conversation (Emily speaks → Listen for response → AI replies)

```
Emily TTS playback → camera_mic.py recording → asr_engine.py recognition → AI generates reply → Loop
```

- `camera_mic.py`: Capture audio stream from camera via RTSP (RTSP → AAC → PCM)
- `asr_engine.py`: Tencent Cloud one-sentence recognition, supports Chinese/English/Cantonese mix
- `conversation.py`: Manages multi-turn dialogue state, turn limits, and fallback strategies

## Data Flow Diagram

```
                    ┌─────────────────────────────────────┐
                    │          config.yaml                 │
                    │  (Centralized secrets & config)      │
                    └──────────┬──────────────────────────┘
                               │
                    ┌──────────▼──────────────────────────┐
                    │       config_loader.py               │
                    │  (Load config + pre-launch checks)   │
                    └──────────┬──────────────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
    ezviz_monitor.py    scheduled_tasks     quiet_hours
    (Alert polling)     (Scheduled tasks)   (Silence protection)
            │                  │
            └───────┬──────────┘
                    ▼
            emily_v2.py (TriggerTracker + Scene matching)
                    │
            ┌───────┴───────┐
            ▼               ▼
       DeepSeek API    OpenClaw API
            │               │
            └───────┬───────┘
                    ▼
            tts_stream.py (WebSocket streaming TTS)
                    │
            ┌───────┴───────┐
            ▼               ▼
        PyAudio         camera_speaker.py
      (Local play)    (RTSP Backchannel)
```

## Resource Usage

Real-world measurements on Xiaomi 7000 Router:

| Metric | Normal | Peak | Limit |
|--------|--------|------|-------|
| Memory | 60-80MB | ~100MB | 128MB (hard limit) |
| CPU | <5% | ~30% (during TTS+push) | 0.5 cores |
| Processes | 8-12 | ~20 | 32 |
| Network | ~1KB/s (polling) | ~50KB/s (TTS stream) | Unlimited |
