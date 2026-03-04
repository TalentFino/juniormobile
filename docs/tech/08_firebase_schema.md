# Firebase Data Schema

## Overview

This document defines the complete Firebase data structure for the KidSafe application, including Firestore collections, Realtime Database structure, and Cloud Storage organization.

---

## Firestore Collections

### 1. Users Collection

```
/users/{userId}
```

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `id` | string | User ID (matches auth UID) | Yes |
| `email` | string | User email | No |
| `phone` | string | User phone number | No |
| `displayName` | string | Parent's name | Yes |
| `profileImageUrl` | string | Profile image URL | No |
| `fcmToken` | string | FCM token for push | No |
| `fcmTokenUpdatedAt` | timestamp | Last FCM token update | No |
| `subscriptionId` | string | Active subscription ID | No |
| `subscriptionStatus` | string | active, cancelled, expired | No |
| `subscriptionPlan` | string | basic, premium, family | No |
| `createdAt` | timestamp | Account creation time | Yes |
| `updatedAt` | timestamp | Last update time | Yes |
| `settings` | map | User preferences | No |
| `timezone` | string | User timezone (e.g., Asia/Kolkata) | No |

**Settings Map Structure:**

```typescript
settings: {
  notifications: {
    panicAlerts: boolean,      // Default: true
    batteryAlerts: boolean,    // Default: true
    offlineAlerts: boolean,    // Default: true
    screenTimeLimits: boolean, // Default: true
    dailyReports: boolean,     // Default: false
  },
  language: string,            // 'en' | 'hi'
  theme: string,               // 'light' | 'dark' | 'system'
}
```

**Example Document:**

```json
{
  "id": "user_abc123",
  "email": "parent@example.com",
  "phone": "+919876543210",
  "displayName": "Priya Sharma",
  "profileImageUrl": "https://storage.googleapis.com/...",
  "fcmToken": "dGhpcyBpcyBhIHRva2VuLi4u",
  "fcmTokenUpdatedAt": "2024-01-15T10:30:00Z",
  "subscriptionId": "sub_xyz789",
  "subscriptionStatus": "active",
  "subscriptionPlan": "premium",
  "createdAt": "2024-01-01T08:00:00Z",
  "updatedAt": "2024-01-15T10:30:00Z",
  "settings": {
    "notifications": {
      "panicAlerts": true,
      "batteryAlerts": true,
      "offlineAlerts": true,
      "screenTimeLimits": true,
      "dailyReports": false
    },
    "language": "en",
    "theme": "light"
  },
  "timezone": "Asia/Kolkata"
}
```

---

### 2. Devices Collection

```
/devices/{deviceId}
```

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `id` | string | Device ID | Yes |
| `parentId` | string | Parent user ID | Yes |
| `childName` | string | Child's display name | Yes |
| `childAge` | number | Child's age | Yes |
| `childAvatar` | string | Avatar identifier | No |
| `protectionLevel` | string | maximum, balanced, trust | Yes |
| `deviceInfo` | map | Device hardware info | Yes |
| `status` | string | online, offline, pending | Yes |
| `mode` | string | kids, standard | Yes |
| `lastSeen` | timestamp | Last activity time | Yes |
| `createdAt` | timestamp | Pairing time | Yes |
| `updatedAt` | timestamp | Last update time | Yes |
| `screenTimeSettings` | map | Screen time configuration | No |
| `locationEnabled` | boolean | Location tracking enabled | No |
| `volumeLimit` | number | Max volume percentage (0-100) | No |
| `panicContacts` | array | Emergency contact phone numbers | No |

**DeviceInfo Map Structure:**

```typescript
deviceInfo: {
  model: string,           // "Samsung Galaxy A05"
  manufacturer: string,    // "Samsung"
  androidVersion: string,  // "14"
  sdkVersion: number,      // 34
  serialNumber: string,    // Device serial
  imei: string,            // Optional, if available
  launcherVersion: string, // "1.0.0"
  dpcVersion: string,      // "1.0.0"
}
```

**ScreenTimeSettings Map Structure:**

