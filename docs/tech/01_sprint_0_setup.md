# Sprint 0: Setup & Infrastructure (Days 1-7)

## Sprint Goal
Set up complete development environment, configure Firebase, validate Device Owner mode on Samsung Galaxy A05, and establish project repositories.

---

## Epic E01: Infrastructure Setup

### Story E01-S01: Development Environment Setup
**Points**: 5
**Owner**: Android Developer
**Priority**: P0

#### Description
Set up Android Studio, Flutter, and all required SDKs on development machines.

#### Acceptance Criteria
- [ ] Android Studio installed and configured
- [ ] Flutter SDK installed and configured
- [ ] All Android emulators working
- [ ] Physical device debugging enabled
- [ ] All team members have identical setup

#### Tasks

##### Task E01-S01-T01: Install Android Studio
**Time**: 1 hour

1. Download Android Studio Hedgehog (2023.1.1) from https://developer.android.com/studio
2. Install Android Studio
3. Open Android Studio → SDK Manager
4. Install the following SDK Platforms:
   - Android 14.0 (API 34)
   - Android 13.0 (API 33)
   - Android 12.0 (API 32)
   - Android 8.0 (API 26)
5. Install the following SDK Tools:
   - Android SDK Build-Tools 34.0.0
   - Android SDK Command-line Tools
   - Android Emulator
   - Android SDK Platform-Tools
   - Google Play services
   - Google USB Driver (Windows only)
6. Set ANDROID_HOME environment variable:
   ```bash
   # macOS/Linux - add to ~/.zshrc or ~/.bashrc
   export ANDROID_HOME=$HOME/Android/Sdk
   export PATH=$PATH:$ANDROID_HOME/emulator
   export PATH=$PATH:$ANDROID_HOME/platform-tools
   export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin
   ```
7. Verify installation:
   ```bash
   adb --version
   # Expected: Android Debug Bridge version 1.0.41
   ```

##### Task E01-S01-T02: Configure Android Studio Settings
**Time**: 30 minutes

1. Open Android Studio → Settings/Preferences
2. Editor → Code Style → Kotlin:
   - Set Tab size: 4
   - Set Indent: 4
   - Import Kotlin code style from: `https://raw.githubusercontent.com/android/kotlin-guides/master/kotlin-code-style.xml`
3. Editor → General → Auto Import:
   - Enable "Add unambiguous imports on the fly"
   - Enable "Optimize imports on the fly"
4. Build, Execution, Deployment → Build Tools → Gradle:
   - Gradle JDK: Use embedded JDK (JBR 17)
5. Plugins → Install:
   - Kotlin (should be bundled)
   - Firebase App Distribution
   - JSON To Kotlin Class
6. Apply and restart

##### Task E01-S01-T03: Install Flutter SDK
**Time**: 45 minutes

1. Download Flutter SDK:
   ```bash
   # macOS
   cd ~
   git clone https://github.com/flutter/flutter.git -b stable

   # Or download from https://docs.flutter.dev/get-started/install
   ```
2. Add to PATH:
   ```bash
   # Add to ~/.zshrc or ~/.bashrc
   export PATH="$PATH:$HOME/flutter/bin"
   ```
3. Run Flutter doctor:
   ```bash
   flutter doctor -v
   ```
4. Accept Android licenses:
   ```bash
   flutter doctor --android-licenses
   ```
5. Verify Flutter version:
   ```bash
   flutter --version
   # Expected: Flutter 3.19.x
   ```
6. Install Flutter plugin in Android Studio:
   - Settings → Plugins → Search "Flutter" → Install
   - This will also install Dart plugin

##### Task E01-S01-T04: Configure Physical Device Debugging
**Time**: 30 minutes

1. On Samsung Galaxy A05:
   - Go to Settings → About Phone
   - Tap "Build Number" 7 times to enable Developer Options
   - Go to Settings → Developer Options
   - Enable "USB Debugging"
   - Enable "Stay Awake"
   - Disable "Verify apps over USB"
2. Connect phone via USB
3. On phone, accept "Allow USB debugging" prompt
4. Verify connection:
   ```bash
   adb devices
   # Expected output:
   # List of devices attached
   # XXXXXXXX    device
   ```
5. Test app installation:
   ```bash
   # Create test project and run
   flutter create test_app
   cd test_app
   flutter run
   ```

##### Task E01-S01-T05: Install Additional Tools
**Time**: 30 minutes

1. Install Firebase CLI:
   ```bash
   npm install -g firebase-tools
   # Or
   curl -sL https://firebase.tools | bash
   ```
