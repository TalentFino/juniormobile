# Testing Checklist

## Overview

This document provides comprehensive testing checklists for the KidSafe application. It covers:
- Manual testing procedures
- Automated test requirements
- Device-specific testing
- End-to-end scenarios

---

## Test Device Matrix

### Required Test Devices

| Device | Android Version | Purpose |
|--------|----------------|---------|
| Samsung Galaxy A05 | Android 14 | Primary target device |
| Samsung Galaxy A14 | Android 13 | Alternate Samsung |
| Redmi Note 12 | Android 13 | MIUI compatibility |
| Generic Android Emulator | Android 14 | Development testing |

### Parent App Test Devices

| Device | OS | Purpose |
|--------|-----|---------|
| Android Phone | Android 10+ | Primary parent device |
| iPhone 12 or newer | iOS 15+ | iOS testing |
| Android Tablet | Android 10+ | Tablet layout |
| iPad | iOS 15+ | iPad layout |

---

## 1. Child Device Testing

### 1.1 Launcher Tests

#### First Launch
- [ ] Launcher displays on device boot
- [ ] Welcome/setup screen shows on first launch
- [ ] Can connect to parent via QR code
- [ ] Device Owner mode is active
- [ ] Cannot uninstall launcher via settings

#### Kids Mode UI
- [ ] Greeting shows correct time of day (morning/afternoon/evening)
- [ ] Child's name displays correctly
- [ ] App grid shows 2x3 layout
- [ ] Only whitelisted apps appear
- [ ] App icons are clear and tappable
- [ ] Panic button is visible and prominent
- [ ] Status bar shows battery and signal

#### App Launching
- [ ] Tap on app icon launches app
- [ ] App opens within 2 seconds
- [ ] Can return to launcher via home button
- [ ] Recent apps shows only allowed apps

#### Standard Mode UI
- [ ] 4x4 app grid displays
- [ ] All whitelisted apps visible
- [ ] Settings icon visible (if allowed)
- [ ] Can switch to Kids mode

#### Mode Switching
- [ ] "Switch Mode" button visible
- [ ] PIN dialog appears on tap
- [ ] Correct PIN (1234) switches mode
- [ ] Wrong PIN shows error message
- [ ] 3 wrong attempts locks for 1 minute
- [ ] Mode switch animation plays
- [ ] Mode persists after screen off/on

#### Panic Button
- [ ] Button clearly visible in Kids mode
- [ ] Long press (3 sec) triggers alert
- [ ] Vibration feedback on trigger
- [ ] Confirmation dialog shows
- [ ] Alert sent to parent app
- [ ] Location included if enabled

### 1.2 DPC (Device Policy Controller) Tests

#### Restrictions - Kids Mode
- [ ] Cannot open Play Store
- [ ] Cannot install APK files
- [ ] Cannot uninstall apps
- [ ] Cannot access developer options
- [ ] Cannot factory reset without PIN
- [ ] USB debugging blocked
- [ ] Safe boot blocked

#### Restrictions - Standard Mode
- [ ] Play Store accessible (if configured)
- [ ] Settings partially accessible
- [ ] Cannot uninstall DPC
- [ ] Cannot uninstall launcher
- [ ] Cannot change device admin

#### Volume Control
- [ ] Volume limit enforced (70% default)
- [ ] Cannot exceed limit via hardware buttons
- [ ] Cannot exceed limit in app
- [ ] Parent can change limit remotely

### 1.3 Remote Command Tests

#### Mode Toggle
- [ ] Parent toggles Kids mode ON → Device switches
- [ ] Parent toggles Kids mode OFF → Device switches
- [ ] Command executes within 5 seconds
- [ ] Device shows notification of mode change

#### App Whitelist
- [ ] Parent adds app → App appears on device
- [ ] Parent removes app → App disappears
- [ ] App limits sync correctly
- [ ] Schedule restrictions apply

#### Screen Time
- [ ] Daily limit set → Timer starts
- [ ] Warning at 10 minutes remaining
- [ ] Device locks at limit
- [ ] Lock screen shows time remaining until reset
- [ ] Per-app limits enforced
- [ ] Bedtime mode activates on schedule
- [ ] Bedtime mode deactivates on schedule

#### Device Lock
- [ ] Lock command → Device immediately locks
- [ ] Lock screen shows parent message
- [ ] Cannot unlock without parent approval
- [ ] Unlock command → Device unlocks

#### Volume Limit
- [ ] Volume limit command received
- [ ] Limit applied immediately
- [ ] Persists after restart

### 1.4 Background Services

#### Usage Tracking
- [ ] Screen time recorded accurately
- [ ] Per-app usage tracked
- [ ] Data syncs to Firebase
- [ ] Survives device restart
- [ ] Low battery impact (< 2%)