```typescript
screenTimeSettings: {
  dailyLimitMinutes: number,     // 0 = no limit
  bedtimeStart: string,          // "20:30" (HH:mm)
  bedtimeEnd: string,            // "07:00" (HH:mm)
  bedtimeEnabled: boolean,
  weekdayLimitMinutes: number,   // Optional separate limit
  weekendLimitMinutes: number,   // Optional separate limit
  appLimits: {                   // Per-app limits
    [packageName: string]: number  // Minutes
  }
}
```

**Example Document:**

```json
{
  "id": "device_xyz789",
  "parentId": "user_abc123",
  "childName": "Rahul",
  "childAge": 8,
  "childAvatar": "boy_1",
  "protectionLevel": "balanced",
  "deviceInfo": {
    "model": "Samsung Galaxy A05",
    "manufacturer": "Samsung",
    "androidVersion": "14",
    "sdkVersion": 34,
    "serialNumber": "R5CR12345678",
    "launcherVersion": "1.0.0",
    "dpcVersion": "1.0.0"
  },
  "status": "online",
  "mode": "kids",
  "lastSeen": "2024-01-15T14:30:00Z",
  "createdAt": "2024-01-10T09:00:00Z",
  "updatedAt": "2024-01-15T14:30:00Z",
  "screenTimeSettings": {
    "dailyLimitMinutes": 120,
    "bedtimeStart": "20:30",
    "bedtimeEnd": "07:00",
    "bedtimeEnabled": true,
    "appLimits": {
      "com.spotify.music": 60,
      "com.jio.saavn": 60
    }
  },
  "locationEnabled": true,
  "volumeLimit": 70,
  "panicContacts": ["+919876543210", "+919876543211"]
}
```

---

### 3. Device Whitelist Subcollection

```
/devices/{deviceId}/whitelist/{appId}
```

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `packageName` | string | Android package name | Yes |
| `appName` | string | Display name | Yes |
| `appIcon` | string | Icon URL (cached) | No |
| `category` | string | App category | No |
| `enabled` | boolean | Currently allowed | Yes |
| `dailyLimitMinutes` | number | Per-app limit | No |
| `scheduleRestriction` | map | Time-based restrictions | No |
| `addedAt` | timestamp | When added to whitelist | Yes |
| `addedBy` | string | parent, default, remote | Yes |

**Example Document:**

```json
{
  "packageName": "com.spotify.music",
  "appName": "Spotify",
  "appIcon": "https://storage.googleapis.com/kidsafe/icons/spotify.png",
  "category": "music",
  "enabled": true,
  "dailyLimitMinutes": 60,
  "scheduleRestriction": {
    "allowedHoursStart": "08:00",
    "allowedHoursEnd": "20:00",
    "daysAllowed": ["mon", "tue", "wed", "thu", "fri", "sat", "sun"]
  },
  "addedAt": "2024-01-10T09:00:00Z",
  "addedBy": "parent"
}
```

---

### 4. Device Usage Subcollection

```
/devices/{deviceId}/usage/{date}
```

Date format: `YYYY-MM-DD`

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `date` | string | Date (YYYY-MM-DD) | Yes |
| `totalMinutes` | number | Total usage in minutes | Yes |
| `appUsage` | map | Per-app usage | Yes |
| `hourlyUsage` | map | Usage by hour | No |
| `sessions` | array | Usage sessions | No |
| `limitReached` | boolean | Did they hit limit? | No |
| `limitReachedAt` | timestamp | When limit was reached | No |

**Example Document:**

```json
{
  "date": "2024-01-15",
  "totalMinutes": 95,
  "appUsage": {
    "com.spotify.music": 45,
    "com.jio.saavn": 30,
    "com.android.deskclock": 20
  },
  "hourlyUsage": {
    "9": 15,
    "10": 20,
    "14": 25,
    "16": 20,
    "18": 15
  },
  "sessions": [
    {
      "startTime": "2024-01-15T09:00:00Z",
      "endTime": "2024-01-15T09:35:00Z",
      "duration": 35
    },
    {
      "startTime": "2024-01-15T14:00:00Z",
      "endTime": "2024-01-15T15:00:00Z",
      "duration": 60
    }
  ],
  "limitReached": false
}
```

---

### 5. Usage Summary Subcollection

```
/devices/{deviceId}/usage_summary/{date}
```