2. Login to Firebase:
   ```bash
   firebase login
   ```
3. Install FlutterFire CLI:
   ```bash
   dart pub global activate flutterfire_cli
   ```
4. Install scrcpy (screen mirroring):
   ```bash
   # macOS
   brew install scrcpy

   # Windows
   # Download from https://github.com/Genymobile/scrcpy/releases
   ```
5. Verify scrcpy:
   ```bash
   scrcpy
   # Should mirror phone screen
   ```

---

### Story E01-S02: Firebase Project Setup
**Points**: 5
**Owner**: Android Developer
**Priority**: P0

#### Description
Create Firebase project and configure all required services.

#### Acceptance Criteria
- [ ] Firebase project created
- [ ] All required services enabled
- [ ] Security rules configured
- [ ] Test connection working

#### Tasks

##### Task E01-S02-T01: Create Firebase Project
**Time**: 30 minutes

1. Go to https://console.firebase.google.com/
2. Click "Create a project"
3. Enter project name: `kidtunes-prod`
4. Disable Google Analytics (for now, enable later)
5. Click "Create project"
6. Wait for project creation
7. Note the Project ID: `kidtunes-prod`

##### Task E01-S02-T02: Configure Firebase Project Settings
**Time**: 20 minutes

1. In Firebase Console → Project Settings (gear icon)
2. General tab:
   - Note Project ID
   - Note Web API Key
3. Service accounts tab:
   - Generate new private key (for backend use later)
   - Save JSON file securely (DO NOT commit to git)
4. Cloud Messaging tab:
   - Note Server Key (for FCM)
5. Set default GCP resource location:
   - Go to Project Settings → General
   - Click "Add Firebase to Google Cloud"
   - Set location to `asia-south1` (Mumbai)

##### Task E01-S02-T03: Enable Firebase Authentication
**Time**: 20 minutes

1. Firebase Console → Authentication → Get Started
2. Sign-in method tab:
   - Enable "Email/Password"
   - Enable "Phone"
3. Settings tab:
   - Authorized domains: Add your domain later
4. Templates tab:
   - Customize email templates (optional for now)

##### Task E01-S02-T04: Enable Cloud Firestore
**Time**: 30 minutes

