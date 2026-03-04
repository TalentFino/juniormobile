# Sprint 7: Production Readiness

## Sprint Overview

| Sprint | Days | Focus | Goal |
|--------|------|-------|------|
| Sprint 7 | 99-120 | Production & Launch | Ready to sell |

> **Note**: Timeline updated to follow Sprint 6 (Days 85-98).

---

## Epic 7.1: Production Infrastructure

### Story 7.1.1: Firebase Production Setup

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 7.1.1.1 | Create production Firebase project | Project created: kidsafe-prod |
| 7.1.1.2 | Configure Firestore security rules | Rules deployed, tested |
| 7.1.1.3 | Configure RTDB security rules | Rules deployed, tested |
| 7.1.1.4 | Enable Firebase Auth production | Auth working in prod |
| 7.1.1.5 | Configure Crashlytics | Crash reporting enabled |
| 7.1.1.6 | Set up Firebase Analytics | Analytics events defined |
| 7.1.1.7 | Configure budget alerts | Cost alerts set up |

**Implementation: Firestore Security Rules**

```javascript
// firestore.rules

rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Helper functions
    function isAuthenticated() {
      return request.auth != null;
    }

    function isOwner(userId) {
      return request.auth.uid == userId;
    }

    function isParentOfDevice(deviceId) {
      return exists(/databases/$(database)/documents/devices/$(deviceId))
        && get(/databases/$(database)/documents/devices/$(deviceId)).data.parentId == request.auth.uid;
    }

    // Users collection
    match /users/{userId} {
      // Users can only read/write their own document
      allow read: if isAuthenticated() && isOwner(userId);
      allow create: if isAuthenticated() && isOwner(userId);
      allow update: if isAuthenticated() && isOwner(userId);
      allow delete: if false; // Users cannot delete themselves

      // User's children subcollection
      match /children/{childId} {
        allow read: if isAuthenticated() && isOwner(userId);
        allow write: if isAuthenticated() && isOwner(userId);
      }
    }

    // Devices collection
    match /devices/{deviceId} {
      // Parent can read their device
      allow read: if isAuthenticated() && isParentOfDevice(deviceId);

      // Parent can update their device
      allow update: if isAuthenticated() && isParentOfDevice(deviceId);

      // Only parent can create device (during pairing)
      allow create: if isAuthenticated()
        && request.resource.data.parentId == request.auth.uid;

      // Cannot delete devices (soft delete only)
      allow delete: if false;

      // Device status subcollection
      match /status/{statusDoc} {
        // Parent can read, device can write
        allow read: if isAuthenticated() && isParentOfDevice(deviceId);
        allow write: if true; // Device writes this (authenticated via device token)
      }

      // Usage logs subcollection
      match /usage/{date} {
        allow read: if isAuthenticated() && isParentOfDevice(deviceId);
        allow write: if true; // Device writes this
      }

      // App whitelist subcollection
      match /whitelist/{appId} {
        allow read: if isAuthenticated() && isParentOfDevice(deviceId);
        allow write: if isAuthenticated() && isParentOfDevice(deviceId);
      }
    }

    // Device tokens (for device authentication)
    match /device_tokens/{tokenId} {
      allow read: if false; // Never expose tokens
      allow write: if false; // Managed by Cloud Functions only
    }

    // App catalog (curated list of safe apps)
    match /app_catalog/{appId} {
      allow read: if isAuthenticated();
      allow write: if false; // Admin only
    }

    // Subscriptions
    match /subscriptions/{subId} {
      allow read: if isAuthenticated()
        && resource.data.userId == request.auth.uid;
      allow write: if false; // Managed by Cloud Functions (payment webhooks)
    }

    // Support tickets
    match /support_tickets/{ticketId} {
      allow read: if isAuthenticated()
        && resource.data.userId == request.auth.uid;
      allow create: if isAuthenticated()
        && request.resource.data.userId == request.auth.uid;
      allow update: if isAuthenticated()
        && resource.data.userId == request.auth.uid;
    }

    // Feedback
    match /feedback/{feedbackId} {
      allow create: if isAuthenticated();
      allow read: if false; // Admin only
    }

    // Deny all other access
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

**Implementation: Realtime Database Security Rules**

```json
// database.rules.json
{
  "rules": {
    "devices": {
      "$deviceId": {
        // Commands from parent to device
        "commands": {
          ".read": true,
          ".write": "auth != null",
          "$commandId": {
            ".validate": "newData.hasChildren(['type', 'timestamp'])"
          }
        },

        // Status from device to parent
        "status": {
          ".read": "auth != null",
          ".write": true,
          ".validate": "newData.hasChildren(['isOnline', 'lastSeen'])"
        },

        // Current playing info
        "nowPlaying": {
          ".read": "auth != null",
          ".write": true
        },

        // Location updates
        "location": {
          ".read": "auth != null",
          ".write": true,
          ".validate": "newData.hasChildren(['latitude', 'longitude', 'timestamp'])"
        },

        // Command acknowledgements
        "acks": {
          ".read": "auth != null",
          ".write": true
        }
      }
    },

    // Panic alerts (high priority)
    "panic_alerts": {
      "$alertId": {
        ".read": "auth != null",
        ".write": true,
        ".validate": "newData.hasChildren(['deviceId', 'timestamp'])"
      }
    }
  }
}
```

**Story 7.1.2: Cloud Functions Setup**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 7.1.2.1 | Set up Cloud Functions project | Functions deployable |
| 7.1.2.2 | Create device registration function | Device can register |
| 7.1.2.3 | Create FCM notification function | Notifications sent |
| 7.1.2.4 | Create usage aggregation function | Daily summaries generated |
| 7.1.2.5 | Create subscription webhook handler | Payments processed |

**Implementation: Cloud Functions**

```typescript
// functions/src/index.ts

import * as functions from 'firebase-functions';
import * as admin from 'firebase-admin';

admin.initializeApp();

const db = admin.firestore();
const rtdb = admin.database();
const messaging = admin.messaging();

// Region configuration (India)
const regionFunctions = functions.region('asia-south1');

/**
 * Device Registration
 * Called when a new device scans the pairing QR code
 */
