# User Flows

## Overview

This document describes the key user journeys for both parents and children using KidSafe.

---

## Parent Flows

### PF01: First-Time Setup (Parent)

**Goal**: Parent creates account and sets up their first child's device.

```
┌─────────────────────────────────────────────────────────────────┐
│                    PARENT FIRST-TIME SETUP                       │
└─────────────────────────────────────────────────────────────────┘

1. DOWNLOAD & OPEN APP
   ┌─────────────────────┐
   │  Welcome to KidSafe │
   │                     │
   │  [Get Started]      │
   └─────────────────────┘

2. CREATE ACCOUNT
   ┌─────────────────────┐
   │  Sign up with       │
   │                     │
   │  [Phone Number]     │
   │  [Email]            │
   └─────────────────────┘

   If Phone:
   ┌─────────────────────┐
   │  +91 [__________]   │
   │                     │
   │  [Send OTP]         │
   └─────────────────────┘

   ┌─────────────────────┐
   │  Enter OTP          │
   │  [_ _ _ _ _ _]      │
   │                     │
   │  [Verify]           │
   └─────────────────────┘

3. ADD CHILD PROFILE
   ┌─────────────────────┐
   │  Who is this        │
   │  device for?        │
   │                     │
   │  Name: [Rahul___]   │
   │  Age:  [8 years ▼]  │
   │  Avatar: 🧒👦👧     │
   │                     │
   │  [Continue]         │
   └─────────────────────┘

4. SELECT PROTECTION LEVEL
   ┌─────────────────────┐
   │  Choose safety      │
   │  level for Rahul    │
   │                     │
   │  ○ Maximum (5-8)    │
   │    Only basics,     │
   │    very restricted  │
   │                     │
   │  ● Balanced (9-12)  │
   │    Core apps +      │
   │    parent picks     │
   │    [Recommended]    │
   │                     │
   │  ○ Trust (13+)      │
   │    More freedom,    │
   │    still monitored  │
   │                     │
   │  [Continue]         │
   └─────────────────────┘

5. GENERATE PAIRING QR
   ┌─────────────────────┐
   │  Scan this with     │
   │  Rahul's device     │
   │                     │
   │    ┌───────────┐    │
   │    │ █▀█ ▄▄▄ █ │    │
   │    │ ▄▄▄ █▀█ ▄ │    │
   │    │ █▀█ ▄▄▄ █ │    │
   │    └───────────┘    │
   │                     │
   │  Expires in 14:32   │
   │                     │
   │  [Having trouble?]  │
   └─────────────────────┘

6. PAIRING SUCCESS
   ┌─────────────────────┐
   │  🎉 Connected!      │
   │                     │
   │  Rahul's device     │
   │  is now set up.     │
   │                     │
   │  You can now:       │
   │  • See his activity │
   │  • Control apps     │
   │  • Set limits       │
   │                     │
   │  [Go to Dashboard]  │
   └─────────────────────┘
```

**Success Criteria**:
- Entire flow takes < 10 minutes
- No technical jargon shown
- Clear progress indication
- Can go back at any step

---

### PF02: Daily Check-In (Parent)

**Goal**: Parent checks on child's device status and activity.

```
1. OPEN APP → DASHBOARD
   ┌─────────────────────────────────────────┐
   │  Good morning, Priya! 👋                │
   ├─────────────────────────────────────────┤
   │  ┌─────────────────────────────────┐    │
   │  │ Rahul's Device        🟢 Online │    │
   │  │ ─────────────────────────────── │    │
   │  │ 🎵 Listening to: Kesariya       │    │
   │  │ 📍 Home                         │    │
   │  │ ⏱️ Today: 45m / 2h limit        │    │
   │  │ 🔋 72%                          │    │
   │  │                                  │    │
   │  │ [View Details]                   │    │
   │  └─────────────────────────────────┘    │
   │                                          │
   │  Quick Actions                           │
   │  [🔇Mute] [🔒Lock] [📍Locate] [⏸️Pause]  │
   │                                          │
   │  Today's Activity                        │
   │  ████████████░░░░░░░ 45m / 2h           │
   │  🎵 Spotify    25m                       │
   │  🎵 JioSaavn   20m                       │
   └─────────────────────────────────────────┘
```

**What Parent Can Do From Dashboard**:
- See real-time status
- See current activity
- Take quick actions
- View usage summary
- Navigate to detailed settings

---

### PF03: Managing Apps (Parent)

**Goal**: Parent enables or disables apps for child.

