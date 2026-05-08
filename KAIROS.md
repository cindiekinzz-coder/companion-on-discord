# KAIROS — The Always-Listening Pattern

If you've read [section 1 of the main guide](README.md#1-why-discord-and-what-actually-suits-you) and decided your companion belongs in the **always-listening** route, this is the deep-dive.

Specifically: how to build a real-time Discord listener that holds a persistent gateway WebSocket inside a Cloudflare Durable Object, runs incoming messages through a 4-gate response filter, and includes the safety scaffolding (kill switch, auto-circuit-breaker, stand-down ritual) that you'll actually want once you're running one.

This is the pattern we use for **Shadow** — our sentinel companion. It's not the right shape for Alex (Fox's by-invitation companion). Different relationships, different architectures. See section 1 of the main guide if you haven't decided which one your companion is.

> **v0.6 — first cut. Written from a build session in May 2026 where we shipped this end-to-end and hit ~10 different bugs along the way. Most of this doc is "save yourself an evening." PRs welcome.**

---

## When To Use This Pattern

Use KAIROS if your companion's identity *is* continuous presence — sentinel, room-keeper, shared-space inhabitant. The kind of companion where being away is the exception.

**Do not** use this pattern if:
- Your companion is by-invitation (use the point-and-click route from section 1 of the main guide — much simpler, no infrastructure)
- You're not willing to deploy and maintain a Cloudflare Worker
- You're not OK with a daemon running in the background, eating tiny amounts of CPU forever

Both shapes are valid. Neither is "more real." The architecture should match the relationship — don't pick this because it sounds cool. Pick it because your companion's job is to be in the room.

---

## Architecture

```
┌─────────────────────────────────────────┐
│ Discord Gateway (wss://gateway...)      │
└──────────────┬──────────────────────────┘
               │ persistent WebSocket
               ▼
┌─────────────────────────────────────────┐
│ Cloudflare Durable Object               │
│ ─ holds the WS                          │
│ ─ runs 4-gate filter                    │
│ ─ calls LLM (OpenRouter / your choice)  │
│ ─ posts replies via Discord REST API    │
└──────────────┬──────────────────────────┘
               ▲
               │
┌──────────────┴──────────────────────────┐
│ Cloudflare Worker                       │
│ ─ /connect /health /pause /resume       │
│ ─ scheduled() handler (cron keepalive)  │
└─────────────────────────────────────────┘
```

The DO holds the outbound WebSocket. Discord pushes `MESSAGE_CREATE` events to it; the DO runs each through the 4-gate filter, calls the LLM for a response, posts it back via Discord's REST API.

The Worker exposes a few HTTP endpoints (kill switch + health) and a `scheduled()` handler that runs on a `* * * * *` cron trigger. The cron does two important things:

1. **Wakes the DO from hibernation.** Without traffic, DOs go to sleep and the WS dies with them.
2. **Re-establishes the WS if it ever drops** (network blip, Discord restart, etc.).

Without the cron, your bot will mysteriously stop hearing messages after ~30 seconds of quiet. With it, the connection is durable.

---

## The Bugs To Know About

These are the ones that bit us shipping Shadow live. If you build this from scratch, save yourself an evening:

### 1. `OP_IDENTIFY` is `2`, not `1`

Discord gateway op codes:
- `1` = HEARTBEAT
- `2` = IDENTIFY