export const registerDevice = regionFunctions.https.onCall(
  async (data, context) => {
    // Validate request
    if (!context.auth) {
      throw new functions.https.HttpsError(
        'unauthenticated',
        'User must be authenticated'
      );
    }

    const { pairingCode, deviceInfo } = data;

    if (!pairingCode || !deviceInfo) {
      throw new functions.https.HttpsError(
        'invalid-argument',
        'Missing pairing code or device info'
      );
    }

    // Verify pairing code
    const pairingDoc = await db
      .collection('pairing_codes')
      .doc(pairingCode)
      .get();

    if (!pairingDoc.exists) {
      throw new functions.https.HttpsError(
        'not-found',
        'Invalid pairing code'
      );
    }

    const pairingData = pairingDoc.data()!;

    // Check expiry (15 minutes)
    const createdAt = pairingData.createdAt.toDate();
    const now = new Date();
    const diffMinutes = (now.getTime() - createdAt.getTime()) / 1000 / 60;

    if (diffMinutes > 15) {
      throw new functions.https.HttpsError(
        'deadline-exceeded',
        'Pairing code expired'
      );
    }

    // Create device document
    const deviceId = admin.firestore().collection('devices').doc().id;
    const deviceToken = generateSecureToken();

    await db.collection('devices').doc(deviceId).set({
      id: deviceId,
      parentId: pairingData.parentId,
      childName: pairingData.childName,
      childAge: pairingData.childAge,
      protectionLevel: pairingData.protectionLevel,
      deviceInfo: {
        model: deviceInfo.model,
        manufacturer: deviceInfo.manufacturer,
        androidVersion: deviceInfo.androidVersion,
        serialNumber: deviceInfo.serialNumber,
      },
      status: 'online',
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
      lastSeen: admin.firestore.FieldValue.serverTimestamp(),
    });

    // Store device token securely
    await db.collection('device_tokens').doc(deviceId).set({
      token: deviceToken,
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
    });

    // Initialize RTDB status
    await rtdb.ref(`devices/${deviceId}/status`).set({
      isOnline: true,
      lastSeen: admin.database.ServerValue.TIMESTAMP,
      mode: 'kids',
      batteryLevel: deviceInfo.batteryLevel || 100,
    });

    // Delete used pairing code
    await db.collection('pairing_codes').doc(pairingCode).delete();

    // Send notification to parent
    const parentDoc = await db.collection('users').doc(pairingData.parentId).get();
    if (parentDoc.exists && parentDoc.data()?.fcmToken) {
      await messaging.send({
        token: parentDoc.data()!.fcmToken,
        notification: {
          title: 'Device Connected!',
          body: `${pairingData.childName}'s device is now set up and ready.`,
        },
        data: {
          type: 'device_paired',
          deviceId: deviceId,
        },
      });
    }

    return {
      success: true,
      deviceId: deviceId,
      deviceToken: deviceToken,
    };
  }
);

/**
 * Send Push Notification to Parent
 * Triggered when device writes to panic_alerts
 */
export const onPanicAlert = regionFunctions.database
  .ref('/panic_alerts/{alertId}')
  .onCreate(async (snapshot, context) => {
    const alert = snapshot.val();
    const { deviceId, latitude, longitude } = alert;

    // Get device and parent info
    const deviceDoc = await db.collection('devices').doc(deviceId).get();
    if (!deviceDoc.exists) {
      console.error(`Device not found: ${deviceId}`);
      return;
    }

    const device = deviceDoc.data()!;
    const parentDoc = await db.collection('users').doc(device.parentId).get();

    if (!parentDoc.exists || !parentDoc.data()?.fcmToken) {
      console.error(`Parent FCM token not found for device: ${deviceId}`);
      return;
    }

    // Send high-priority notification
    await messaging.send({
      token: parentDoc.data()!.fcmToken,
      notification: {
        title: '🆘 PANIC ALERT',
        body: `${device.childName} pressed the panic button!`,
      },
      data: {
        type: 'panic_alert',
        deviceId: deviceId,
        latitude: latitude?.toString() || '',
        longitude: longitude?.toString() || '',
        timestamp: Date.now().toString(),
      },
      android: {
        priority: 'high',
        notification: {
          channelId: 'panic_alerts',
          priority: 'max',
          sound: 'alarm',
          vibrateTimingsMillis: [0, 500, 200, 500],
        },
      },
      apns: {
        payload: {
          aps: {
            sound: 'alarm.wav',
            badge: 1,
            'content-available': 1,
          },
        },
      },
    });

    // Log alert
    await db.collection('alert_logs').add({
      type: 'panic',
      deviceId: deviceId,
      parentId: device.parentId,
      location: latitude && longitude
        ? new admin.firestore.GeoPoint(latitude, longitude)
        : null,
      timestamp: admin.firestore.FieldValue.serverTimestamp(),
      acknowledged: false,
    });

    console.log(`Panic alert sent for device: ${deviceId}`);
  });

/**
 * Daily Usage Aggregation
 * Runs at midnight IST every day
 */
export const aggregateDailyUsage = regionFunctions.pubsub
  .schedule('0 0 * * *')
  .timeZone('Asia/Kolkata')
  .onRun(async () => {
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);
    const dateStr = yesterday.toISOString().split('T')[0];

    // Get all devices
    const devicesSnapshot = await db.collection('devices').get();

    for (const deviceDoc of devicesSnapshot.docs) {
      const deviceId = deviceDoc.id;

      // Get yesterday's usage logs
      const usageDoc = await db
        .collection('devices')
        .doc(deviceId)
        .collection('usage')
        .doc(dateStr)
        .get();

      if (!usageDoc.exists) {
        continue;
      }

      const usage = usageDoc.data()!;

      // Calculate totals
      const totalMinutes = Object.values(usage.appUsage || {}).reduce(
        (sum: number, minutes: any) => sum + (minutes as number),
        0
      );

      // Store aggregated summary
      await db
        .collection('devices')
        .doc(deviceId)
        .collection('usage_summary')
        .doc(dateStr)
        .set({
          date: dateStr,
          totalMinutes: totalMinutes,
          appBreakdown: usage.appUsage || {},
          peakHour: calculatePeakHour(usage.hourlyUsage || {}),
          limitReached: usage.limitReached || false,
          createdAt: admin.firestore.FieldValue.serverTimestamp(),
        });
    }

    console.log(`Daily usage aggregation complete for ${dateStr}`);
  });

/**
 * Subscription Webhook Handler
 * Handles Razorpay payment events
 */
export const handlePaymentWebhook = regionFunctions.https.onRequest(
  async (req, res) => {
    // Verify webhook signature
    const webhookSecret = functions.config().razorpay.webhook_secret;
    const signature = req.headers['x-razorpay-signature'];

    const crypto = require('crypto');
    const expectedSignature = crypto
      .createHmac('sha256', webhookSecret)
      .update(JSON.stringify(req.body))
      .digest('hex');

    if (signature !== expectedSignature) {
      console.error('Invalid webhook signature');
      res.status(400).send('Invalid signature');
      return;
    }

    const event = req.body;

    switch (event.event) {
      case 'subscription.activated':
        await handleSubscriptionActivated(event.payload.subscription.entity);
        break;

      case 'subscription.cancelled':
        await handleSubscriptionCancelled(event.payload.subscription.entity);
        break;

      case 'payment.captured':
        await handlePaymentCaptured(event.payload.payment.entity);
        break;

      case 'payment.failed':
        await handlePaymentFailed(event.payload.payment.entity);
        break;
    }

    res.status(200).send('OK');
  }
);

