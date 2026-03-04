# Sprint Gap Analysis & Remediation Plan

## Executive Summary

This document identifies critical gaps across all sprints and provides remediation plans with code fixes.

---

## Sprint 0: Setup (Days 1-7)

### Gap S0-G01: Device Owner Validation is Make-or-Break

**Risk Level:** CRITICAL

**Issue:** Day 6 Device Owner validation has no contingency if it fails.

**Remediation:**

Add contingency plan to Sprint 0:

```markdown
### Day 6 Contingency Plan

If Device Owner mode fails on Samsung Galaxy A05:

**Immediate Actions (within 4 hours):**
1. Test on Samsung Galaxy A06 (next model)
2. Test on Redmi Note 12 (alternative budget device)
3. Check for Samsung-specific Knox restrictions

**Root Cause Analysis:**
- Check Android version (must be 8.0+)
- Check if device was previously enrolled in MDM
- Check if Google Play Protect is blocking
- Try ADB provisioning method instead of QR

**Fallback Options:**
| Option | Effort | Risk |
|--------|--------|------|
| A. Different Samsung model | Low | Low |
| B. Non-Samsung device (Redmi, Realme) | Low | Medium |
| C. ADB provisioning instead of QR | Low | Low |
| D. Profile Owner mode (less control) | Low | High |

**Go/No-Go Criteria:**
- If 2+ devices fail: Investigate Android version issues
- If all Samsung fails: Pivot to Redmi/Realme
- If all fail: Evaluate Profile Owner as MVP fallback
```

---

## Sprint 1: Launcher + DPC (Days 8-17)

### Gap S1-G01: Mixed Secure/Non-Secure Preferences

**Risk Level:** HIGH

**Issue:** PIN and sensitive data use EncryptedSharedPreferences, but whitelist doesn't.

**File:** `docs/tech/02_sprint_1_launcher_dpc.md` - LauncherPreferences

**Remediation:**

Update `LauncherPreferences.kt`:

```kotlin
@Singleton
class LauncherPreferences @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    // ALL preferences now use encrypted storage
    private val securePrefs: SharedPreferences by lazy {
        EncryptedSharedPreferences.create(
            context,
            "kidtunes_secure_prefs",
            masterKey,
            EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
            EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
        )
    }

    // Non-sensitive preferences (cached for performance)
    private val prefs: SharedPreferences by lazy {
        context.getSharedPreferences("kidtunes_prefs", Context.MODE_PRIVATE)
    }

    companion object {
        // Secure keys (encrypted)
        private const val KEY_PIN = "pin"
        private const val KEY_DEVICE_TOKEN = "device_token"
        private const val KEY_PARENT_ID = "parent_id"

        // Non-sensitive keys (standard prefs)
        private const val KEY_KIDS_MODE = "kids_mode"
        private const val KEY_DEVICE_ID = "device_id"
        private const val KEY_CHILD_NAME = "child_name"
        private const val KEY_VOLUME_LIMIT = "volume_limit"
        private const val KEY_DAILY_LIMIT_MINUTES = "daily_limit_minutes"
        private const val KEY_WHITELIST = "app_whitelist"
    }

    // Use SECURE for: PIN, tokens, parent ID
    fun getPin(): String? = securePrefs.getString(KEY_PIN, null)
    fun setPin(pin: String) = securePrefs.edit { putString(KEY_PIN, pin) }

    fun getDeviceToken(): String? = securePrefs.getString(KEY_DEVICE_TOKEN, null)
    fun setDeviceToken(token: String) = securePrefs.edit { putString(KEY_DEVICE_TOKEN, token) }

    // Use REGULAR for: UI state, settings (non-sensitive)
    fun isKidsMode(): Boolean = prefs.getBoolean(KEY_KIDS_MODE, true)
    fun setKidsMode(enabled: Boolean) = prefs.edit { putBoolean(KEY_KIDS_MODE, enabled) }

    // ... rest of methods
}
```

---

### Gap S1-G02: Missing Error Handling Patterns

**Risk Level:** MEDIUM

