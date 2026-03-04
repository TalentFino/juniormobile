# Developer Plan Overview

## Product Vision

**KidTunes is a complete kid-safe phone solution** - not just a music player. Parents have full control over:
- Which apps the child can access (ANY app - music, payments, education, games)
- Remote app installation via managed Google Play
- Screen time, location, volume, and device settings
- Two modes: Kids Mode (restricted) and Standard Mode (more freedom)

---

## Key Capabilities

| Capability | Description |
|------------|-------------|
| **App Whitelisting** | Parent controls which apps are visible/accessible |
| **Remote App Install** | Parent can push apps from Play Store to child device |
| **Remote App Removal** | Parent can remove apps remotely |
| **Screen Time Control** | Daily limits, per-app limits, schedules |
| **Kids Mode** | Simplified UI with large icons, only approved apps |
| **Standard Mode** | Full UI with all approved apps |
| **Volume Protection** | Max volume limit for hearing safety |
| **Location Tracking** | Real-time GPS location (opt-in) |
| **Panic Button** | One-tap emergency alert to parents |
| **Bedtime Mode** | Device locks during sleep hours |
| **Remote Lock/Unlock** | Parent can lock device instantly |
| **Usage Reports** | Detailed app usage analytics |

---

## Project Structure

```
kidapp/
├── android/
│   ├── launcher/                    # Custom Launcher App (Kotlin)
│   │   ├── app/
│   │   │   ├── src/main/
│   │   │   │   ├── java/com/kidtunes/launcher/
│   │   │   │   │   ├── LauncherApplication.kt
│   │   │   │   │   ├── ui/
│   │   │   │   │   │   ├── LauncherActivity.kt
│   │   │   │   │   │   ├── kids/
│   │   │   │   │   │   │   ├── KidsModeFragment.kt
│   │   │   │   │   │   │   └── KidsModeAdapter.kt
│   │   │   │   │   │   ├── standard/
│   │   │   │   │   │   │   ├── StandardModeFragment.kt
│   │   │   │   │   │   │   └── StandardModeAdapter.kt
│   │   │   │   │   │   ├── pin/
│   │   │   │   │   │   │   └── PinDialogFragment.kt
│   │   │   │   │   │   ├── panic/
│   │   │   │   │   │   │   └── PanicActivity.kt
│   │   │   │   │   │   └── lockscreen/
│   │   │   │   │   │       ├── ScreenTimeLockActivity.kt
│   │   │   │   │   │       └── BedtimeLockActivity.kt
│   │   │   │   │   ├── services/
│   │   │   │   │   │   ├── UsageTrackingService.kt
│   │   │   │   │   │   ├── VolumeMonitorService.kt
│   │   │   │   │   │   ├── LocationService.kt
│   │   │   │   │   │   ├── ScreenTimeService.kt
│   │   │   │   │   │   ├── FirebaseSyncService.kt
│   │   │   │   │   │   └── FCMService.kt
│   │   │   │   │   ├── receivers/
│   │   │   │   │   │   ├── BootReceiver.kt
│   │   │   │   │   │   ├── PackageChangeReceiver.kt
│   │   │   │   │   │   └── ScreenStateReceiver.kt
│   │   │   │   │   ├── data/
│   │   │   │   │   │   ├── models/
│   │   │   │   │   │   ├── local/
│   │   │   │   │   │   └── remote/
│   │   │   │   │   ├── di/
│   │   │   │   │   │   └── AppModule.kt
│   │   │   │   │   └── utils/
│   │   │   │   ├── res/
│   │   │   │   └── AndroidManifest.xml
│   │   │   └── build.gradle.kts
│   │   ├── build.gradle.kts
│   │   └── settings.gradle.kts
│   │
│   └── dpc/                         # Device Policy Controller (Kotlin)
│       ├── app/
│       │   ├── src/main/
│       │   │   ├── java/com/kidtunes/dpc/
│       │   │   │   ├── DPCApplication.kt
│       │   │   │   ├── receivers/
│       │   │   │   │   └── KidDeviceAdminReceiver.kt
│       │   │   │   ├── provisioning/
│       │   │   │   │   ├── ProvisioningActivity.kt
│       │   │   │   │   └── QRScanActivity.kt
│       │   │   │   ├── policies/
│       │   │   │   │   ├── PolicyManager.kt
│       │   │   │   │   ├── AppRestrictionManager.kt
│       │   │   │   │   └── SettingsRestrictionManager.kt
│       │   │   │   ├── apps/
│       │   │   │   │   ├── ManagedPlayManager.kt
│       │   │   │   │   └── AppInstallManager.kt
│       │   │   │   └── utils/
│       │   │   ├── res/
│       │   │   └── AndroidManifest.xml
│       │   └── build.gradle.kts
│       ├── build.gradle.kts
│       └── settings.gradle.kts
│
├── flutter/
│   └── parent_app/                  # Parent Companion App (Flutter)
│       ├── lib/
│       │   ├── main.dart
│       │   ├── app/
│       │   │   ├── app.dart
│       │   │   └── router.dart
│       │   ├── core/
│       │   │   ├── constants/
│       │   │   ├── theme/
│       │   │   ├── extensions/
│       │   │   └── utils/
│       │   ├── data/
│       │   │   ├── models/
│       │   │   │   ├── user.dart
│       │   │   │   ├── child_profile.dart
│       │   │   │   ├── device.dart
│       │   │   │   ├── device_status.dart
│       │   │   │   ├── app_info.dart
│       │   │   │   ├── usage_stats.dart
│       │   │   │   ├── screen_time_policy.dart
│       │   │   │   └── location.dart
│       │   │   ├── repositories/
│       │   │   │   ├── auth_repository.dart
│       │   │   │   ├── device_repository.dart
│       │   │   │   ├── app_repository.dart
│       │   │   │   └── settings_repository.dart
│       │   │   └── services/
│       │   │       ├── firebase_service.dart
│       │   │       ├── notification_service.dart
│       │   │       └── location_service.dart
│       │   ├── presentation/
│       │   │   ├── providers/
│       │   │   ├── screens/
│       │   │   │   ├── splash/
│       │   │   │   ├── auth/
│       │   │   │   ├── onboarding/
│       │   │   │   ├── dashboard/
│       │   │   │   ├── device_detail/
│       │   │   │   ├── app_management/
│       │   │   │   ├── screen_time/
│       │   │   │   ├── location/
│       │   │   │   ├── alerts/
│       │   │   │   ├── usage_reports/
│       │   │   │   └── settings/
│       │   │   └── widgets/
│       │   └── l10n/
│       ├── android/
│       ├── ios/
│       ├── pubspec.yaml
│       └── test/
│
├── firebase/
│   ├── firestore.rules
│   ├── database.rules.json
│   ├── firestore.indexes.json
│   └── functions/
│       ├── src/
│       │   └── index.ts
│       ├── package.json
│       └── tsconfig.json
│
└── docs/
    └── developer/
```