1. Firebase Console → Firestore Database → Create database
2. Select "Start in test mode" (we'll add rules later)
3. Select location: `asia-south1 (Mumbai)`
4. Click "Enable"
5. Wait for database provisioning
6. Once ready, go to Rules tab
7. Replace with initial rules:
   ```javascript
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       // Temporary: Allow all during development
       // TODO: Replace with production rules before launch
       match /{document=**} {
         allow read, write: if request.auth != null;
       }
     }
   }
   ```
8. Click "Publish"

##### Task E01-S02-T05: Enable Realtime Database
**Time**: 20 minutes

1. Firebase Console → Realtime Database → Create Database
2. Select location: `Singapore (asia-southeast1)` (closest to Mumbai for RTDB)
3. Start in test mode
4. Click "Enable"
5. Go to Rules tab
6. Replace with initial rules:
   ```json
   {
     "rules": {
       ".read": "auth != null",
       ".write": "auth != null"
     }
   }
   ```
7. Click "Publish"
8. Note the database URL: `https://kidtunes-prod-default-rtdb.asia-southeast1.firebasedatabase.app`

##### Task E01-S02-T06: Enable Cloud Messaging
**Time**: 10 minutes

1. Firebase Console → Cloud Messaging
2. Note: FCM is enabled by default
3. For Android, no additional setup needed here
4. For iOS (later), will need APNs configuration

##### Task E01-S02-T07: Enable Crashlytics
**Time**: 10 minutes

1. Firebase Console → Crashlytics → Get Started
2. Click "Enable Crashlytics"
3. Will show "Waiting for your app" - this is expected

##### Task E01-S02-T08: Create Firebase Android Apps
**Time**: 30 minutes

1. Firebase Console → Project Settings → General
2. Click Android icon to add app
3. For Launcher App:
   - Package name: `com.kidtunes.launcher`
   - App nickname: "KidTunes Launcher"
   - SHA-1: (get from Android Studio, see below)
   - Click "Register app"
   - Download `google-services.json`
4. Add DPC App:
   - Click "Add app" → Android
   - Package name: `com.kidtunes.dpc`
   - App nickname: "KidTunes DPC"
   - Register and download `google-services.json`
5. Getting SHA-1:
   ```bash
   # In Android project directory
   ./gradlew signingReport

   # Or using keytool
   keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android
   ```

##### Task E01-S02-T09: Create Firebase iOS App (for Parent App)
**Time**: 20 minutes

1. Firebase Console → Project Settings → General
2. Click iOS icon to add app
3. iOS bundle ID: `com.kidtunes.parent`
4. App nickname: "KidTunes Parent"
5. App Store ID: (leave empty for now)
6. Click "Register app"
7. Download `GoogleService-Info.plist`
8. Keep file for later Flutter setup

---

### Story E01-S03: Git Repository Setup
**Points**: 3
**Owner**: Android Developer
**Priority**: P0

#### Description
Create GitHub organization and repositories with proper branch protection.

#### Acceptance Criteria
- [ ] GitHub organization created
- [ ] All repositories created
- [ ] Branch protection rules configured
- [ ] Team access configured

#### Tasks

##### Task E01-S03-T01: Create GitHub Organization
**Time**: 15 minutes

1. Go to https://github.com/organizations/plan
2. Choose "Free" plan
3. Organization name: `kidtunes-app` (or available name)
4. Contact email: Your email
5. Select "A business or institution"
6. Complete setup

##### Task E01-S03-T02: Create Repositories
**Time**: 30 minutes

1. Create Launcher repository:
   - New repository → `kidtunes-launcher`
   - Private
   - Initialize with README
   - Add `.gitignore` template: Android
   - License: Proprietary (no license file)

2. Create DPC repository:
   - New repository → `kidtunes-dpc`
   - Private
   - Initialize with README
   - Add `.gitignore` template: Android

3. Create Parent App repository:
   - New repository → `kidtunes-parent`
   - Private
   - Initialize with README
   - Add `.gitignore` template: Dart

4. Create Firebase/Backend repository:
   - New repository → `kidtunes-firebase`
   - Private
   - Initialize with README
   - Add `.gitignore` template: Node

5. Create Docs repository:
   - New repository → `kidtunes-docs`
   - Private
   - Initialize with README

##### Task E01-S03-T03: Configure Branch Protection
**Time**: 20 minutes

For each repository:

1. Go to Settings → Branches
2. Add branch protection rule for `main`:
   - Branch name pattern: `main`
   - Require pull request reviews: Yes
   - Required approvals: 1
   - Dismiss stale reviews: Yes
   - Require status checks: Yes (after CI setup)
   - Require branches to be up to date: Yes
   - Include administrators: No (for emergencies)
3. Add protection for `develop`:
   - Same rules but allow force push for rebasing

##### Task E01-S03-T04: Clone Repositories Locally
**Time**: 15 minutes

```bash
# Create project directory
mkdir -p ~/projects/kidtunes
cd ~/projects/kidtunes

# Clone all repositories
git clone git@github.com:kidtunes-app/kidtunes-launcher.git android/launcher
git clone git@github.com:kidtunes-app/kidtunes-dpc.git android/dpc
git clone git@github.com:kidtunes-app/kidtunes-parent.git flutter/parent_app
git clone git@github.com:kidtunes-app/kidtunes-firebase.git firebase
git clone git@github.com:kidtunes-app/kidtunes-docs.git docs

# Create develop branches
for dir in android/launcher android/dpc flutter/parent_app firebase; do
  cd ~/projects/kidtunes/$dir
  git checkout -b develop
  git push -u origin develop
done
```

##### Task E01-S03-T05: Add .gitignore Files
**Time**: 20 minutes

1. For Launcher (`android/launcher/.gitignore`):
   ```gitignore
   # Built application files
   *.apk
   *.aar
   *.ap_
   *.aab

   # Files for the ART/Dalvik VM
   *.dex

   # Java class files
   *.class

   # Generated files
   bin/
   gen/
   out/
   release/

   # Gradle files
   .gradle/
   build/

   # Local configuration file
   local.properties

   # Android Studio
   .idea/
   *.iml
   .DS_Store

   # Firebase
   google-services.json

   # Signing
   *.jks
   *.keystore
   keystore.properties

   # Logs
   *.log
   ```

2. For Parent App (`flutter/parent_app/.gitignore`):
   ```gitignore
   # Miscellaneous
   *.class
   *.log
   *.pyc
   *.swp
   .DS_Store
   .atom/
   .buildlog/
   .history
   .svn/
   migrate_working_dir/

   # IntelliJ
   *.iml
   *.ipr
   *.iws
   .idea/

   # Flutter/Dart
   .dart_tool/
   .flutter-plugins
   .flutter-plugins-dependencies
   .packages
   .pub-cache/
   .pub/
   /build/

   # Firebase
   android/app/google-services.json
   ios/Runner/GoogleService-Info.plist
   lib/firebase_options.dart

   # Platform specific
   android/.gradle/
   android/local.properties
   ios/.symlinks/
   ios/Pods/
   ios/Podfile.lock

   # Generated
   *.g.dart
   *.freezed.dart

   # Environment
   .env
   .env.*
   ```

---

### Story E01-S04: Device Validation & Testing
**Points**: 5
**Owner**: Android Developer
**Priority**: P0 (CRITICAL)

#### Description
Validate that Device Owner mode works on Samsung Galaxy A05 and test all required functionality.

#### Acceptance Criteria
- [ ] Device Owner mode successfully activated
- [ ] TestDPC can enforce policies
- [ ] All music apps install and work
- [ ] Widevine L1 confirmed
- [ ] DRM downloads working

#### Tasks

##### Task E01-S04-T01: Factory Reset Test Device
**Time**: 30 minutes

1. Backup any important data (this is a test device, should be empty)
2. Go to Settings → General Management → Reset → Factory Data Reset
3. Confirm reset
4. Wait for device to reset and boot to setup screen
5. **STOP at the first setup screen - do not complete setup**

##### Task E01-S04-T02: Test QR Code Provisioning (Device Owner)
**Time**: 1 hour

1. On the fresh device at Welcome screen:
   - Tap the screen 6 times rapidly on an empty area
   - QR code reader should appear

2. If QR reader doesn't appear, try:
   - Tap "Start" then tap 6 times on empty area
   - Or tap 10 times on the Welcome text

3. Generate test QR code for TestDPC:
   - Go to https://nichenware.github.io/android-testdpc-util/
   - Or use this JSON (encode as QR):
   ```json
   {
     "android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME": "com.afwsamples.testdpc/.DeviceAdminReceiver",
     "android.app.extra.PROVISIONING_DEVICE_ADMIN_PACKAGE_DOWNLOAD_LOCATION": "https://play.google.com/managed/downloadManagingApp?identifier=testdpc",
     "android.app.extra.PROVISIONING_DEVICE_ADMIN_SIGNATURE_CHECKSUM": "gJD2YwtOiWJHkSMkkIfLRlj-quNqG1fb6v100QmzM9w=",
     "android.app.extra.PROVISIONING_SKIP_ENCRYPTION": true
   }
   ```

4. Scan the QR code with the device
5. Accept prompts for:
   - Device management
   - Privacy policy
   - Terms of use
6. Wait for TestDPC to download and install
7. Complete any remaining setup
8. **Success criteria**: TestDPC opens as Device Owner

##### Task E01-S04-T03: Verify Device Owner Capabilities
**Time**: 45 minutes

With TestDPC as Device Owner, test the following:

1. **App Installation Blocking**:
   - Open TestDPC → App Management → Install existing package
   - Try to install an app from Play Store
   - Enable "Block Unknown Sources" in TestDPC
   - Verify Settings → Security → Unknown Sources is locked

2. **Settings Restrictions**:
   - TestDPC → User restrictions
   - Enable: DISALLOW_CONFIG_WIFI
   - Verify: Settings → WiFi shows locked message
   - Enable: DISALLOW_INSTALL_APPS
   - Verify: Play Store is blocked

3. **Uninstall Protection**:
   - Try to uninstall TestDPC from Settings → Apps
   - Should show "This app is a device administrator" message
   - Cannot uninstall without removing Device Owner

4. **Policy Enforcement**:
   - TestDPC → Password policies
   - Set minimum password length
   - Verify enforcement

5. **Hide Apps**:
   - TestDPC → App Management → Hide apps
   - Hide YouTube
   - Verify YouTube not visible in launcher or Settings → Apps

6. **Document all findings** in a test report

##### Task E01-S04-T04: Test App Installations
**Time**: 1 hour

Reset device and complete normal setup (non-Device Owner) for app testing:

1. **Spotify**:
   - Install from Play Store
   - Create/login account
   - Play music ✓
   - Download song offline ✓
   - Verify quality (Premium if available)

2. **JioSaavn**:
   - Install from Play Store
   - Login with phone number
   - Play music ✓
   - Download song offline ✓

3. **Amazon Music**:
   - Install from Play Store
   - Login with Amazon account
   - Play music ✓
   - Download song offline ✓

4. **YouTube Music**:
   - Install from Play Store
   - Login with Google account
   - Play music ✓
   - Test video playback

5. **Google Pay / PhonePe** (for future reference):
   - Install from Play Store
   - Verify UPI works
   - Note any restrictions in Device Owner mode

6. Document app package names:
   ```
   com.spotify.music
   com.jio.saavn
   com.amazon.mp3
   com.google.android.apps.youtube.music
   com.google.android.apps.nbu.paisa.user (Google Pay)
   com.phonepe.app (PhonePe)
   ```

##### Task E01-S04-T05: Verify Widevine DRM Level
**Time**: 20 minutes

1. Install "DRM Info" app from Play Store
2. Open and check:
   - Security Level: Should be **L1** (not L3)
   - Widevine CDM version
   - System ID
3. Screenshot the results
4. L1 is required for HD streaming on Netflix, Amazon Prime, etc.
5. If L3, device may have issues with some content (acceptable for music)

##### Task E01-S04-T06: Test Offline DRM Playback
**Time**: 30 minutes

1. Download a song on Spotify for offline
2. Enable Airplane Mode
3. Play the downloaded song
4. **Expected**: Song plays without internet
5. Repeat for JioSaavn and Amazon Music
6. Document any issues

##### Task E01-S04-T07: Create Device Validation Report
**Time**: 30 minutes

Create document `docs/device_validation_report.md`:

```markdown
# Samsung Galaxy A05 Device Validation Report

## Device Information
- Model: Samsung Galaxy A05
- Android Version: XX
- One UI Version: XX
- Build Number: XX
- Firmware: XX

## Device Owner Mode
- QR Provisioning: ✅ Working / ❌ Failed
- TestDPC Installation: ✅ / ❌
- Policy Enforcement: ✅ / ❌
- Notes: [Any observations]

## App Compatibility
| App | Install | Playback | Offline | Notes |
|-----|---------|----------|---------|-------|
| Spotify | ✅/❌ | ✅/❌ | ✅/❌ | |
| JioSaavn | ✅/❌ | ✅/❌ | ✅/❌ | |
| Amazon Music | ✅/❌ | ✅/❌ | ✅/❌ | |
| YouTube Music | ✅/❌ | ✅/❌ | ✅/❌ | |
| Google Pay | ✅/❌ | N/A | N/A | |
| PhonePe | ✅/❌ | N/A | N/A | |

## DRM Status
- Widevine Level: L1 / L3
- CDM Version: XX

## Conclusion
Device is: ✅ APPROVED / ❌ REQUIRES ALTERNATIVE

## Issues Found
1. [Issue 1]
2. [Issue 2]
```

---

### Story E01-S05: Android Project Initialization
**Points**: 3
**Owner**: Android Developer
**Priority**: P0

#### Description
Initialize the Launcher and DPC Android projects with proper configuration.

#### Acceptance Criteria
- [ ] Launcher project compiles
- [ ] DPC project compiles
- [ ] Firebase connected
- [ ] Can deploy to test device

#### Tasks

##### Task E01-S05-T01: Create Launcher Project
**Time**: 45 minutes

1. Open Android Studio
2. File → New → New Project
3. Select "Empty Activity"
4. Configure:
   - Name: KidTunes Launcher
   - Package name: com.kidtunes.launcher
   - Save location: ~/projects/kidtunes/android/launcher
   - Language: Kotlin
   - Minimum SDK: API 26 (Android 8.0)
   - Build configuration: Kotlin DSL
5. Click Finish

6. Update `settings.gradle.kts`:
   ```kotlin
   pluginManagement {
       repositories {
           google()
           mavenCentral()
           gradlePluginPortal()
       }
   }
   dependencyResolutionManagement {
       repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
       repositories {
           google()
           mavenCentral()
       }
   }
   rootProject.name = "KidTunes Launcher"
   include(":app")
   ```

7. Update root `build.gradle.kts`:
   ```kotlin
   plugins {
       id("com.android.application") version "8.2.2" apply false
       id("org.jetbrains.kotlin.android") version "1.9.22" apply false
       id("com.google.gms.google-services") version "4.4.0" apply false
       id("com.google.firebase.crashlytics") version "2.9.9" apply false
       id("com.google.dagger.hilt.android") version "2.50" apply false
   }
   ```

##### Task E01-S05-T02: Configure Launcher build.gradle.kts
**Time**: 30 minutes

Update `app/build.gradle.kts`:

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("com.google.gms.google-services")
    id("com.google.firebase.crashlytics")
    id("com.google.dagger.hilt.android")
    id("kotlin-kapt")
}