#### Location Service
- [ ] Location updates sent to parent
- [ ] Accuracy is reasonable (< 50m)
- [ ] Updates on significant location change
- [ ] Works in background
- [ ] Respects enable/disable toggle

#### Firebase Sync
- [ ] Status syncs in real-time
- [ ] Commands received via FCM
- [ ] Reconnects after network loss
- [ ] Queues commands when offline
- [ ] Processes queue on reconnect

### 1.5 Edge Cases

#### Network Conditions
- [ ] Works on WiFi
- [ ] Works on mobile data
- [ ] Graceful offline behavior
- [ ] Reconnects automatically
- [ ] Offline commands queue

#### Battery States
- [ ] Functions at low battery (< 10%)
- [ ] Doze mode doesn't break functionality
- [ ] Battery optimization handled
- [ ] Charging detection works

#### Device States
- [ ] Survives restart
- [ ] Survives force stop
- [ ] Recovers from crash
- [ ] Handles time zone change
- [ ] Handles date change

---

## 2. Parent App Testing

### 2.1 Authentication

#### Phone Login
- [ ] Can enter phone number
- [ ] Country code selection works
- [ ] OTP sent within 30 seconds
- [ ] OTP entry works
- [ ] Resend OTP works
- [ ] Auto-fill OTP works (Android)
- [ ] Login successful with valid OTP
- [ ] Error shown for invalid OTP

#### Email Login
- [ ] Can enter email
- [ ] Can enter password
- [ ] Password visibility toggle works
- [ ] Login successful with valid credentials
- [ ] Error shown for wrong password
- [ ] "Forgot Password" flow works

#### Account Management
- [ ] Profile shows user info
- [ ] Can update display name
- [ ] Can update profile picture
- [ ] Can change password
- [ ] Logout works
- [ ] Delete account works (with confirmation)

### 2.2 Device Pairing

#### Add Child Flow
- [ ] "Add Child" button visible
- [ ] Can enter child name
- [ ] Can enter child age (slider)
- [ ] Can select avatar
- [ ] Can select protection level
- [ ] Protection level explanation shown

#### QR Code Generation
- [ ] QR code generates successfully
- [ ] QR code is scannable
- [ ] Expiry timer shows (15 min)
- [ ] Can regenerate code
- [ ] Code expires after use

#### Pairing Completion
- [ ] Device appears in dashboard after pairing
- [ ] Correct child name shown
- [ ] Correct protection level applied
- [ ] Success notification shown

### 2.3 Dashboard

#### Device Card
- [ ] Child name displayed
- [ ] Device status (online/offline) shown
- [ ] Battery level displayed
- [ ] Current mode shown (Kids/Standard)
- [ ] "Now Playing" shows when applicable
- [ ] Last seen time when offline

#### Quick Actions
- [ ] Lock device button works
- [ ] Locate device button works
- [ ] Toggle mode button works
- [ ] Mute device button works

#### Activity Summary
- [ ] Today's screen time shown
- [ ] Progress bar accurate
- [ ] Top apps listed
- [ ] "See More" navigates to details

### 2.4 App Management

#### App List
- [ ] All installed apps shown
- [ ] App icons load correctly
- [ ] Categories displayed
- [ ] Search/filter works

#### App Toggle
- [ ] Toggle ON adds to whitelist
- [ ] Toggle OFF removes from whitelist
- [ ] Change syncs to device within 5 seconds
- [ ] Pending state shown while syncing

#### App Limits
- [ ] Can set per-app time limit
- [ ] Limit dropdown works (No limit, 15m, 30m, 1h, 2h)
- [ ] Limit syncs to device

#### App Schedule
- [ ] Can set allowed hours
- [ ] Can select allowed days
- [ ] Schedule syncs to device

### 2.5 Screen Time

#### Daily Limit
- [ ] Slider works (30 min - 4 hours)
- [ ] Value label updates
- [ ] "Same every day" option
- [ ] "Different by day" option
- [ ] Changes sync to device

#### Bedtime
- [ ] Start time picker works
- [ ] End time picker works
- [ ] Enable/disable toggle works
- [ ] Settings sync to device

#### Usage History
- [ ] Today's usage shown
- [ ] Past 7 days shown
- [ ] Past 30 days shown
- [ ] Per-app breakdown accurate
- [ ] Charts render correctly

### 2.6 Location

#### Map View
- [ ] Map loads correctly
- [ ] Current location marker shown
- [ ] Location accuracy displayed
- [ ] Last update time shown
- [ ] Can refresh location

#### Safe Zones
- [ ] Can add safe zone
- [ ] Can set zone radius
- [ ] Can name zone
- [ ] Can delete zone
- [ ] Alerts trigger when child leaves zone

#### Location History
- [ ] History trail shown
- [ ] Can view by date
- [ ] Timestamps displayed

