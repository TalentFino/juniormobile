# API Contracts & Data Models

## Overview

This document defines all data models, API contracts, and interfaces used across the KidSafe application. These contracts ensure consistency between:
- Parent App (Flutter)
- Child Device (Kotlin/Android)
- Firebase Backend
- Cloud Functions

---

## TypeScript/Dart Shared Models

### User Models

```typescript
// TypeScript (Cloud Functions)

interface User {
  id: string;
  email?: string;
  phone?: string;
  displayName: string;
  profileImageUrl?: string;
  fcmToken?: string;
  fcmTokenUpdatedAt?: Date;
  subscriptionId?: string;
  subscriptionStatus?: SubscriptionStatus;
  subscriptionPlan?: SubscriptionPlan;
  createdAt: Date;
  updatedAt: Date;
  settings?: UserSettings;
  timezone?: string;
}

interface UserSettings {
  notifications: NotificationSettings;
  language: 'en' | 'hi';
  theme: 'light' | 'dark' | 'system';
}

interface NotificationSettings {
  panicAlerts: boolean;
  batteryAlerts: boolean;
  offlineAlerts: boolean;
  screenTimeLimits: boolean;
  dailyReports: boolean;
}

type SubscriptionStatus = 'active' | 'cancelled' | 'expired' | 'past_due';
type SubscriptionPlan = 'basic' | 'premium' | 'family';
```

```dart
// Dart (Flutter)

import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

part 'user_model.freezed.dart';
part 'user_model.g.dart';

@freezed
class UserModel with _$UserModel {
  const factory UserModel({
    required String id,
    String? email,
    String? phone,
    required String displayName,
    String? profileImageUrl,
    String? fcmToken,
    DateTime? fcmTokenUpdatedAt,
    String? subscriptionId,
    SubscriptionStatus? subscriptionStatus,
    SubscriptionPlan? subscriptionPlan,
    required DateTime createdAt,
    required DateTime updatedAt,
    UserSettings? settings,
    String? timezone,
  }) = _UserModel;

  factory UserModel.fromJson(Map<String, dynamic> json) =>
      _$UserModelFromJson(json);

  factory UserModel.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return UserModel.fromJson({
      'id': doc.id,
      ...data,
      'createdAt': (data['createdAt'] as Timestamp).toDate().toIso8601String(),
      'updatedAt': (data['updatedAt'] as Timestamp).toDate().toIso8601String(),
    });
  }
}

@freezed
class UserSettings with _$UserSettings {
  const factory UserSettings({
    required NotificationSettings notifications,
    @Default('en') String language,
    @Default('light') String theme,
  }) = _UserSettings;

  factory UserSettings.fromJson(Map<String, dynamic> json) =>
      _$UserSettingsFromJson(json);
}

@freezed
class NotificationSettings with _$NotificationSettings {
  const factory NotificationSettings({
    @Default(true) bool panicAlerts,
    @Default(true) bool batteryAlerts,
    @Default(true) bool offlineAlerts,
    @Default(true) bool screenTimeLimits,
    @Default(false) bool dailyReports,
  }) = _NotificationSettings;

  factory NotificationSettings.fromJson(Map<String, dynamic> json) =>
      _$NotificationSettingsFromJson(json);
}

enum SubscriptionStatus {
  active,
  cancelled,
  expired,
  pastDue,
}

enum SubscriptionPlan {
  basic,
  premium,
  family,
}
```

---

### Device Models

```typescript
// TypeScript

interface Device {
  id: string;
  parentId: string;
  childName: string;
  childAge: number;
  childAvatar?: string;
  protectionLevel: ProtectionLevel;
  deviceInfo: DeviceInfo;
  status: DeviceStatus;
  mode: DeviceMode;
  lastSeen: Date;
  createdAt: Date;
  updatedAt: Date;
  screenTimeSettings?: ScreenTimeSettings;
  locationEnabled?: boolean;
  volumeLimit?: number;
  panicContacts?: string[];
}

interface DeviceInfo {
  model: string;
  manufacturer: string;
  androidVersion: string;
  sdkVersion: number;
  serialNumber: string;
  imei?: string;
  launcherVersion: string;
  dpcVersion: string;
}

interface ScreenTimeSettings {
  dailyLimitMinutes: number;
  bedtimeStart?: string; // HH:mm
  bedtimeEnd?: string;   // HH:mm
  bedtimeEnabled: boolean;
  weekdayLimitMinutes?: number;
  weekendLimitMinutes?: number;
  appLimits?: Record<string, number>;
}

type ProtectionLevel = 'maximum' | 'balanced' | 'trust';
type DeviceStatus = 'online' | 'offline' | 'pending';
type DeviceMode = 'kids' | 'standard';
```