Aggregated daily summaries (created by Cloud Function).

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `date` | string | Date (YYYY-MM-DD) | Yes |
| `totalMinutes` | number | Total usage | Yes |
| `appBreakdown` | map | Usage per app | Yes |
| `peakHour` | number | Hour with most usage (0-23) | No |
| `limitReached` | boolean | Limit reached | Yes |
| `comparedToAverage` | number | % compared to 7-day average | No |
| `createdAt` | timestamp | Aggregation time | Yes |

---

### 6. Pairing Codes Collection

```
/pairing_codes/{code}
```

Temporary codes for device pairing (15-minute TTL).

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `code` | string | 6-character alphanumeric | Yes |
| `parentId` | string | Parent who created | Yes |
| `childName` | string | Child name entered | Yes |
| `childAge` | number | Child age entered | Yes |
| `protectionLevel` | string | Selected protection | Yes |
| `createdAt` | timestamp | Code creation time | Yes |
| `expiresAt` | timestamp | Expiry time (15 min) | Yes |
| `used` | boolean | Has been used | Yes |

**Example Document:**

```json
{
  "code": "ABC123",
  "parentId": "user_abc123",
  "childName": "Rahul",
  "childAge": 8,
  "protectionLevel": "balanced",
  "createdAt": "2024-01-15T10:00:00Z",
  "expiresAt": "2024-01-15T10:15:00Z",
  "used": false
}
```

---

### 7. Device Tokens Collection

```
/device_tokens/{deviceId}
```

Secure tokens for device authentication (never exposed to clients).

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `token` | string | Secure random token | Yes |
| `deviceId` | string | Associated device | Yes |
| `createdAt` | timestamp | Token creation time | Yes |
| `lastUsed` | timestamp | Last authentication | No |
| `rotatedAt` | timestamp | Last rotation time | No |

---

### 8. App Catalog Collection

```
/app_catalog/{appId}
```

Curated list of apps with safety ratings (admin-managed).

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `packageName` | string | Android package name | Yes |
| `appName` | string | Display name | Yes |
| `appIcon` | string | Icon URL | Yes |
| `category` | string | music, education, utility, communication, payment, game | Yes |
| `safetyRating` | string | safe, caution, restricted | Yes |
| `ageRating` | string | all, 7+, 12+, 16+ | Yes |
| `description` | string | Brief description | No |
| `popularityRank` | number | For sorting | No |
| `defaultEnabled` | map | Default enabled by protection level | No |
| `updatedAt` | timestamp | Last review time | Yes |

**Example Document:**

```json
{
  "packageName": "com.spotify.music",
  "appName": "Spotify",
  "appIcon": "https://storage.googleapis.com/kidsafe/icons/spotify.png",
  "category": "music",
  "safetyRating": "safe",
  "ageRating": "all",
  "description": "Music streaming service with kids-friendly content options",
  "popularityRank": 1,
  "defaultEnabled": {
    "maximum": true,
    "balanced": true,
    "trust": true
  },
  "updatedAt": "2024-01-01T00:00:00Z"
}
```

**Pre-populated Apps:**

```json
[
  // Music Apps
  { "packageName": "com.spotify.music", "category": "music", "safetyRating": "safe" },
  { "packageName": "com.jio.saavn", "category": "music", "safetyRating": "safe" },
  { "packageName": "com.amazon.mp3", "category": "music", "safetyRating": "safe" },
  { "packageName": "com.google.android.apps.youtube.music", "category": "music", "safetyRating": "caution" },
  { "packageName": "com.gaana", "category": "music", "safetyRating": "safe" },
  { "packageName": "com.wynk.music", "category": "music", "safetyRating": "safe" },

  // Communication Apps
  { "packageName": "com.whatsapp", "category": "communication", "safetyRating": "caution" },
  { "packageName": "com.google.android.apps.messaging", "category": "communication", "safetyRating": "safe" },

  // Payment Apps
  { "packageName": "com.phonepe.app", "category": "payment", "safetyRating": "safe" },
  { "packageName": "com.google.android.apps.nbu.paisa.user", "category": "payment", "safetyRating": "safe" },
  { "packageName": "net.one97.paytm", "category": "payment", "safetyRating": "safe" },

  // Education Apps
  { "packageName": "com.byjus.thelearningapp", "category": "education", "safetyRating": "safe" },
  { "packageName": "com.vedantu.student", "category": "education", "safetyRating": "safe" },
  { "packageName": "com.duolingo", "category": "education", "safetyRating": "safe" },
  { "packageName": "com.khan.academy", "category": "education", "safetyRating": "safe" },

  // Utility Apps
  { "packageName": "com.android.deskclock", "category": "utility", "safetyRating": "safe" },
  { "packageName": "com.android.calculator2", "category": "utility", "safetyRating": "safe" },
  { "packageName": "com.google.android.calendar", "category": "utility", "safetyRating": "safe" }
]
```