---

## Technology Stack (Exact Versions)

### Android (Child Device)

| Technology | Version | Purpose |
|------------|---------|---------|
| Kotlin | 1.9.22 | Primary language |
| Android Gradle Plugin | 8.2.2 | Build system |
| Gradle | 8.4 | Build tool |
| Min SDK | 26 (Android 8.0) | Minimum support |
| Target SDK | 34 (Android 14) | Target version |
| Compile SDK | 34 | Compilation target |
| Firebase BOM | 32.7.2 | Firebase dependencies |
| Material Design 3 | 1.11.0 | UI components |
| Coroutines | 1.7.3 | Async operations |
| Hilt | 2.50 | Dependency injection |
| Room | 2.6.1 | Local database |
| DataStore | 1.0.0 | Preferences storage |
| WorkManager | 2.9.0 | Background work |
| Play Services | 21.0.1 | Google services |

### Flutter (Parent App)

| Technology | Version | Purpose |
|------------|---------|---------|
| Flutter | 3.19.x | Framework |
| Dart | 3.3.x | Language |
| firebase_core | 2.27.0 | Firebase initialization |
| firebase_auth | 4.17.0 | Authentication |
| cloud_firestore | 4.15.0 | Database |
| firebase_database | 10.4.0 | Realtime DB |
| firebase_messaging | 14.7.0 | Push notifications |
| flutter_riverpod | 2.5.0 | State management |
| go_router | 13.2.0 | Navigation |
| mobile_scanner | 4.0.1 | QR scanning |
| geolocator | 11.0.0 | Location |
| google_maps_flutter | 2.5.3 | Map display |
| fl_chart | 0.66.0 | Usage charts |
| cached_network_image | 3.3.1 | Image caching |
| flutter_local_notifications | 17.0.0 | Local notifications |
| shared_preferences | 2.2.2 | Local storage |