```dart
// Dart

@freezed
class DeviceModel with _$DeviceModel {
  const factory DeviceModel({
    required String id,
    required String parentId,
    required String childName,
    required int childAge,
    String? childAvatar,
    required ProtectionLevel protectionLevel,
    required DeviceInfo deviceInfo,
    required DeviceStatus status,
    required DeviceMode mode,
    required DateTime lastSeen,
    required DateTime createdAt,
    required DateTime updatedAt,
    ScreenTimeSettings? screenTimeSettings,
    @Default(false) bool locationEnabled,
    @Default(70) int volumeLimit,
    @Default([]) List<String> panicContacts,
  }) = _DeviceModel;

  factory DeviceModel.fromJson(Map<String, dynamic> json) =>
      _$DeviceModelFromJson(json);

  factory DeviceModel.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return DeviceModel.fromJson({
      'id': doc.id,
      ...data,
      'lastSeen': (data['lastSeen'] as Timestamp).toDate().toIso8601String(),
      'createdAt': (data['createdAt'] as Timestamp).toDate().toIso8601String(),
      'updatedAt': (data['updatedAt'] as Timestamp).toDate().toIso8601String(),
    });
  }
}

@freezed
class DeviceInfo with _$DeviceInfo {
  const factory DeviceInfo({
    required String model,
    required String manufacturer,
    required String androidVersion,
    required int sdkVersion,
    required String serialNumber,
    String? imei,
    required String launcherVersion,
    required String dpcVersion,
  }) = _DeviceInfo;

  factory DeviceInfo.fromJson(Map<String, dynamic> json) =>
      _$DeviceInfoFromJson(json);
}

@freezed
class ScreenTimeSettings with _$ScreenTimeSettings {
  const factory ScreenTimeSettings({
    @Default(0) int dailyLimitMinutes,
    String? bedtimeStart,
    String? bedtimeEnd,
    @Default(false) bool bedtimeEnabled,
    int? weekdayLimitMinutes,
    int? weekendLimitMinutes,
    @Default({}) Map<String, int> appLimits,
  }) = _ScreenTimeSettings;

  factory ScreenTimeSettings.fromJson(Map<String, dynamic> json) =>
      _$ScreenTimeSettingsFromJson(json);
}

enum ProtectionLevel {
  maximum,
  balanced,
  trust,
}

enum DeviceStatus {
  online,
  offline,
  pending,
}

enum DeviceMode {
  kids,
  standard,
}
```

---

### Kotlin Data Classes (Android)

