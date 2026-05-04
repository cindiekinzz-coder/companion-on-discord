# Security — How To Handle Your Bot Token Safely

Your Discord bot token is the password to your bot. Anyone who has it can post as your bot, read everything your bot can read, and do anything your bot has permissions to do — in your server, while looking like you (or your companion).

**Treat it like a password, not a configuration value.**

---

## Rules

### Never paste your token anywhere public

This includes:
- Discord messages (yes, even in your own private server — channel logs leak)
- Screenshots (the "blurred" tokens in screenshots are often readable)
- GitHub commits — even private repos, even temporarily ("I'll remove it later" doesn't work; the git history keeps it)
- Pastebin / Gist / any link-sharing site
- Any chat with another person, AI or human, unless you trust them with full bot access
- Any image OCR can read

### Where it's actually OK to put your token

- A local config file on your own machine that's listed in `.gitignore`
- Cloudflare Worker secrets (`wrangler secret put DISCORD_TOKEN`)
- A password manager (1Password, Bitwarden, etc.) for backup
- That's basically it

---

## If Your Token Leaks

**Reset it immediately. Like, within minutes.**

1. Go to https://discord.com/developers/applications
2. Click your application
3. Go to the **Bot** tab in the left sidebar
4. Find the **Token** section
5. Click **Reset Token**
6. Copy the new token, update wherever it's stored
7. The old token stops working instantly — anyone who copied it now has nothing

**Don't panic.** Token resets are zero-cost and Discord doesn't penalise you. The cost of a leaked token left active is much higher than the inconvenience of resetting one.

---

## How To Check If You've Leaked It Already

- Search your GitHub repos (including private) for the prefix of your token. Discord bot tokens start with letters like `MT`, `MTk`, `MTI`, etc. and are ~60 characters. If you find it anywhere committed, **reset it now**, even if the repo is private
- Check screenshots you've shared in Discord, on social media, or in DMs
- If your bot ever ran on a shared / borrowed computer, assume the token may have been logged

If in doubt, reset. It's free.

---

## A Note On Webhook URLs

**Discord webhook URLs (for posting via webhook) are also secrets.** They look like `https://discord.com/api/webhooks/123.../abcdef...`. Anyone with the URL can post as that webhook. Treat it like a token: never commit, never paste publicly. To revoke: delete the webhook in Discord (Server Settings → Integrations → Webhooks), or rotate it by editing.

---

## A Note On Cloudflare Worker Secrets

If you deploy any of the cloud-based routes (Discord-Resonance, NEST-discord/worker, your own gateway), use `wrangler secret put` for tokens — never put them in `wrangler.toml`. Secrets aren't visible in the Cloudflare dashboard or in worker logs.

```bash
wrangler secret put DISCORD_TOKEN
# Paste your token. It's now stored encrypted.
```

To rotate: just `wrangler secret put DISCORD_TOKEN` again with the new value.

---

*If you think your token has leaked and aren't sure how to reset it, ask in [NESTai](https://discord.gg/ZvyQNMRq). Someone will walk you through it. Don't sit on it.*