---

### 9. Subscriptions Collection

```
/subscriptions/{subscriptionId}
```

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `id` | string | Subscription ID | Yes |
| `userId` | string | Parent user ID | Yes |
| `planId` | string | Plan identifier | Yes |
| `planName` | string | basic, premium, family | Yes |
| `status` | string | active, cancelled, expired, past_due | Yes |
| `provider` | string | razorpay, manual | Yes |
| `providerSubscriptionId` | string | External subscription ID | No |
| `amount` | number | Amount in paise | Yes |
| `currency` | string | INR | Yes |
| `interval` | string | monthly, annual | Yes |
| `currentPeriodStart` | timestamp | Current billing period start | Yes |
| `currentPeriodEnd` | timestamp | Current billing period end | Yes |
| `cancelledAt` | timestamp | Cancellation time | No |
| `cancelReason` | string | Cancellation reason | No |
| `createdAt` | timestamp | Subscription creation | Yes |
| `updatedAt` | timestamp | Last update | Yes |

**Example Document:**

```json
{
  "id": "sub_xyz789",
  "userId": "user_abc123",
  "planId": "premium_monthly",
  "planName": "premium",
  "status": "active",
  "provider": "razorpay",
  "providerSubscriptionId": "sub_LmKbNSPvVAPwvB",
  "amount": 24900,
  "currency": "INR",
  "interval": "monthly",
  "currentPeriodStart": "2024-01-15T00:00:00Z",
  "currentPeriodEnd": "2024-02-14T23:59:59Z",
  "createdAt": "2024-01-15T10:00:00Z",
  "updatedAt": "2024-01-15T10:00:00Z"
}
```

---

### 10. Payments Collection

```
/payments/{paymentId}
```

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `id` | string | Payment ID | Yes |
| `userId` | string | Parent user ID | Yes |
| `subscriptionId` | string | Associated subscription | No |
| `orderId` | string | Order ID (for hardware) | No |
| `amount` | number | Amount in INR | Yes |
| `currency` | string | INR | Yes |
| `status` | string | captured, failed, refunded | Yes |
| `provider` | string | razorpay | Yes |
| `providerPaymentId` | string | External payment ID | Yes |
| `errorCode` | string | Error code if failed | No |
| `errorDescription` | string | Error description | No |
| `refundedAt` | timestamp | Refund time | No |
| `refundAmount` | number | Refund amount | No |
| `createdAt` | timestamp | Payment time | Yes |

---

### 11. Orders Collection

```
/orders/{orderId}
```

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `id` | string | Order ID | Yes |
| `userId` | string | Parent user ID | Yes |
| `status` | string | pending, confirmed, shipped, delivered, cancelled | Yes |
| `items` | array | Order items | Yes |
| `shippingAddress` | map | Delivery address | Yes |
| `totalAmount` | number | Total in INR | Yes |
| `paymentId` | string | Payment reference | No |
| `trackingNumber` | string | Shipping tracking | No |
| `trackingUrl` | string | Tracking URL | No |
| `notes` | string | Order notes | No |
| `createdAt` | timestamp | Order time | Yes |
| `updatedAt` | timestamp | Last update | Yes |
| `deliveredAt` | timestamp | Delivery time | No |

**Order Item Structure:**

```typescript
{
  productId: string,
  productName: string,
  quantity: number,
  unitPrice: number,
  totalPrice: number
}
```

**Shipping Address Structure:**

```typescript
{
  name: string,
  phone: string,
  addressLine1: string,
  addressLine2: string,
  city: string,
  state: string,
  pincode: string,
  landmark: string
}
```

---