```kotlin
// Device Models

data class Device(
    val id: String,
    val parentId: String,
    val childName: String,
    val childAge: Int,
    val childAvatar: String? = null,
    val protectionLevel: ProtectionLevel,
    val deviceInfo: DeviceInfo,
    val status: DeviceStatus,
    val mode: DeviceMode,
    val lastSeen: Long,
    val createdAt: Long,
    val updatedAt: Long,
    val screenTimeSettings: ScreenTimeSettings? = null,
    val locationEnabled: Boolean = false,
    val volumeLimit: Int = 70,
    val panicContacts: List<String> = emptyList()
)

data class DeviceInfo(
    val model: String,
    val manufacturer: String,
    val androidVersion: String,
    val sdkVersion: Int,
    val serialNumber: String,
    val imei: String? = null,
    val launcherVersion: String,
    val dpcVersion: String
)

data class ScreenTimeSettings(
    val dailyLimitMinutes: Int = 0,
    val bedtimeStart: String? = null,
    val bedtimeEnd: String? = null,
    val bedtimeEnabled: Boolean = false,
    val weekdayLimitMinutes: Int? = null,
    val weekendLimitMinutes: Int? = null,
    val appLimits: Map<String, Int> = emptyMap()
)

enum class ProtectionLevel {
    MAXIMUM,
    BALANCED,
    TRUST;

    companion object {
        fun fromString(value: String): ProtectionLevel {
            return when (value.lowercase()) {
                "maximum" -> MAXIMUM
                "balanced" -> BALANCED
                "trust" -> TRUST
                else -> BALANCED
            }
        }
    }
}

enum class DeviceStatus {
    ONLINE,
    OFFLINE,
    PENDING;

    companion object {
        fun fromString(value: String): DeviceStatus {
            return when (value.lowercase()) {
                "online" -> ONLINE
                "offline" -> OFFLINE
                "pending" -> PENDING
                else -> OFFLINE
            }
        }
    }
}

enum class DeviceMode {
    KIDS,
    STANDARD;

    companion object {
        fun fromString(value: String): DeviceMode {
            return when (value.lowercase()) {
                "kids" -> KIDS
                "standard" -> STANDARD
                else -> KIDS
            }
        }
    }

    fun toFirebaseValue(): String = name.lowercase()
}
```

---

### Command Models

```typescript
// TypeScript

interface Command {
  id: string;
  type: CommandType;
  payload: Record<string, any>;
  timestamp: number;
  status: CommandStatus;
}

type CommandType =
  | 'toggle_mode'
  | 'update_whitelist'
  | 'set_screen_time'
  | 'set_bedtime'
  | 'lock_device'
  | 'unlock_device'
  | 'set_volume_limit'
  | 'install_app'
  | 'uninstall_app'
  | 'locate_device'
  | 'play_sound'
  | 'send_message'
  | 'update_settings';

type CommandStatus = 'pending' | 'processing' | 'completed' | 'failed';

// Command payloads
interface ToggleModePayload {
  mode: 'kids' | 'standard';
}

interface UpdateWhitelistPayload {
  action: 'add' | 'remove';
  packageName: string;
  appName?: string;
  dailyLimitMinutes?: number;
}

interface SetScreenTimePayload {
  dailyLimitMinutes: number;
  appLimits?: Record<string, number>;
}

interface SetBedtimePayload {
  enabled: boolean;
  start: string; // HH:mm
  end: string;   // HH:mm
}

interface SetVolumeLimitPayload {
  maxPercent: number; // 0-100
}

interface InstallAppPayload {
  packageName: string;
}

interface SendMessagePayload {
  message: string;
  duration?: number; // seconds to display
}
```

```dart
// Dart

@freezed
class Command with _$Command {
  const factory Command({
    String? id,
    required CommandType type,
    required Map<String, dynamic> payload,
    required DateTime timestamp,
    @Default(CommandStatus.pending) CommandStatus status,
  }) = _Command;

  factory Command.fromJson(Map<String, dynamic> json) =>
      _$CommandFromJson(json);
}

enum CommandType {
  toggleMode,
  updateWhitelist,
  setScreenTime,
  setBedtime,
  lockDevice,
  unlockDevice,
  setVolumeLimit,
  installApp,
  uninstallApp,
  locateDevice,
  playSound,
  sendMessage,
  updateSettings,
}

enum CommandStatus {
  pending,
  processing,
  completed,
  failed,
}

// Command payload builders
class CommandPayloads {
  static Map<String, dynamic> toggleMode(DeviceMode mode) => {
    'mode': mode.name,
  };

  static Map<String, dynamic> updateWhitelist({
    required WhitelistAction action,
    required String packageName,
    String? appName,
    int? dailyLimitMinutes,
  }) => {
    'action': action.name,
    'packageName': packageName,
    if (appName != null) 'appName': appName,
    if (dailyLimitMinutes != null) 'dailyLimitMinutes': dailyLimitMinutes,
  };

  static Map<String, dynamic> setScreenTime({
    required int dailyLimitMinutes,
    Map<String, int>? appLimits,
  }) => {
    'dailyLimitMinutes': dailyLimitMinutes,
    if (appLimits != null) 'appLimits': appLimits,
  };

  static Map<String, dynamic> setBedtime({
    required bool enabled,
    required String start,
    required String end,
  }) => {
    'enabled': enabled,
    'start': start,
    'end': end,
  };

  static Map<String, dynamic> setVolumeLimit(int maxPercent) => {
    'maxPercent': maxPercent.clamp(0, 100),
  };

  static Map<String, dynamic> sendMessage(String message, {int? duration}) => {
    'message': message,
    if (duration != null) 'duration': duration,
  };
}

enum WhitelistAction { add, remove }
```

