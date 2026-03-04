# UI Specifications

## Design Principles

### For Parent App

| Principle | Implementation |
|-----------|----------------|
| **No jargon** | Never use "MDM", "DPC", "whitelist", "policy" |
| **Visual controls** | Sliders, toggles, icons instead of text inputs |
| **Instant feedback** | Show loading states, confirmations |
| **Scannable** | Key info visible at a glance |
| **Trust** | Professional, reliable, secure feeling |

### For Child Device (Launcher)

| Principle | Implementation |
|-----------|----------------|
| **Age-appropriate** | Large icons, simple words |
| **Friendly** | Warm greetings, playful colors |
| **Clear boundaries** | Child knows limits exist |
| **Non-frustrating** | Clear feedback when restricted |
| **Safe** | Panic button always accessible |

---

## Color System

### Parent App Colors

```
Primary:       #6366F1 (Indigo)      - Actions, links
Secondary:     #10B981 (Green)       - Success, online
Error:         #EF4444 (Red)         - Errors, alerts
Warning:       #F59E0B (Amber)       - Warnings
Background:    #F9FAFB (Light Gray)  - Page background
Surface:       #FFFFFF (White)       - Cards, modals
Text Primary:  #111827 (Dark Gray)   - Main text
Text Secondary:#6B7280 (Gray)        - Secondary text

Status Colors:
Online:        #22C55E (Green)
Offline:       #9CA3AF (Gray)
Kids Mode:     #8B5CF6 (Purple)
Standard Mode: #3B82F6 (Blue)
Panic Alert:   #DC2626 (Red)
```

### Child Device Colors (Kids Mode)

```
Background:    #FEF3C7 (Warm Yellow)  - Friendly, warm
App Icons:     Bright, saturated colors
Text:          #1F2937 (Dark)         - High contrast
Panic Button:  #DC2626 (Red)          - Attention
Greeting:      #059669 (Green)        - Positive
```

### Child Device Colors (Standard Mode)

```
Background:    #F3F4F6 (Neutral Gray)
App Icons:     Standard Android style
Text:          #1F2937 (Dark)
Accent:        #3B82F6 (Blue)
```

---

## Typography

### Parent App

| Element | Font | Size | Weight |
|---------|------|------|--------|
| Heading 1 | System | 24sp | Bold |
| Heading 2 | System | 20sp | SemiBold |
| Heading 3 | System | 18sp | SemiBold |
| Body | System | 16sp | Regular |
| Caption | System | 14sp | Regular |
| Button | System | 16sp | SemiBold |

### Child Device

| Element | Font | Size | Weight |
|---------|------|------|--------|
| Greeting | System | 24sp | Bold |
| App Label | System | 14sp | Medium |
| Status | System | 12sp | Regular |
| Panic Button | System | 18sp | Bold |

---

## Parent App Screens

### PA01: Dashboard

```
┌─────────────────────────────────────────────────────────────┐
│  ≡                              [Profile Avatar]            │  <- Nav bar
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Good morning, Priya! 👋                                    │  <- Greeting
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                       │   │
│  │  [Avatar]  Rahul's Device           🟢 Online        │   │
│  │                                                       │   │
│  │  🎵 Now: Kesariya - Arijit Singh                     │   │  <- Device Card
│  │  📍 Home                                              │   │
│  │  ⏱️ 45m / 2h                    🔋 72%               │   │
│  │                                                       │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │  [🔇 Mute] [🔒 Lock] [📍 Locate] [⏸️ Pause]    │  │   │  <- Quick Actions
│  │  └────────────────────────────────────────────────┘  │   │
│  │                                                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Today's Activity                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  ████████████████░░░░░░░░░░░░  45m / 2h (37%)       │   │  <- Progress Bar
│  │                                                       │   │
│  │  🎵 Spotify           25m    ████████████           │   │
│  │  🎵 JioSaavn          15m    ██████                 │   │  <- App Usage
│  │  💬 WhatsApp           5m    ██                     │   │
│  │                                                       │   │
│  │  [See Full Report →]                                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  [+ Add Another Device]                                      │  <- Add Device
│                                                              │
├─────────────────────────────────────────────────────────────┤
│  [🏠 Home]    [📱 Apps]    [⏱️ Time]    [📍 Location]      │  <- Bottom Nav
└─────────────────────────────────────────────────────────────┘
```

**Specifications**:
- Device card: 16dp corner radius, subtle shadow
- Status dot: 8dp diameter
- Quick action buttons: 48dp touch target
- Progress bar: 8dp height, rounded caps
- Bottom nav: 56dp height, icons 24dp

---

### PA02: App Management