### 2.7 Alerts & Notifications

#### Notification Settings
- [ ] Panic alerts toggle
- [ ] Battery alerts toggle
- [ ] Offline alerts toggle
- [ ] Screen time alerts toggle
- [ ] Daily report toggle

#### Receiving Notifications
- [ ] Panic alert received
- [ ] Battery low alert received
- [ ] Device offline alert received
- [ ] Screen time limit alert received
- [ ] Can tap notification to open app

### 2.8 Settings

#### Account
- [ ] Profile editing works
- [ ] Password change works
- [ ] Delete account works

#### Subscription
- [ ] Current plan shown
- [ ] Upgrade option available
- [ ] Payment history accessible
- [ ] Cancel subscription works

#### Support
- [ ] FAQ accessible
- [ ] Contact support works
- [ ] Submit feedback works

---

## 3. End-to-End Scenarios

### 3.1 Complete Setup Flow

```
1. Parent downloads app
2. Parent creates account
3. Parent adds child profile
4. Parent generates QR code
5. Child device scans QR code
6. Device Owner mode activated
7. Launcher set as default
8. Device appears in parent dashboard
9. Parent customizes settings
10. Child uses device normally
```

**Test each step and verify:**
- [ ] All steps complete without errors
- [ ] Process takes < 10 minutes
- [ ] Clear instructions at each step

### 3.2 Daily Usage Scenario

```
Morning:
1. Child wakes up, device is in bedtime lock
2. 7:00 AM - Bedtime ends, device unlocks
3. Child opens Spotify, listens to music
4. Parent checks dashboard - sees "Now Playing"

Afternoon:
5. Child continues using device
6. Usage approaches limit - warning shown
7. Limit reached - device locks
8. Child sees "Time's up" message
9. Parent receives notification

Evening:
10. Parent extends time limit
11. Device unlocks for 30 more minutes
12. 8:30 PM - Bedtime mode activates
13. Device locks for the night
```

**Verify:**
- [ ] All transitions work correctly
- [ ] Times are accurate
- [ ] Notifications received
- [ ] Data matches between parent app and device

### 3.3 Emergency Scenario

```
1. Child is in unfamiliar location
2. Child presses panic button (3 second hold)
3. Device vibrates, confirms alert
4. Parent receives high-priority notification
5. Parent opens notification
6. App shows child's location on map
7. Parent can call child
8. Parent acknowledges alert
```

**Verify:**
- [ ] Alert received within 5 seconds
- [ ] Location accurate
- [ ] Notification is unmissable (sound, vibration)
- [ ] Works when child device has low battery

### 3.4 Offline Recovery Scenario

```
1. Parent sets new daily limit
2. Child device goes offline (airplane mode)
3. Command queued on Firebase
4. Child device comes back online
5. Command delivered and executed
6. Device confirms execution
7. Parent app shows "Applied"
```

**Verify:**
- [ ] Commands queue correctly
- [ ] Commands apply in order
- [ ] Device state syncs after reconnect

### 3.5 Multi-Child Scenario

```
1. Parent has 2 children
2. Each child has own device
3. Parent customizes each device separately
4. Dashboard shows both devices
5. Can switch between device controls
6. Alerts differentiate by child
```

**Verify:**
- [ ] Both devices show correctly
- [ ] Settings are independent
- [ ] No data mixing between devices

---

## 4. Performance Testing

### 4.1 Launcher Performance

| Metric | Target | Test Method |
|--------|--------|-------------|
| Cold start time | < 500ms | Stopwatch from tap |
| App grid render | < 100ms | Profile in Android Studio |
| Mode switch | < 2s | Stopwatch |
| Memory usage | < 50MB | Android Studio Profiler |
| Battery drain (idle) | < 1%/hr | Battery stats |
| Battery drain (active) | < 3%/hr | Battery stats |

### 4.2 Parent App Performance

| Metric | Target | Test Method |
|--------|--------|-------------|
| Cold start time | < 2s | Stopwatch from tap |
| Dashboard load | < 1s | Profile in DevTools |
| Image load time | < 500ms | Network tab |
| Memory usage | < 100MB | Flutter DevTools |
| Frame rate | > 55fps | Flutter DevTools |

### 4.3 Network Performance

| Operation | Target | Test Method |
|-----------|--------|-------------|
| Command delivery | < 3s | Timestamp comparison |
| Status sync | < 2s | Timestamp comparison |
| Location update | < 5s | Timestamp comparison |
| Image upload | < 5s for 1MB | Network profiler |

### 4.4 Load Testing

| Scenario | Target |
|----------|--------|
| 100 concurrent parents | No degradation |
| 1000 devices online | No degradation |
| 10 commands/second | All processed |
| 1000 FCM/minute | All delivered |

---

## 5. Security Testing

### 5.1 Authentication