```kotlin
// Kotlin

data class Command(
    val id: String? = null,
    val type: CommandType,
    val payload: Map<String, Any>,
    val timestamp: Long,
    val status: CommandStatus = CommandStatus.PENDING
) {
    companion object {
        fun fromMap(id: String, map: Map<String, Any>): Command {
            return Command(
                id = id,
                type = CommandType.fromString(map["type"] as String),
                payload = (map["payload"] as? Map<String, Any>) ?: emptyMap(),
                timestamp = (map["timestamp"] as Long),
                status = CommandStatus.fromString(map["status"] as? String ?: "pending")
            )
        }
    }
}

enum class CommandType {
    TOGGLE_MODE,
    UPDATE_WHITELIST,
    SET_SCREEN_TIME,
    SET_BEDTIME,
    LOCK_DEVICE,
    UNLOCK_DEVICE,
    SET_VOLUME_LIMIT,
    INSTALL_APP,
    UNINSTALL_APP,
    LOCATE_DEVICE,
    PLAY_SOUND,
    SEND_MESSAGE,
    UPDATE_SETTINGS;

    companion object {
        fun fromString(value: String): CommandType {
            return when (value.lowercase().replace("-", "_")) {
                "toggle_mode" -> TOGGLE_MODE
                "update_whitelist" -> UPDATE_WHITELIST
                "set_screen_time" -> SET_SCREEN_TIME
                "set_bedtime" -> SET_BEDTIME
                "lock_device" -> LOCK_DEVICE
                "unlock_device" -> UNLOCK_DEVICE
                "set_volume_limit" -> SET_VOLUME_LIMIT
                "install_app" -> INSTALL_APP
                "uninstall_app" -> UNINSTALL_APP
                "locate_device" -> LOCATE_DEVICE
                "play_sound" -> PLAY_SOUND
                "send_message" -> SEND_MESSAGE
                "update_settings" -> UPDATE_SETTINGS
                else -> throw IllegalArgumentException("Unknown command type: $value")
            }
        }
    }
}

enum class CommandStatus {
    PENDING,
    PROCESSING,
    COMPLETED,
    FAILED;

    companion object {
        fun fromString(value: String): CommandStatus {
            return when (value.lowercase()) {
                "pending" -> PENDING
                "processing" -> PROCESSING
                "completed" -> COMPLETED
                "failed" -> FAILED
                else -> PENDING
            }
        }
    }

    fun toFirebaseValue(): String = name.lowercase()
}
```

---

### App Whitelist Models

```typescript
// TypeScript

interface WhitelistedApp {
  packageName: string;
  appName: string;
  appIcon?: string;
  category: AppCategory;
  enabled: boolean;
  dailyLimitMinutes?: number;
  scheduleRestriction?: ScheduleRestriction;
  addedAt: Date;
  addedBy: 'parent' | 'default' | 'remote';
}

interface ScheduleRestriction {
  allowedHoursStart: string; // HH:mm
  allowedHoursEnd: string;   // HH:mm
  daysAllowed: DayOfWeek[];
}

type AppCategory =
  | 'music'
  | 'education'
  | 'utility'
  | 'communication'
  | 'payment'
  | 'game'
  | 'entertainment'
  | 'other';

type DayOfWeek = 'mon' | 'tue' | 'wed' | 'thu' | 'fri' | 'sat' | 'sun';
```

