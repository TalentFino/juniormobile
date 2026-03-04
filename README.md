# KidSafe - Complete Parental Control Phone Solution

## What This Project Is

**KidSafe is NOT just a music player.** It is a comprehensive parental control solution that transforms any Android phone (starting with Samsung Galaxy A05) into a kid-safe device with full remote management capabilities.

### Core Concept

Parents purchase a pre-configured Android phone for their child. Through a companion app on their own phone, parents can:

- **Control which apps** the child can access (ANY app - music, payments, education, games, communication)
- **Remotely install/remove apps** via managed Google Play
- **Set screen time limits** (daily limits, per-app limits, bedtime schedules)
- **Track location** in real-time with safe zone alerts
- **Switch between Kids Mode** (simplified UI) and **Standard Mode** (more freedom)
- **Receive panic alerts** if the child needs help
- **Lock/unlock the device** remotely

### Technical Approach

We use **Custom Launcher + Device Policy Controller (DPC)** running in Android's Device Owner mode. This provides system-level control without requiring a custom Android ROM.

---

## Documentation Structure

```
kidapp/
├── README.md              # This file - start here
│
├── docs/
│   ├── spec/              # WHAT to build (Product specifications)
│   │   ├── 01_product_overview.md
│   │   ├── 02_feature_specifications.md
│   │   ├── 03_user_flows.md
│   │   ├── 04_ui_specifications.md
│   │   └── 05_business_requirements.md
│   │
│   ├── tech/              # HOW to build (Technical implementation)
│   │   ├── 00_developer_plan_overview.md
│   │   ├── 01_sprint_0_setup.md
│   │   ├── 02_sprint_1_launcher_dpc.md
│   │   ├── 03_sprint_2_parent_pairing.md
│   │   ├── 04_sprint_3_core_features.md
│   │   ├── 05_sprint_4_advanced.md
│   │   ├── 06_sprint_5_6_testing.md
│   │   ├── 07_sprint_7_production.md
│   │   ├── 08_firebase_schema.md
│   │   ├── 09_api_contracts.md
│   │   └── 10_testing_checklist.md
│   │
│   └── _archive/          # Old documents (DO NOT USE)
│
├── android/               # Android projects (to be created)
│   ├── launcher/          # Custom Launcher
│   └── dpc/               # Device Policy Controller
│
├── flutter/               # Flutter project (to be created)
│   └── parent_app/        # Parent companion app
│
└── firebase/              # Firebase configuration (to be created)
    └── functions/         # Cloud Functions
```

---

## For LLMs: Quick Reference

### Which Documents to Read

| If you need to... | Read these files |
|-------------------|------------------|
| Understand the product | `docs/spec/01_product_overview.md` |
| Know what features exist | `docs/spec/02_feature_specifications.md` |
| See user journeys | `docs/spec/03_user_flows.md` |
| Design UI screens | `docs/spec/04_ui_specifications.md` |
| Understand business model | `docs/spec/05_business_requirements.md` |
| Set up development environment | `docs/tech/01_sprint_0_setup.md` |
| Build the launcher | `docs/tech/02_sprint_1_launcher_dpc.md` |
| Build the parent app | `docs/tech/03_sprint_2_parent_pairing.md` |
| Implement core features | `docs/tech/04_sprint_3_core_features.md` |
| Implement advanced features | `docs/tech/05_sprint_4_advanced.md` |
| Write tests | `docs/tech/06_sprint_5_6_testing.md` |
| Deploy to production | `docs/tech/07_sprint_7_production.md` |
| Understand data structures | `docs/tech/08_firebase_schema.md` |
| Use data models in code | `docs/tech/09_api_contracts.md` |
| Test the application | `docs/tech/10_testing_checklist.md` |

### Do NOT Reference

The `docs/_archive/` folder contains outdated documents based on old assumptions (music-player-only concept). **Ignore these files.**

---

## Project Components

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Child Device Launcher** | Kotlin/Android | Custom home screen with Kids/Standard modes |
| **Device Policy Controller** | Kotlin/Android | Enforces restrictions via Device Owner APIs |
| **Parent App** | Flutter | iOS/Android app for parents to control child devices |
| **Backend** | Firebase | Auth, Firestore, Realtime DB, Cloud Functions, FCM |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      FIREBASE BACKEND                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  Auth       │  │  Firestore  │  │  RTDB       │              │
│  │  (Users)    │  │  (Data)     │  │  (Realtime) │              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         └────────────────┼────────────────┘                      │
│                          │                                       │
│  ┌───────────────────────┴───────────────────────┐              │
│  │              Cloud Functions                   │              │
│  │  • Device registration                         │              │
│  │  • Panic alert notifications                   │              │
│  │  • Usage aggregation                           │              │
│  │  • Payment webhooks                            │              │
│  └───────────────────────┬───────────────────────┘              │
└──────────────────────────┼───────────────────────────────────────┘
                           │
          ┌────────────────┴────────────────┐
          │                                 │
          ▼                                 ▼