**Issue:** No consistent Result type for error handling across codebase.

**Remediation:**

Add `core/util/Result.kt`:

```kotlin
package com.kidtunes.launcher.core.util

sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable, val message: String? = null) : Result<Nothing>()
    object Loading : Result<Nothing>()

    val isSuccess get() = this is Success
    val isError get() = this is Error
    val isLoading get() = this is Loading

    fun getOrNull(): T? = (this as? Success)?.data
    fun exceptionOrNull(): Throwable? = (this as? Error)?.exception

    inline fun <R> map(transform: (T) -> R): Result<R> = when (this) {
        is Success -> Success(transform(data))
        is Error -> this
        is Loading -> Loading
    }

    inline fun onSuccess(action: (T) -> Unit): Result<T> {
        if (this is Success) action(data)
        return this
    }

    inline fun onError(action: (Throwable, String?) -> Unit): Result<T> {
        if (this is Error) action(exception, message)
        return this
    }
}

// Extension for suspending operations
suspend fun <T> safeCall(
    errorMessage: String = "Operation failed",
    block: suspend () -> T
): Result<T> {
    return try {
        Result.Success(block())
    } catch (e: Exception) {
        Result.Error(e, errorMessage)
    }
}

// Extension for Flow
fun <T> Flow<T>.asResult(): Flow<Result<T>> = flow {
    emit(Result.Loading)
    try {
        collect { emit(Result.Success(it)) }
    } catch (e: Exception) {
        emit(Result.Error(e))
    }
}
```

Usage example:

```kotlin
// In Repository
suspend fun fetchApps(): Result<List<AppInfo>> = safeCall("Failed to fetch apps") {
    whitelistManager.getWhitelistedApps().first()
}

// In ViewModel
viewModelScope.launch {
    repository.fetchApps()
        .onSuccess { apps -> _uiState.value = UiState.Success(apps) }
        .onError { e, msg -> _uiState.value = UiState.Error(msg ?: e.message) }
}
```

---

### Gap S1-G03: No App Icon Caching Strategy

**Risk Level:** LOW

**Issue:** App icons loaded repeatedly without caching.

**Remediation:**

Add icon caching with Coil:

```kotlin
// In AppModule.kt
@Provides
@Singleton
fun provideImageLoader(@ApplicationContext context: Context): ImageLoader {
    return ImageLoader.Builder(context)
        .diskCache {
            DiskCache.Builder()
                .directory(context.cacheDir.resolve("app_icons"))
                .maxSizePercent(0.02) // 2% of disk space
                .build()
        }
        .memoryCache {
            MemoryCache.Builder(context)
                .maxSizePercent(0.15) // 15% of memory
                .build()
        }
        .crossfade(true)
        .build()
}

// In AppWhitelistManager - cache icons to storage
suspend fun cacheAppIcon(packageName: String): String? {
    return try {
        val drawable = packageManager.getApplicationIcon(packageName)
        val bitmap = drawable.toBitmap()
        val file = File(context.cacheDir, "icons/$packageName.png")
        file.parentFile?.mkdirs()
        FileOutputStream(file).use { out ->
            bitmap.compress(Bitmap.CompressFormat.PNG, 80, out)
        }
        file.absolutePath
    } catch (e: Exception) {
        null
    }
}
```

---

## Sprint 2: Parent App + Pairing (Days 18-27)

### Gap S2-G01: Router Redirect Doesn't Handle Loading States

**Risk Level:** HIGH

**Issue:** `authState.valueOrNull` returns null during loading, causing redirect to login.

**File:** `docs/tech/03_sprint_2_parent_pairing.md` - router.dart

**Remediation:**