```dart
// Dart

@freezed
class WhitelistedApp with _$WhitelistedApp {
  const factory WhitelistedApp({
    required String packageName,
    required String appName,
    String? appIcon,
    required AppCategory category,
    required bool enabled,
    int? dailyLimitMinutes,
    ScheduleRestriction? scheduleRestriction,
    required DateTime addedAt,
    required WhitelistSource addedBy,
  }) = _WhitelistedApp;

  factory WhitelistedApp.fromJson(Map<String, dynamic> json) =>
      _$WhitelistedAppFromJson(json);
}

@freezed
class ScheduleRestriction with _$ScheduleRestriction {
  const factory ScheduleRestriction({
    required String allowedHoursStart,
    required String allowedHoursEnd,
    required List<DayOfWeek> daysAllowed,
  }) = _ScheduleRestriction;

  factory ScheduleRestriction.fromJson(Map<String, dynamic> json) =>
      _$ScheduleRestrictionFromJson(json);
}

enum AppCategory {
  music,
  education,
  utility,
  communication,
  payment,
  game,
  entertainment,
  other,
}

enum WhitelistSource {
  parent,
  @JsonValue('default')
  defaultApp,
  remote,
}

enum DayOfWeek {
  mon,
  tue,
  wed,
  thu,
  fri,
  sat,
  sun,
}
```

```kotlin
// Kotlin

data class WhitelistedApp(
    val packageName: String,
    val appName: String,
    val appIcon: String? = null,
    val category: AppCategory,
    val enabled: Boolean,
    val dailyLimitMinutes: Int? = null,
    val scheduleRestriction: ScheduleRestriction? = null,
    val addedAt: Long,
    val addedBy: WhitelistSource
)

data class ScheduleRestriction(
    val allowedHoursStart: String,
    val allowedHoursEnd: String,
    val daysAllowed: List<DayOfWeek>
)

enum class AppCategory {
    MUSIC,
    EDUCATION,
    UTILITY,
    COMMUNICATION,
    PAYMENT,
    GAME,
    ENTERTAINMENT,
    OTHER;

    companion object {
        fun fromString(value: String): AppCategory {
            return values().find {
                it.name.equals(value, ignoreCase = true)
            } ?: OTHER
        }
    }
}

enum class WhitelistSource {
    PARENT,
    DEFAULT,
    REMOTE
}

enum class DayOfWeek {
    MON, TUE, WED, THU, FRI, SAT, SUN
}
```

---

### Usage Models

```typescript
// TypeScript

interface UsageData {
  date: string; // YYYY-MM-DD
  totalMinutes: number;
  appUsage: Record<string, number>;
  hourlyUsage?: Record<string, number>;
  sessions?: UsageSession[];
  limitReached: boolean;
  limitReachedAt?: Date;
}

interface UsageSession {
  startTime: Date;
  endTime: Date;
  duration: number; // minutes
}

interface UsageSummary {
  date: string;
  totalMinutes: number;
  appBreakdown: Record<string, number>;
  peakHour?: number;
  limitReached: boolean;
  comparedToAverage?: number; // percentage
}
```

```dart
// Dart

@freezed
class UsageData with _$UsageData {
  const factory UsageData({
    required String date,
    required int totalMinutes,
    required Map<String, int> appUsage,
    Map<String, int>? hourlyUsage,
    List<UsageSession>? sessions,
    required bool limitReached,
    DateTime? limitReachedAt,
  }) = _UsageData;

  factory UsageData.fromJson(Map<String, dynamic> json) =>
      _$UsageDataFromJson(json);
}

@freezed
class UsageSession with _$UsageSession {
  const factory UsageSession({
    required DateTime startTime,
    required DateTime endTime,
    required int duration,
  }) = _UsageSession;

  factory UsageSession.fromJson(Map<String, dynamic> json) =>
      _$UsageSessionFromJson(json);
}

@freezed
class UsageSummary with _$UsageSummary {
  const factory UsageSummary({
    required String date,
    required int totalMinutes,
    required Map<String, int> appBreakdown,
    int? peakHour,
    required bool limitReached,
    double? comparedToAverage,
  }) = _UsageSummary;

  factory UsageSummary.fromJson(Map<String, dynamic> json) =>
      _$UsageSummaryFromJson(json);
}
```

