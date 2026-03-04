# Feature Specifications

## Feature Priority Matrix

| Priority | Definition | MVP Scope |
|----------|------------|-----------|
| **P0** | Must have for launch | Yes |
| **P1** | Important, needed soon | Yes |
| **P2** | Nice to have | Partial |
| **P3** | Future consideration | No |

---

## P0 Features (Must Have)

### F01: Device Setup & Pairing

**Description**: Parent pairs child device to their account via QR code.

| Aspect | Specification |
|--------|---------------|
| **Trigger** | Parent taps "Add Device" in parent app |
| **Input** | Child name, age, protection level |
| **Output** | QR code valid for 15 minutes |
| **Child Action** | Scan QR during device setup |
| **Result** | Device linked to parent account |

**Acceptance Criteria**:
- [ ] QR code generates in < 2 seconds
- [ ] QR code is scannable from 30cm distance
- [ ] Pairing completes in < 30 seconds
- [ ] Device appears in parent dashboard immediately
- [ ] Works on first boot (factory reset device)

---

### F02: Custom Launcher

**Description**: Replacement home screen that shows only approved apps.

| Aspect | Specification |
|--------|---------------|
| **Default Home** | Launcher is set as default, cannot be changed |
| **App Display** | Only whitelisted apps visible |
| **Modes** | Kids Mode (2x3 grid), Standard Mode (4x4 grid) |
| **Mode Switch** | Requires 4-digit PIN |
| **Persist** | Survives restart, cannot be uninstalled |

**Acceptance Criteria**:
- [ ] Launcher loads in < 500ms
- [ ] Only whitelisted apps appear
- [ ] Home button always returns to launcher
- [ ] Cannot access other launchers
- [ ] Recent apps only shows whitelisted apps

---

### F03: Device Policy Controller (DPC)

**Description**: System-level control via Android Device Owner APIs.

| Restriction | Kids Mode | Standard Mode |
|-------------|-----------|---------------|
| Install apps | Blocked | Blocked (unless remote install) |
| Uninstall apps | Blocked | Blocked |
| Access Play Store | Blocked | Blocked |
| Access Settings | Blocked | Limited |
| Factory Reset | PIN required | PIN required |
| USB Debugging | Blocked | Blocked |
| Safe Boot | Blocked | Blocked |

**Acceptance Criteria**:
- [ ] Device Owner mode active after setup
- [ ] Cannot uninstall DPC or Launcher
- [ ] Restrictions survive device restart
- [ ] Cannot bypass via ADB

---

### F04: App Whitelist Management

**Description**: Parent controls which apps child can access.

| Aspect | Specification |
|--------|---------------|
| **View Apps** | Parent sees all installed apps on child device |
| **Toggle** | ON = visible to child, OFF = hidden |
| **Sync Time** | Changes reflect on device within 5 seconds |
| **Default** | New devices have preset app list based on age |
| **Categories** | Music, Education, Utility, Communication, Payment, Games |

**Default Whitelist by Protection Level**:

| Protection Level | Default Apps |
|-----------------|--------------|
| Maximum (5-8) | Clock, Calculator, 2 music apps |
| Balanced (9-12) | Above + messaging, educational |
| Trust (13+) | Above + more apps, fewer restrictions |

**Acceptance Criteria**:
- [ ] All installed apps listed in parent app
- [ ] Toggle state syncs to device in < 5 seconds
- [ ] Hidden apps not visible in launcher
- [ ] Hidden apps cannot be launched via shortcuts

---

### F05: Remote Mode Toggle

**Description**: Parent can switch child device between Kids and Standard mode.

| Aspect | Specification |
|--------|---------------|
| **Trigger** | Parent taps mode toggle in dashboard |
| **Confirmation** | "Switch to Kids Mode?" dialog |
| **Execution** | Command sent via Firebase RTDB |
| **Result** | Device switches mode immediately |
| **Notification** | Child sees brief notification |