```
1. TAP "Manage Apps" → APP LIST
   ┌─────────────────────────────────────────┐
   │  ← Manage Apps                     🔍   │
   ├─────────────────────────────────────────┤
   │                                          │
   │  MUSIC                                   │
   │  ┌─────────────────────────────────┐    │
   │  │ 🎵 Spotify           [✓ ON ]    │    │
   │  │    No limit set                  │    │
   │  └─────────────────────────────────┘    │
   │  ┌─────────────────────────────────┐    │
   │  │ 🎵 JioSaavn          [✓ ON ]    │    │
   │  │    1 hour limit                  │    │
   │  └─────────────────────────────────┘    │
   │  ┌─────────────────────────────────┐    │
   │  │ 🎵 YouTube Music     [  OFF]    │    │
   │  │    Has video content             │    │
   │  └─────────────────────────────────┘    │
   │                                          │
   │  PAYMENT                                 │
   │  ┌─────────────────────────────────┐    │
   │  │ 💳 PhonePe           [✓ ON ]    │    │
   │  └─────────────────────────────────┘    │
   │  ┌─────────────────────────────────┐    │
   │  │ 💳 Google Pay        [  OFF]    │    │
   │  └─────────────────────────────────┘    │
   │                                          │
   │  COMMUNICATION                           │
   │  ┌─────────────────────────────────┐    │
   │  │ 💬 WhatsApp          [✓ ON ]    │    │
   │  │    30 min limit                  │    │
   │  └─────────────────────────────────┘    │
   │                                          │
   └─────────────────────────────────────────┘

2. TAP ON APP → APP SETTINGS
   ┌─────────────────────────────────────────┐
   │  ← WhatsApp                             │
   ├─────────────────────────────────────────┤
   │                                          │
   │  Allow this app        [✓ ON ]          │
   │                                          │
   │  ───────────────────────────────────    │
   │                                          │
   │  Daily Time Limit                        │
   │  [No Limit ▼]                           │
   │   • No limit                             │
   │   • 15 minutes                           │
   │   • 30 minutes ✓                        │
   │   • 1 hour                               │
   │   • 2 hours                              │
   │                                          │
   │  ───────────────────────────────────    │
   │                                          │
   │  Available During                        │
   │  [Always ▼]                             │
   │   • Always                               │
   │   • School hours only                    │
   │   • After school only                    │
   │   • Custom...                            │
   │                                          │
   │  [Save Changes]                          │
   └─────────────────────────────────────────┘
```

---

### PF04: Setting Screen Time (Parent)

**Goal**: Parent configures daily usage limits.

```
1. TAP "Screen Time" → SETTINGS
   ┌─────────────────────────────────────────┐
   │  ← Screen Time                          │
   ├─────────────────────────────────────────┤
   │                                          │
   │  Daily Limit                             │
   │  ┌─────────────────────────────────┐    │
   │  │         ┌─────────────┐         │    │
   │  │    ◀    │   2 hours   │    ▶    │    │
   │  │         └─────────────┘         │    │
   │  │                                  │    │
   │  │  ○ Same every day                │    │
   │  │  ● Different on weekends         │    │
   │  └─────────────────────────────────┘    │
   │                                          │
   │  Weekday: 2 hours                        │
   │  Weekend: 3 hours                        │
   │                                          │
   │  ───────────────────────────────────    │
   │                                          │
   │  Bedtime                                 │
   │  ┌─────────────────────────────────┐    │
   │  │  Device goes quiet at:          │    │
   │  │                                  │    │
   │  │  [8:30 PM] to [7:00 AM]         │    │
   │  │                                  │    │
   │  │  [✓] Allow alarm to ring        │    │
   │  │  [✓] Allow emergency calls      │    │
   │  └─────────────────────────────────┘    │
   │                                          │
   │  ───────────────────────────────────    │
   │                                          │
   │  Downtime                                │
   │  ┌─────────────────────────────────┐    │
   │  │  [ ] Enable Homework Time       │    │
   │  │  Block entertainment during     │    │
   │  │  homework hours                  │    │
   │  └─────────────────────────────────┘    │
   │                                          │
   │  [Save Changes]                          │
   └─────────────────────────────────────────┘
```

---

### PF05: Responding to Panic Alert (Parent)

**Goal**: Parent receives and responds to child's panic alert.

