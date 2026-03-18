# OpenClaw Emily Persona Setup Guide

## What is OpenClaw?

[OpenClaw](https://openclaw.com) is an open-source AI Agent platform. WallWhisper uses OpenClaw to build a complete persona and memory system, enabling Emily to:

- 🧠 **Remember** words she's taught, avoiding repetition
- 👨‍👩‍👧 **Recognize** your family members, adjusting difficulty for each person
- 💝 **Grow** — Over time, Emily's teaching becomes increasingly personalized

> 💡 **WallWhisper works without OpenClaw!** When OpenClaw is not configured, Emily calls the DeepSeek API directly, generating independent content each time. OpenClaw provides persona consistency and long-term memory.

## Setup Steps

### 1. Deploy OpenClaw

Refer to the [OpenClaw official documentation](https://openclaw.com/docs) to deploy OpenClaw on your server.

Recommended: Tencent Cloud Lighthouse (2C2G is sufficient).

### 2. Create Emily Agent

Create an agent named `emily` in OpenClaw.

### 3. Upload Persona Files

Upload the files from `examples/openclaw-config/` to OpenClaw:

| File | Upload Location | Description |
|------|----------------|-------------|
| `SOUL.md` | Emily Agent persona config | Emily's personality, teaching strategy, speaking style |
| `USER.md` | Emily Agent user config | Your family information (⚠️ Must customize!) |
| `TOOLS.md` | Emily Agent tools config | Hardware environment info |
| `MEMORY.md` | Emily Agent memory file | Teaching progress (Emily auto-updates this) |
| `HEARTBEAT.md` | Emily Agent heartbeat config | Periodic maintenance tasks |

### 4. Customize USER.md (Important!)

`USER.md` is the only way Emily learns about your family. Be sure to modify it:

```markdown
## Your Child
- **Name:** YourKidName        # ← Replace with your child's name
- **Age:** 3 years old         # ← Replace with your child's age
- **Interests:** Frozen, Elsa  # ← Replace with your child's interests
```

Emily will personalize content based on this info. For example:
- If the child loves Frozen → Emily will frequently use Elsa in examples
- If the child is 5 years old → Emily will teach slightly more complex vocabulary

### 5. Customize TOOLS.md

Update your hardware environment information:

```markdown
### Camera & Speaker
- **Camera IP:** 192.168.1.100   # ← Your camera IP
```

### 6. Deploy emily-api.py

`emily-api.py` is a lightweight HTTP gateway, deployed on the same server as OpenClaw:

```bash
# Upload to server
scp examples/openclaw-config/emily-api.py your-server:/root/

# Set environment variables (optional)
export EMILY_API_PORT=8901
export EMILY_API_TOKEN="your-secure-token"

# Start
nohup python3 /root/emily-api.py > /root/emily-api.log 2>&1 &

# Verify
curl http://your-server:8901/api/emily/health
# Should return: {"status": "ok"}
```

### 7. Configure Emily to Connect to OpenClaw

Enable OpenClaw in `config.yaml`:

```yaml
openclaw_emily:
  enabled: true
  api_url: "http://your-server:8901/api/emily/speak"
  api_token: "your-secure-token"  # Must match EMILY_API_TOKEN in emily-api.py
  timeout: 30
```

### 8. Verify

```bash
# End-to-end test
python run.py test_full

# Check logs, you should see:
# [INFO] [Emily] 🧠 AI mode: OpenClaw Emily API (unified persona + memory)
```

## Using sync_openclaw.py to Sync Persona

After modifying persona files, use `sync_openclaw.py` for one-click sync:

```bash
# Check diff
python sync_openclaw.py --dry-run

# Sync changed files
python sync_openclaw.py

# View diff for a specific file
python sync_openclaw.py --diff SOUL.md

# Force re-upload all
python sync_openclaw.py --force

# Specify server
python sync_openclaw.py --host your-server-ip
```

## Persona File Details

### SOUL.md — Emily's Soul

Defines Emily's personality, speaking style, and teaching strategy:

- **Personality**: Gentle, patient, playful
- **Vocabulary Level**: Limited to toddler level (mainly monosyllabic words)
- **Speaking Format**: Bilingual (English first, Chinese explanation after)
- **Interaction Modes**: pass_by (short), interact (medium), scheduled (long)
- **Teaching Strategy**: One word at a time, connect to child's world, repetition, encouragement

> 💡 You can customize SOUL.md for your needs — for example, changing it to adult-oriented English teaching.

### USER.md — Family Profile

Tells Emily about your family for personalization:

- Child's age, English level, interests
- Parent's basic information
- Family daily routine

### MEMORY.md — Teaching Memory

Emily automatically updates this file:

- Words taught
- Learning progress for each family member
- Observation notes
- Milestones

> 📌 On first use, MEMORY.md is an empty template. As Emily interacts with your family, she will automatically populate it.

## FAQ

**Q: Can I use WallWhisper without OpenClaw?**

Yes! Set `openclaw_emily.enabled: false` in `config.yaml`. Emily will use the DeepSeek API directly. You just won't have long-term memory or a unified persona.

**Q: What server specs does OpenClaw need?**

Minimum 2C2G is sufficient. Tencent Cloud Lighthouse, AWS EC2, Google Cloud, etc. all work.

**Q: How secure is emily-api.py?**

- Supports Bearer Token authentication
- Token auto-generated and saved to `~/.emily-api-token`
- HTTPS recommended (via Nginx reverse proxy)