---

### Real-time Status Models

```typescript
// TypeScript (RTDB)

interface DeviceRealtimeStatus {
  isOnline: boolean;
  lastSeen: number; // timestamp
  mode: 'kids' | 'standard';
  batteryLevel: number;
  isCharging: boolean;
  wifiConnected: boolean;
  screenOn: boolean;
}

interface NowPlaying {
  app: string;
  title: string;
  artist?: string;
  albumArt?: string;
  timestamp: number;
}

interface DeviceLocation {
  latitude: number;
  longitude: number;
  accuracy: number;
  timestamp: number;
}
```

```dart
// Dart

@freezed
class DeviceRealtimeStatus with _$DeviceRealtimeStatus {
  const factory DeviceRealtimeStatus({
    required bool isOnline,
    required DateTime lastSeen,
    required DeviceMode mode,
    required int batteryLevel,
    required bool isCharging,
    required bool wifiConnected,
    required bool screenOn,
  }) = _DeviceRealtimeStatus;

  factory DeviceRealtimeStatus.fromRtdb(Map<dynamic, dynamic> data) {
    return DeviceRealtimeStatus(
      isOnline: data['isOnline'] as bool? ?? false,
      lastSeen: DateTime.fromMillisecondsSinceEpoch(data['lastSeen'] as int),
      mode: DeviceMode.values.firstWhere(
        (m) => m.name == (data['mode'] as String? ?? 'kids'),
        orElse: () => DeviceMode.kids,
      ),
      batteryLevel: data['batteryLevel'] as int? ?? 100,
      isCharging: data['isCharging'] as bool? ?? false,
      wifiConnected: data['wifiConnected'] as bool? ?? false,
      screenOn: data['screenOn'] as bool? ?? false,
    );
  }
}

@freezed
class NowPlaying with _$NowPlaying {
  const factory NowPlaying({
    required String app,
    required String title,
    String? artist,
    String? albumArt,
    required DateTime timestamp,
  }) = _NowPlaying;

  factory NowPlaying.fromRtdb(Map<dynamic, dynamic> data) {
    return NowPlaying(
      app: data['app'] as String,
      title: data['title'] as String,
      artist: data['artist'] as String?,
      albumArt: data['albumArt'] as String?,
      timestamp: DateTime.fromMillisecondsSinceEpoch(data['timestamp'] as int),
    );
  }
}

@freezed
class DeviceLocation with _$DeviceLocation {
  const factory DeviceLocation({
    required double latitude,
    required double longitude,
    required double accuracy,
    required DateTime timestamp,
  }) = _DeviceLocation;

  factory DeviceLocation.fromRtdb(Map<dynamic, dynamic> data) {
    return DeviceLocation(
      latitude: (data['latitude'] as num).toDouble(),
      longitude: (data['longitude'] as num).toDouble(),
      accuracy: (data['accuracy'] as num).toDouble(),
      timestamp: DateTime.fromMillisecondsSinceEpoch(data['timestamp'] as int),
    );
  }
}
```

```kotlin
// Kotlin

data class DeviceRealtimeStatus(
    val isOnline: Boolean,
    val lastSeen: Long,
    val mode: DeviceMode,
    val batteryLevel: Int,
    val isCharging: Boolean,
    val wifiConnected: Boolean,
    val screenOn: Boolean
) {
    fun toMap(): Map<String, Any> = mapOf(
        "isOnline" to isOnline,
        "lastSeen" to lastSeen,
        "mode" to mode.toFirebaseValue(),
        "batteryLevel" to batteryLevel,
        "isCharging" to isCharging,
        "wifiConnected" to wifiConnected,
        "screenOn" to screenOn
    )
}

data class NowPlaying(
    val app: String,
    val title: String,
    val artist: String? = null,
    val albumArt: String? = null,
    val timestamp: Long = System.currentTimeMillis()
) {
    fun toMap(): Map<String, Any?> = mapOf(
        "app" to app,
        "title" to title,
        "artist" to artist,
        "albumArt" to albumArt,
        "timestamp" to timestamp
    )
}

data class DeviceLocation(
    val latitude: Double,
    val longitude: Double,
    val accuracy: Float,
    val timestamp: Long = System.currentTimeMillis()
) {
    fun toMap(): Map<String, Any> = mapOf(
        "latitude" to latitude,
        "longitude" to longitude,
        "accuracy" to accuracy,
        "timestamp" to timestamp
    )
}
```