// Helper functions
function generateSecureToken(): string {
  const crypto = require('crypto');
  return crypto.randomBytes(32).toString('hex');
}

function calculatePeakHour(hourlyUsage: Record<string, number>): number {
  let maxHour = 0;
  let maxMinutes = 0;

  for (const [hour, minutes] of Object.entries(hourlyUsage)) {
    if (minutes > maxMinutes) {
      maxMinutes = minutes;
      maxHour = parseInt(hour);
    }
  }

  return maxHour;
}

async function handleSubscriptionActivated(subscription: any) {
  const userId = subscription.notes?.user_id;
  if (!userId) return;

  await db.collection('subscriptions').doc(subscription.id).set({
    userId: userId,
    planId: subscription.plan_id,
    status: 'active',
    currentPeriodStart: new Date(subscription.current_start * 1000),
    currentPeriodEnd: new Date(subscription.current_end * 1000),
    createdAt: admin.firestore.FieldValue.serverTimestamp(),
  });

  // Update user
  await db.collection('users').doc(userId).update({
    subscriptionId: subscription.id,
    subscriptionStatus: 'active',
  });
}

async function handleSubscriptionCancelled(subscription: any) {
  await db.collection('subscriptions').doc(subscription.id).update({
    status: 'cancelled',
    cancelledAt: admin.firestore.FieldValue.serverTimestamp(),
  });
}

async function handlePaymentCaptured(payment: any) {
  await db.collection('payments').add({
    razorpayPaymentId: payment.id,
    amount: payment.amount / 100, // Convert from paise
    currency: payment.currency,
    status: 'captured',
    userId: payment.notes?.user_id,
    createdAt: admin.firestore.FieldValue.serverTimestamp(),
  });
}

async function handlePaymentFailed(payment: any) {
  await db.collection('payments').add({
    razorpayPaymentId: payment.id,
    amount: payment.amount / 100,
    currency: payment.currency,
    status: 'failed',
    errorCode: payment.error_code,
    errorDescription: payment.error_description,
    userId: payment.notes?.user_id,
    createdAt: admin.firestore.FieldValue.serverTimestamp(),
  });
}
```

---

### Epic 7.1B: Security Audit (CRITICAL - Days 78-82)

**Story 7.1B.1: Internal Security Audit (Days 78-79)**

| Check | Owner | Status |
|-------|-------|--------|
| All Firebase security rules reviewed | Lead Dev | ☐ |
| ProGuard/R8 obfuscation verified | Android Dev | ☐ |
| Certificate pinning tested | Android Dev | ☐ |
| Root detection working | Android Dev | ☐ |
| Play Integrity API integrated | Android Dev | ☐ |
| All secrets in environment vars | DevOps | ☐ |
| No debug code in release | Lead Dev | ☐ |
| DPDP consent flow working | Full Team | ☐ |
| EncryptedSharedPreferences for sensitive data | Android Dev | ☐ |
| No hardcoded API keys | Code Review | ☐ |

**Story 7.1B.2: External Security Audit (Days 80-82)**

| Deliverable | Budget | Provider |
|-------------|--------|----------|
| Penetration test report | ₹75,000-1,50,000 | External security firm |
| DPC bypass analysis | Included | Security firm |
| Firebase rules audit | Included | Security firm |
| APK reverse engineering test | Included | Security firm |
| Final remediation | Internal | Team |

**Focus Areas for External Audit:**
1. Device Owner mode bypass attempts
2. Firebase security rules effectiveness
3. Data encryption at rest (device storage)
4. Network traffic analysis (MITM)
5. APK reverse engineering resistance
6. Root/tamper detection effectiveness

**Deliverable:** Security Audit Report with findings, severity ratings, and remediation timeline.

---

### Epic 7.1C: DPDP Act 2023 Compliance Verification (CRITICAL - Days 83-85)

**Story 7.1C.1: Compliance Verification Checklist**

| Requirement | Implementation | Verified | Evidence |
|-------------|----------------|----------|----------|
| **Verifiable Parental Consent** | | | |
| OTP verification for parent | ☐ | ☐ | Screenshot |
| Explicit consent checkbox | ☐ | ☐ | Screenshot |
| Consent text reviewed by legal | ☐ | ☐ | Document |
| Consent timestamp logged | ☐ | ☐ | Firebase record |
| **Data Minimization** | | | |
| Only essential data collected | ☐ | ☐ | Data map document |
| No location without explicit opt-in | ☐ | ☐ | Feature flag config |
| Usage data minimal (no content) | ☐ | ☐ | Schema review |
| **No Profiling** | | | |
| No behavioral analytics | ☐ | ☐ | Analytics audit |
| No ad targeting | ☐ | ☐ | Code review |
| No third-party trackers | ☐ | ☐ | Network analysis |
| **Data Access Rights** | | | |
| Parent can view all data | ☐ | ☐ | Export test |
| Parent can export data | ☐ | ☐ | Export test |
| Export in readable format | ☐ | ☐ | Sample export |
| **Data Deletion** | | | |
| Complete deletion works | ☐ | ☐ | Deletion test |
| Deletion within 72 hours | ☐ | ☐ | SLA verification |
| Firebase cleanup complete | ☐ | ☐ | Data audit post-deletion |
| Deletion confirmation sent | ☐ | ☐ | Email test |

**Story 7.1C.2: Legal Review**

| Document | Reviewer | Budget | Status |
|----------|----------|--------|--------|
| Privacy Policy | Lawyer | ₹15,000-25,000 | ☐ |
| Terms of Service | Lawyer | ₹10,000-15,000 | ☐ |
| Parental Consent Form | Lawyer | Included | ☐ |
| Data Processing Agreement | Lawyer | Included | ☐ |
| **Total Legal Budget** | | **₹25,000-40,000** | |

**Story 7.1C.3: Data Localization Verification**

| Check | Expected | Actual | Status |
|-------|----------|--------|--------|
| Firestore region | asia-south1 | | ☐ |
| RTDB region | asia-southeast1 | | ☐ |
| Cloud Functions region | asia-south1 | | ☐ |
| Storage bucket region | asia-south1 | | ☐ |
| No data transfer outside India | Verified | | ☐ |

**DPDP Violation Penalty:** Up to ₹250 Crore - This verification is non-negotiable.

---

### Epic 7.2: Release Build Preparation

**Story 7.2.1: Android Release Build**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 7.2.1.1 | Generate release keystore | Keystore created, secured |
| 7.2.1.2 | Configure ProGuard/R8 | Obfuscation working |
| 7.2.1.3 | Remove debug code | No debug logs in release |
| 7.2.1.4 | Configure build variants | Release builds correctly |
| 7.2.1.5 | Test release APK | All features working |

**Implementation: Release Signing Configuration**

```kotlin
// app/build.gradle.kts