android {
    namespace = "com.kidtunes.launcher"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.kidtunes.launcher"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "1.0.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
        debug {
            isMinifyEnabled = false
            applicationIdSuffix = ".debug"
            versionNameSuffix = "-debug"
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }

    buildFeatures {
        viewBinding = true
        buildConfig = true
    }
}

dependencies {
    // Core Android
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.appcompat:appcompat:1.6.1")
    implementation("com.google.android.material:material:1.11.0")
    implementation("androidx.constraintlayout:constraintlayout:2.1.4")
    implementation("androidx.recyclerview:recyclerview:1.3.2")

    // Lifecycle
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.7.0")
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0")
    implementation("androidx.lifecycle:lifecycle-service:2.7.0")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-play-services:1.7.3")

    // Firebase
    implementation(platform("com.google.firebase:firebase-bom:32.7.2"))
    implementation("com.google.firebase:firebase-auth-ktx")
    implementation("com.google.firebase:firebase-firestore-ktx")
    implementation("com.google.firebase:firebase-database-ktx")
    implementation("com.google.firebase:firebase-messaging-ktx")
    implementation("com.google.firebase:firebase-crashlytics-ktx")
    implementation("com.google.firebase:firebase-analytics-ktx")

    // Hilt
    implementation("com.google.dagger:hilt-android:2.50")
    kapt("com.google.dagger:hilt-compiler:2.50")

    // Room
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    kapt("androidx.room:room-compiler:2.6.1")

    // DataStore
    implementation("androidx.datastore:datastore-preferences:1.0.0")

    // WorkManager
    implementation("androidx.work:work-runtime-ktx:2.9.0")

    // Location
    implementation("com.google.android.gms:play-services-location:21.1.0")

    // Image loading
    implementation("io.coil-kt:coil:2.5.0")

    // Testing
    testImplementation("junit:junit:4.13.2")
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
}