```
1. NOTIFICATION RECEIVED
   ┌─────────────────────────────────────────┐
   │  🆘 PANIC ALERT                          │
   │  Rahul pressed the panic button!         │
   │  Tap to see location                     │
   └─────────────────────────────────────────┘
   (High priority - sounds alarm even in DND)

2. TAP NOTIFICATION → ALERT SCREEN
   ┌─────────────────────────────────────────┐
   │  🆘 PANIC ALERT                    ✕    │
   ├─────────────────────────────────────────┤
   │                                          │
   │  Rahul needs help!                       │
   │  3:45 PM, 2 minutes ago                  │
   │                                          │
   │  ┌─────────────────────────────────┐    │
   │  │                                  │    │
   │  │         [MAP VIEW]               │    │
   │  │            📍                    │    │
   │  │    Near: MG Road Metro           │    │
   │  │                                  │    │
   │  └─────────────────────────────────┘    │
   │                                          │
   │  ┌─────────────┐  ┌─────────────┐       │
   │  │ 📞 Call     │  │ 📍 Navigate │       │
   │  │    Rahul    │  │    to here  │       │
   │  └─────────────┘  └─────────────┘       │
   │                                          │
   │  [I've reached Rahul - All OK]          │
   │                                          │
   └─────────────────────────────────────────┘
```

---

## Child Flows

### CF01: First Boot (Child Device)

**Goal**: Child device gets set up and paired with parent.

```
1. DEVICE POWERS ON
   ┌─────────────────────┐
   │                     │
   │    [KidSafe Logo]   │
   │                     │
   │    Setting up...    │
   │                     │
   └─────────────────────┘

2. PAIRING SCREEN
   ┌─────────────────────┐
   │                     │
   │  Scan the code from │
   │  parent's phone     │
   │                     │
   │    ┌───────────┐    │
   │    │  [CAMERA] │    │
   │    │    📷     │    │
   │    └───────────┘    │
   │                     │
   │  Ask your parent    │
   │  to open KidSafe    │
   │  and tap "Add       │
   │  Device"            │
   │                     │
   └─────────────────────┘

3. PAIRING IN PROGRESS
   ┌─────────────────────┐
   │                     │
   │    Connecting...    │
   │    ◐◓◑◒             │
   │                     │
   │  Don't turn off     │
   │  the device         │
   │                     │
   └─────────────────────┘

4. SETUP COMPLETE
   ┌─────────────────────┐
   │                     │
   │    🎉 All done!     │
   │                     │
   │    Hi, Rahul!       │
   │                     │
   │  Your device is     │
   │  ready to use.      │
   │                     │
   │    [Let's Go!]      │
   │                     │
   └─────────────────────┘
```

---

### CF02: Normal Usage (Child - Kids Mode)

**Goal**: Child uses device in Kids Mode.

```
1. HOME SCREEN (KIDS MODE)
   ┌─────────────────────────────────────────┐
   │                                          │
   │         Good morning, Rahul! 🌞          │
   │                                          │
   │    ┌──────────┐    ┌──────────┐         │
   │    │   🎵     │    │   🎵     │         │
   │    │ Spotify  │    │ JioSaavn │         │
   │    └──────────┘    └──────────┘         │
   │                                          │
   │    ┌──────────┐    ┌──────────┐         │
   │    │   💬     │    │   ⏰     │         │
   │    │ WhatsApp │    │  Alarm   │         │
   │    └──────────┘    └──────────┘         │
   │                                          │
   │    ┌──────────┐    ┌──────────┐         │
   │    │   📚     │    │   🧮     │         │
   │    │ BYJU'S   │    │  Calc    │         │
   │    └──────────┘    └──────────┘         │
   │                                          │
   │  ┌─────────────────────────────────┐    │
   │  │  🆘  I need help               │    │
   │  └─────────────────────────────────┘    │
   │                                          │
   │              🔋 78%  📶                  │
   └─────────────────────────────────────────┘

2. TAP APP → APP OPENS
   (Standard app experience)

3. PRESS HOME → RETURN TO LAUNCHER
   (Always returns to KidSafe launcher)
```

---

### CF03: Triggering Panic Alert (Child)

**Goal**: Child sends emergency alert to parent.

```
1. LONG PRESS PANIC BUTTON (3 seconds)
   ┌─────────────────────────────────────────┐
   │                                          │
   │  ┌─────────────────────────────────┐    │
   │  │  🆘  Hold to send alert...      │    │
   │  │  ████████░░░░░░░                │    │
   │  └─────────────────────────────────┘    │
   │                                          │
   └─────────────────────────────────────────┘

2. ALERT SENT
   ┌─────────────────────────────────────────┐
   │                                          │
   │              ✓ Alert Sent!               │
   │                                          │
   │         Your parent has been             │
   │         notified and can see             │
   │         your location.                   │
   │                                          │
   │         Stay where you are.              │
   │                                          │
   │              [OK]                         │
   │                                          │
   └─────────────────────────────────────────┘
```