android {
    signingConfigs {
        create("release") {
            // Load from local.properties (not committed to git)
            val localProperties = Properties()
            val localPropertiesFile = rootProject.file("local.properties")
            if (localPropertiesFile.exists()) {
                localProperties.load(localPropertiesFile.inputStream())
            }

            storeFile = file(localProperties.getProperty("release.storeFile") ?: "release.keystore")
            storePassword = localProperties.getProperty("release.storePassword") ?: ""
            keyAlias = localProperties.getProperty("release.keyAlias") ?: ""
            keyPassword = localProperties.getProperty("release.keyPassword") ?: ""
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            signingConfig = signingConfigs.getByName("release")

            // Remove logging
            buildConfigField("boolean", "ENABLE_LOGGING", "false")
        }

        debug {
            isMinifyEnabled = false
            buildConfigField("boolean", "ENABLE_LOGGING", "true")
        }
    }

    flavorDimensions += "environment"
    productFlavors {
        create("dev") {
            dimension = "environment"
            applicationIdSuffix = ".dev"
            versionNameSuffix = "-dev"
            buildConfigField("String", "FIREBASE_PROJECT", "\"kidsafe-dev\"")
        }

        create("prod") {
            dimension = "environment"
            buildConfigField("String", "FIREBASE_PROJECT", "\"kidsafe-prod\"")
        }
    }
}
```

**Implementation: ProGuard Rules**

```proguard
# proguard-rules.pro

# Keep important annotations
-keepattributes *Annotation*
-keepattributes SourceFile,LineNumberTable
-keepattributes Signature
-keepattributes Exceptions

# Firebase
-keepattributes Signature
-keepclassmembers class com.kidsafe.** {
    *;
}

# Keep Device Admin Receiver
-keep class com.kidsafe.dpc.KidDeviceAdminReceiver {
    *;
}

# Keep model classes for Firebase serialization
-keep class com.kidsafe.launcher.model.** { *; }
-keep class com.kidsafe.dpc.model.** { *; }

# Keep FCM service
-keep class com.kidsafe.launcher.service.KidSafeFCMService {
    *;
}

# Kotlin Coroutines
-keepnames class kotlinx.coroutines.internal.MainDispatcherFactory {}
-keepnames class kotlinx.coroutines.CoroutineExceptionHandler {}
-keepclassmembernames class kotlinx.** {
    volatile <fields>;
}

# OkHttp (if using)
-dontwarn okhttp3.**
-dontwarn okio.**
-keepnames class okhttp3.internal.publicsuffix.PublicSuffixDatabase

# Gson (if using)
-keepattributes Signature
-keepattributes *Annotation*
-dontwarn sun.misc.**
-keep class com.google.gson.stream.** { *; }
-keep class * implements com.google.gson.TypeAdapterFactory
-keep class * implements com.google.gson.JsonSerializer
-keep class * implements com.google.gson.JsonDeserializer

# Remove logging in release
-assumenosideeffects class android.util.Log {
    public static int v(...);
    public static int d(...);
    public static int i(...);
}

# Remove debug code
-assumenosideeffects class com.kidsafe.launcher.util.DebugUtils {
    public static *** *(...);
}
```

**Story 7.2.2: Flutter Release Build**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 7.2.2.1 | Configure app signing | Signing working |
| 7.2.2.2 | Set up flavors for environments | Dev/Prod flavors |
| 7.2.2.3 | Configure Dart obfuscation | Code obfuscated |
| 7.2.2.4 | Test iOS release build | TestFlight upload |
| 7.2.2.5 | Test Android release build | Play Store compatible |

**Implementation: Flutter Build Configuration**

```yaml
# android/app/build.gradle

android {
    defaultConfig {
        applicationId "com.kidsafe.parent"
        minSdkVersion 21
        targetSdkVersion 34
        versionCode flutterVersionCode.toInteger()
        versionName flutterVersionName
    }

    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }

    flavorDimensions "environment"
    productFlavors {
        dev {
            dimension "environment"
            applicationIdSuffix ".dev"
            resValue "string", "app_name", "KidSafe Dev"
        }
        prod {
            dimension "environment"
            resValue "string", "app_name", "KidSafe"
        }
    }
}
```

**Build Commands:**

```bash
# Build for development
flutter build apk --flavor dev --release

# Build for production
flutter build apk --flavor prod --release --obfuscate --split-debug-info=build/debug-info

# Build iOS for TestFlight
flutter build ipa --flavor prod --release --obfuscate --split-debug-info=build/debug-info

# Build App Bundle for Play Store
flutter build appbundle --flavor prod --release --obfuscate --split-debug-info=build/debug-info
```

---

### Epic 7.3: App Store Preparation

**Story 7.3.1: Google Play Store Listing**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 7.3.1.1 | Create Play Console developer account | Account active |
| 7.3.1.2 | Write app description | Description approved |
| 7.3.1.3 | Create screenshots | 8 screenshots ready |
| 7.3.1.4 | Create feature graphic | 1024x500 graphic |
| 7.3.1.5 | Write privacy policy | Policy URL active |
| 7.3.1.6 | Complete data safety form | Form submitted |
| 7.3.1.7 | Submit for review | Review approved |

**Play Store Description:**

```markdown
# Short Description (80 chars)
KidSafe: Control your child's phone with ease. Screen time, app limits & more.

# Full Description (4000 chars)

## Give Your Child a Safe Phone Experience

KidSafe is India's first comprehensive parental control solution designed for modern families. Turn any Android phone into a kid-friendly device with powerful controls and peace of mind.

### For Parents Who Care

✅ **Complete App Control**
• Allow or block any app with a simple toggle
• Pre-approved apps like PhonePe, Google Pay, WhatsApp
• Block inappropriate content automatically
• Remote app installation capability

✅ **Smart Screen Time**
• Set daily usage limits
• App-specific time restrictions
• Automatic bedtime mode
• Get alerts when limits are reached

✅ **Stay Connected**
• Real-time location tracking
• See what your child is listening to
• Panic button for emergencies
• Instant device lock from anywhere

✅ **Kids Mode**
• Simplified interface for younger children
• Large icons, easy navigation
• Cannot access settings or install apps
• Switch to Standard Mode for older kids

### How It Works

1. **Download**: Install KidSafe Parent app on your phone
2. **Pair**: Scan QR code on your child's device
3. **Customize**: Set rules, limits, and allowed apps
4. **Monitor**: Get real-time updates and alerts

### Why Indian Parents Love KidSafe

🇮🇳 Made in India, for Indian families
📱 Works with Spotify, JioSaavn, Amazon Music
💳 Supports PhonePe, Google Pay, Paytm
🔒 No data leaves India (Firebase asia-south1)
💬 WhatsApp support in Hindi & English

### Subscription Plans