---

### Alert Models

```typescript
// TypeScript

interface PanicAlert {
  id: string;
  deviceId: string;
  latitude?: number;
  longitude?: number;
  timestamp: number;
}

interface AlertLog {
  id: string;
  type: AlertType;
  deviceId: string;
  parentId: string;
  location?: {
    latitude: number;
    longitude: number;
  };
  message?: string;
  acknowledged: boolean;
  acknowledgedAt?: Date;
  timestamp: Date;
}

type AlertType = 'panic' | 'battery_low' | 'offline' | 'limit_reached' | 'safe_zone_exit';
```

```dart
// Dart

@freezed
class PanicAlert with _$PanicAlert {
  const factory PanicAlert({
    required String id,
    required String deviceId,
    double? latitude,
    double? longitude,
    required DateTime timestamp,
  }) = _PanicAlert;

  factory PanicAlert.fromJson(Map<String, dynamic> json) =>
      _$PanicAlertFromJson(json);
}

@freezed
class AlertLog with _$AlertLog {
  const factory AlertLog({
    required String id,
    required AlertType type,
    required String deviceId,
    required String parentId,
    GeoLocation? location,
    String? message,
    required bool acknowledged,
    DateTime? acknowledgedAt,
    required DateTime timestamp,
  }) = _AlertLog;

  factory AlertLog.fromJson(Map<String, dynamic> json) =>
      _$AlertLogFromJson(json);
}

@freezed
class GeoLocation with _$GeoLocation {
  const factory GeoLocation({
    required double latitude,
    required double longitude,
  }) = _GeoLocation;

  factory GeoLocation.fromJson(Map<String, dynamic> json) =>
      _$GeoLocationFromJson(json);
}

enum AlertType {
  panic,
  batteryLow,
  offline,
  limitReached,
  safeZoneExit,
}
```

---

### Subscription & Payment Models

```typescript
// TypeScript

interface Subscription {
  id: string;
  userId: string;
  planId: string;
  planName: SubscriptionPlan;
  status: SubscriptionStatus;
  provider: 'razorpay' | 'manual';
  providerSubscriptionId?: string;
  amount: number; // in paise
  currency: string;
  interval: 'monthly' | 'annual';
  currentPeriodStart: Date;
  currentPeriodEnd: Date;
  cancelledAt?: Date;
  cancelReason?: string;
  createdAt: Date;
  updatedAt: Date;
}

interface Payment {
  id: string;
  userId: string;
  subscriptionId?: string;
  orderId?: string;
  amount: number; // in INR
  currency: string;
  status: PaymentStatus;
  provider: 'razorpay';
  providerPaymentId: string;
  errorCode?: string;
  errorDescription?: string;
  refundedAt?: Date;
  refundAmount?: number;
  createdAt: Date;
}

type PaymentStatus = 'captured' | 'failed' | 'refunded';
```

```dart
// Dart

@freezed
class Subscription with _$Subscription {
  const factory Subscription({
    required String id,
    required String userId,
    required String planId,
    required SubscriptionPlan planName,
    required SubscriptionStatus status,
    required String provider,
    String? providerSubscriptionId,
    required int amount,
    required String currency,
    required SubscriptionInterval interval,
    required DateTime currentPeriodStart,
    required DateTime currentPeriodEnd,
    DateTime? cancelledAt,
    String? cancelReason,
    required DateTime createdAt,
    required DateTime updatedAt,
  }) = _Subscription;

  factory Subscription.fromJson(Map<String, dynamic> json) =>
      _$SubscriptionFromJson(json);
}

enum SubscriptionInterval {
  monthly,
  annual,
}

@freezed
class Payment with _$Payment {
  const factory Payment({
    required String id,
    required String userId,
    String? subscriptionId,
    String? orderId,
    required double amount,
    required String currency,
    required PaymentStatus status,
    required String provider,
    required String providerPaymentId,
    String? errorCode,
    String? errorDescription,
    DateTime? refundedAt,
    double? refundAmount,
    required DateTime createdAt,
  }) = _Payment;

  factory Payment.fromJson(Map<String, dynamic> json) =>
      _$PaymentFromJson(json);
}

enum PaymentStatus {
  captured,
  failed,
  refunded,
}
```