**Acceptance Criteria**:
- [ ] Mode switches within 3 seconds
- [ ] Works when device is online
- [ ] Queued when device is offline
- [ ] Parent sees confirmation

---

### F06: Panic Button

**Description**: Child can send emergency alert to parent.

| Aspect | Specification |
|--------|---------------|
| **Location** | Prominent button in Kids Mode |
| **Activation** | Long press for 3 seconds |
| **Feedback** | Vibration + visual confirmation |
| **Data Sent** | Device ID, timestamp, GPS location |
| **Parent Alert** | High-priority push notification |

**Acceptance Criteria**:
- [ ] Panic button always visible in Kids Mode
- [ ] Cannot be accidentally triggered (3 sec hold)
- [ ] Alert received by parent within 5 seconds
- [ ] Works even with low battery
- [ ] Includes location if GPS enabled

---

### F07: Real-time Device Status

**Description**: Parent sees current state of child device.

| Status | Displayed |
|--------|-----------|
| Online/Offline | Green/gray dot |
| Battery Level | Percentage + icon |
| Current Mode | Kids/Standard |
| Last Seen | "2 minutes ago" |
| Screen State | On/Off |

**Acceptance Criteria**:
- [ ] Status updates within 10 seconds
- [ ] Offline shown if no update for 2 minutes
- [ ] Battery level accurate within 5%

---

## P1 Features (Important)

### F08: Screen Time Limits

**Description**: Parent sets daily usage limits.

| Aspect | Specification |
|--------|---------------|
| **Daily Limit** | 30 min to 4 hours (or unlimited) |
| **Per-App Limit** | Optional limit for specific apps |
| **Warning** | Alert at 10 minutes remaining |
| **Enforcement** | Device locks when limit reached |
| **Reset** | Daily at midnight (device timezone) |

**Lock Screen When Limit Reached**:
- Shows "Time's up for today!"
- Shows time until reset
- Emergency call still possible
- Parent can extend via app

**Acceptance Criteria**:
- [ ] Time tracked accurately (± 1 minute)
- [ ] Warning notification at 10 min remaining
- [ ] Device locks when limit hit
- [ ] Parent can extend remotely
- [ ] Resets at midnight correctly

---

### F09: Bedtime Mode

**Description**: Device locks during sleep hours.

| Aspect | Specification |
|--------|---------------|
| **Schedule** | Start time + End time |
| **Days** | Can set different for weekdays/weekends |
| **Lock** | Device shows bedtime screen |
| **Exceptions** | Alarm app still works |
| **Emergency** | Panic button still works |

**Acceptance Criteria**:
- [ ] Locks at exact scheduled time
- [ ] Unlocks at exact scheduled time
- [ ] Alarm sounds even when locked
- [ ] Panic button works when locked

---

### F10: Remote Device Lock

**Description**: Parent can instantly lock child device.

| Aspect | Specification |
|--------|---------------|
| **Trigger** | Parent taps "Lock Now" |
| **Execution** | Immediate (< 3 seconds) |
| **Lock Screen** | Shows parent-defined message |
| **Unlock** | Parent taps "Unlock" or schedules |

**Acceptance Criteria**:
- [ ] Device locks within 3 seconds
- [ ] Custom message displayed
- [ ] Cannot be bypassed by child
- [ ] Parent can unlock remotely

---

### F11: Volume Limiting

**Description**: Maximum volume enforced for hearing protection.

| Aspect | Specification |
|--------|---------------|
| **Default Limit** | 70% of max volume |
| **Range** | 50% to 100% |
| **Enforcement** | Cannot exceed via buttons or apps |
| **Setting** | Parent configures in app |

**Acceptance Criteria**:
- [ ] Volume cannot exceed set limit
- [ ] Works with all audio apps
- [ ] Works with headphones and speaker
- [ ] Persists after restart

---