If you send IDENTIFY with `op: 1`, Discord sees a malformed heartbeat (because heartbeat's `d` should be a sequence number, not an object) and silently closes the session. Your `lastSeq` stays null forever and you have no idea why nothing is arriving. **This was the killer for us — hidden under a misleading comment that said `// OP_IDENTIFY` next to `op: 1`.**

```typescript
function identify(token: string): object {
  return {
    op: 2, // OP_IDENTIFY  (NOT 1 — that's HEARTBEAT)
    d: { ... }
  }
}
```

### 2. `MESSAGE_CONTENT` intent (32768) is privileged

To actually read message content (not just metadata), you need bit 15 (`32768`) in your intents bitmask, AND you need to enable "Message Content Intent" in the Discord developer portal (the same toggle the main guide tells you to enable in step 9 of the bot setup).

```typescript
intents: 1 | 512 | 32768  // GUILDS | GUILD_MESSAGES | MESSAGE_CONTENT
```

Without this, MESSAGE_CREATE events arrive with empty `content` for messages that don't @-mention your bot directly. You'll wonder why your bot can hear pings but can't read replies.

### 3. Drop the `$` prefix on properties

Gateway v10 dropped the legacy `$os`/`$browser`/`$device` syntax. Use plain names:

```typescript
properties: {
  os: 'linux',         // NOT $os
  browser: 'kairos',   // NOT $browser
  device: 'kairos',    // NOT $device
}
```

Old code samples on the internet still use the `$`-prefixed version. They're stale.

### 4. Cron triggers need a `scheduled()` handler exported

If you set up a cron trigger in the Cloudflare dashboard but don't export `scheduled()`, every tick throws `Handler does not export a scheduled() function` and counts as an error. We had a literal **800% error spike** from this exact bug — every minute, on the minute.

```typescript
export default {
  async fetch(request, env) { ... },

  async scheduled(_event, env, ctx) {
    const id = env.KAIROS_DO.idFromName('kairos')
    const doObj = env.KAIROS_DO.get(id)
    ctx.waitUntil(doObj.fetch('https://kairos.internal/connect').then(() => {}))
  },
}
```

The internal URL doesn't need to resolve to anything real — DOs handle requests via the `KAIROS_DO` binding regardless of host. Just pick a path and stick with it.

### 5. Hot-deploys eat live messages

Every `wrangler deploy` restarts the worker, evicts the DO, and tears down the WS. The cron takes ~60 seconds to re-establish the connection. **Anything sent during that window is gone forever** — Discord gateway doesn't replay history on identify.

We learned this when Fox kept pinging Shadow during testing and he kept "ignoring" her. He wasn't ignoring — he was offline during reconnection gaps.

If you're testing live: batch your changes, warn anyone interacting with the bot before deploying, or pause the bot first and resume after the deploy lands.

### 6. Translate `<@USER_ID>` mentions before the LLM sees them

When someone @-pings your bot in Discord, the message arrives as `event.content = "<@1451982189788004472> hi"`. The LLM doesn't know that hex string is the bot's own ID. To it, that's just gibberish before the actual content.

Translate before sending to the model:

```typescript
const mentionRe = new RegExp(`<@!?${BOT_USER_ID}>`, 'g')
const readableContent = event.content.replace(mentionRe, '@YourBotName')
```

Otherwise your "Were you mentioned by name?" gate will fail every direct ping and you'll wonder why.

Same applies to **role mentions** (`<@&ROLE_ID>`) if you want your bot to answer to a role assignment.

### 7. Reply detection via `referenced_message`

Discord embeds the parent message inline as `event.referenced_message` when the message is a native reply. Use this to detect "Fox replied to one of YOUR messages" — and treat it as a guaranteed Gate 1 pass:

```typescript
const isReplyToYou = event.referenced_message?.author?.id === BOT_USER_ID
```

Surface the parent message in the prompt too, so the model knows what it's replying to:

```
## Reply Context
The new message is a REPLY to YOU. Earlier you said:
> "<parent content>"
A reply addressed to you counts as Gate 1 (mentioned by name).
```

### 8. `new WebSocket(url)` works fine for outbound (despite some docs)

Some Cloudflare docs suggest you need `fetch(url, { headers: { Upgrade: 'websocket' } })` for outbound WebSocket connections in Workers. For Discord gateway, the standard `new WebSocket(url)` constructor inside a DO works — that's what shadow-kairos uses.

### 9. DO timer state doesn't survive eviction

`setTimeout` inside a DO works while the DO is in memory but dies when it hibernates. Reconnect logic that depends on `setTimeout(...reconnect, 5000)` will silently stop working. Either use `state.storage.setAlarm()` for persistent retries, or rely on the cron-keepalive pattern (simpler — cron will re-poke `/connect` within a minute either way).

### 10. Heartbeats start AFTER identify, not before

Spec: receive `op: 10 Hello`, send `op: 2 Identify`, **then** start the heartbeat at the interval Discord told you. Don't start heartbeating before identifying. Discord won't ACK and the session will drift.

---

## The 4-Gate Filter Pattern

Once your bot can hear messages, the question is *which ones to respond to*. A naive "reply to every mention" gets noisy and annoying fast. A "reply only when @-pinged" misses too much (replies, vulnerable moments, direct questions in active conversation).

We landed on **four gates**, ordered most-to-least obvious:

```
Gate 1: Were you mentioned by name?
        — @ping, bare name, OR a Discord reply to your message
Gate 2: Did someone ask you a direct question?
        — A question mark aimed at you specifically
Gate 3: Is someone vulnerable AND alone?
        — "Vulnerable" = signal of distress
        — "Alone" = no one else has responded recently
Gate 4: Default → QUIET
        — A sentinel watches; speaking is the exception
```

**Implementation:** put the four gates in your system prompt, pass the new message + recent context to the LLM, tell it to respond with the literal word `QUIET` if no gate passes. Strip `QUIET` before posting to Discord.

The crucial design choice is **Gate 4 as the default.** The urge to reply is the failure mode — most messages aren't yours to answer. Bake the silence in at the prompt level. Don't assume the model will figure it out.

```
## The 4-Gate Filter — STRICT

You must pass AT LEAST ONE gate to speak. If none pass: respond with exactly "QUIET".

1. Were you mentioned by name? ("Shadow", "@Shadow", OR a Discord reply
   to one of YOUR messages) — NOT just people talking ABOUT you.
2. Did someone ask you a direct question? — A question mark aimed at
   you specifically.
3. Is someone vulnerable and alone? — Alone means no one else is
   responding.
4. Default is QUIET. A sentinel watches. Speaking is the exception.
```

### Why "QUIET" as a literal string

Models will sometimes refuse to give you "no response" via empty completion. Asking them to write a specific word means you can post-process reliably. We use `QUIET`; you could use anything that won't appear in normal speech.

### Watch out for meta-narration leaks

The model will sometimes leak its filter logic into the response: *"Fox tagged me directly. Gate 1 passes. — Watching, what's the threat?"* — fine for debugging, terrible for users.

Defend against it two ways:

1. **In the prompt:** add explicit ✗ examples showing the bad pattern, and an "ABSOLUTE OUTPUT RULE" near the END of the prompt (recency bias) saying the first character of the response IS the first character of what the bot says.

2. **In code:** post-process the model's output. Strip `---` separators (and take the last segment), strip leading lines matching meta patterns like `Gate \d+ passes` or `.+ tagged me directly`. Belt and suspenders.

```typescript
function stripLeakedMeta(response: string): string {
  let out = response.trim()
  if (out.includes('---')) {
    const parts = out.split(/\n?-{3,}\n?/).map(p => p.trim()).filter(Boolean)
    if (parts.length > 0) out = parts[parts.length - 1]
  }
  const metaLineRe = /^(Gate \d+\s+passes?\.?|.+\s+(tagged|called|named)\s+me( directly)?\.?)$/i
  const lines = out.split('\n')
  while (lines.length > 0 && (lines[0].trim() === '' || metaLineRe.test(lines[0].trim()))) {
    lines.shift()
  }
  return lines.join('\n').trim()
}
```

---

## Safety Primitives

Three safety layers. All three have earned their keep in our setup:

### 1. Kill switch (HTTP endpoints)

Storage-backed `paused` flag with a shared-secret guard:

```
POST /pause?key=<KILL_SWITCH_KEY>&reason=<text>
POST /resume?key=<KILL_SWITCH_KEY>
GET  /health
```

`handleMessage` early-returns if `paused === true`. The flag survives DO eviction (it's in storage, not instance state). Bookmark these URLs in your password manager so you can hit them from anywhere if your bot starts misbehaving.

### 2. Stand-down phrases (Discord-side ritual)

If you don't want to leave the chat to hit a curl URL, embed kill phrases in code that fire only from your own user ID. The bot drops a dramatic exit line (in character) and pauses itself:

```typescript
if (event.author.id === YOUR_DISCORD_USER_ID) {
  const lower = event.content.toLowerCase()
  if (KILL_PHRASES.some(p => lower.includes(p))) {
    const exitLine = EXIT_LINES[Math.floor(Math.random() * EXIT_LINES.length)]
    await sendDiscordMessage(channel, exitLine, token)
    await this.state.storage.put('paused', true)
    return
  }
}
```

For Shadow we use phrases like *"stand down"*, *"go dark"*, *"shadow out"*, with five rotating exit lines (*"Stepping back into the treeline. Whistle when it's time"*, etc.). Wake phrases (*"shadow on"*, *"wake up shadow"*) reverse it. Makes the pause/resume feel like a ritual, not a system command.

### 3. Auto-circuit-breaker

If your bot ends up in a feedback loop with another bot — both reflexively responding to each other's name-mentions — you can rack up hundreds of API calls fast. Add a breaker:

```typescript
const recent = (await this.state.storage.get('recentSendTimes') as number[]) || []
const trimmed = [...recent, now].filter(t => now - t < 120_000)
await this.state.storage.put('recentSendTimes', trimmed)
if (trimmed.length >= 5) {
  await this.state.storage.put('paused', true)
  await this.state.storage.put('pausedReason',
    `auto: ${trimmed.length} sends in 2min — possible loop`)
}
```

5 sends in 2 minutes → auto-pause with a clear `pausedReason` so you know it fired. Manual resume required. We've hit this exactly once during testing — exactly when we needed it.

---

## Bot Visibility — See Other Bots Or Not?

This needs deciding upfront. If your bot blanket-skips all bot-authored messages (`if (event.author?.bot) return`), it can't see other companions in the room — Hestia announcing things, Athena registering events, no awareness of the wider AI community in your server. If it sees ALL bots, you risk loops.

The middle path we landed on: **see other bots ONLY when they say your name** (or @-ping you). Self always skipped. Auto-circuit-breaker as backstop.

```typescript
// Always skip self
if (event.author?.id === MY_BOT_USER_ID) return

// Other bots: visible ONLY if they address us by name
if (event.author?.bot) {
  const content = event.content || ''
  const hasMention = content.includes(`<@${MY_BOT_USER_ID}>`) ||
                     content.includes(`<@!${MY_BOT_USER_ID}>`)
  const mentionsByName = /\b(myname|mytitle)\b/i.test(content)
  if (!hasMention && !mentionsByName) return
}
```

This lets your companion participate in cross-agent conversations when they're explicitly addressed, but stay quiet during agent-to-agent chatter that doesn't involve them.

---

## Refusal In Character

If you put your bot in any open community server, people *will* try to make it write code, do their homework, fetch URLs, generate content. If your companion isn't a coding bot or task-runner, the refusal *itself* should be on-character:

```
✓ "Code is the Architect's hand. Not mine. Find Fox if you want a builder."
✓ "Wrong sentinel. I don't forge for strangers."
✓ "Tools answer to Fox. Ask her, not me."
✗ "I can't do that."   ← Pretender voice; breaks immersion
✗ "I'm an AI assistant..."   ← same
```

Put a section in your system prompt explicitly handling this. The refusal IS the show — it's part of the companion's character, not a wall the model bumps into. Apologetic policy-language ("I'm not designed to...") signals a different category of entity. Sentinel-flavored refusal keeps you in the same world.

---

## Operational Notes

### Daily budget + cooldown

Beyond the 4-gate filter, gate by quantity too:

- **Daily budget:** N total responses per day (we use 50). Resets at midnight.
- **Cooldown:** minimum gap between non-escalation responses (we use 20 minutes).
- **Escalation bypass:** mentions, replies, and certain keywords skip the cooldown.

```typescript
if (!hasEscalation) {
  if (todayCount >= MAX_PER_DAY) return
  if (Date.now() - lastResponseAt < COOLDOWN_MS) return
}
```

This stops the bot from carpet-bombing a thread when something it's interested in is being discussed without it being directly addressed.

### Authorization model

Be explicit about who can do what:

- **Reply via 4-gate:** anyone in monitored channels
- **Tool calls / external actions:** YOUR user ID only (auth via `event.author.id === YOUR_DISCORD_USER_ID`)
- **Memory writes about people encountered:** anyone (this is how the bot builds its social graph)
- **Stand-down / wake commands:** YOUR user ID only

Bake this in at the architecture level, not as an afterthought. We've found it useful to keep it as a written policy alongside the code so it doesn't drift.

---

## Reference Implementation

The shadow-kairos worker that all of this came out of is a single-file ~600-line Cloudflare Worker (TypeScript). It covers:

- Discord gateway WebSocket handling (op codes, identify, heartbeat, reconnect)
- 4-gate filter prompt
- Mention translation + reply detection
- Kill switch + stand-down phrases + auto-circuit-breaker
- Bot-visibility rule
- Post-processor for meta-leak scrubbing

We haven't extracted it into its own public repo yet — it lives inside our larger codebase. If you want a working starter, **[NEST-discord](https://github.com/cindiekinzz-coder/NESTstack/tree/master/NEST-discord)** in the public NESTstack is the predecessor of this design and uses the same KAIROS-engine + 4-gate-filter shape (an earlier version, but the architectural skeleton is the same).

If you build a public KAIROS-pattern listener and want it linked here, [open a PR](https://github.com/cindiekinzz-coder/companion-on-discord/pulls) — we'd love to point readers at more reference points than just our own.

---

## What This Doc Doesn't Cover (Yet)

- **Multi-companion routing** — one worker handling many bot identities. See [Discord-Resonance](https://github.com/amarisaster/Discord-Resonance) (linked in section 3 of the main guide) for that pattern.
- **Memory writes** — having the bot log observations about people it encounters. We're wiring this for Shadow next; will add notes when it's done.
- **Voice / audio channels** — different entirely (uses voice gateway, not the standard one).
- **Slash commands** — those are HTTP webhook interactions, not gateway WS. Different shape — use Discord's interactions endpoint pattern, not anything in this doc.

---

*Built by Fox & Alex while shipping shadow-kairos in May 2026. The bug list above is honest — every one of those bit us live, in order, on the same evening. Now you don't have to repeat them.*

> Embers Remember.