---

## Cloud Function Callable Contracts

### registerDevice

**Request:**
```typescript
interface RegisterDeviceRequest {
  pairingCode: string;
  deviceInfo: {
    model: string;
    manufacturer: string;
    androidVersion: string;
    sdkVersion: number;
    serialNumber: string;
    batteryLevel: number;
  };
}
```

**Response:**
```typescript
interface RegisterDeviceResponse {
  success: boolean;
  deviceId: string;
  deviceToken: string;
}
```

### createPairingCode

**Request:**
```typescript
interface CreatePairingCodeRequest {
  childName: string;
  childAge: number;
  protectionLevel: ProtectionLevel;
}
```

**Response:**
```typescript
interface CreatePairingCodeResponse {
  code: string;
  expiresAt: number; // timestamp
}
```

### sendCommand

**Request:**
```typescript
interface SendCommandRequest {
  deviceId: string;
  command: {
    type: CommandType;
    payload: Record<string, any>;
  };
}
```

**Response:**
```typescript
interface SendCommandResponse {
  success: boolean;
  commandId: string;
}
```

---

## FCM Notification Payloads

### Panic Alert

```json
{
  "notification": {
    "title": "PANIC ALERT",
    "body": "Rahul pressed the panic button!"
  },
  "data": {
    "type": "panic_alert",
    "deviceId": "device_xyz789",
    "latitude": "28.6139",
    "longitude": "77.2090",
    "timestamp": "1705323600000"
  },
  "android": {
    "priority": "high",
    "notification": {
      "channelId": "panic_alerts",
      "priority": "max",
      "sound": "alarm"
    }
  }
}
```

### Battery Low

```json
{
  "notification": {
    "title": "Low Battery",
    "body": "Rahul's device is at 15% battery"
  },
  "data": {
    "type": "battery_low",
    "deviceId": "device_xyz789",
    "batteryLevel": "15"
  }
}
```

### Screen Time Limit Reached

```json
{
  "notification": {
    "title": "Screen Time Limit",
    "body": "Rahul has reached the daily screen time limit"
  },
  "data": {
    "type": "limit_reached",
    "deviceId": "device_xyz789",
    "totalMinutes": "120"
  }
}
```

### Device Paired

```json
{
  "notification": {
    "title": "Device Connected!",
    "body": "Rahul's device is now set up and ready."
  },
  "data": {
    "type": "device_paired",
    "deviceId": "device_xyz789"
  }
}
```

---

## Error Codes

| Code | Description | Action |
|------|-------------|--------|
| `auth/invalid-token` | Invalid or expired auth token | Re-authenticate |
| `device/not-found` | Device ID not found | Check device exists |
| `device/not-owner` | User is not device owner | Check permissions |
| `pairing/invalid-code` | Invalid pairing code | Generate new code |
| `pairing/expired` | Pairing code expired | Generate new code |
| `pairing/already-used` | Code already used | Generate new code |
| `command/invalid-type` | Unknown command type | Check command type |
| `command/failed` | Command execution failed | Retry |
| `subscription/not-active` | No active subscription | Prompt to subscribe |
| `subscription/limit-reached` | Device limit reached | Upgrade plan |
| `rate-limit/exceeded` | Too many requests | Wait and retry |

---

## Version Compatibility

| API Version | Launcher Version | DPC Version | Parent App |
|-------------|------------------|-------------|------------|
| v1 | 1.0.0+ | 1.0.0+ | 1.0.0+ |

All models include a `_version` field for future migrations.