### Firebase

| Service | Purpose |
|---------|---------|
| Authentication | User auth (phone/email) |
| Firestore | Structured data storage |
| Realtime Database | Live device status |
| Cloud Messaging | Push notifications |
| Cloud Functions | Server-side logic |
| Crashlytics | Crash reporting |
| Analytics | Usage analytics |
| Remote Config | Feature flags |

---

## Development Phases & Sprints

| Phase | Duration | Sprint Count | Focus |
|-------|----------|--------------|-------|
| **Phase 0** | Days 1-7 | Sprint 0 | Setup & Infrastructure |
| **Phase 1** | Days 8-49 | Sprints 1-4 | Core Development |
| **Phase 2** | Days 50-77 | Sprints 5-6 | Testing & Polish |
| **Phase 3** | Days 78-100 | Sprint 7 | Production Prep |

### Sprint Calendar

| Sprint | Days | Duration | Focus |
|--------|------|----------|-------|
| Sprint 0 | 1-7 | 7 days | Environment setup, Firebase, device validation |
| Sprint 1 | 8-17 | 10 days | Launcher foundation + DPC basics |
| Sprint 2 | 18-27 | 10 days | Parent app foundation + pairing |
| Sprint 3 | 28-37 | 10 days | Core features (controls, screen time) |
| Sprint 4 | 38-49 | 12 days | Advanced features + polish |
| Sprint 5 | 50-63 | 14 days | Internal testing + bug fixes |
| Sprint 6 | 64-77 | 14 days | Beta testing + feedback |
| Sprint 7 | 78-100 | 23 days | Production readiness |

---

## Epic Overview

| Epic ID | Epic Name | Sprint | Owner |
|---------|-----------|--------|-------|
| E01 | Infrastructure Setup | 0 | Both |
| E02 | Custom Launcher | 1 | Android |
| E03 | Device Policy Controller | 1-2 | Android |
| E04 | Parent App Foundation | 2 | Flutter |
| E05 | Device Pairing | 2 | Both |
| E06 | Remote Controls | 3 | Both |
| E07 | App Management | 3 | Both |
| E08 | Screen Time | 3-4 | Both |
| E09 | Location & Safety | 4 | Both |
| E10 | Usage Analytics | 4 | Both |
| E11 | Remote App Installation | 4 | Android |
| E12 | Testing & QA | 5-6 | Both |
| E13 | Production Prep | 7 | Both |

---

## Story Point Reference

| Points | Complexity | Time Estimate | Example |
|--------|------------|---------------|---------|
| 1 | Trivial | 1-2 hours | Add a button |
| 2 | Simple | 2-4 hours | Simple screen UI |
| 3 | Medium | 4-8 hours | Form with validation |
| 5 | Complex | 1-2 days | Firebase integration |
| 8 | Very Complex | 2-3 days | Background service |
| 13 | Epic | 3-5 days | Full feature |