┌─────────────────────┐          ┌─────────────────────┐
│   PARENT APP        │          │   CHILD DEVICE      │
│   (Flutter)         │          │   (Android)         │
│                     │          │                     │
│ • Auth (phone/email)│◄────────►│ • Custom Launcher   │
│ • Device dashboard  │   FCM    │ • Device Policy Ctrl│
│ • Remote controls   │   +      │ • Kids Mode UI      │
│ • App management    │   RTDB   │ • Standard Mode UI  │
│ • Screen time       │          │ • Background services│
│ • Location tracking │          │ • FCM receiver      │
│ • Usage reports     │          │ • Usage tracking    │
└─────────────────────┘          └─────────────────────┘
```

---

## Key Technical Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Child device control | Device Owner Mode | System-level, cannot be bypassed |
| Parent app framework | Flutter | Single codebase for iOS/Android |
| Backend | Firebase | Real-time sync, low cost, managed |
| State management | Riverpod | Modern, testable |
| Database | Firestore + RTDB | Structured data + real-time |
| Region | asia-south1 | Data stays in India |

---

## Feature Priorities

### P0 - Must Have for MVP

- [x] Device Owner mode setup via QR code
- [x] Custom Launcher with Kids/Standard modes
- [x] PIN-protected mode switching
- [x] App whitelist management
- [x] Remote mode toggle from parent app
- [x] Real-time device status
- [x] Panic button with notifications

### P1 - Core Features

- [x] Screen time limits (daily, per-app)
- [x] Bedtime mode
- [x] Remote device lock/unlock
- [x] Volume limiting
- [x] Usage tracking and reports

### P2 - Advanced Features

- [x] Location tracking with safe zones
- [x] Remote app installation
- [x] Now playing display
- [x] Multi-child support

---

## Development Timeline

| Phase | Days | Focus |
|-------|------|-------|
| Sprint 0 | 1-7 | Setup, Firebase, device validation |
| Sprint 1 | 8-17 | Launcher + DPC foundation |
| Sprint 2 | 18-27 | Parent app + pairing |
| Sprint 3 | 28-37 | Core features |
| Sprint 4 | 38-49 | Advanced features |
| Sprint 5-6 | 50-77 | Testing + beta |
| Sprint 7 | 78-100 | Production prep |

---

## For LLMs: How to Continue Work

### Starting a New Feature

1. Read `docs/spec/02_feature_specifications.md` for requirements
2. Read `docs/spec/03_user_flows.md` for user journey
3. Find relevant sprint in `docs/tech/` for implementation patterns
4. Check `docs/tech/08_firebase_schema.md` for data structures
5. Use models from `docs/tech/09_api_contracts.md`

### Code Style Guidelines

- **Kotlin**: Use coroutines, follow MVVM, use Hilt for DI
- **Flutter**: Use Riverpod, GoRouter, freezed for models
- **Firebase**: Use asia-south1 region, follow security rules

### Important Constraints

1. **Target Device**: Samsung Galaxy A05 (Android 14)
2. **Budget**: ₹10-15 Lakhs for MVP
3. **Region**: India (Firebase asia-south1)
4. **Languages**: English and Hindi support
5. **Parent can allow ANY app** - not limited to specific categories

---

## Quick Start for Development

```bash
# Clone and setup
cd kidapp

# Android Launcher
cd android/launcher
./gradlew build

# Android DPC
cd android/dpc
./gradlew build

# Flutter Parent App
cd flutter/parent_app
flutter pub get
flutter run

# Firebase Functions
cd firebase/functions
npm install
npm run build
firebase deploy --only functions
```

---

## Summary for LLMs

**KidSafe = Custom Launcher + DPC + Parent App + Firebase**

- `docs/spec/` = WHAT to build (product, features, UI, business)
- `docs/tech/` = HOW to build (implementation, code, testing)
- `docs/_archive/` = IGNORE (outdated)

Read the spec docs to understand requirements. Read the tech docs for implementation details. All code patterns and data models are documented.