```
┌─────────────────────────────────────────────────────────────┐
│  ←  Manage Apps                                    🔍       │  <- Header
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [All] [Music] [Education] [Communication] [Other]          │  <- Filter Tabs
│                                                              │
│  MUSIC                                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  [🎵]  Spotify                                       │   │
│  │        Music streaming                    [✓ ON ]   │   │
│  │        No time limit                                 │   │
│  │  ─────────────────────────────────────────────────  │   │  <- App Row
│  │  [🎵]  JioSaavn                                      │   │
│  │        Music streaming                    [✓ ON ]   │   │
│  │        1 hour daily limit                            │   │
│  │  ─────────────────────────────────────────────────  │   │
│  │  [🎵]  YouTube Music                                 │   │
│  │        Music + video                      [  OFF]   │   │
│  │        ⚠️ Contains video content                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  PAYMENT                                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  [💳]  PhonePe                                       │   │
│  │        Payment app                        [✓ ON ]   │   │
│  │  ─────────────────────────────────────────────────  │   │
│  │  [💳]  Google Pay                                    │   │
│  │        Payment app                        [  OFF]   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  COMMUNICATION                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  [💬]  WhatsApp                                      │   │
│  │        Messaging                          [✓ ON ]   │   │
│  │        30 min daily limit                            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Specifications**:
- App icon: 40dp
- App row height: 72dp minimum
- Toggle: Material 3 switch
- Section headers: 14sp, medium weight, gray
- Category filter: Horizontal scroll, pill shape

---

### PA03: App Detail

```
┌─────────────────────────────────────────────────────────────┐
│  ←  WhatsApp                                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│         [💬 Large App Icon - 72dp]                          │
│         WhatsApp                                             │
│         Communication                                        │
│                                                              │
│  ───────────────────────────────────────────────────────   │
│                                                              │
│  Allow this app                             [✓ ON ]         │
│                                                              │
│  ───────────────────────────────────────────────────────   │
│                                                              │
│  USAGE LIMITS                                                │
│                                                              │
│  Daily time limit                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │   [No limit]  [15m]  [30m✓]  [1h]  [2h]             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ───────────────────────────────────────────────────────   │
│                                                              │
│  SCHEDULE                                                    │
│                                                              │
│  Available during                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  ○ Always available                                  │   │
│  │  ○ Only after school (3 PM - 8 PM)                  │   │
│  │  ● Custom schedule                                   │   │
│  │     Mon-Fri: 4:00 PM - 8:00 PM                      │   │
│  │     Sat-Sun: 10:00 AM - 8:00 PM                     │   │
│  │     [Edit Schedule]                                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ───────────────────────────────────────────────────────   │
│                                                              │
│  USAGE THIS WEEK                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Mon  Tue  Wed  Thu  Fri  Sat  Sun                  │   │
│  │  ██   ███  █    ██   █    ████ ███                  │   │
│  │  12m  28m  5m   15m  8m   45m  32m                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

### PA04: Screen Time Settings

```
┌─────────────────────────────────────────────────────────────┐
│  ←  Screen Time                                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  DAILY LIMIT                                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                       │   │
│  │         ◀─────────●─────────────────────────▶        │   │  <- Slider
│  │                                                       │   │
│  │                  2 hours                              │   │
│  │                                                       │   │
│  │  30m   1h   1.5h   2h   2.5h   3h   3.5h   4h   ∞   │   │  <- Labels
│  │                                                       │   │
│  │  ○ Same every day                                    │   │
│  │  ● Different for weekends                            │   │
│  │                                                       │   │
│  │  Weekday limit: 2 hours                              │   │
│  │  Weekend limit: 3 hours  [Edit]                      │   │
│  │                                                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  BEDTIME                                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                       │   │
│  │  Enable bedtime                          [✓ ON ]     │   │
│  │                                                       │   │
│  │  Device goes quiet from:                             │   │
│  │                                                       │   │
│  │  ┌───────────┐          ┌───────────┐               │   │
│  │  │  8:30 PM  │    to    │  7:00 AM  │               │   │  <- Time Pickers
│  │  └───────────┘          └───────────┘               │   │
│  │                                                       │   │
│  │  During bedtime:                                     │   │
│  │  [✓] Alarm can still ring                           │   │
│  │  [✓] Emergency button works                         │   │
│  │  [ ] Allow reading apps                             │   │
│  │                                                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  DOWNTIME                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                       │   │
│  │  Enable homework time                    [  OFF]     │   │
│  │                                                       │   │
│  │  Block entertainment apps during                     │   │
│  │  homework hours. Only educational                    │   │
│  │  apps will be available.                             │   │
│  │                                                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│                    [Save Changes]                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

### PA05: Location Screen

```
┌─────────────────────────────────────────────────────────────┐
│  ←  Location                                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                       │   │
│  │                    [MAP]                              │   │
│  │                                                       │   │
│  │              📍 Rahul                                 │   │
│  │               Current                                 │   │
│  │                                                       │   │
│  │          🏠──────────┐                               │   │  <- Map View
│  │          Home        │                               │   │
│  │          Safe Zone   │                               │   │
│  │                      │                               │   │
│  │                      └──────────🏫                   │   │
│  │                              School                   │   │
│  │                              Safe Zone                │   │
│  │                                                       │   │
│  │  [−]  [+]                           [📍 Center]      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  CURRENT LOCATION                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  📍 Near MG Road Metro Station                       │   │
│  │     Last updated: 2 min ago                          │   │
│  │     Accuracy: ±15m                                   │   │
│  │                                                       │   │
│  │  [🔄 Refresh Now]                                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  SAFE ZONES                                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  🏠 Home                                     [Edit]  │   │
│  │     200m radius • Alert when leaves                  │   │
│  │  ─────────────────────────────────────────────────  │   │
│  │  🏫 School                                   [Edit]  │   │
│  │     300m radius • Alert when leaves                  │   │
│  │  ─────────────────────────────────────────────────  │   │
│  │  [+ Add Safe Zone]                                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

