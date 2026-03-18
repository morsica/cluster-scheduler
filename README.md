# Cal.com Cluster Scheduling — n8n Workflow

Stop clients from scattering your day. This workflow forces all Cal.com bookings to stack as one unbroken block, no matter when clients book.

---

## The Problem

You share a booking link, by Monday, your calendar looks like this:

```
16:00 ░░░░ free
16:30 ████ CLIENT A
17:00 ░░░░ free
17:30 ░░░░ free
18:00 ████ CLIENT B
18:30 ░░░░ free
19:00 ████ CLIENT C
```

Three sessions. Four hours blocked. Two useless gaps in between.

Clients are not doing anything wrong — they just pick whatever time looks good to them. The problem is that Cal.com shows all available slots at once, so nothing stops them from spreading out.

---

## The Solution

This workflow controls which slots Cal.com shows. At any given moment, visitors only ever see **two slots**: one just before the current block of bookings, and one just after. Every new booking can only attach to the front or the back of the existing block — never in the middle, never somewhere random.

The result:

```
16:00 ████ CLIENT A   ← attached to front
16:30 ████ CLIENT B
17:00 ████ CLIENT C   ← attached to back
17:30 ░░░░ ← only slot visible to next client
```

It works by writing "fake busy" events to a second Google Calendar (called `BLOCKED - Auto`). Cal.com is connected to both your real calendar and this blocking calendar. Any slot that has a blocking event becomes invisible to visitors. The workflow rewrites these blocking events every 15 minutes so the page always reflects the current state.

---

## How It Works

### The Three States

**State 1 — No bookings yet**
Cal.com shows only the first two slots of your window (e.g. 16:00 and 16:30). Everything else is hidden. Two adjacent slots means two clients booking at the exact same time still can't create a gap.

**State 2 — At least one booking exists**
The workflow finds where your bookings start and end (the "cluster"), then opens exactly one slot before it and one slot after it. Everything else is hidden. Next client can only attach to one end or the other.

**State 3 — Window is full**
No action needed. Cal.com shows the day as fully booked on its own.

### What Counts as a Booking

Only Cal.com bookings shape the cluster. Google Calendar events that came from email invites or recurring blocks do not affect the cluster — they are walls that block themselves. This means an email invite at 19:30 does not shrink the open slots for your actual clients.

### The Schedule

A workflow runs every 15 minutes and processes the next 7 days. It does not wait for something to happen — it rewrites the blocking calendar continuously. This means a visitor at 4am, 9pm, or any time in between always sees an accurate booking page.

---

## Requirements

- [n8n](https://n8n.io) (self-hosted or cloud)
- A [Cal.com](https://cal.com) account (free tier works)
- A Google account with Google Calendar

---

## Setup

### Step 1 — Create the Blocking Calendar

1. Go to [Google Calendar](https://calendar.google.com)
2. In the left sidebar, click the **+** next to "Other calendars"
3. Select **Create new calendar**
4. Name it `BLOCKED - Auto`
5. Click **Create calendar**
6. Find the new calendar in the sidebar, click the three dots next to it → **Settings and sharing**
7. Scroll down to **Integrate calendar**
8. Copy the **Calendar ID** — it looks like `abc123xyz@group.calendar.google.com`
9. Save this ID — you will need it in Step 4

### Step 2 — Connect Both Calendars to Cal.com

1. Log into Cal.com
2. Go to **Settings → Calendars**
3. Connect your primary Google Calendar if not already connected
4. Under **Check for conflicts**, make sure both your primary calendar AND `BLOCKED - Auto` are checked
5. Go to **Availability** in the left sidebar
6. Set your booking window — e.g. Monday–Friday, 16:00–20:00
7. Set weekends to unavailable

> Cal.com will now treat any event in either calendar as busy time. The workflow writes to `BLOCKED - Auto` to hide slots. Your real bookings hide themselves through your primary calendar.

### Step 3 — Import the Workflow into n8n

1. Download `cluster_scheduler.json` from this repository
2. Open n8n → click **Workflows** → **Import from file**
3. Select the downloaded file
4. The workflow will open with a warning that credentials are missing — that is expected

### Step 4 — Connect Your Google Account

1. Inside the workflow, click any Google Calendar node
2. Click **Credentials → Create new**
3. Follow the OAuth prompt to connect your Google account
4. Apply the same credential to every Google Calendar node in the workflow

*Note*: If you have a self-hosted instance of n8n, you need to set up Google credentials using Google Cloud. Watch this youtube tutorial:
https://www.youtube.com/watch?v=lXNIteL16Z0

### Step 5 — Configure the Workflow

Open the **Config** node at the start of the workflow and fill in:

| Setting | Description | Example |
|---|---|---|
| `TZ` | Your timezone | `America/New_York` |
| `WINDOW_START_H` | Hour your booking window opens (24h) | `16` |
| `WINDOW_END_H` | Hour your booking window closes (24h) | `20` |
| `SLOT_MIN` | Session length in minutes | `30` |
| `DAYS_AHEAD` | How many days ahead to manage | `7` |
| `PRIMARY_CAL_ID` | Your main Google Calendar ID — usually your Gmail address | `you@gmail.com` |
| `BLOCKED_CAL_ID` | Calendar ID from Step 1 | `abc123@group.calendar.google.com` |

### Step 6 — Set the Schedule

1. Click the **Schedule Trigger** node
2. Set it to run every **15 minutes**
3. Make sure it is active

### Step 7 — Test It

Run the workflow manually once by clicking **Test workflow**. Then check your `BLOCKED - Auto` calendar in Google Calendar — you should see blocking events covering all slots except the first two for each upcoming weekday.

To verify Cal.com is working, open your booking page in an incognito window. You should only see two available slots for today (or the next available weekday).

---

## Workflow Structure

```
Schedule Trigger (every 15 min)
  → Generate Next 7 Days
  → Loop over each day
      → Get Primary Calendar Events (16:00–20:00 for that day)
      → Cluster Logic (Code Node)
      → Filter: reason = 'fake'
      → Delete existing BLOCKED events for that day
      → Create new BLOCKED events
```

---

## Frequently Asked Questions

**Will this affect bookings that already exist?**
No. The workflow only writes to `BLOCKED - Auto`. It never touches your primary calendar or existing Cal.com bookings.

**What happens if an existing booking gets cancelled?**
Within 15 minutes the workflow will run, see the cancellation, recalculate, and update the blocking calendar. The booking page will reflect the change within that window.

**What if I want to offer a slot outside the cluster to a specific client?**
Book them manually through the Cal.com admin panel. The workflow will pick up the new booking on its next run and adjust the cluster around it.

**What does the workflow do on weekends?**
Nothing. It detects weekends by day of week and skips them automatically.

**Does it handle email invites and recurring calendar blocks?**
Yes. Those events block themselves through Cal.com's primary calendar conflict check. The workflow recognises them by their calendar signature and does not let them affect the cluster shape — only Cal.com bookings do that.

**Can I change the window or slot length later?**
Yes. Update the values in the Config node and save. The next scheduled run will use the new settings.

---

## Identifying Cal.com Events

Cal.com events in Google Calendar have an `iCalUID` field that ends with `@Cal.com`. Google Calendar events (email invites, recurring blocks, manual entries) end with `@google.com`. The workflow uses this to tell them apart.

If this ever breaks — for example if Cal.com changes their event format — check the raw event data in the n8n execution log and update the filter in the Code node accordingly:

```javascript
const clientSessions = allEvents.filter(e =>
  e.iCalUID?.endsWith('@Cal.com')
);
```

---

## License

MIT — use it, change it, share it.
