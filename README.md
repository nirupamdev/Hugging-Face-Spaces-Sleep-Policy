# Hugging-Face-Spaces-Sleep-Policy
Hugging Face Spaces Sleep Policy

Based on the official Hugging Face documentation and community resources I found, here's a comprehensive guide on keeping your Hugging Face Space alive without a Pro subscription:

---

## Understanding Hugging Face Spaces Sleep Policy

According to the [official Hugging Face documentation](https://huggingface.co/docs/hub/spaces-gpus#set-a-custom-sleep-timesleep-time):

- **Free `cpu-basic` hardware**: Your Space will automatically go to sleep after **48 hours of inactivity**
- **You cannot configure a custom sleep time** on free hardware — it's fixed at 48 hours
- **Upgraded/paid hardware**: Never sleeps by default (or you can set custom sleep time)
- Anyone visiting your Space will restart it automatically, but there's a cold start delay

---

## Method 1: Self Keep-Alive with GitHub Actions (Recommended)

The most popular free approach is using **GitHub Actions** to ping your Space periodically.

### How it works:
- Schedule a GitHub Action to run every **~40 hours** (safely within the 48-hour window)
- Send a `curl` request to your Space URL
- This resets the inactivity timer and prevents pausing

### Sample GitHub Actions Workflow

Create `.github/workflows/keep-alive.yml` in a public GitHub repository:

```yaml
name: Keep HF Space Alive

on:
  schedule:
    # Runs every 40 hours (within the 48-hour free tier limit)
    - cron: '0 */40 * * *'
  workflow_dispatch:  # Allows manual triggering

jobs:
  ping:
    runs-on: ubuntu-latest
    steps:
      - name: Ping Hugging Face Space
        run: |
          curl -s -L -o /dev/null -w "%{http_code}" \
            https://your-username-your-space.hf.space
```

### Key Points from Community Solutions:

| Repository | Approach | Interval |
|---|---|---|
| [FahimFBA/ping-hf-safelicensing](https://github.com/FahimFBA/ping-hf-safelicensing) | Simple curl ping | Every 40 hours |
| [BibekG1/Huggingface-Mine](https://github.com/BibekG1/Huggingface-Mine) | Smart wake-up + ping | Every 25 minutes |

The **25-minute interval** approach is more robust because:
- GitHub Actions cron jobs can be delayed 5-15 minutes during peak hours
- Provides a safety buffer so you never hit the inactivity limit

---

## Method 2: UptimeRobot Setup

**UptimeRobot** is a free external monitoring service that can keep your Space alive by sending periodic HTTP requests.

### Setup Steps:

1. **Create a free UptimeRobot account** at [uptimerobot.com](https://uptimerobot.com)

2. **Add a new monitor**:
   - Monitor Type: HTTP(s)
   - Friendly Name: `HF Space Keep-Alive`
   - URL: `https://your-username-your-space.hf.space`
   - Monitoring Interval: **Every 5 minutes** (free tier minimum)

3. **Optional**: Add alert contacts (email, Discord, Slack, etc.)

### ⚠️ Important Warnings:

Based on community reports (e.g., [this forum post](https://discuss.huggingface.co/t/keepalive-ping-get-health-ready-every-2-minutes/176238) and [this GitHub repo](https://github.com/F4bC0d3/huggingmes-hermes-webui)):

> **Hugging Face accounts are getting suspended for aggressive keep-alive practices.**

- A user was **flagged as abusive** and blocked by Hugging Face's abuse handler for pinging `/health/ready` every 2 minutes from a Cloudflare Worker
- Running always-on agents on free Spaces is **against the spirit (and increasingly the letter) of HF's ToS**

**Recommendations to avoid suspension:**
- Use a **reasonable interval** — every 25-40 hours is much safer than every 2 minutes
- Don't use Cloudflare Workers or aggressive cron jobs
- Use **GitHub Actions** with a conservative schedule instead of persistent ping services
- Consider using a **throwaway HF account** if you're experimenting

---

## Method 3: Advanced Self Keep-Alive (Python Script)

If you want more control, you can run a Python script that:
1. Checks your Space status via the Hugging Face API
2. Wakes it up if sleeping
3. Pings the public URL to simulate traffic

```python
import os
import time
import requests

HF_TOKEN = os.getenv("HF_TOKEN")  # Your Hugging Face token
SPACE_ID = "your-username/your-space"

def keep_alive():
    headers = {"Authorization": f"Bearer {HF_TOKEN}"}
    
    # Check Space status
    status_url = f"https://huggingface.co/api/spaces/{SPACE_ID}"
    response = requests.get(status_url, headers=headers)
    data = response.json()
    
    if data.get("runtime", {}).get("stage") == "SLEEPING":
        # Restart the Space
        restart_url = f"https://huggingface.co/api/spaces/{SPACE_ID}/restart"
        requests.post(restart_url, headers=headers)
        print("Space was sleeping, restarted it")
        time.sleep(30)  # Wait for boot
    
    # Ping the public URL to reset inactivity timer
    space_url = f"https://{SPACE_ID.replace('/', '-')}.hf.space"
    ping = requests.get(space_url)
    print(f"Pinged Space: {ping.status_code}")

if __name__ == "__main__":
    keep_alive()
```

Run this on a free cron service or your own server every 25 minutes.

---

## Summary Comparison

| Method | Cost | Interval | Risk Level | Best For |
|---|---|---|---|---|
| **GitHub Actions (40h)** | Free | Every 40h | Low | Simple, safe keep-alive |
| **GitHub Actions (25m)** | Free | Every 25m | Low-Medium | Critical apps (n8n, etc.) |
| **UptimeRobot (5m)** | Free | Every 5m | Medium | Uptime monitoring + keep-alive |
| **Python Script + API** | Free/VPS | Custom | Low | Full control over wake-up |

---

## Key Takeaways from HF Docs

1. **Free tier sleep is 48 hours** — no way to change it without upgrading ([source](https://huggingface.co/docs/hub/spaces-gpus#set-a-custom-sleep-timesleep-time))
2. **Paid hardware never sleeps by default** — Pro subscription removes this issue entirely
3. **Paused time is not billed** — you can manually pause when not using
4. **Static Spaces are free for everyone** — but only host static HTML, not Gradio/Docker apps


<img src="https://user-images.githubusercontent.com/74038190/212284100-561aa473-3905-4a80-b561-0d28506553ee.gif" width="100%" height="3px">



A comprehensive GitHub Actions template repository for keeping your Hugging Face Space alive. 

A complete GitHub Actions template repository for keeping your Hugging Face Space alive. Here's what's included:

## 📦 What's in the Template

### **Three Workflow Options:**

1. **Safe (Every 40 Hours)** ⭐ — Recommended
   - Minimal GitHub Actions usage (~18 runs/month)
   - Very low risk of HF flagging
   - Safely within the 48-hour sleep window

2. **Warm (Every 6 Hours)**
   - Keeps Space consistently warm
   - Retry logic for wake-up delays
   - ~120 runs/month

3. **Aggressive (Every 25 Minutes)** ⚠️
   - Near-zero cold start
   - ~1,700 runs/month
   - Higher risk — use with caution

### **Key Features:**
- ✅ Multiple ping strategies to choose from
- ✅ Retry logic with exponential backoff
- ✅ Health endpoint targeting (`/health`)
- ✅ Manual trigger support (`workflow_dispatch`)
- ✅ Concurrency control to prevent overlapping runs
- ✅ Proper error handling without false failures
- ✅ Ready-to-copy workflow files

## 🚀 How to Use

1. **Create a new GitHub repository** using this template
2. **Add two secrets** in Settings → Actions:
   - `SPACE_URL`: Your HF Space URL (e.g., `https://username-space-name.hf.space`)
   - `HF_TOKEN`: Your Hugging Face token (get it [here](https://huggingface.co/settings/tokens))
3. **Enable the workflow** in the Actions tab
4. **Test manually** with "Run workflow" before relying on the schedule

## ⚠️ Critical Warnings

Based on the research:
- A user was **flagged as abusive and blocked** for pinging every 2 minutes via Cloudflare Worker ([source](https://discuss.huggingface.co/t/keepalive-ping-get-health-ready-every-2-minutes/176238))
- GitHub Actions **auto-disables scheduled workflows after 60 days** of repo inactivity
- The **Safe (40h)** strategy is the sweet spot — enough to prevent sleep, minimal risk