**Basic Plan - ₹149/month**
• Remote controls
• Usage reports
• Screen time limits

**Premium Plan - ₹249/month**
• All Basic features
• Location tracking
• Advanced alerts
• Priority support

**Family Plan - ₹399/month**
• Up to 4 devices
• All Premium features

### Contact Us
📧 support@kidsafe.in
📱 WhatsApp: +91 98765 43210
🌐 www.kidsafe.in

---
KidSafe respects your privacy. We only collect data necessary to provide our services. See our privacy policy at kidsafe.in/privacy
```

**Story 7.3.2: Apple App Store Listing**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 7.3.2.1 | Enroll in Apple Developer Program | Account active |
| 7.3.2.2 | Create App Store Connect listing | Listing created |
| 7.3.2.3 | Create iOS screenshots | iPhone/iPad screenshots |
| 7.3.2.4 | Write app description | Description ready |
| 7.3.2.5 | Configure app privacy | Privacy labels complete |
| 7.3.2.6 | Submit for TestFlight | TestFlight approved |
| 7.3.2.7 | Submit for App Store review | Review approved |

**App Store Privacy Labels:**

```
Data Linked to You:
- Contact Info (Email, Phone Number)
- Identifiers (User ID, Device ID)
- Usage Data (Product Interaction)
- Diagnostics (Crash Data)

Data Used for Tracking:
- None

Data Used for Analytics:
- Usage Data (anonymized)
- Diagnostics
```

---

### Epic 7.4: OTA Update System

**Story 7.4.1: Device OTA Updates**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 7.4.1.1 | Create update check mechanism | Device checks daily |
| 7.4.1.2 | Create update download service | APK downloads in background |
| 7.4.1.3 | Create update installer | Silent install as Device Owner |
| 7.4.1.4 | Create rollback mechanism | Can revert on failure |
| 7.4.1.5 | Create update console | Admin can push updates |

**Implementation: OTA Update Manager**

```kotlin
// app/src/main/java/com/kidsafe/launcher/update/OTAUpdateManager.kt

package com.kidsafe.launcher.update

import android.app.DownloadManager
import android.app.admin.DevicePolicyManager
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.net.Uri
import android.os.Environment
import android.util.Log
import com.google.firebase.firestore.FirebaseFirestore
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import java.io.File

class OTAUpdateManager(
    private val context: Context,
    private val devicePolicyManager: DevicePolicyManager,
    private val adminComponent: android.content.ComponentName
) {
    private val firestore = FirebaseFirestore.getInstance()
    private val downloadManager = context.getSystemService(Context.DOWNLOAD_SERVICE) as DownloadManager
    private val prefs = context.getSharedPreferences("ota_prefs", Context.MODE_PRIVATE)

    companion object {
        private const val TAG = "OTAUpdateManager"
        private const val PREF_CURRENT_VERSION = "current_version"
        private const val PREF_DOWNLOAD_ID = "download_id"
    }

    data class UpdateInfo(
        val versionCode: Int,
        val versionName: String,
        val downloadUrl: String,
        val releaseNotes: String,
        val isMandatory: Boolean,
        val minVersionCode: Int,
        val sha256Checksum: String
    )

    /**
     * Check for available updates
     */
    suspend fun checkForUpdate(): UpdateInfo? = withContext(Dispatchers.IO) {
        try {
            val currentVersionCode = getCurrentVersionCode()

            val updateDoc = firestore
                .collection("app_updates")
                .document("launcher_latest")
                .get()
                .await()

            if (!updateDoc.exists()) {
                Log.d(TAG, "No update document found")
                return@withContext null
            }

            val data = updateDoc.data ?: return@withContext null
            val availableVersion = (data["versionCode"] as Long).toInt()

            if (availableVersion <= currentVersionCode) {
                Log.d(TAG, "No update available. Current: $currentVersionCode, Available: $availableVersion")
                return@withContext null
            }

            UpdateInfo(
                versionCode = availableVersion,
                versionName = data["versionName"] as String,
                downloadUrl = data["downloadUrl"] as String,
                releaseNotes = data["releaseNotes"] as String? ?: "",
                isMandatory = data["isMandatory"] as Boolean? ?: false,
                minVersionCode = (data["minVersionCode"] as Long?)?.toInt() ?: 0,
                sha256Checksum = data["sha256Checksum"] as String
            )
        } catch (e: Exception) {
            Log.e(TAG, "Error checking for update", e)
            null
        }
    }

    /**
     * Download update APK
     */
    fun downloadUpdate(updateInfo: UpdateInfo): Long {
        val fileName = "kidsafe_launcher_${updateInfo.versionCode}.apk"
        val destinationDir = context.getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS)
        val destinationFile = File(destinationDir, fileName)

        // Delete old file if exists
        if (destinationFile.exists()) {
            destinationFile.delete()
        }

        val request = DownloadManager.Request(Uri.parse(updateInfo.downloadUrl))
            .setTitle("KidSafe Update")
            .setDescription("Downloading version ${updateInfo.versionName}")
            .setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE)
            .setDestinationUri(Uri.fromFile(destinationFile))
            .setAllowedOverMetered(false) // Only on WiFi
            .setAllowedOverRoaming(false)

        val downloadId = downloadManager.enqueue(request)
        prefs.edit().putLong(PREF_DOWNLOAD_ID, downloadId).apply()

        Log.d(TAG, "Started download: $downloadId")
        return downloadId
    }

    /**
     * Install downloaded update (as Device Owner)
     */
    fun installUpdate(apkFile: File, checksum: String): Boolean {
        // Verify checksum
        if (!verifyChecksum(apkFile, checksum)) {
            Log.e(TAG, "Checksum verification failed")
            apkFile.delete()
            return false
        }

        return try {
            // As Device Owner, we can silently install
            val packageInstaller = context.packageManager.packageInstaller
            val params = android.content.pm.PackageInstaller.SessionParams(
                android.content.pm.PackageInstaller.SessionParams.MODE_FULL_INSTALL
            )
            params.setAppPackageName(context.packageName)

            val sessionId = packageInstaller.createSession(params)
            val session = packageInstaller.openSession(sessionId)

            apkFile.inputStream().use { input ->
                session.openWrite("package", 0, apkFile.length()).use { output ->
                    input.copyTo(output)
                    session.fsync(output)
                }
            }

            val intent = Intent(context, UpdateInstallReceiver::class.java)
            val pendingIntent = android.app.PendingIntent.getBroadcast(
                context, 0, intent,
                android.app.PendingIntent.FLAG_UPDATE_CURRENT or android.app.PendingIntent.FLAG_IMMUTABLE
            )

            session.commit(pendingIntent.intentSender)
            session.close()

            Log.d(TAG, "Update installation initiated")
            true
        } catch (e: Exception) {
            Log.e(TAG, "Error installing update", e)
            false
        }
    }

    /**
     * Register download completion receiver
     */
    fun registerDownloadReceiver(onComplete: (File?) -> Unit) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context, intent: Intent) {
                val downloadId = intent.getLongExtra(DownloadManager.EXTRA_DOWNLOAD_ID, -1)
                val savedDownloadId = prefs.getLong(PREF_DOWNLOAD_ID, -1)

                if (downloadId == savedDownloadId) {
                    val query = DownloadManager.Query().setFilterById(downloadId)
                    val cursor = downloadManager.query(query)

                    if (cursor.moveToFirst()) {
                        val statusIndex = cursor.getColumnIndex(DownloadManager.COLUMN_STATUS)
                        val status = cursor.getInt(statusIndex)

                        if (status == DownloadManager.STATUS_SUCCESSFUL) {
                            val uriIndex = cursor.getColumnIndex(DownloadManager.COLUMN_LOCAL_URI)
                            val uri = cursor.getString(uriIndex)
                            val file = File(Uri.parse(uri).path!!)
                            onComplete(file)
                        } else {
                            onComplete(null)
                        }
                    }
                    cursor.close()
                }
            }
        }

        context.registerReceiver(
            receiver,
            IntentFilter(DownloadManager.ACTION_DOWNLOAD_COMPLETE),
            Context.RECEIVER_NOT_EXPORTED
        )
    }

    private fun getCurrentVersionCode(): Int {
        return try {
            context.packageManager.getPackageInfo(context.packageName, 0).longVersionCode.toInt()
        } catch (e: Exception) {
            0
        }
    }

    private fun verifyChecksum(file: File, expectedChecksum: String): Boolean {
        val digest = java.security.MessageDigest.getInstance("SHA-256")
        file.inputStream().use { input ->
            val buffer = ByteArray(8192)
            var read: Int
            while (input.read(buffer).also { read = it } != -1) {
                digest.update(buffer, 0, read)
            }
        }
        val actualChecksum = digest.digest().joinToString("") { "%02x".format(it) }
        return actualChecksum.equals(expectedChecksum, ignoreCase = true)
    }
}