### F12: Usage Reports

**Description**: Parent sees how child used the device.

| Report | Content |
|--------|---------|
| **Today** | Total time, per-app breakdown |
| **This Week** | Daily totals, trends |
| **This Month** | Weekly totals, most used apps |

**Acceptance Criteria**:
- [ ] Accurate within 1 minute
- [ ] Updates in near real-time
- [ ] Shows app icons and names
- [ ] Visual charts (bar/pie)

---

## P2 Features (Nice to Have)

### F13: Location Tracking

**Description**: Parent sees child device location.

| Aspect | Specification |
|--------|---------------|
| **Update Frequency** | Every 5 minutes (configurable) |
| **Accuracy** | Within 50 meters |
| **History** | Last 7 days of locations |
| **Privacy** | Opt-in, can be disabled |

**Acceptance Criteria**:
- [ ] Location accurate within 50m
- [ ] Map displays correctly
- [ ] History scrollable by date
- [ ] Battery impact < 3%/hour

---

### F14: Safe Zones

**Description**: Parent defines allowed locations, alerts when child leaves.

| Aspect | Specification |
|--------|---------------|
| **Create** | Drop pin on map, set radius |
| **Radius** | 100m to 2km |
| **Alert** | Push notification when child leaves |
| **Zones** | Up to 5 zones (Home, School, etc.) |

**Acceptance Criteria**:
- [ ] Geofence triggers within 100m accuracy
- [ ] Alert received within 1 minute of exit
- [ ] Can enable/disable per zone

---

### F15: Remote App Installation

**Description**: Parent can install apps on child device remotely.

| Aspect | Specification |
|--------|---------------|
| **Method** | Via Managed Google Play |
| **Trigger** | Parent selects app in parent app |
| **Execution** | App installs silently on child device |
| **Catalog** | Pre-approved safe apps |

**Acceptance Criteria**:
- [ ] App installs without child interaction
- [ ] Parent sees installation status
- [ ] Only works with approved apps

---

### F16: Now Playing Display

**Description**: Parent sees what child is currently listening to.

| Aspect | Specification |
|--------|---------------|
| **Display** | Song title, artist, app |
| **Update** | Real-time when playing |
| **Apps** | Spotify, JioSaavn, Amazon Music, YouTube Music |

**Acceptance Criteria**:
- [ ] Shows current song within 5 seconds
- [ ] Works with major music apps
- [ ] Shows "Not playing" when idle

---

## P3 Features (Future)

| Feature | Description |
|---------|-------------|
| **Content Filtering** | Block explicit content in music/video |
| **Call Monitoring** | See incoming/outgoing calls |
| **SMS Monitoring** | View text messages |
| **Social Media Controls** | App-specific restrictions |
| **Multiple Parents** | Both parents can control |
| **Chore Rewards** | Earn screen time for tasks |
| **AI Alerts** | Detect concerning patterns |

---

## Feature Dependencies

```
F01 Device Setup
 └── F02 Custom Launcher
      └── F03 DPC
           └── F04 App Whitelist
                └── F05 Remote Mode Toggle
                     └── F08 Screen Time
                          └── F09 Bedtime
                               └── F10 Remote Lock

F06 Panic Button (independent, P0)
F07 Device Status (independent, P0)
F11 Volume Limit (depends on F03)
F12 Usage Reports (depends on F04)
F13 Location (depends on F01)
F14 Safe Zones (depends on F13)
F15 Remote Install (depends on F03)
F16 Now Playing (depends on F07)
```

---

## Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Latency** | Commands execute within 5 seconds |
| **Availability** | 99.5% uptime for cloud services |
| **Battery** | < 5% daily drain from our services |
| **Storage** | < 100MB total app size |
| **Offline** | Core features work offline, sync when online |
| **Security** | All data encrypted in transit and at rest |
| **Privacy** | Data stored in India (Firebase asia-south1) |