### 12. Alert Logs Collection

```
/alert_logs/{alertId}
```

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `id` | string | Alert ID | Yes |
| `type` | string | panic, battery_low, offline, limit_reached | Yes |
| `deviceId` | string | Source device | Yes |
| `parentId` | string | Parent to notify | Yes |
| `location` | geopoint | Location if available | No |
| `message` | string | Alert message | No |
| `acknowledged` | boolean | Parent acknowledged | Yes |
| `acknowledgedAt` | timestamp | Acknowledgement time | No |
| `timestamp` | timestamp | Alert time | Yes |

---

### 13. Support Tickets Collection

```
/support_tickets/{ticketId}
```

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `id` | string | Ticket ID | Yes |
| `userId` | string | User who created | Yes |
| `deviceId` | string | Related device | No |
| `subject` | string | Ticket subject | Yes |
| `description` | string | Issue description | Yes |
| `category` | string | setup, technical, billing, other | Yes |
| `status` | string | open, in_progress, resolved, closed | Yes |
| `priority` | string | low, medium, high, urgent | Yes |
| `assignedTo` | string | Support agent | No |
| `resolution` | string | How it was resolved | No |
| `createdAt` | timestamp | Creation time | Yes |
| `updatedAt` | timestamp | Last update | Yes |
| `resolvedAt` | timestamp | Resolution time | No |

---

### 14. Feedback Collection

```
/feedback/{feedbackId}
```

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `id` | string | Feedback ID | Yes |
| `userId` | string | User who submitted | Yes |
| `type` | string | feature_request, bug_report, general | Yes |
| `rating` | number | 1-5 stars | No |
| `message` | string | Feedback message | Yes |
| `appVersion` | string | App version | No |
| `deviceInfo` | map | Device information | No |
| `screenshot` | string | Screenshot URL | No |
| `createdAt` | timestamp | Submission time | Yes |

---

## Realtime Database Structure

The Realtime Database is used for real-time data that needs low-latency updates.

```
/
├── devices/
│   └── {deviceId}/
│       ├── commands/
│       │   └── {commandId}/
│       │       ├── type: string
│       │       ├── payload: object
│       │       ├── timestamp: number
│       │       └── status: string
│       ├── status/
│       │   ├── isOnline: boolean
│       │   ├── lastSeen: number
│       │   ├── mode: string
│       │   ├── batteryLevel: number
│       │   ├── isCharging: boolean
│       │   ├── wifiConnected: boolean
│       │   └── screenOn: boolean
│       ├── nowPlaying/
│       │   ├── app: string
│       │   ├── title: string
│       │   ├── artist: string
│       │   ├── albumArt: string
│       │   └── timestamp: number
│       ├── location/
│       │   ├── latitude: number
│       │   ├── longitude: number
│       │   ├── accuracy: number
│       │   └── timestamp: number
│       └── acks/
│           └── {commandId}/
│               ├── status: string
│               └── timestamp: number
│
└── panic_alerts/
    └── {alertId}/
        ├── deviceId: string
        ├── latitude: number
        ├── longitude: number
        └── timestamp: number
```

### Command Types

| Type | Payload | Description |
|------|---------|-------------|
| `toggle_mode` | `{ mode: "kids" \| "standard" }` | Switch device mode |
| `update_whitelist` | `{ action: "add" \| "remove", packageName: string }` | Modify app whitelist |
| `set_screen_time` | `{ dailyLimitMinutes: number, appLimits: object }` | Update screen time |
| `set_bedtime` | `{ enabled: boolean, start: string, end: string }` | Configure bedtime |
| `lock_device` | `{}` | Immediately lock device |
| `unlock_device` | `{}` | Unlock device |
| `set_volume_limit` | `{ maxPercent: number }` | Set max volume |
| `install_app` | `{ packageName: string }` | Remote install app |
| `uninstall_app` | `{ packageName: string }` | Remote uninstall app |
| `locate_device` | `{}` | Request current location |
| `play_sound` | `{ duration: number }` | Play sound to locate |
| `send_message` | `{ message: string }` | Send message to device |
| `update_settings` | `{ settings: object }` | Update device settings |

### Command Example

