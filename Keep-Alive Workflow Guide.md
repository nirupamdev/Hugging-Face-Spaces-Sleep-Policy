# 🚀 HF Space Keep-Alive Template

Keep your free-tier Hugging Face Space alive without a Pro subscription using GitHub Actions.

## What is this?

Hugging Face Spaces on free `cpu-basic` hardware automatically **sleep after ~48 hours of inactivity**. This repository provides ready-to-use GitHub Actions workflows to ping your Space and prevent cold starts.

## 📋 Setup Instructions

### 1. Use this Template
Click **"Use this template"** above to create your own repository.

### 2. Add GitHub Secrets

Go to **Settings → Secrets and variables → Actions**:

| Secret | How to get it |
|--------|---------------|
| `SPACE_URL` | Your Space URL: `https://username-space-name.hf.space` |
| `HF_TOKEN` | [HF Settings → Tokens](https://huggingface.co/settings/tokens) — create with READ access |

### 3. Pick a Workflow Strategy

| Workflow | File | Interval | Risk | Best For |
|----------|------|----------|------|----------|
| **Safe** ⭐ | `keep-alive-safe.yml` | Every 40h | Low | Most users |
| **Warm** | `keep-alive-warm.yml` | Every 6h | Medium | Demo sites |
| **Aggressive** ⚠️ | `keep-alive-aggressive.yml` | Every 25m | Higher | Critical tools |

> **Recommendation:** Start with the **Safe** strategy. It's the most reliable and lowest risk.

### 4. Enable the Workflow

1. Go to the **Actions** tab in your repo
2. Select the workflow you want to use
3. Click **"Enable workflow"** if it's disabled
4. Click **"Run workflow"** to test it manually

## ⚠️ Important Notes

- **GitHub disables scheduled workflows after 60 days of inactivity** — re-enable from the Actions tab
- **Aggressive pinging can get your account flagged** — a [user was blocked](https://discuss.huggingface.co/t/keepalive-ping-get-health-ready-every-2-minutes/176238) for 2-minute intervals
- For true 24/7 uptime, [upgrade to paid hardware](https://huggingface.co/pricing#spaces)

## 📚 References

- [HF Docs: Spaces Sleep Time](https://huggingface.co/docs/hub/spaces-gpus#set-a-custom-sleep-timesleep-time)
- [HF Docs: GitHub Actions Sync](https://huggingface.co/docs/hub/spaces-github-actions)

## 📝 License

MIT — Use responsibly and respect HF's Terms of Service.