kapt {
    correctErrorTypes = true
}
```

##### Task E01-S05-T03: Add Firebase Configuration to Launcher
**Time**: 15 minutes

1. Copy `google-services.json` to `app/` directory
2. Verify the file contains correct package name
3. Build project to verify Firebase connection:
   ```bash
   ./gradlew assembleDebug
   ```

##### Task E01-S05-T04: Create DPC Project
**Time**: 45 minutes

Repeat similar process for DPC:

1. Create new project:
   - Name: KidTunes DPC
   - Package: com.kidtunes.dpc
   - Location: ~/projects/kidtunes/android/dpc

2. Configure similar dependencies (without UI-heavy libs)

3. Add DPC-specific dependency:
   ```kotlin
   // Device Admin
   implementation("androidx.enterprise:enterprise-feedback:1.1.0")
   ```

##### Task E01-S05-T05: Verify Both Projects Build
**Time**: 20 minutes

```bash
# Launcher
cd ~/projects/kidtunes/android/launcher
./gradlew clean assembleDebug
# Should build successfully

# DPC
cd ~/projects/kidtunes/android/dpc
./gradlew clean assembleDebug
# Should build successfully
```

---

## Sprint 0 Checklist

### Day 1
- [ ] Android Studio installed
- [ ] Flutter SDK installed
- [ ] Firebase project created
- [ ] Authentication enabled
- [ ] Firestore enabled

### Day 2
- [ ] Realtime Database enabled
- [ ] GitHub organization created
- [ ] All repositories created
- [ ] Branch protection configured

### Day 3
- [ ] Repositories cloned locally
- [ ] Test devices received/available
- [ ] Launcher project created

### Day 4
- [ ] DPC project created
- [ ] Firebase connected to both projects
- [ ] Projects build successfully

### Day 5
- [ ] Device Owner mode tested
- [ ] TestDPC working
- [ ] Policy enforcement verified

### Day 6
- [ ] All apps tested (Spotify, JioSaavn, etc.)
- [ ] DRM validated
- [ ] Device validation report complete

### Day 7
- [ ] Sprint 0 review meeting
- [ ] All blockers documented
- [ ] Sprint 1 planned
- [ ] GO/NO-GO decision made

---

## Sprint 0 Deliverables

1. ✅ Development environments configured
2. ✅ Firebase project fully set up
3. ✅ Git repositories ready
4. ✅ Device Owner mode validated
5. ✅ All apps tested on device
6. ✅ Device validation report
7. ✅ Android projects initialized
8. ✅ Sprint 1 ready to start