- [ ] Cannot access app without login
- [ ] Session expires appropriately
- [ ] Cannot use expired tokens
- [ ] Brute force protection works
- [ ] Password requirements enforced

### 5.2 Authorization

- [ ] Cannot access other user's devices
- [ ] Cannot send commands to other devices
- [ ] Cannot view other user's data
- [ ] API rejects unauthorized requests

### 5.3 Data Security

- [ ] Data encrypted in transit (HTTPS)
- [ ] Sensitive data encrypted at rest
- [ ] No sensitive data in logs
- [ ] No secrets in APK
- [ ] Secure key storage

### 5.4 Device Security

- [ ] Cannot bypass Device Owner
- [ ] Cannot uninstall DPC via ADB
- [ ] Cannot disable DPC via settings
- [ ] Cannot bypass PIN dialog
- [ ] Cannot escape Kids mode

---

## 6. Accessibility Testing

### 6.1 Visual Accessibility

- [ ] Text readable at default size
- [ ] Text readable at large size
- [ ] Sufficient color contrast (4.5:1)
- [ ] Not relying on color alone
- [ ] Icons have labels

### 6.2 Screen Reader

- [ ] TalkBack navigation works
- [ ] VoiceOver navigation works
- [ ] All elements labeled
- [ ] Logical focus order
- [ ] Announcements meaningful

### 6.3 Motor Accessibility

- [ ] Touch targets >= 48dp
- [ ] Adequate spacing between elements
- [ ] No rapid tapping required
- [ ] Gestures have alternatives

---

## 7. Localization Testing

### 7.1 English (en)

- [ ] All strings in English
- [ ] Grammar correct
- [ ] No truncation
- [ ] Date format correct (DD/MM/YYYY)
- [ ] Time format correct (12/24 based on device)

### 7.2 Hindi (hi)

- [ ] All strings translated
- [ ] Hindi renders correctly
- [ ] No layout breaking
- [ ] Culturally appropriate

---

## 8. Regression Testing

### After Each Build

Run automated tests:
```bash
# Android Unit Tests
./gradlew testDebugUnitTest

# Flutter Unit Tests
flutter test

# Android Instrumented Tests
./gradlew connectedDebugAndroidTest

# Flutter Integration Tests
flutter test integration_test/app_test.dart
```

### Smoke Test Checklist

Quick manual verification after deployment:

- [ ] Can login
- [ ] Can see dashboard
- [ ] Can send command
- [ ] Device receives command
- [ ] Can view usage data
- [ ] Notifications work

---

## 9. Bug Report Template

```markdown
## Bug Report

**Title:** [Brief description]

**Severity:** [P0/P1/P2/P3]

**Environment:**
- Device: [Model, OS version]
- App Version: [Version number]
- Date/Time: [When occurred]

**Steps to Reproduce:**
1. Step 1
2. Step 2
3. Step 3

**Expected Behavior:**
[What should happen]

**Actual Behavior:**
[What actually happened]

**Screenshots/Video:**
[Attach if applicable]

**Logs:**
```
[Paste relevant logs]
```

**Additional Context:**
[Any other information]
```

---

## 10. Test Automation

### Unit Test Coverage Requirements

| Module | Minimum Coverage |
|--------|-----------------|
| Launcher Core | 80% |
| DPC | 70% |
| Parent App Services | 80% |
| Parent App Repositories | 80% |
| Parent App Providers | 70% |
| Cloud Functions | 80% |

### CI/CD Pipeline Tests

```yaml
# .github/workflows/test.yml

name: Test

on: [push, pull_request]

jobs:
  android-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - run: cd android && ./gradlew testDebugUnitTest

  flutter-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.19.0'
      - run: cd flutter/parent_app && flutter test --coverage
      - uses: codecov/codecov-action@v3
        with:
          files: flutter/parent_app/coverage/lcov.info

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd android && ./gradlew lint
      - run: cd flutter/parent_app && flutter analyze
```

---

## Sign-Off Checklist

### Pre-Release Sign-Off

| Area | QA Sign-Off | Dev Sign-Off | Date |
|------|-------------|--------------|------|
| Launcher | ☐ | ☐ | |
| DPC | ☐ | ☐ | |
| Parent App Android | ☐ | ☐ | |
| Parent App iOS | ☐ | ☐ | |
| Cloud Functions | ☐ | ☐ | |
| Security | ☐ | ☐ | |
| Performance | ☐ | ☐ | |
| Accessibility | ☐ | ☐ | |

### Release Criteria

- [ ] Zero P0 bugs
- [ ] Zero P1 bugs
- [ ] P2 bugs documented with workarounds
- [ ] All automated tests passing
- [ ] Manual test pass rate > 95%
- [ ] Performance targets met
- [ ] Security audit passed
- [ ] Accessibility audit passed
- [ ] Documentation complete