### PA06: Panic Alert Screen

```
┌─────────────────────────────────────────────────────────────┐
│                                                    [✕]      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                         🆘                                   │
│                                                              │
│                    PANIC ALERT                               │
│                                                              │
│              Rahul needs your help!                          │
│                                                              │
│              3:45 PM • 2 minutes ago                         │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                       │   │
│  │                    [MAP]                              │   │
│  │                      📍                               │   │
│  │                                                       │   │  <- Map View
│  │              Near: MG Road Metro                      │   │
│  │              Accuracy: ±10m                           │   │
│  │                                                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌───────────────────┐    ┌───────────────────┐            │
│  │                   │    │                   │            │
│  │   📞 Call Rahul   │    │  📍 Navigate      │            │  <- Action Buttons
│  │                   │    │                   │            │
│  └───────────────────┘    └───────────────────┘            │
│                                                              │
│  ─────────────────────────────────────────────────────     │
│                                                              │
│  [ ✓ I've reached Rahul - Mark as resolved ]                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Specifications**:
- Red accent color (#DC2626)
- Prominent, unmissable design
- Large touch targets for action buttons
- Auto-center map on child location

---

## Child Device Screens

### CD01: Kids Mode Home

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│                                                              │
│              Good morning, Rahul! 🌞                         │
│                                                              │
│                                                              │
│       ┌────────────────┐     ┌────────────────┐            │
│       │                │     │                │            │
│       │      🎵        │     │      🎵        │            │
│       │                │     │                │            │
│       │    Spotify     │     │   JioSaavn     │            │
│       │                │     │                │            │
│       └────────────────┘     └────────────────┘            │
│                                                              │
│       ┌────────────────┐     ┌────────────────┐            │
│       │                │     │                │            │
│       │      💬        │     │      ⏰        │            │
│       │                │     │                │            │
│       │   WhatsApp     │     │    Alarm       │            │
│       │                │     │                │            │
│       └────────────────┘     └────────────────┘            │
│                                                              │
│       ┌────────────────┐     ┌────────────────┐            │
│       │                │     │                │            │
│       │      📚        │     │      🧮        │            │
│       │                │     │                │            │
│       │    BYJU'S      │     │   Calculator   │            │
│       │                │     │                │            │
│       └────────────────┘     └────────────────┘            │
│                                                              │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                       │   │
│  │       🆘   I need help (hold for 3 seconds)          │   │  <- Panic Button
│  │                                                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│                    🔋 78%    📶    🔊                       │  <- Status Bar
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Specifications**:
- App icons: 80dp with 16dp padding
- Grid: 2 columns, 3 rows maximum
- App labels: 14sp, centered below icon
- Panic button: Full width minus 32dp margins, 64dp height
- Background: Warm gradient (#FEF3C7 to #FFFBEB)
- Greeting: 24sp bold, centered

---

### CD02: Standard Mode Home

```
┌─────────────────────────────────────────────────────────────┐
│  12:34                                   🔋 78%  📶  🔊     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [🎵]  [🎵]  [💬]  [💳]                                    │
│  Spot  Jio   WApp  PPe                                      │
│                                                              │
│  [📚]  [🧮]  [⏰]  [⚙️]                                     │
│  BYJU  Calc  Alrm  Set                                      │
│                                                              │
│  [📖]  [🎮]  [📷]  [📁]                                     │
│  Read  Game  Cam   File                                     │
│                                                              │
│  (Empty slots if fewer apps)                                │
│                                                              │
│                                                              │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  🎵 Now Playing: Kesariya                           │   │
│  │  ▶  ─────────────●─────── 2:34                      │   │  <- Mini Player
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│                                                              │
│  ───────────────────────────────────────────────────────   │
│         ○              ○              ○                     │
│        Home         Recent         Search                   │  <- Nav Dots
└─────────────────────────────────────────────────────────────┘
```

**Specifications**:
- App icons: 56dp
- Grid: 4 columns, dynamic rows
- App labels: 12sp, centered
- Background: Neutral (#F3F4F6)
- Mini player: Only shown when music playing
- Mode switch: Small icon in settings area

---

### CD03: Screen Time Limit Screen

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│                                                              │
│                          ⏰                                  │
│                                                              │
│                   Time's up for today!                       │
│                                                              │
│                                                              │
│              You've used your 2 hours.                       │
│                                                              │
│              Come back tomorrow!                             │
│                                                              │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                       │   │
│  │            ⏱️ Resets in 6h 23m                       │   │
│  │                                                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│                                                              │
│                                                              │
│                                                              │
│  ─────────────────────────────────────────────────────     │
│                                                              │
│                    [🆘 Emergency]                            │
│              (Panic button still works)                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

### CD04: Bedtime Screen

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│                                                              │
│                                                              │
│                          🌙                                  │
│                                                              │
│                                                              │
│                    Time for bed!                             │
│                                                              │
│                                                              │
│             Your device will wake up at                      │
│                     7:00 AM                                  │
│                                                              │
│                                                              │
│  ─────────────────────────────────────────────────────     │
│                                                              │
│           ⏰ Your alarm will still ring                      │
│                                                              │
│           🆘 Emergency button still works                    │
│                                                              │
│  ─────────────────────────────────────────────────────     │
│                                                              │
│                                                              │
│                  Good night, Rahul! 💤                       │
│                                                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Specifications**:
- Background: Dark blue gradient (#1E3A5F to #0F172A)
- Text: White (#FFFFFF)
- Moon emoji: Large, centered
- Calming, non-stimulating design

---

### CD05: PIN Entry Dialog

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│                    Enter Parent PIN                          │
│                                                              │
│                    ● ● ○ ○                                   │
│                                                              │
│         ┌─────┐   ┌─────┐   ┌─────┐                         │
│         │  1  │   │  2  │   │  3  │                         │
│         └─────┘   └─────┘   └─────┘                         │
│                                                              │
│         ┌─────┐   ┌─────┐   ┌─────┐                         │
│         │  4  │   │  5  │   │  6  │                         │
│         └─────┘   └─────┘   └─────┘                         │
│                                                              │
│         ┌─────┐   ┌─────┐   ┌─────┐                         │
│         │  7  │   │  8  │   │  9  │                         │
│         └─────┘   └─────┘   └─────┘                         │
│                                                              │
│                   ┌─────┐   ┌─────┐                         │
│                   │  0  │   │  ⌫  │                         │
│                   └─────┘   └─────┘                         │
│                                                              │
│                     [Cancel]                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Specifications**:
- Number buttons: 64dp diameter, 56dp touch target
- PIN dots: 16dp diameter
- Modal overlay: 50% black
- Dialog: White background, 24dp corner radius

---

## Component Library

### Buttons

| Type | Usage | Style |
|------|-------|-------|
| Primary | Main actions | Filled, primary color |
| Secondary | Secondary actions | Outlined |
| Text | Tertiary actions | No background |
| Destructive | Dangerous actions | Red filled |
| Panic | Emergency only | Large red, pulsing |

### Toggles

| State | Visual |
|-------|--------|
| ON | Primary color track, white thumb |
| OFF | Gray track, gray thumb |
| Disabled | 50% opacity |

### Cards

| Type | Style |
|------|-------|
| Device Card | White, 16dp radius, subtle shadow |
| App Card | White, 12dp radius, border |
| Alert Card | Red tinted, prominent |

### Inputs

| Type | Style |
|------|-------|
| Text Field | Outlined, 12dp radius |
| Search | Filled gray, search icon |
| PIN | Numeric keypad |
| Time Picker | Wheel or dropdown |

---

## Accessibility Requirements

| Requirement | Implementation |
|-------------|----------------|
| Touch targets | Minimum 48dp |
| Color contrast | WCAG AA (4.5:1 text) |
| Screen reader | All elements labeled |
| Font scaling | Support up to 200% |
| Focus indicators | Visible focus rings |
| Motion | Respect reduced motion |