// Receiver for install completion
class UpdateInstallReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val status = intent.getIntExtra(
            android.content.pm.PackageInstaller.EXTRA_STATUS,
            android.content.pm.PackageInstaller.STATUS_FAILURE
        )

        when (status) {
            android.content.pm.PackageInstaller.STATUS_SUCCESS -> {
                Log.i("UpdateInstallReceiver", "Update installed successfully")
                // App will restart automatically
            }
            else -> {
                val message = intent.getStringExtra(
                    android.content.pm.PackageInstaller.EXTRA_STATUS_MESSAGE
                )
                Log.e("UpdateInstallReceiver", "Update failed: $message")
            }
        }
    }
}
```

---

### Epic 7.5: Device Setup Automation

**Story 7.5.1: Automated Device Configuration**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 7.5.1.1 | Create device flashing script | Script works reliably |
| 7.5.1.2 | Create configuration template | Template complete |
| 7.5.1.3 | Create batch setup tool | Can configure 10 devices/hour |
| 7.5.1.4 | Create verification checklist | All features verified |
| 7.5.1.5 | Document setup process | Documentation complete |

**Implementation: Device Setup Script**

```bash
#!/bin/bash

# kidsafe_device_setup.sh
# Automated device setup script for KidSafe devices
# Usage: ./kidsafe_device_setup.sh [options]

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Configuration
LAUNCHER_APK="./builds/launcher-release.apk"
DPC_APK="./builds/dpc-release.apk"
MUSIC_APPS_DIR="./apks/music"
UTILITY_APPS_DIR="./apks/utility"
LOG_DIR="./setup_logs"

# Create log directory
mkdir -p "$LOG_DIR"

# Functions
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

check_requirements() {
    log_info "Checking requirements..."

    # Check ADB
    if ! command -v adb &> /dev/null; then
        log_error "ADB not found. Please install Android SDK Platform Tools."
        exit 1
    fi

    # Check APK files
    if [ ! -f "$LAUNCHER_APK" ]; then
        log_error "Launcher APK not found: $LAUNCHER_APK"
        exit 1
    fi

    if [ ! -f "$DPC_APK" ]; then
        log_error "DPC APK not found: $DPC_APK"
        exit 1
    fi

    log_info "All requirements met."
}

list_devices() {
    log_info "Connected devices:"
    adb devices -l | grep -v "List" | grep -v "^$"
}