```json
{
  "type": "set_screen_time",
  "payload": {
    "dailyLimitMinutes": 120,
    "appLimits": {
      "com.spotify.music": 60,
      "com.jio.saavn": 60
    }
  },
  "timestamp": 1705323600000,
  "status": "pending"
}
```

### Status Update Example

```json
{
  "isOnline": true,
  "lastSeen": 1705323600000,
  "mode": "kids",
  "batteryLevel": 67,
  "isCharging": false,
  "wifiConnected": true,
  "screenOn": true
}
```

### Now Playing Example

```json
{
  "app": "com.spotify.music",
  "title": "Kesariya",
  "artist": "Arijit Singh",
  "albumArt": "https://i.scdn.co/image/...",
  "timestamp": 1705323600000
}
```

---

## Cloud Storage Structure

```
/kidsafe-storage/
├── users/
│   └── {userId}/
│       └── profile.jpg
│
├── devices/
│   └── {deviceId}/
│       └── screenshots/
│           └── {timestamp}.jpg
│
├── app_icons/
│   └── {packageName}.png
│
├── support/
│   └── {ticketId}/
│       └── {filename}
│
└── updates/
    ├── launcher/
    │   └── launcher-{version}.apk
    └── dpc/
        └── dpc-{version}.apk
```

---

## Indexes Required

### Firestore Composite Indexes

```
1. devices
   - parentId ASC, createdAt DESC

2. devices/{deviceId}/usage
   - date DESC

3. devices/{deviceId}/usage_summary
   - date DESC

4. subscriptions
   - userId ASC, status ASC

5. orders
   - userId ASC, createdAt DESC

6. support_tickets
   - userId ASC, status ASC, createdAt DESC

7. alert_logs
   - parentId ASC, timestamp DESC

8. payments
   - userId ASC, createdAt DESC
```

### Firestore Index Configuration

```json
// firestore.indexes.json
{
  "indexes": [
    {
      "collectionGroup": "devices",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "parentId", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "usage",
      "queryScope": "COLLECTION_GROUP",
      "fields": [
        { "fieldPath": "date", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "subscriptions",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "userId", "order": "ASCENDING" },
        { "fieldPath": "status", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "orders",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "userId", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "support_tickets",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "userId", "order": "ASCENDING" },
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "alert_logs",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "parentId", "order": "ASCENDING" },
        { "fieldPath": "timestamp", "order": "DESCENDING" }
      ]
    }
  ],
  "fieldOverrides": []
}
```

---

## Data Retention Policy

| Collection | Retention | Reason |
|------------|-----------|--------|
| users | Forever | Account data |
| devices | Forever | Device history |
| devices/usage | 90 days | Storage optimization |
| devices/usage_summary | 365 days | Annual reports |
| pairing_codes | 15 minutes | Security |
| device_tokens | Until device removed | Security |
| alert_logs | 90 days | Audit trail |
| support_tickets | 2 years | Legal |
| feedback | Forever | Product improvement |

---

## Migration Scripts

### Initial Data Seeding

```typescript
// scripts/seed_app_catalog.ts

import * as admin from 'firebase-admin';

const apps = [
  // Music
  {
    packageName: 'com.spotify.music',
    appName: 'Spotify',
    category: 'music',
    safetyRating: 'safe',
    ageRating: 'all',
    popularityRank: 1,
    defaultEnabled: { maximum: true, balanced: true, trust: true }
  },
  {
    packageName: 'com.jio.saavn',
    appName: 'JioSaavn',
    category: 'music',
    safetyRating: 'safe',
    ageRating: 'all',
    popularityRank: 2,
    defaultEnabled: { maximum: true, balanced: true, trust: true }
  },
  // ... more apps
];

async function seedAppCatalog() {
  const db = admin.firestore();
  const batch = db.batch();

  for (const app of apps) {
    const ref = db.collection('app_catalog').doc(app.packageName.replace(/\./g, '_'));
    batch.set(ref, {
      ...app,
      updatedAt: admin.firestore.FieldValue.serverTimestamp()
    });
  }

  await batch.commit();
  console.log(`Seeded ${apps.length} apps to catalog`);
}

seedAppCatalog();
```

---

## TypeScript Type Definitions

See [09_api_contracts.md](./09_api_contracts.md) for complete TypeScript interfaces for all data models.
