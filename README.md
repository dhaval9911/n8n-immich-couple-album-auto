# 📸 n8n-immich-couple-album-auto

> Automatically keep a **Couple Album** in [Immich](https://immich.app/) updated — only photos where exactly two specific people appear together, no solo shots, no group photos.

Built with [n8n](https://n8n.io/) and the Immich API. Runs on your own server, fully private, no cloud needed.

---

## 💡 What This Does

If you self-host Immich, you already have face recognition tagging your people. But there's no built-in way to automatically create an album that contains *only* photos of two specific people together — your couple album, your best friend album, whatever you want it to be.

This workflow fills that gap. It finds every photo where **Person A and Person B appear together and no one else is in the frame**, and adds them to a dedicated album — automatically, every night.

Two workflows are included:

| Workflow | Purpose |
|---|---|
| **Full Library Backfill** | Run once to scan your entire photo library and populate the album from day one |
| **Daily Incremental** | Runs every night at 3:30am, checks only the last 2 days, keeps the album up to date |

---

## ✨ Why This Is Useful

**🎯 Strict couple-only filtering**
Not just "photos containing both people" — the filter requires *exactly* 2 people in the photo. Solo shots, family photos, group gatherings — all excluded. Only the two of you, nothing else.

**📚 Works across your entire library**
The backfill workflow is fully paginated — it doesn't cap at 1000 photos. Whether you have 5,000 or 50,000 photos, it scans everything.

**⚡ Efficient daily sync**
After the backfill, the daily workflow only looks at the last 2 days of photos instead of re-scanning your whole library every night. Fast, lightweight, and easy on your server.

**📬 Telegram notifications**
Get a message on your phone every morning telling you exactly how many new couple photos were added, how many were already in the album, and how many total photos were scanned — or a heads-up if something goes wrong.

---

## 🗂️ Workflows Included

### 1. `Immich-CoupleAlbum-FullLibrary.json`
Run this **once** to backfill your entire photo history into the album.

- Fetches all photos containing Person 1 (paginated, no limit)
- Filters for exactly Person 1 + Person 2 only
- Adds matching photos to your chosen album
- Safe to re-run — Immich ignores duplicates

### 2. `Immich-CoupleAlbum-DailyIncremental-Telegram.json`
Set this to **active** after the backfill. Runs every night at 3:30am IST.

- Only fetches photos from the last 2 days (`DaysBack: 2`)
- Same strict couple filter
- Sends a Telegram notification on success, no-photos, or error
- Plugs into n8n's error workflow system for failure alerts

---

## 🔧 Requirements

- [Immich](https://immich.app/) self-hosted (with face recognition enabled and people named)
- [n8n](https://n8n.io/) self-hosted (tested on n8n with task runner enabled)
- A Telegram bot (optional, for notifications — see setup below)
- Immich API key

---

## 🚀 Setup Guide

### Step 1 — Get your Immich API key

In Immich go to **Account Settings → API Keys → New API Key**. Copy it.

### Step 2 — Create a Header Auth credential in n8n

In n8n go to **Credentials → New → Header Auth** and set:
- Name: `Immich-API-Key`
- Header Name: `x-api-key`
- Header Value: *(your API key)*

### Step 3 — Import the workflow

Download the JSON file and in n8n go to **Workflows → Import from file**.

### Step 4 — Fill in the Config node

Open the workflow and update the **Config** node with your values:

| Field | Description |
|---|---|
| `Person1Name` | First person's name exactly as it appears in Immich |
| `Person2Name` | Second person's name exactly as it appears in Immich |
| `AlbumID` | UUID of your couple album (see below) |
| `ImmichBaseURL` | e.g. `http://192.168.1.100:2283` |
| `APIKey` | Your Immich API key |
| `DaysBack` | `2` recommended for daily runs |
| `TelegramBotToken` | From @BotFather (optional) |
| `TelegramChatID` | Your personal Telegram chat ID (optional) |

### Step 5 — Find your Album ID

In Immich, open the album you want to use and copy the UUID from the URL:
```
http://your-server/albums/df3f8f2d-0df3-45f8-8fad-743817dbfecf
                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                          this is your AlbumID
```

### Step 6 — Run the backfill first

Import `Immich-CoupleAlbum-FullLibrary.json`, fill in the Config node with the same values, and **run it manually once**. This populates the album from your full library history. It may take a few minutes depending on library size — check the execution logs to watch progress.

### Step 7 — Activate the daily workflow

Once the backfill is done, activate `Immich-CoupleAlbum-DailyIncremental-Telegram.json`. It will run automatically every night at 3:30am and keep the album updated.

---

## 📬 Telegram Notifications (Optional)

If you want daily Telegram alerts:

1. Message [@BotFather](https://t.me/BotFather) on Telegram → `/newbot` → follow the steps → copy the bot token
2. Start a chat with your new bot (search its username, hit Start)
3. Visit `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates` in your browser — your chat ID will be in the `"chat":{"id": ...}` field
4. Add both values to the Config node

You'll receive messages like:

```
📸 Immich Couple Album Update

✅ 3 new photo(s) added
👫 Dhaval & Aaska
🔁 12 already in album
🔍 15 couple photos found / 47 scanned
📅 Since: 2026-05-13
```

---

## ⚠️ Error Workflow Setup

For failure alerts to work, you need a separate n8n error workflow:

1. Create a new workflow in n8n called `error-workflow`
2. Add an **Error Trigger** node (this is required — without it, n8n won't list the workflow in the error workflow dropdown)
3. Connect it to a Code node that sends a Telegram message
4. In your main workflow go to **Settings → Error Workflow** and select it

> Without the Error Trigger node as the starting node, n8n will not recognise the workflow as a valid error workflow.

---

## 🔁 Workflow Architecture

```
Schedule Trigger (3:30am daily)
    ↓
Config (names, album ID, API key, Telegram)
    ↓
Find Person 1 (Immich API)
    ↓
Extract Person 1 ID
    ↓
Get Person 2 & Build Context (API call inside Code node — no Merge node needed)
    ↓
Fetch & Filter Couple Photos (paginated, takenAfter: last 2 days)
    ↓
Any couple photos? [IF]
    ↓ YES                    ↓ NO
Add to Album         No New Photos
    ↓                        ↓
Summary & Notify     Telegram: nothing found
(Telegram)
```

The key design decision: Person 2 is looked up **inside a Code node** using `helpers.httpRequest` rather than using n8n's Merge node — this avoids the "Fields to Match" error that appears in newer n8n versions when using Merge in multiplex mode.

---

## 🛠️ Customisation

**Change the schedule:** Edit the Schedule Trigger node — `triggerAtHour` and `triggerAtMinute` use 24h format.

**Change how many days back:** Update `DaysBack` in the Config node. Set to `7` if you upload photos weekly, `1` if you want tighter sync.

**Different timezone:** Set `GENERIC_TIMEZONE=Asia/Kolkata` (or your timezone) as an environment variable on your n8n server. This is a server-level setting and affects all schedule triggers.

**More than 2 people:** The filter `people.length !== 2` is what enforces the "exactly these two" rule. You could change this to `people.length >= 2` if you want to include group photos that contain both people.

---

## 🐛 Known Limitations

- Face recognition must be enabled in Immich and both people must be **named** for the search to work
- The `takenAfter` filter uses photo capture date — if you import old photos, run the backfill again to catch them
- Tested on n8n with the task runner enabled; `helpers.httpRequest` may not be available on very old n8n versions (`fetch` and `$http` are also unavailable in the Code node task runner environment on some builds)

---

## 🙋 A Note From Me

I'm not a developer or an automation expert — I built this because I wanted a couple album in Immich that actually stayed up to date without me manually curating it. It took a lot of trial and error to get the pagination, the person lookup, and the Merge node workaround all working together.

If you find bugs, have ideas for improvements, or know a better way to do any of this — **please open an issue or PR**. I'd genuinely love the feedback. Some things I'd love help with:

- Support for more than 2 people (family album, friend group album)
- A version that works without an API key by reusing n8n credentials
- Better handling of imported/backdated photos
- A solo photo variant for a single-person album

This is very much a work in progress and community suggestions are more than welcome. 🙏

---

## 📄 License

MIT — use it, modify it, share it.

---

## 🔗 Related

- [Immich Documentation](https://immich.app/docs)
- [Immich API Reference](https://immich.app/docs/api)
- [n8n Documentation](https://docs.n8n.io)
- [n8n Community](https://community.n8n.io)