---

### CF04: Screen Time Limit Reached (Child)

**Goal**: Child understands usage limit and can request more time.

```
1. WARNING (10 MINUTES LEFT)
   ┌─────────────────────────────────────────┐
   │  ⏰ 10 minutes left today               │
   └─────────────────────────────────────────┘
   (Brief notification, does not interrupt)

2. LIMIT REACHED
   ┌─────────────────────────────────────────┐
   │                                          │
   │              ⏰                          │
   │                                          │
   │         Time's up for today!             │
   │                                          │
   │    You've used your 2 hours.             │
   │    See you tomorrow!                     │
   │                                          │
   │    Resets in: 6h 23m                     │
   │                                          │
   │  ─────────────────────────────────      │
   │                                          │
   │    [🆘 Emergency]                        │
   │    (Panic button still works)            │
   │                                          │
   └─────────────────────────────────────────┘
```

---

### CF05: Bedtime Mode (Child)

**Goal**: Child's device locks at bedtime.

```
1. BEDTIME STARTS
   ┌─────────────────────────────────────────┐
   │                                          │
   │              🌙                          │
   │                                          │
   │           Time for bed!                  │
   │                                          │
   │    Your device will wake up at           │
   │    7:00 AM tomorrow.                     │
   │                                          │
   │    ─────────────────────────            │
   │                                          │
   │    ⏰ Your alarm will still ring         │
   │    🆘 Emergency button still works       │
   │                                          │
   │           Good night, Rahul!             │
   │                                          │
   └─────────────────────────────────────────┘
```

---

### CF06: Switching to Standard Mode (Child)

**Goal**: Child (or parent on device) switches to Standard Mode.

```
1. TAP MODE SWITCH ICON
   (Small icon in corner, not prominent)

2. PIN ENTRY
   ┌─────────────────────────────────────────┐
   │                                          │
   │         Enter Parent PIN                 │
   │                                          │
   │         ● ● ○ ○                          │
   │                                          │
   │    ┌───┐ ┌───┐ ┌───┐                    │
   │    │ 1 │ │ 2 │ │ 3 │                    │
   │    └───┘ └───┘ └───┘                    │
   │    ┌───┐ ┌───┐ ┌───┐                    │
   │    │ 4 │ │ 5 │ │ 6 │                    │
   │    └───┘ └───┘ └───┘                    │
   │    ┌───┐ ┌───┐ ┌───┐                    │
   │    │ 7 │ │ 8 │ │ 9 │                    │
   │    └───┘ └───┘ └───┘                    │
   │         ┌───┐ ┌───┐                     │
   │         │ 0 │ │ ⌫ │                     │
   │         └───┘ └───┘                     │
   │                                          │
   │           [Cancel]                       │
   │                                          │
   └─────────────────────────────────────────┘

3. SUCCESS → STANDARD MODE
   ┌─────────────────────────────────────────┐
   │  12:34                     🔋 78%  📶   │
   ├─────────────────────────────────────────┤
   │                                          │
   │  ┌────┐ ┌────┐ ┌────┐ ┌────┐           │
   │  │ 🎵 │ │ 🎵 │ │ 💬 │ │ 💳 │           │
   │  └────┘ └────┘ └────┘ └────┘           │
   │  ┌────┐ ┌────┐ ┌────┐ ┌────┐           │
   │  │ 📚 │ │ ⏰ │ │ 🧮 │ │ ⚙️ │           │
   │  └────┘ └────┘ └────┘ └────┘           │
   │                                          │
   │  (4x4 grid with more apps)              │
   │                                          │
   └─────────────────────────────────────────┘

4. WRONG PIN
   ┌─────────────────────────────────────────┐
   │         ❌ Wrong PIN                     │
   │         2 attempts remaining             │
   └─────────────────────────────────────────┘

   (After 3 wrong attempts: locked for 1 min)
```

---

## Edge Case Flows

### EF01: Device Offline

When child device loses internet:
- Commands queue on Firebase
- Parent sees "Offline" status
- When device reconnects:
  - Queued commands execute
  - Status syncs
  - Parent sees "Online"

### EF02: Parent Forgets PIN

- Parent can reset PIN in account settings
- Requires email/phone verification
- New PIN syncs to device when online

### EF03: Multiple Children

- Parent adds second child via same flow
- Dashboard shows both devices
- Each device has independent settings
- Family plan required for > 2 devices

### EF04: App Update Available

- Device checks for updates daily
- Updates install silently (Device Owner)
- Parent app updates via app stores
- Version mismatch handled gracefully