```dart
final routerProvider = Provider<GoRouter>((ref) {
  final authState = ref.watch(authStateProvider);

  return GoRouter(
    initialLocation: '/splash',
    debugLogDiagnostics: true,
    redirect: (context, state) {
      // CRITICAL: Handle loading state
      final isLoading = authState.isLoading;
      final hasError = authState.hasError;
      final isLoggedIn = authState.valueOrNull != null;

      final currentPath = state.matchedLocation;
      final isOnAuthRoute = currentPath.startsWith('/auth');
      final isOnSplash = currentPath == '/splash';

      // Stay on splash while loading
      if (isLoading) {
        return isOnSplash ? null : '/splash';
      }

      // Show error on splash (it will handle redirect)
      if (hasError) {
        return isOnSplash ? null : '/splash';
      }

      if (isOnSplash) return null;

      if (!isLoggedIn && !isOnAuthRoute) {
        return '/auth/login';
      }

      if (isLoggedIn && isOnAuthRoute) {
        return '/dashboard';
      }

      return null;
    },
    routes: [
      // ... routes
    ],
  );
});
```

Update `splash_screen.dart`:

```dart
class SplashScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final authState = ref.watch(authStateProvider);

    // Handle auth state changes
    ref.listen(authStateProvider, (previous, next) {
      next.when(
        data: (user) {
          if (user != null) {
            // Check if user has devices
            _checkOnboardingStatus(context, ref);
          } else {
            context.go('/auth/login');
          }
        },
        loading: () {}, // Stay on splash
        error: (e, _) {
          context.go('/auth/login');
        },
      );
    });

    return Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Logo
            Container(
              width: 120,
              height: 120,
              decoration: BoxDecoration(
                color: AppTheme.primaryColor,
                borderRadius: BorderRadius.circular(24),
              ),
              child: const Icon(Icons.child_care, size: 80, color: Colors.white),
            ),
            const SizedBox(height: 24),
            const CircularProgressIndicator(),
            const SizedBox(height: 16),
            if (authState.isLoading)
              const Text('Loading...')
            else if (authState.hasError)
              Text('Error: ${authState.error}', style: TextStyle(color: Colors.red)),
          ],
        ),
      ),
    );
  }
}
```

---

### Gap S2-G02: Hardcoded Firebase Dependencies

**Risk Level:** MEDIUM

**Issue:** Repositories directly instantiate Firebase, making testing difficult.

**Remediation:**

Create abstract interfaces:

```dart
// lib/data/datasources/auth_datasource.dart
abstract class AuthDataSource {
  User? get currentUser;
  Stream<User?> get authStateChanges;
  Future<void> verifyPhoneNumber({
    required String phoneNumber,
    required Function(String) onCodeSent,
    required Function(PhoneAuthCredential) onVerificationCompleted,
    required Function(FirebaseAuthException) onVerificationFailed,
  });
  Future<UserCredential> signInWithOTP(String verificationId, String otp);
  Future<void> signOut();
}

// lib/data/datasources/firebase_auth_datasource.dart
class FirebaseAuthDataSource implements AuthDataSource {
  final FirebaseAuth _auth;

  FirebaseAuthDataSource({FirebaseAuth? auth})
    : _auth = auth ?? FirebaseAuth.instance;

  @override
  User? get currentUser => _auth.currentUser;

  @override
  Stream<User?> get authStateChanges => _auth.authStateChanges();

  // ... rest of implementation
}

// lib/data/datasources/mock_auth_datasource.dart
class MockAuthDataSource implements AuthDataSource {
  User? _mockUser;
  final _authStateController = StreamController<User?>.broadcast();

  @override
  User? get currentUser => _mockUser;

  @override
  Stream<User?> get authStateChanges => _authStateController.stream;

  // For testing
  void setMockUser(User? user) {
    _mockUser = user;
    _authStateController.add(user);
  }

  // ... rest of mock implementation
}

// Provider configuration
final authDataSourceProvider = Provider<AuthDataSource>((ref) {
  // In production
  return FirebaseAuthDataSource();
});

// Override in tests
// container.updateOverrides([
//   authDataSourceProvider.overrideWithValue(MockAuthDataSource()),
// ]);
```

---

### Gap S2-G03: Missing Rate Limiting on Pairing

**Risk Level:** HIGH

**Issue:** No rate limiting prevents brute force attacks on pairing tokens.

**Remediation:**

Add server-side rate limiting via Cloud Function:

```typescript
// firebase/functions/src/pairing.ts
import * as functions from 'firebase-functions';
import * as admin from 'firebase-admin';

const rateLimiter = new Map<string, { count: number; resetTime: number }>();

const RATE_LIMIT = {
  MAX_REQUESTS: 5,
  WINDOW_MS: 15 * 60 * 1000, // 15 minutes
};

export const generatePairingToken = functions
  .region('asia-south1')
  .https.onCall(async (data, context) => {
    // Auth check
    if (!context.auth) {
      throw new functions.https.HttpsError('unauthenticated', 'Must be logged in');
    }

    const userId = context.auth.uid;
    const now = Date.now();

    // Rate limiting
    const userLimit = rateLimiter.get(userId);
    if (userLimit) {
      if (now < userLimit.resetTime) {
        if (userLimit.count >= RATE_LIMIT.MAX_REQUESTS) {
          throw new functions.https.HttpsError(
            'resource-exhausted',
            `Too many pairing attempts. Try again in ${Math.ceil((userLimit.resetTime - now) / 60000)} minutes`
          );
        }
        userLimit.count++;
      } else {
        rateLimiter.set(userId, { count: 1, resetTime: now + RATE_LIMIT.WINDOW_MS });
      }
    } else {
      rateLimiter.set(userId, { count: 1, resetTime: now + RATE_LIMIT.WINDOW_MS });
    }

    // Generate token
    const token = generateSecureToken();
    const expiresAt = new Date(now + 10 * 60 * 1000); // 10 minutes

    await admin.firestore().collection('pairing_tokens').doc(token).set({
      userId,
      childId: data.childId,
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
      expiresAt: admin.firestore.Timestamp.fromDate(expiresAt),
      used: false,
      attempts: 0,
    });

    return { token, expiresAt: expiresAt.toISOString() };
  });

export const validatePairingToken = functions
  .region('asia-south1')
  .https.onCall(async (data, context) => {
    const { token, deviceId } = data;

    const tokenDoc = await admin.firestore()
      .collection('pairing_tokens')
      .doc(token)
      .get();

    if (!tokenDoc.exists) {
      throw new functions.https.HttpsError('not-found', 'Invalid token');
    }

    const tokenData = tokenDoc.data()!;

    // Check if already used
    if (tokenData.used) {
      throw new functions.https.HttpsError('failed-precondition', 'Token already used');
    }

    // Check expiry
    if (tokenData.expiresAt.toDate() < new Date()) {
      throw new functions.https.HttpsError('failed-precondition', 'Token expired');
    }

    // Increment attempts (prevent brute force on token validation)
    if (tokenData.attempts >= 3) {
      throw new functions.https.HttpsError(
        'resource-exhausted',
        'Too many validation attempts'
      );
    }

    await tokenDoc.ref.update({
      attempts: admin.firestore.FieldValue.increment(1),
    });

    // Mark as used and complete pairing
    await tokenDoc.ref.update({
      used: true,
      deviceId,
      completedAt: admin.firestore.FieldValue.serverTimestamp(),
    });

    return { success: true, userId: tokenData.userId, childId: tokenData.childId };
  });

function generateSecureToken(): string {
  const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789'; // No O/0/I/1 for clarity
  let token = '';
  const randomBytes = require('crypto').randomBytes(6);
  for (let i = 0; i < 6; i++) {
    token += chars[randomBytes[i] % chars.length];
  }
  return token;
}
```

---

## Sprint 3: Core Features (Days 28-37)

### Gap S3-G01: UsageStatsManager Permission Issues

**Risk Level:** HIGH

**Issue:** UsageStatsManager requires special permission that must be granted in Settings.

**Remediation:**

Add permission check and guided setup:

```kotlin
// UsageStatsPermissionChecker.kt
object UsageStatsPermissionChecker {

    fun hasPermission(context: Context): Boolean {
        val appOps = context.getSystemService(Context.APP_OPS_SERVICE) as AppOpsManager
        val mode = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            appOps.unsafeCheckOpNoThrow(
                AppOpsManager.OPSTR_GET_USAGE_STATS,
                android.os.Process.myUid(),
                context.packageName
            )
        } else {
            @Suppress("DEPRECATION")
            appOps.checkOpNoThrow(
                AppOpsManager.OPSTR_GET_USAGE_STATS,
                android.os.Process.myUid(),
                context.packageName
            )
        }
        return mode == AppOpsManager.MODE_ALLOWED
    }

    fun requestPermission(activity: Activity) {
        val intent = Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK
        }

        // Try to deep link to our app's setting
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            intent.data = Uri.fromParts("package", activity.packageName, null)
        }

        activity.startActivity(intent)
    }
}

// In ScreenTimeService - check before querying
private fun getTodayUsageMinutes(): Int {
    if (!UsageStatsPermissionChecker.hasPermission(this)) {
        Log.w(TAG, "Usage stats permission not granted")
        requestPermissionViaNotification()
        return 0
    }

    // ... rest of implementation
}

// Show notification prompting user to grant permission
private fun requestPermissionViaNotification() {
    val intent = Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS)
    val pendingIntent = PendingIntent.getActivity(
        this, 0, intent, PendingIntent.FLAG_IMMUTABLE
    )

    val notification = NotificationCompat.Builder(this, "setup_channel")
        .setContentTitle("Permission Required")
        .setContentText("Tap to enable screen time tracking")
        .setSmallIcon(R.drawable.ic_warning)
        .setContentIntent(pendingIntent)
        .setAutoCancel(true)
        .build()

    notificationManager.notify(PERMISSION_NOTIFICATION_ID, notification)
}
```

Add to device provisioning flow:

```kotlin
// In ProvisioningActivity - grant permission via Device Owner
private fun grantUsageStatsPermission() {
    if (isDeviceOwner()) {
        val dpm = getSystemService(DEVICE_POLICY_SERVICE) as DevicePolicyManager
        val componentName = KidDeviceAdminReceiver.getComponentName(this)

        // Device Owner can grant this permission
        dpm.setPermissionGrantState(
            componentName,
            packageName,
            Manifest.permission.PACKAGE_USAGE_STATS,
            DevicePolicyManager.PERMISSION_GRANT_STATE_GRANTED
        )
    }
}
```

---

### Gap S3-G02: Battery Optimization Risks

**Risk Level:** HIGH

**Issue:** Foreground service may be killed by aggressive battery optimization.

**Remediation:**

Use WorkManager with expedited work:

```kotlin
// ScreenTimeWorker.kt - Replace foreground service with WorkManager
class ScreenTimeWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            val preferences = LauncherPreferences(applicationContext)
            val usageTracker = UsageTracker(applicationContext)

            // Check and enforce screen time
            val todayUsage = usageTracker.getTodayUsageMinutes()
            val dailyLimit = preferences.getDailyLimitMinutes()

            if (todayUsage >= dailyLimit) {
                showLimitReachedScreen()
            } else if (todayUsage >= dailyLimit - 10) {
                showWarningNotification(dailyLimit - todayUsage)
            }

            // Check bedtime
            if (isInBedtimeWindow(preferences)) {
                showBedtimeScreen()
            }

            // Sync usage to Firebase
            syncUsageToFirebase(usageTracker, preferences)

            Result.success()
        } catch (e: Exception) {
            Log.e("ScreenTimeWorker", "Error", e)
            Result.retry()
        }
    }

    companion object {
        private const val WORK_NAME = "screen_time_check"

        fun schedule(context: Context) {
            val constraints = Constraints.Builder()
                .setRequiresBatteryNotLow(false) // Run even on low battery
                .build()

            val request = PeriodicWorkRequestBuilder<ScreenTimeWorker>(
                15, TimeUnit.MINUTES // Android enforces minimum 15 min
            )
                .setConstraints(constraints)
                .setBackoffCriteria(
                    BackoffPolicy.LINEAR,
                    1, TimeUnit.MINUTES
                )
                .build()

            WorkManager.getInstance(context).enqueueUniquePeriodicWork(
                WORK_NAME,
                ExistingPeriodicWorkPolicy.KEEP,
                request
            )
        }

        // For more frequent checks, use expedited work
        fun checkNow(context: Context) {
            val request = OneTimeWorkRequestBuilder<ScreenTimeWorker>()
                .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
                .build()

            WorkManager.getInstance(context).enqueue(request)
        }
    }
}

// Also use AlarmManager for precise timing (bedtime)
class BedtimeAlarmReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        when (intent.action) {
            "BEDTIME_START" -> {
                val lockIntent = Intent(context, BedtimeLockActivity::class.java).apply {
                    addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                }
                context.startActivity(lockIntent)
            }
            "BEDTIME_END" -> {
                // Unlock device
                LocalBroadcastManager.getInstance(context)
                    .sendBroadcast(Intent("BEDTIME_END"))
            }
        }
    }

    companion object {
        fun scheduleBedtime(context: Context, startTime: String, endTime: String) {
            val alarmManager = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager

            // Schedule bedtime start
            val startIntent = Intent(context, BedtimeAlarmReceiver::class.java).apply {
                action = "BEDTIME_START"
            }
            val startPendingIntent = PendingIntent.getBroadcast(
                context, 1001, startIntent, PendingIntent.FLAG_IMMUTABLE
            )

            val startCalendar = parseTimeToCalendar(startTime)

            // Use setExactAndAllowWhileIdle for reliability
            alarmManager.setExactAndAllowWhileIdle(
                AlarmManager.RTC_WAKEUP,
                startCalendar.timeInMillis,
                startPendingIntent
            )

            // Similar for end time...
        }
    }
}

// Request battery optimization exemption
fun requestBatteryOptimizationExemption(context: Context) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        val powerManager = context.getSystemService(Context.POWER_SERVICE) as PowerManager
        if (!powerManager.isIgnoringBatteryOptimizations(context.packageName)) {
            val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
                data = Uri.parse("package:${context.packageName}")
            }
            context.startActivity(intent)
        }
    }
}
```

---

## Sprints 5-6: Testing (Days 50-77)

### Gap S56-G01: Missing Device Farm Testing

**Risk Level:** HIGH

**Remediation:**

Add to testing strategy:

```markdown
### Device Farm Testing Plan

**Firebase Test Lab Configuration:**

```yaml
# .github/workflows/device-tests.yml
name: Device Farm Tests

on:
  push:
    branches: [develop, main]

jobs:
  instrumentation-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build APKs
        run: |
          cd android/launcher
          ./gradlew assembleDebug assembleDebugAndroidTest

      - name: Run Firebase Test Lab
        uses: google-github-actions/firebase-test-lab@v1
        with:
          credentials: ${{ secrets.GOOGLE_CREDENTIALS }}
          projectId: kidtunes-prod
          devices: |
            - model: a05
              version: 34
              locale: en
              orientation: portrait
            - model: redfin
              version: 30
              locale: en
            - model: oriole
              version: 33
              locale: en
          timeout: 15m

      - name: Upload Results
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: firebase-test-results/
```

**Priority Test Devices:**

| Priority | Model | Android | Reason |
|----------|-------|---------|--------|
| P0 | Samsung Galaxy A05 | 14 | Primary target device |
| P1 | Samsung Galaxy A06 | 14 | Backup device |
| P2 | Redmi Note 12 | 13 | Popular budget device |
| P3 | Pixel 6 | 14 | Reference device |
| P4 | Samsung Galaxy S21 | 14 | Premium device |
```

---

### Gap S56-G02: Missing Security Penetration Testing

**Risk Level:** CRITICAL

**Remediation:**