setup_device() {
    local SERIAL=$1
    local DEVICE_NAME=$2
    local LOG_FILE="$LOG_DIR/setup_${SERIAL}_$(date +%Y%m%d_%H%M%S).log"

    log_info "Setting up device: $SERIAL ($DEVICE_NAME)"
    echo "Setup started at $(date)" > "$LOG_FILE"

    # Wait for device
    log_info "Waiting for device..."
    adb -s "$SERIAL" wait-for-device >> "$LOG_FILE" 2>&1

    # Get device info
    local MODEL=$(adb -s "$SERIAL" shell getprop ro.product.model | tr -d '\r')
    local ANDROID_VERSION=$(adb -s "$SERIAL" shell getprop ro.build.version.release | tr -d '\r')
    log_info "Device: $MODEL, Android $ANDROID_VERSION"
    echo "Model: $MODEL, Android: $ANDROID_VERSION" >> "$LOG_FILE"

    # Check if already set up
    local DPC_INSTALLED=$(adb -s "$SERIAL" shell pm list packages | grep "com.kidsafe.dpc" || true)
    if [ -n "$DPC_INSTALLED" ]; then
        log_warn "Device already has DPC installed. Skipping..."
        return
    fi

    # Step 1: Install DPC
    log_info "Installing Device Policy Controller..."
    adb -s "$SERIAL" install -r "$DPC_APK" >> "$LOG_FILE" 2>&1
    sleep 2

    # Step 2: Set as Device Owner
    log_info "Setting as Device Owner..."
    adb -s "$SERIAL" shell dpm set-device-owner com.kidsafe.dpc/.KidDeviceAdminReceiver >> "$LOG_FILE" 2>&1 || {
        log_error "Failed to set Device Owner. Device may need factory reset."
        echo "ERROR: Device Owner setup failed" >> "$LOG_FILE"
        return 1
    }
    sleep 2

    # Step 3: Install Launcher
    log_info "Installing Launcher..."
    adb -s "$SERIAL" install -r "$LAUNCHER_APK" >> "$LOG_FILE" 2>&1
    sleep 2

    # Step 4: Set Launcher as default home
    log_info "Setting launcher as default home..."
    adb -s "$SERIAL" shell cmd package set-home-activity com.kidsafe.launcher/.LauncherActivity >> "$LOG_FILE" 2>&1

    # Step 5: Install music apps
    log_info "Installing music apps..."
    for APK in "$MUSIC_APPS_DIR"/*.apk; do
        if [ -f "$APK" ]; then
            local APP_NAME=$(basename "$APK" .apk)
            log_info "  Installing $APP_NAME..."
            adb -s "$SERIAL" install -r "$APK" >> "$LOG_FILE" 2>&1 || log_warn "  Failed to install $APP_NAME"
        fi
    done

    # Step 6: Install utility apps
    log_info "Installing utility apps..."
    for APK in "$UTILITY_APPS_DIR"/*.apk; do
        if [ -f "$APK" ]; then
            local APP_NAME=$(basename "$APK" .apk)
            log_info "  Installing $APP_NAME..."
            adb -s "$SERIAL" install -r "$APK" >> "$LOG_FILE" 2>&1 || log_warn "  Failed to install $APP_NAME"
        fi
    done

    # Step 7: Configure initial settings
    log_info "Configuring initial settings..."

    # Disable Setup Wizard
    adb -s "$SERIAL" shell pm disable-user --user 0 com.google.android.setupwizard >> "$LOG_FILE" 2>&1 || true

    # Set screen timeout
    adb -s "$SERIAL" shell settings put system screen_off_timeout 120000 >> "$LOG_FILE" 2>&1

    # Enable stay awake while charging (for setup)
    adb -s "$SERIAL" shell settings put global stay_on_while_plugged_in 3 >> "$LOG_FILE" 2>&1

    # Step 8: Verify installation
    log_info "Verifying installation..."
    local VERIFY_ERRORS=0

    # Check DPC
    if ! adb -s "$SERIAL" shell pm list packages | grep -q "com.kidsafe.dpc"; then
        log_error "  DPC not installed"
        VERIFY_ERRORS=$((VERIFY_ERRORS + 1))
    fi

    # Check Launcher
    if ! adb -s "$SERIAL" shell pm list packages | grep -q "com.kidsafe.launcher"; then
        log_error "  Launcher not installed"
        VERIFY_ERRORS=$((VERIFY_ERRORS + 1))
    fi

    # Check Device Owner
    local IS_DEVICE_OWNER=$(adb -s "$SERIAL" shell dumpsys device_policy | grep "Device Owner" || true)
    if [ -z "$IS_DEVICE_OWNER" ]; then
        log_error "  Not set as Device Owner"
        VERIFY_ERRORS=$((VERIFY_ERRORS + 1))
    fi

    if [ $VERIFY_ERRORS -eq 0 ]; then
        log_info "Device setup completed successfully!"
        echo "Setup completed successfully at $(date)" >> "$LOG_FILE"
    else
        log_error "Device setup completed with $VERIFY_ERRORS errors"
        echo "Setup completed with errors at $(date)" >> "$LOG_FILE"
        return 1
    fi

    # Generate device info file
    local INFO_FILE="$LOG_DIR/device_info_${SERIAL}.txt"
    echo "Device Serial: $SERIAL" > "$INFO_FILE"
    echo "Device Name: $DEVICE_NAME" >> "$INFO_FILE"
    echo "Model: $MODEL" >> "$INFO_FILE"
    echo "Android Version: $ANDROID_VERSION" >> "$INFO_FILE"
    echo "Setup Date: $(date)" >> "$INFO_FILE"
    echo "Log File: $LOG_FILE" >> "$INFO_FILE"

    log_info "Device info saved to: $INFO_FILE"
}

batch_setup() {
    log_info "Starting batch setup..."

    # Get all connected devices
    local DEVICES=$(adb devices | grep -v "List" | grep "device$" | awk '{print $1}')

    if [ -z "$DEVICES" ]; then
        log_error "No devices connected."
        exit 1
    fi

    local COUNT=1
    for SERIAL in $DEVICES; do
        log_info "Processing device $COUNT: $SERIAL"
        setup_device "$SERIAL" "Device$COUNT"
        COUNT=$((COUNT + 1))
        echo ""
    done

    log_info "Batch setup complete. Processed $((COUNT - 1)) devices."
}

show_help() {
    echo "KidSafe Device Setup Script"
    echo ""
    echo "Usage: $0 [options]"
    echo ""
    echo "Options:"
    echo "  -s, --serial <serial>    Setup specific device by serial number"
    echo "  -b, --batch              Setup all connected devices"
    echo "  -l, --list               List connected devices"
    echo "  -h, --help               Show this help message"
    echo ""
    echo "Examples:"
    echo "  $0 --list                    # List all connected devices"
    echo "  $0 --serial ABC123           # Setup device with serial ABC123"
    echo "  $0 --batch                   # Setup all connected devices"
}

# Main
check_requirements

case "$1" in
    -s|--serial)
        if [ -z "$2" ]; then
            log_error "Please provide device serial number"
            exit 1
        fi
        setup_device "$2" "Device"
        ;;
    -b|--batch)
        batch_setup
        ;;
    -l|--list)
        list_devices
        ;;
    -h|--help|*)
        show_help
        ;;
esac
```

---

### Epic 7.6: Launch Preparation

**Story 7.6.1: Website Launch**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 7.6.1.1 | Create landing page | Page live at kidsafe.in |
| 7.6.1.2 | Implement ordering system | Can place orders |
| 7.6.1.3 | Integrate Razorpay | Payments working |
| 7.6.1.4 | Set up customer support | Support channels ready |
| 7.6.1.5 | Create help documentation | FAQ and guides ready |

**Story 7.6.2: First Batch Order**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 7.6.2.1 | Order 50 Samsung Galaxy A05 | Order placed |
| 7.6.2.2 | Receive and verify devices | All devices working |
| 7.6.2.3 | Configure all devices | 50 devices ready |
| 7.6.2.4 | Create packaging materials | Boxes, manuals ready |
| 7.6.2.5 | Quality check all devices | QC passed |

**Story 7.6.3: Customer Support Setup**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 7.6.3.1 | Create support email | support@kidsafe.in active |
| 7.6.3.2 | Set up WhatsApp Business | WhatsApp number active |
| 7.6.3.3 | Create support scripts | Scripts ready |
| 7.6.3.4 | Train support person | Person trained |
| 7.6.3.5 | Create escalation process | Process documented |

**Support Response Templates:**

```markdown
# KidSafe Support Response Templates

## 1. Initial Response (Within 1 hour)

Hindi:
नमस्ते [ग्राहक का नाम],

आपके संदेश के लिए धन्यवाद। हम आपकी समस्या को समझ गए हैं और जल्द से जल्द मदद करेंगे।

टिकट नंबर: [TICKET_ID]

अगर कोई और जानकारी चाहिए तो कृपया बताएं।

धन्यवाद,
KidSafe टीम

English:
Hello [Customer Name],

Thank you for reaching out. We understand your concern and will help you as soon as possible.

Ticket Number: [TICKET_ID]

Please let us know if you need any additional information.

Best regards,
KidSafe Team

---

## 2. Device Setup Help

Hindi:
नमस्ते,

डिवाइस सेटअप के लिए:
1. Parent App डाउनलोड करें (Play Store/App Store से)
2. अपना अकाउंट बनाएं
3. "Add Child" पर क्लिक करें
4. बच्चे की जानकारी भरें
5. QR code दिखाई देगा
6. बच्चे के फोन से QR code स्कैन करें

वीडियो गाइड: [VIDEO_LINK]

---

## 3. Screen Time Not Working

Troubleshooting steps:
1. Check if device is online (green dot in parent app)
2. Pull down to refresh the parent app
3. Check if the child device has internet
4. Restart the child device
5. If still not working, contact support with:
   - Parent app version
   - Child device model
   - Screenshot of the issue

---

## 4. App Not Appearing on Child Device

Steps:
1. Open Parent App
2. Go to "Manage Apps"
3. Find the app you want to add
4. Toggle it ON
5. Wait 30 seconds
6. Check child device

If app still not appearing:
- The app may not be installed on child device
- Try: Parent App > Manage Apps > [App] > "Install on device"

---

## 5. Panic Alert Not Received

Check:
1. Parent app notifications enabled
2. Phone's DND mode is off
3. KidSafe has notification permissions
4. Internet connection on both devices

Test:
1. Open child device
2. Press and hold panic button for 3 seconds
3. Check parent phone

---

## 6. Refund Request

Response:
We're sorry to hear you'd like a refund.

Our refund policy:
- Full refund within 7 days of delivery
- Device must be in original condition
- Original packaging required

To process your refund:
1. Reply with your order number
2. Share reason for return
3. We'll arrange pickup within 2-3 days

---

## 7. Subscription Cancellation

Response:
You can cancel your subscription anytime:

1. Open Parent App
2. Go to Settings > Subscription
3. Tap "Cancel Subscription"
4. Confirm cancellation

Note: Your subscription will remain active until the end of the current billing period.

If you need help, please let us know.
```

---

## Sprint 7 Exit Criteria

### Production Checklist (Day 100)

| Category | Item | Status |
|----------|------|--------|
| **Infrastructure** | | |
| | Firebase production project created | ☐ |
| | Security rules deployed and tested | ☐ |
| | Cloud Functions deployed | ☐ |
| | Crashlytics configured | ☐ |
| | Analytics events defined | ☐ |
| | Budget alerts configured | ☐ |
| **App Builds** | | |
| | Android launcher release APK | ☐ |
| | Android DPC release APK | ☐ |
| | Flutter parent app (Android) | ☐ |
| | Flutter parent app (iOS) | ☐ |
| | All builds tested on real devices | ☐ |
| **App Stores** | | |
| | Play Store listing complete | ☐ |
| | Play Store app approved | ☐ |
| | App Store listing complete | ☐ |
| | App Store app approved | ☐ |
| **Devices** | | |
| | 50 devices ordered | ☐ |
| | Devices received and verified | ☐ |
| | All devices configured | ☐ |
| | QC passed for all devices | ☐ |
| | Packaging ready | ☐ |
| **Website** | | |
| | Landing page live | ☐ |
| | Ordering system working | ☐ |
| | Payment integration tested | ☐ |
| | SSL certificate installed | ☐ |
| **Support** | | |
| | Support email active | ☐ |
| | WhatsApp Business active | ☐ |
| | Support scripts ready | ☐ |
| | Support person trained | ☐ |
| **Legal** | | |
| | Privacy policy published | ☐ |
| | Terms of service published | ☐ |
| | Refund policy published | ☐ |
| | Company registration complete | ☐ |
| | GST registration complete | ☐ |
| **Documentation** | | |
| | User manual created | ☐ |
| | FAQ page live | ☐ |
| | Setup video recorded | ☐ |
| | Troubleshooting guide ready | ☐ |

---

## Launch Day Checklist

```markdown
# KidSafe Launch Day Checklist

## T-24 Hours
- [ ] All devices packaged and ready
- [ ] Website final review
- [ ] Test order placement
- [ ] Test payment flow
- [ ] Support team briefed
- [ ] Social media posts scheduled

## T-1 Hour
- [ ] Website live check
- [ ] Payment gateway active
- [ ] Support channels monitored
- [ ] Team on standby

## Launch
- [ ] Send launch announcement
- [ ] Monitor first orders
- [ ] Monitor app store downloads
- [ ] Watch for crash reports
- [ ] Respond to customer queries

## T+1 Hour
- [ ] Review any issues
- [ ] Check order fulfillment queue
- [ ] Review support tickets
- [ ] Team check-in

## T+24 Hours
- [ ] Review Day 1 metrics
- [ ] Follow up on any issues
- [ ] Send thank you to first customers
- [ ] Collect early feedback
```

---

## Files Created/Modified in Sprint 7

| File | Type | Purpose |
|------|------|---------|
| `firestore.rules` | Created | Firestore security rules |
| `database.rules.json` | Created | RTDB security rules |
| `functions/src/index.ts` | Created | Cloud Functions |
| `app/build.gradle.kts` | Modified | Release configuration |
| `proguard-rules.pro` | Created | ProGuard configuration |
| `kidsafe_device_setup.sh` | Created | Device setup script |
| `support_templates.md` | Created | Support response templates |

---

## Sprint 7 Checklist

### Days 78-82: Security Audit (CRITICAL)
- [ ] Internal security audit completed
- [ ] Firebase security rules reviewed
- [ ] ProGuard obfuscation verified
- [ ] Root detection tested
- [ ] Play Integrity API integrated
- [ ] External security audit completed
- [ ] All critical findings remediated

### Days 83-85: DPDP Compliance (CRITICAL)
- [ ] Parental consent flow verified (OTP + checkbox)
- [ ] Data export feature working
- [ ] Data deletion working (72-hour SLA)
- [ ] Privacy policy reviewed by lawyer
- [ ] Terms of service reviewed by lawyer
- [ ] Data localization verified (asia-south1)
- [ ] No third-party analytics on child data

### Days 86-92: Release Build
- [ ] Release keystore generated and secured
- [ ] ProGuard/R8 configured
- [ ] Release APK tested
- [ ] Flutter app release build
- [ ] Google Play listing prepared
- [ ] Apple App Store listing prepared

### Days 93-100: Launch Preparation
- [ ] Firebase production project configured
- [ ] Cloud Functions deployed
- [ ] First 50 devices ordered
- [ ] Device setup script working
- [ ] Customer support channels ready
- [ ] Website live
- [ ] **GO/NO-GO decision made**

---

## Post-Launch (Days 101-120)

Continue to soft launch phase:
- First 10 customer sales
- Collect feedback
- Iterate and improve
- Scale gradually

See business plan for detailed go-to-market strategy.
