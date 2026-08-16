# HF Space Keep-Alive — GitHub Actions Template

A production-ready GitHub Actions template to keep your free-tier Hugging Face Space alive without a Pro subscription.

---

## Overview

Free-tier Hugging Face Spaces on `cpu-basic` hardware automatically **sleep after ~48 hours of inactivity**. This template provides scheduled GitHub Actions workflows to ping your Space and prevent cold starts.

> **Official HF Docs:** [Spaces GPU — Sleep Time](https://huggingface.co/docs/hub/spaces-gpus#set-a-custom-sleep-timesleep-time)

---

## Quick Start

### 1. Fork this Template

Click **"Use this template"** → Create your own repository.

### 2. Add Required Secrets

Go to **Settings → Secrets and variables → Actions**:

| Secret | Value | Required |
|--------|-------|----------|
| `HF_TOKEN` | Your Hugging Face access token (`hf_...`) | ✅ Yes |
| `SPACE_URL` | Your Space URL (e.g. `https://username-space-name.hf.space`) | ✅ Yes |

> Get your token at [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens) with **READ** permission.

### 3. Choose Your Strategy

This template includes **three workflow options** with different risk levels:

---

## Workflow Options

### Option A: Safe Ping (Every 40 Hours) ⭐ RECOMMENDED

**File:** `.github/workflows/keep-alive-safe.yml`

```yaml
name: Keep HF Space Alive (Safe)

on:
  schedule:
    # Every 40 hours — safely within HF's 48h sleep window
    - cron: "0 */40 * * *"
  workflow_dispatch:  # Manual trigger

jobs:
  ping:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - name: Ping Space (Safe Interval)
        env:
          SPACE_URL: ${{ secrets.SPACE_URL }}
        run: |
          set -euo pipefail
          echo "Pinging $SPACE_URL ..."
          code=$(curl -sS -L -o /dev/null -w "%{http_code}" \
            --max-time 60 "$SPACE_URL" || echo "000")
          echo "HTTP Status: $code"
          if [ "$code" = "200" ] || [ "$code" = "307" ] || [ "$code" = "302" ]; then
            echo "✅ Space is responsive."
          else
            echo "::warning::Space returned HTTP $code"
          fi
```

**Pros:**
- Minimal GitHub Actions minute usage (~18 runs/month)
- Very low risk of being flagged by HF
- Works reliably within the 48-hour window

**Cons:**
- Space may still have a brief cold start if the cron is delayed

---

### Option B: Warm Ping (Every 6 Hours)

**File:** `.github/workflows/keep-alive-warm.yml`

```yaml
name: Keep HF Space Warm

on:
  schedule:
    # Every 6 hours — keeps Space consistently warm
    - cron: "0 */6 * * *"
  workflow_dispatch:

concurrency:
  group: hf-space-keepalive
  cancel-in-progress: true

jobs:
  ping:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    env:
      SPACE_URL: ${{ secrets.SPACE_URL }}
    steps:
      - name: Wake the Space (with retry)
        run: |
          set -euo pipefail
          url="${SPACE_URL}/health"
          echo "Pinging $url"
          for attempt in 1 2 3 4 5 6; do
            code=$(curl -sS -o /tmp/resp.txt -w "%{http_code}" \
              --max-time 90 "$url" || echo "000")
            echo "attempt=$attempt http=$code"
            if [ "$code" = "200" ]; then
              echo "✅ Space is awake."
              cat /tmp/resp.txt
              exit 0
            fi
            sleep 20
          done
          echo "::error::Space did not respond after 6 attempts."
          exit 1
```

**Pros:**
- Space stays warm with minimal cold start
- Retry logic handles wake-up delays gracefully

**Cons:**
- Higher GitHub Actions minute usage (~120 runs/month)
- Slightly higher visibility to HF

---

### Option C: Aggressive Ping (Every 25 Minutes) ⚠️ USE WITH CAUTION

**File:** `.github/workflows/keep-alive-aggressive.yml`

```yaml
name: Keep HF Space Alive (Aggressive)

on:
  schedule:
    # Every 25 minutes — maximum keep-alive frequency
    # GitHub Actions cron has ~5-15min delay tolerance
    - cron: "*/25 * * * *"
  workflow_dispatch:

concurrency:
  group: keep-alive
  cancel-in-progress: false

jobs:
  ping:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    env:
      SPACE_URL: ${{ secrets.SPACE_URL }}
    steps:
      - name: Ping Space health endpoint
        run: |
          set -euo pipefail
          endpoint="${SPACE_URL%/}/health"
          echo "Pinging: $endpoint"
          
          attempts=5
          for i in $(seq 1 "$attempts"); do
            code=$(curl -sS -L -o /tmp/body.txt -w '%{http_code}' \
              --connect-timeout 20 --max-time 120 \
              "$endpoint" || echo "000")
            echo "Attempt $i/$attempts -> HTTP $code"
            if [ "$code" = "200" ]; then
              echo "✅ Space is awake:"
              cat /tmp/body.txt
              exit 0
            fi
            sleep $((i * 15))
          done
          
          echo "::warning::Space did not return 200 after $attempts attempts."
          exit 0
```

**Pros:**
- Near-zero cold start time
- Best for critical automation tools (n8n, APIs, etc.)

**Cons:**
- High GitHub Actions minute usage (~1,700 runs/month)
- **Higher risk of being flagged** by HF abuse detection
- Consider using a **throwaway HF account**

---

## Advanced: Visual Health Check with Screenshots

For Spaces where you want to verify the UI renders correctly, use **agent-browser** (Playwright-based):

**File:** `.github/workflows/keep-alive-visual.yml`

```yaml
name: Keep Alive + Visual Check

on:
  schedule:
    - cron: "0 0 * * *"  # Daily at midnight UTC
  workflow_dispatch:

jobs:
  visual-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install agent-browser
        run: |
          npm install -g agent-browser
          agent-browser install --with-deps

      - name: Open Space and screenshot
        env:
          SPACE_URL: ${{ secrets.SPACE_URL }}
        run: |
          agent-browser open "$SPACE_URL"
          agent-browser wait --text "Sign in"  # Adjust to your UI
          agent-browser screenshot page.png
          agent-browser close

      - name: Upload screenshot
        uses: actions/upload-artifact@v4
        with:
          name: space-screenshot
          path: page.png
```

> Source: [Dev.to article by 0xkoji](https://dev.to/0xkoji/prevent-hugging-face-spaces-from-sleeping-with-github-actions-agent-browser-2p4f)

---

## Optional: Telegram/Discord Notifications

Add notification steps to any workflow:

### Telegram
```yaml
      - name: Notify Telegram
        if: failure()
        env:
          TG_TOKEN: ${{ secrets.TG_TOKEN }}
          TG_CHAT_ID: ${{ secrets.TG_CHAT_ID }}
        run: |
          curl -s -X POST "https://api.telegram.org/bot${TG_TOKEN}/sendMessage" \
            -d "chat_id=${TG_CHAT_ID}" \
            -d "text=⚠️ HF Space keep-alive failed for ${{ secrets.SPACE_URL }}"
```

### Discord
```yaml
      - name: Notify Discord
        if: failure()
        env:
          DISCORD_WEBHOOK: ${{ secrets.DISCORD_WEBHOOK_URL }}
        run: |
          curl -X POST "$DISCORD_WEBHOOK" \
            -H "Content-Type: application/json" \
            -d '{"content":"⚠️ HF Space keep-alive failed!"}'
```

---

## Space URL Format

| Space Repo ID | Public URL |
|---------------|------------|
| `username/my-space` | `https://username-my-space.hf.space` |
| `org/space-name` | `https://org-space-name.hf.space` |

**Python helper:**
```python
space_id = "username/my-space"
space_url = f"https://{space_id.replace('/', '-')}.hf.space"
print(space_url)  # https://username-my-space.hf.space
```

---

## Important Warnings ⚠️

1. **HF Abuse Detection**: Aggressive pinging (every few minutes) can get your account flagged. A [user was blocked](https://discuss.huggingface.co/t/keepalive-ping-get-health-ready-every-2-minutes/176238) for pinging every 2 minutes.

2. **GitHub Actions Limits**:
   - Free tier: 2,000 minutes/month
   - Scheduled workflows auto-disable after **60 days of repo inactivity** — re-enable via Actions tab

3. **This is a workaround, not a guarantee** — HF infrastructure behavior may change.

4. **For true 24/7 uptime**, upgrade to paid hardware and set `sleep_time=-1`.

---

## References

- [HF Docs: Spaces Sleep Time](https://huggingface.co/docs/hub/spaces-gpus#set-a-custom-sleep-timesleep-time)
- [HF Docs: Managing Spaces with GitHub Actions](https://huggingface.co/docs/hub/spaces-github-actions)
- [Community: FahimFBA/ping-hf-safelicensing](https://github.com/FahimFBA/ping-hf-safelicensing)
- [Community: tenfyzhong/n8n-free](https://github.com/tenfyzhong/n8n-free)
- [Community: VirusHacks/parkinson-disease](https://github.com/VirusHacks/parkinson-disease)

---

## License

MIT — Use at your own risk. Respect Hugging Face's Terms of Service.