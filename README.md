# WhatsApp Broker Group Moderation Bot

Watches your broker WhatsApp groups, flags spam / wrong-area / wrong-type posts using AI,
sends you the decision to approve on WhatsApp itself (your own "Message Yourself" chat),
then executes the action (delete + message the broker) and logs everything to your
existing Google Sheet.

**Nothing happens automatically without your approval** — every delete/message action
waits for you to reply `1`, `2`, or `3` in your own chat.

---

## 1. What you need before starting

1. A VPS (any small Linux box — 1 vCPU / 1GB RAM is enough) with Node.js 18+ installed.
2. An Anthropic API key (from console.anthropic.com) — this powers the text understanding.
3. A Google Cloud "service account" with access to your Google Sheet (steps below).
4. The WhatsApp number that is already a member/admin of all 50+ groups.

---

## 2. Google Sheets setup

### 2a. Create a service account (one-time)

1. Go to https://console.cloud.google.com/ → create a new project (or use an existing one).
2. Enable the **Google Sheets API** for that project (APIs & Services → Enable APIs → search "Google Sheets API").
3. Go to **IAM & Admin → Service Accounts → Create Service Account**. Name it anything (e.g. `broker-bot`).
4. Once created, open it → **Keys → Add Key → Create new key → JSON**. This downloads a `.json` file.
5. Rename that file to `service-account.json` and put it in this project's `config/` folder.
6. Open the JSON file and copy the `client_email` value (looks like `broker-bot@your-project.iam.gserviceaccount.com`).
7. Open your actual Google Sheet → click **Share** → paste that email in → give it **Editor** access.

### 2b. Add a "Group Config" tab to your existing sheet

This is the one new tab the bot needs — it's the lookup table that tells the bot which
area(s), deal type, and property type each group covers. Add a tab named exactly
`Group Config` with these columns (row 1 = header):

| Group Name | WhatsApp Group JID | Area(s) | Deal Type | Property Type | Status | Invite Link |
|---|---|---|---|---|---|---|
| Arabian Ranches - Sale | | Arabian Ranches | sale | residential | active | https://chat.whatsapp.com/xxxx |
| Downtown - Sale | | Downtown, Downtown Dubai, Burj Khalifa | sale | residential | active | https://chat.whatsapp.com/yyyy |
| Business Bay - Rentals | | Business Bay | rent | residential | pending | |

Notes:
- **Group Name** should match (or closely resemble) the actual WhatsApp group name.
- **WhatsApp Group JID** — leave blank at first. Once the bot sees a message from that
  group, you can fill this in manually (see "Finding a group's JID" below) so that adding
  people to it automatically works.
- **Area(s)**: comma-separated. You only need to list the master community — the AI
  automatically understands that "Arabian Ranches 3" or "Saheel" belong to
  "Arabian Ranches", so you don't need to list every sub-community.
- **Status**: `active` (enough members, ready to receive posts) or `pending` (matches
  your existing "wrong requests & group pending" tab) — this controls which message
  template variant gets used (with or without the invite link).

You do **not** need to change any of your existing tabs — the bot writes to `correct`
and `wrong requests & group ready` (and `spam & ads` for spam) using the same columns
you already have, plus one extra "Decision" column at the end.

---

## 3. Install & configure

```bash
# on your VPS
git clone <wherever you put this project> whatsapp-broker-bot
cd whatsapp-broker-bot
npm install

cp .env.example .env
nano .env   # fill in ANTHROPIC_API_KEY, GOOGLE_SHEET_ID, OWNER_WHATSAPP_NUMBER
```

`GOOGLE_SHEET_ID` is the long ID in your sheet's URL:
`https://docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`

`OWNER_WHATSAPP_NUMBER` is your own number, digits only, with country code, no `+` or
spaces (e.g. `971501234567`).

---

## 4. First run — scanning the QR code

```bash
npm start
```

A QR code will print in the terminal. Open WhatsApp on the phone whose number is in
the groups → **Settings → Linked Devices → Link a Device** → scan it.

The session is saved to `data/auth/`, so you only need to do this once — as long as
that folder isn't deleted, restarts won't ask for the QR again.

---

## 5. Keeping it running 24/7 on the VPS

Use `pm2` so the bot restarts automatically if it crashes or the VPS reboots:

```bash
npm install -g pm2
pm2 start src/index.js --name broker-bot
pm2 save
pm2 startup   # follow the printed instructions to enable on-boot start
```

Useful commands:
```bash
pm2 logs broker-bot     # see live logs
pm2 restart broker-bot  # restart after editing .env or Group Config
```

---

## 6. How approvals work

Whenever the bot flags something, it sends a message to **your own "Message Yourself"
chat** in WhatsApp, like:

```
🚩 #12 — Wrong Area
Group: Business Bay - Sale
From: 971581433893
Extracted: 3BR villa, Downtown, 4.2M
Area mentioned: Downtown
Suggested correct group: Downtown - Sale (active)

Suggested action: Delete post + message broker with the correct-group template

Reply: 12 1 = do it | 12 2 = leave it | 12 3 = decide manually
```

Reply in that same chat with `12 1` (or just `1` if it's the only thing pending) to
execute, `2` to leave it as-is, or `3` to skip and handle it yourself.

---

## 7. Finding a group's JID (to fill in the Group Config tab)

The easiest way: once the bot is running, check its logs (`pm2 logs broker-bot`) —
when a message comes in from a group without a JID configured, you'll see the group's
name and JID together in the console output. Copy the JID (looks like
`123456789-987654321@g.us`) into the matching row's "WhatsApp Group JID" column.

Until a group's JID is filled in, the bot will still flag wrong-area/wrong-type posts
and message the broker with the invite link — it just won't be able to auto-add them
to the group (WhatsApp requires the JID for that), so the broker joins via the link
instead. That's a fine fallback, not a blocker.

---

## 8. What's not fully automatic (by design)

- **Whether the property actually exists in the stated area** — the bot only checks
  that the *text* of the post matches the group's configured area. It cannot verify
  real-world facts. Use the quick manual check you already do for that.
- **The "sale in rental group, group not active" template** — you hadn't sent me the
  exact wording for this one yet, so `src/templates.js` currently reuses the closest
  equivalent phrasing. Send me the real wording whenever you have it and I'll drop it in.
- **The encouragement/"Prive property" message** is a manual quick-send, not part of
  the automated flow — see `templates.encouragement()`, which you'd wire up to a small
  script or command whenever you want to send it to a specific broker.

---

## 9. Cost & rate-limit notes

- Every non-spam post triggers one Claude API call (cheap — a few post's-worth of text).
  At high volume across 50+ groups this is still a small monthly cost, but worth
  keeping an eye on via the Anthropic console.
- WhatsApp itself has no official rate limit for reading messages, but deleting
  messages and adding participants in quick succession across many groups can look
  bot-like. Since every action already waits for your manual approval, this naturally
  paces things — just avoid mass-approving dozens of tickets in a few seconds.