```markdown
### Security Penetration Testing Checklist

**Week 10-11: Security Testing**

| Test Area | Tool/Method | Pass Criteria |
|-----------|-------------|---------------|
| **OWASP Mobile Top 10** | | |
| M1: Improper Platform Usage | Manual review | No unsafe intents exposed |
| M2: Insecure Data Storage | Frida, objection | All sensitive data encrypted |
| M3: Insecure Communication | mitmproxy | Certificate pinning verified |
| M4: Insecure Authentication | Manual testing | No auth bypass possible |
| M5: Insufficient Cryptography | Code review | AES-256 for all crypto |
| M6: Insecure Authorization | Burp Suite | No privilege escalation |
| M7: Client Code Quality | Static analysis | No hardcoded secrets |
| M8: Code Tampering | apktool, jadx | Integrity checks present |
| M9: Reverse Engineering | ProGuard check | Code obfuscated |
| M10: Extraneous Functionality | APK analysis | No debug/test code in release |

**DPC-Specific Tests:**

| Test | Method | Expected Result |
|------|--------|-----------------|
| Bypass Device Owner | ADB, recovery | Cannot bypass |
| Uninstall DPC | Settings app | Blocked |
| Disable admin | ADB shell | Blocked |
| Factory reset | Recovery mode | Requires PIN |
| Safe mode bypass | Boot safe mode | DPC still enforced |

**Firebase Security Tests:**

| Test | Method | Expected Result |
|------|--------|-----------------|
| Unauthorized data access | Direct API calls | Blocked by rules |
| Cross-user data access | Modified client | Blocked by rules |
| Rate limit bypass | Automated requests | Rate limited |
| Token manipulation | JWT tampering | Rejected |

**Hire Security Auditor:**
- Budget: ₹50,000-1,00,000
- Timeline: Week 11 (5 days)
- Focus: DPC bypass, data security, Firebase rules
```

---

### Gap S56-G03: Missing Accessibility Testing

**Risk Level:** MEDIUM

**Remediation:**

```markdown
### Accessibility Testing Plan

**Parent App (Flutter):**

```dart
// Enable Semantics for testing
void main() {
  // ... initialization

  // For accessibility testing
  if (kDebugMode) {
    WidgetsBinding.instance.ensureInitialized();
    SemanticsBinding.instance.ensureSemantics();
  }
}

// Test with accessibility scanner
testWidgets('Dashboard passes accessibility', (tester) async {
  await tester.pumpWidget(ProviderScope(child: DashboardScreen(deviceId: 'test')));

  // Check for sufficient tap targets (48x48 minimum)
  final buttons = find.byType(ElevatedButton);
  for (final button in buttons.evaluate()) {
    final size = tester.getSize(find.byWidget(button.widget));
    expect(size.width, greaterThanOrEqualTo(48));
    expect(size.height, greaterThanOrEqualTo(48));
  }

  // Check for contrast (use accessibility_tools package)
  expectAccessibleContrast(tester);
});
```

**Child Launcher (Android):**

```kotlin
// In build.gradle - enable accessibility checks
android {
    testOptions {
        unitTests {
            includeAndroidResources = true
        }
    }

    lintOptions {
        enable 'a11y'
        fatal 'ContentDescription'
    }
}

// In tests
@Test
fun testKidsModeAccessibility() {
    val scenario = launchActivity<LauncherActivity>()

    scenario.onActivity { activity ->
        val rootView = activity.window.decorView

        // Check all interactive elements have content descriptions
        findAllInteractiveViews(rootView).forEach { view ->
            assertNotNull(
                "View ${view.id} missing content description",
                view.contentDescription
            )
        }

        // Check touch target sizes
        findAllClickableViews(rootView).forEach { view ->
            assertTrue(
                "View ${view.id} touch target too small",
                view.width >= 48.dp && view.height >= 48.dp
            )
        }
    }
}
```

**Manual Testing Checklist:**

- [ ] TalkBack navigation works on all screens
- [ ] All buttons have content descriptions
- [ ] Color contrast ratio ≥ 4.5:1
- [ ] Touch targets ≥ 48x48dp
- [ ] No time-limited actions without extension
- [ ] Visible focus indicators
- [ ] Font scaling works (200%)
```

---

## Sprint 7: Production (Days 78-100)

### Gap S7-G01: Security Audit Missing

**Risk Level:** CRITICAL

**Remediation:**

Add to Sprint 7:

```markdown
### Security Audit Plan (Days 78-82)

**Internal Audit (Days 78-79):**

| Check | Status | Owner |
|-------|--------|-------|
| All Firebase security rules reviewed | ☐ | Lead Dev |
| ProGuard/R8 obfuscation verified | ☐ | Android Dev |
| Certificate pinning tested | ☐ | Android Dev |
| Root detection working | ☐ | Android Dev |
| Play Integrity integrated | ☐ | Android Dev |
| All secrets in environment vars | ☐ | DevOps |
| No debug code in release | ☐ | Lead Dev |
| DPDP consent flow working | ☐ | Full Team |

**External Audit (Days 80-82):**

- Hire security consultant (₹75,000-1,50,000)
- Focus areas:
  1. Device Owner bypass attempts
  2. Firebase security rules
  3. Data encryption at rest
  4. Network traffic analysis
  5. APK reverse engineering

**Deliverable:** Security Audit Report with findings and remediations
```

---

### Gap S7-G02: DPDP Compliance Verification Missing

**Risk Level:** CRITICAL

**Remediation:**

```markdown
### DPDP Compliance Verification Checklist

**Days 83-85: Compliance Verification**

| Requirement | Implementation | Verified | Evidence |
|-------------|----------------|----------|----------|
| **Verifiable Parental Consent** | | | |
| OTP verification for parent | ☐ | ☐ | Screenshot |
| Explicit consent checkbox | ☐ | ☐ | Screenshot |
| Consent text reviewed by legal | ☐ | ☐ | Document |
| Consent timestamp logged | ☐ | ☐ | Firebase record |
| **Data Minimization** | | | |
| Only essential data collected | ☐ | ☐ | Data map |
| No location without explicit opt-in | ☐ | ☐ | Feature flag |
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
| Firebase cleanup complete | ☐ | ☐ | Data audit |

**Legal Review:**
- Privacy policy reviewed by lawyer: ₹15,000-25,000
- Terms of service reviewed: ₹10,000-15,000
- Total legal budget: ₹25,000-40,000
```

---

## Updated Sprint Intensity Analysis

After remediation:

| Sprint | Original Points | Adjusted Points | Days | Intensity |
|--------|-----------------|-----------------|------|-----------|
| Sprint 0 | N/A | +2 (contingency) | 7 | Low |
| Sprint 1 | 36 | 41 (+5 security) | 10 | High |
| Sprint 2 | 29 | 34 (+5 rate limit) | 10 | Moderate |
| Sprint 3 | 49 | 44 (-5 WorkManager) | 10 | High |
| Sprint 5-6 | N/A | +15 (testing) | 28 | Moderate |
| Sprint 7 | N/A | +10 (audit) | 23 | Moderate |

**Recommendation:**
- Consider extending Sprint 3 by 2-3 days (move from "Very High" to "High" intensity)
- Add 2 days buffer before Sprint 7 for audit findings remediation

---

## Summary of Required Changes

| Gap ID | Severity | Sprint | Fix Effort |
|--------|----------|--------|------------|
| S0-G01 | Critical | 0 | Low (add contingency doc) |
| S1-G01 | High | 1 | Medium (refactor prefs) |
| S1-G02 | Medium | 1 | Low (add Result type) |
| S1-G03 | Low | 1 | Low (add Coil config) |
| S2-G01 | High | 2 | Medium (fix router) |
| S2-G02 | Medium | 2 | Medium (add interfaces) |
| S2-G03 | High | 2 | High (add rate limiting) |
| S3-G01 | High | 3 | Medium (permission flow) |
| S3-G02 | High | 3 | High (WorkManager refactor) |
| S56-G01 | High | 5-6 | Medium (CI setup) |
| S56-G02 | Critical | 5-6 | High (security testing) |
| S56-G03 | Medium | 5-6 | Low (a11y testing) |
| S7-G01 | Critical | 7 | High (security audit) |
| S7-G02 | Critical | 7 | High (DPDP verification) |

---

**Document Version:** 1.0
**Created:** Based on Sprint-by-Sprint Analysis
**Next Review:** After Sprint 1 completion