---

## Git Workflow

### Branch Strategy

```
main                      # Production-ready code
├── develop               # Integration branch
├── feature/LA-xxx-desc   # Launcher features
├── feature/DPC-xxx-desc  # DPC features
├── feature/PA-xxx-desc   # Parent app features
├── feature/FB-xxx-desc   # Firebase features
├── bugfix/BUG-xxx-desc   # Bug fixes
└── release/v1.0.0        # Release branches
```

### Commit Format

```
[Component] Brief description (max 72 chars)

- Detailed point 1
- Detailed point 2

Refs: LA-001
```

Components: `[Launcher]`, `[DPC]`, `[Parent]`, `[Firebase]`, `[Docs]`, `[Config]`

---

## Definition of Done

### For Code Tasks
- [ ] Code compiles without errors or warnings
- [ ] Unit tests written and passing (min 70% coverage for logic)
- [ ] Integration test passing
- [ ] Code reviewed and approved
- [ ] No TODO comments left (convert to tickets)
- [ ] Logging added for debugging
- [ ] Error handling implemented

### For Features
- [ ] All acceptance criteria met
- [ ] Works on physical test device
- [ ] Works offline (where applicable)
- [ ] UI matches design specs
- [ ] Accessibility basics met
- [ ] Performance acceptable (no jank)

### For Sprints
- [ ] All committed stories complete
- [ ] Demo prepared and delivered
- [ ] Known issues documented
- [ ] Next sprint planned

---

## Documentation Index

| Document | Description |
|----------|-------------|
| `01_sprint_0_setup.md` | Complete environment and infrastructure setup |
| `02_sprint_1_launcher_dpc.md` | Launcher and DPC foundation |
| `03_sprint_2_parent_pairing.md` | Parent app and device pairing |
| `04_sprint_3_core_features.md` | Remote controls, app management, screen time |
| `05_sprint_4_advanced.md` | Location, analytics, remote install, polish |
| `06_sprint_5_6_testing.md` | Testing procedures and bug fixes |
| `07_sprint_7_production.md` | Production preparation |
| `08_firebase_schema.md` | Complete Firebase data structure |
| `09_api_contracts.md` | All data models and contracts |
| `10_testing_checklist.md` | Manual and automated testing guide |

---

## Risk Register

| Risk | Probability | Impact | Mitigation | Owner |
|------|-------------|--------|------------|-------|
| Device Owner mode not working | Low | Critical | Test Day 6, have Work Profile backup | Android |
| Samsung firmware breaks DPC | Medium | High | Test multiple firmware, document versions | Android |
| Play Store rejects parent app | Medium | High | Follow guidelines, prepare appeal | Flutter |
| Firebase costs spike | Low | Medium | Set alerts, optimize, implement caching | Both |
| Pairing flow confusing | Medium | Medium | User testing in Sprint 3 | Both |
| Battery drain from services | Medium | High | Optimize in Sprint 4, WorkManager | Android |
| Child bypasses restrictions | Low | Critical | Security audit Sprint 5 | Android |

---

## Go/No-Go Checkpoints

| Day | Checkpoint | Criteria | Decision Point |
|-----|------------|----------|----------------|
| 6 | Device Admin | Device Owner mode works on A05 | Continue/Pivot to Work Profile |
| 17 | Launcher MVP | Mode switch, whitelist, basic UI | Continue/Add Android resources |
| 27 | Pairing | Parent-child connection works | Continue/Debug Firebase |
| 37 | Controls | Remote commands execute | Continue/Cut scope |
| 49 | Beta Ready | All P0 features work | Start beta/Delay |
| 77 | Beta Complete | NPS > 30, no P0 bugs | Continue/Fix |
| 100 | Launch Ready | 50 devices, all tests pass | Launch/Delay |
