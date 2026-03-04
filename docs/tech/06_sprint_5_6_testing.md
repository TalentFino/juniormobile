# Sprint 5 & 6: Testing, Bug Fixes & Polish

## Sprint Overview

| Sprint | Days | Focus | Goal |
|--------|------|-------|------|
| Sprint 5 | 71-84 | Internal Testing & Bug Fixes | All features working, <5 P1 bugs |
| Sprint 6 | 85-98 | Beta Testing & Polish | 20 beta families, NPS > 40 |

> **Note**: Timeline updated to follow Sprint 4 (Days 50-70) which now includes advanced parental controls.

---

## Sprint 5: Internal Testing & Bug Fixes (Days 71-84)

### Epic 5.1: Unit Testing Setup

**Story 5.1.1: Android Unit Testing Infrastructure**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 5.1.1.1 | Configure JUnit 5 for launcher project | `./gradlew test` runs successfully |
| 5.1.1.2 | Configure MockK for mocking | MockK available in test scope |
| 5.1.1.3 | Configure Robolectric for Android tests | Can test Android components without device |
| 5.1.1.4 | Set up code coverage with JaCoCo | Coverage report generates |

**Implementation: build.gradle.kts (Module: launcher)**

```kotlin
// app/build.gradle.kts

plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("jacoco")
}

android {
    // ... existing config

    testOptions {
        unitTests {
            isIncludeAndroidResources = true
            isReturnDefaultValues = true
        }
    }

    buildTypes {
        debug {
            enableUnitTestCoverage = true
            enableAndroidTestCoverage = true
        }
    }
}

dependencies {
    // Testing
    testImplementation("junit:junit:4.13.2")
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.1")
    testImplementation("io.mockk:mockk:1.13.9")
    testImplementation("org.robolectric:robolectric:4.11.1")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
    testImplementation("app.cash.turbine:turbine:1.0.0")
    testImplementation("com.google.truth:truth:1.2.0")

    // Android Instrumented Tests
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
    androidTestImplementation("androidx.test:runner:1.5.2")
    androidTestImplementation("androidx.test:rules:1.5.0")
    androidTestImplementation("io.mockk:mockk-android:1.13.9")
}

tasks.withType<Test> {
    useJUnitPlatform()
}

// JaCoCo configuration
jacoco {
    toolVersion = "0.8.11"
}

tasks.register<JacocoReport>("jacocoTestReport") {
    dependsOn("testDebugUnitTest")

    reports {
        xml.required.set(true)
        html.required.set(true)
    }

    val fileFilter = listOf(
        "**/R.class",
        "**/R\$*.class",
        "**/BuildConfig.*",
        "**/Manifest*.*",
        "**/*Test*.*"
    )

    val debugTree = fileTree("${buildDir}/tmp/kotlin-classes/debug") {
        exclude(fileFilter)
    }

    sourceDirectories.setFrom(files("src/main/java", "src/main/kotlin"))
    classDirectories.setFrom(files(debugTree))
    executionData.setFrom(files("${buildDir}/jacoco/testDebugUnitTest.exec"))
}
```

**Story 5.1.2: Flutter Unit Testing Infrastructure**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 5.1.2.1 | Configure flutter_test | `flutter test` runs successfully |
| 5.1.2.2 | Configure mockito for mocking | Can mock dependencies |
| 5.1.2.3 | Configure bloc_test for Riverpod testing | Can test providers |
| 5.1.2.4 | Set up code coverage | `flutter test --coverage` works |

**Implementation: pubspec.yaml (dev_dependencies)**

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.1

  # Testing
  mockito: ^5.4.4
  build_runner: ^2.4.8
  mocktail: ^1.0.3

  # State management testing
  riverpod_test: ^0.1.0

  # Golden tests
  golden_toolkit: ^0.15.0

  # Integration testing
  integration_test:
    sdk: flutter

  # Coverage
  test_coverage: ^0.0.5
```

**Implementation: test/test_helper.dart**

```dart
// test/test_helper.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:parent_app/core/services/auth_service.dart';
import 'package:parent_app/core/services/device_service.dart';
import 'package:parent_app/core/services/command_service.dart';

// Mock classes
class MockAuthService extends Mock implements AuthService {}
class MockDeviceService extends Mock implements DeviceService {}
class MockCommandService extends Mock implements CommandService {}

// Test widget wrapper
Widget createTestWidget({
  required Widget child,
  List<Override>? overrides,
}) {
  return ProviderScope(
    overrides: overrides ?? [],
    child: MaterialApp(
      home: child,
    ),
  );
}

// Common test setups
void setupTestDependencies() {
  // Register fallback values for mocktail
  registerFallbackValue(Uri());
}

// Provider container for unit tests
ProviderContainer createContainer({
  List<Override>? overrides,
}) {
  return ProviderContainer(
    overrides: overrides ?? [],
  );
}
```

---

### Epic 5.2: Launcher Unit Tests

**Story 5.2.1: AppWhitelistManager Tests**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 5.2.1.1 | Test whitelist loading from SharedPreferences | Load returns saved apps |
| 5.2.1.2 | Test whitelist saving | Save persists to SharedPreferences |
| 5.2.1.3 | Test app filtering | Only whitelisted apps returned |
| 5.2.1.4 | Test default whitelist creation | New install has default apps |

**Implementation: AppWhitelistManagerTest.kt**

```kotlin
// app/src/test/java/com/kidsafe/launcher/manager/AppWhitelistManagerTest.kt

package com.kidsafe.launcher.manager

import android.content.Context
import android.content.SharedPreferences
import android.content.pm.ApplicationInfo
import android.content.pm.PackageManager
import com.google.common.truth.Truth.assertThat
import io.mockk.*
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.DisplayName

class AppWhitelistManagerTest {

    private lateinit var context: Context
    private lateinit var sharedPreferences: SharedPreferences
    private lateinit var editor: SharedPreferences.Editor
    private lateinit var packageManager: PackageManager
    private lateinit var manager: AppWhitelistManager

    @BeforeEach
    fun setup() {
        context = mockk(relaxed = true)
        sharedPreferences = mockk(relaxed = true)
        editor = mockk(relaxed = true)
        packageManager = mockk(relaxed = true)

        every { context.getSharedPreferences(any(), any()) } returns sharedPreferences
        every { sharedPreferences.edit() } returns editor
        every { editor.putStringSet(any(), any()) } returns editor
        every { editor.apply() } just Runs
        every { context.packageManager } returns packageManager

        manager = AppWhitelistManager(context)
    }

    @Test
    @DisplayName("getWhitelistedApps returns saved packages")
    fun testGetWhitelistedApps() = runTest {
        // Given
        val savedApps = setOf(
            "com.spotify.music",
            "com.jio.saavn",
            "com.amazon.mp3"
        )
        every { sharedPreferences.getStringSet("whitelisted_apps", any()) } returns savedApps

        // When
        val result = manager.getWhitelistedApps()

        // Then
        assertThat(result).containsExactlyElementsIn(savedApps)
    }

    @Test
    @DisplayName("addToWhitelist adds package to existing list")
    fun testAddToWhitelist() = runTest {
        // Given
        val existingApps = mutableSetOf("com.spotify.music")
        every { sharedPreferences.getStringSet("whitelisted_apps", any()) } returns existingApps

        // When
        manager.addToWhitelist("com.jio.saavn")

        // Then
        verify {
            editor.putStringSet("whitelisted_apps", match {
                it.contains("com.spotify.music") && it.contains("com.jio.saavn")
            })
        }
    }

    @Test
    @DisplayName("removeFromWhitelist removes package from list")
    fun testRemoveFromWhitelist() = runTest {
        // Given
        val existingApps = mutableSetOf("com.spotify.music", "com.jio.saavn")
        every { sharedPreferences.getStringSet("whitelisted_apps", any()) } returns existingApps

        // When
        manager.removeFromWhitelist("com.jio.saavn")

        // Then
        verify {
            editor.putStringSet("whitelisted_apps", match {
                it.contains("com.spotify.music") && !it.contains("com.jio.saavn")
            })
        }
    }

    @Test
    @DisplayName("isWhitelisted returns true for whitelisted apps")
    fun testIsWhitelisted() = runTest {
        // Given
        val savedApps = setOf("com.spotify.music")
        every { sharedPreferences.getStringSet("whitelisted_apps", any()) } returns savedApps

        // When & Then
        assertThat(manager.isWhitelisted("com.spotify.music")).isTrue()
        assertThat(manager.isWhitelisted("com.unknown.app")).isFalse()
    }

    @Test
    @DisplayName("getWhitelistedInstalledApps filters by whitelist")
    fun testGetWhitelistedInstalledApps() = runTest {
        // Given
        val savedApps = setOf("com.spotify.music", "com.jio.saavn")
        every { sharedPreferences.getStringSet("whitelisted_apps", any()) } returns savedApps

        val installedApps = listOf(
            createMockAppInfo("com.spotify.music", "Spotify"),
            createMockAppInfo("com.jio.saavn", "JioSaavn"),
            createMockAppInfo("com.blocked.app", "BlockedApp")
        )
        every { packageManager.getInstalledApplications(any<Int>()) } returns installedApps

        // When
        val result = manager.getWhitelistedInstalledApps()

        // Then
        assertThat(result.map { it.packageName }).containsExactly(
            "com.spotify.music",
            "com.jio.saavn"
        )
    }

    @Test
    @DisplayName("Default whitelist contains essential apps")
    fun testDefaultWhitelist() {
        // Given
        every { sharedPreferences.getStringSet("whitelisted_apps", any()) } returns null

        // When
        val defaults = manager.getDefaultWhitelist()

        // Then
        assertThat(defaults).contains("com.android.deskclock") // Alarm
        assertThat(defaults).contains("com.android.calculator2") // Calculator
    }

    private fun createMockAppInfo(packageName: String, label: String): ApplicationInfo {
        return ApplicationInfo().apply {
            this.packageName = packageName
        }
    }
}
```

**Story 5.2.2: ModeManager Tests**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 5.2.2.1 | Test mode persistence | Mode survives app restart |
| 5.2.2.2 | Test mode switching logic | Correct mode returned |
| 5.2.2.3 | Test PIN validation | Correct PIN allows switch |
| 5.2.2.4 | Test wrong PIN handling | Wrong PIN blocks switch |

**Implementation: ModeManagerTest.kt**

```kotlin
// app/src/test/java/com/kidsafe/launcher/manager/ModeManagerTest.kt

package com.kidsafe.launcher.manager

import android.content.Context
import android.content.SharedPreferences
import com.google.common.truth.Truth.assertThat
import io.mockk.*
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.DisplayName

class ModeManagerTest {

    private lateinit var context: Context
    private lateinit var sharedPreferences: SharedPreferences
    private lateinit var editor: SharedPreferences.Editor
    private lateinit var manager: ModeManager

    @BeforeEach
    fun setup() {
        context = mockk(relaxed = true)
        sharedPreferences = mockk(relaxed = true)
        editor = mockk(relaxed = true)

        every { context.getSharedPreferences(any(), any()) } returns sharedPreferences
        every { sharedPreferences.edit() } returns editor
        every { editor.putString(any(), any()) } returns editor
        every { editor.putBoolean(any(), any()) } returns editor
        every { editor.apply() } just Runs

        manager = ModeManager(context)
    }

    @Test
    @DisplayName("getCurrentMode returns saved mode")
    fun testGetCurrentMode() {
        // Given - Kids mode saved
        every { sharedPreferences.getBoolean("is_kids_mode", true) } returns true

        // When
        val result = manager.getCurrentMode()

        // Then
        assertThat(result).isEqualTo(DeviceMode.KIDS)
    }

    @Test
    @DisplayName("setMode persists mode to SharedPreferences")
    fun testSetMode() {
        // When
        manager.setMode(DeviceMode.STANDARD)

        // Then
        verify { editor.putBoolean("is_kids_mode", false) }
        verify { editor.apply() }
    }

    @Test
    @DisplayName("validatePin returns true for correct PIN")
    fun testValidatePinCorrect() {
        // Given
        val hashedPin = manager.hashPin("1234")
        every { sharedPreferences.getString("parent_pin", any()) } returns hashedPin

        // When
        val result = manager.validatePin("1234")

        // Then
        assertThat(result).isTrue()
    }

    @Test
    @DisplayName("validatePin returns false for wrong PIN")
    fun testValidatePinWrong() {
        // Given
        val hashedPin = manager.hashPin("1234")
        every { sharedPreferences.getString("parent_pin", any()) } returns hashedPin

        // When
        val result = manager.validatePin("0000")

        // Then
        assertThat(result).isFalse()
    }

    @Test
    @DisplayName("setPin hashes and stores PIN")
    fun testSetPin() {
        // When
        manager.setPin("5678")

        // Then
        verify {
            editor.putString("parent_pin", match {
                it != "5678" && it.isNotEmpty() // Should be hashed
            })
        }
    }

    @Test
    @DisplayName("isKidsMode returns true when in kids mode")
    fun testIsKidsMode() {
        // Given
        every { sharedPreferences.getBoolean("is_kids_mode", true) } returns true

        // When & Then
        assertThat(manager.isKidsMode()).isTrue()
    }

    @Test
    @DisplayName("toggleMode switches between modes with valid PIN")
    fun testToggleMode() = runTest {
        // Given - Currently in kids mode
        every { sharedPreferences.getBoolean("is_kids_mode", true) } returns true
        val hashedPin = manager.hashPin("1234")
        every { sharedPreferences.getString("parent_pin", any()) } returns hashedPin

        // When
        val result = manager.toggleMode("1234")

        // Then
        assertThat(result).isTrue()
        verify { editor.putBoolean("is_kids_mode", false) } // Switched to standard
    }
}

enum class DeviceMode {
    KIDS, STANDARD
}
```

**Story 5.2.3: PolicyManager Tests**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 5.2.3.1 | Test app installation blocking | Install attempts blocked |
| 5.2.3.2 | Test settings restriction | Restricted settings blocked |
| 5.2.3.3 | Test uninstall protection | Cannot uninstall protected apps |
| 5.2.3.4 | Test policy application | Policies apply without crash |

**Implementation: PolicyManagerTest.kt**

```kotlin
// dpc/src/test/java/com/kidsafe/dpc/manager/PolicyManagerTest.kt

package com.kidsafe.dpc.manager

import android.app.admin.DevicePolicyManager
import android.content.ComponentName
import android.content.Context
import android.os.UserManager
import com.google.common.truth.Truth.assertThat
import io.mockk.*
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.DisplayName

class PolicyManagerTest {

    private lateinit var context: Context
    private lateinit var devicePolicyManager: DevicePolicyManager
    private lateinit var adminComponent: ComponentName
    private lateinit var manager: PolicyManager

    @BeforeEach
    fun setup() {
        context = mockk(relaxed = true)
        devicePolicyManager = mockk(relaxed = true)
        adminComponent = mockk(relaxed = true)

        every { context.getSystemService(Context.DEVICE_POLICY_SERVICE) } returns devicePolicyManager

        manager = PolicyManager(context, adminComponent)
    }

    @Test
    @DisplayName("blockAppInstallation sets user restriction")
    fun testBlockAppInstallation() {
        // When
        manager.blockAppInstallation()

        // Then
        verify {
            devicePolicyManager.addUserRestriction(
                adminComponent,
                UserManager.DISALLOW_INSTALL_APPS
            )
        }
    }

    @Test
    @DisplayName("allowAppInstallation removes user restriction")
    fun testAllowAppInstallation() {
        // When
        manager.allowAppInstallation()

        // Then
        verify {
            devicePolicyManager.clearUserRestriction(
                adminComponent,
                UserManager.DISALLOW_INSTALL_APPS
            )
        }
    }

    @Test
    @DisplayName("restrictSettings blocks settings access")
    fun testRestrictSettings() {
        // When
        manager.restrictSettings()

        // Then
        verify {
            devicePolicyManager.addUserRestriction(
                adminComponent,
                UserManager.DISALLOW_CONFIG_WIFI
            )
        }
        verify {
            devicePolicyManager.addUserRestriction(
                adminComponent,
                UserManager.DISALLOW_CONFIG_BLUETOOTH
            )
        }
    }

    @Test
    @DisplayName("setUninstallBlocked prevents app uninstall")
    fun testSetUninstallBlocked() {
        // Given
        val packageName = "com.kidsafe.launcher"

        // When
        manager.setUninstallBlocked(packageName, true)

        // Then
        verify {
            devicePolicyManager.setUninstallBlocked(
                adminComponent,
                packageName,
                true
            )
        }
    }

    @Test
    @DisplayName("isDeviceOwner returns correct status")
    fun testIsDeviceOwner() {
        // Given
        every { devicePolicyManager.isDeviceOwnerApp(any()) } returns true

        // When
        val result = manager.isDeviceOwner()

        // Then
        assertThat(result).isTrue()
    }

    @Test
    @DisplayName("setMaximumVolume enforces volume limit")
    fun testSetMaximumVolume() {
        // When
        manager.setMaximumVolume(70) // 70%

        // Then
        verify {
            devicePolicyManager.setMasterVolumeMuted(adminComponent, false)
        }
        // Note: Volume limiting requires AudioManager, tested in integration
    }

    @Test
    @DisplayName("applyKidsModeRestrictions sets all restrictions")
    fun testApplyKidsModeRestrictions() {
        // When
        manager.applyKidsModeRestrictions()

        // Then
        verify {
            devicePolicyManager.addUserRestriction(
                adminComponent,
                UserManager.DISALLOW_INSTALL_APPS
            )
        }
        verify {
            devicePolicyManager.addUserRestriction(
                adminComponent,
                UserManager.DISALLOW_UNINSTALL_APPS
            )
        }
        verify {
            devicePolicyManager.addUserRestriction(
                adminComponent,
                UserManager.DISALLOW_SAFE_BOOT
            )
        }
    }

    @Test
    @DisplayName("removeKidsModeRestrictions clears all restrictions")
    fun testRemoveKidsModeRestrictions() {
        // When
        manager.removeKidsModeRestrictions()

        // Then
        verify {
            devicePolicyManager.clearUserRestriction(
                adminComponent,
                UserManager.DISALLOW_INSTALL_APPS
            )
        }
        verify {
            devicePolicyManager.clearUserRestriction(
                adminComponent,
                UserManager.DISALLOW_UNINSTALL_APPS
            )
        }
    }
}
```

---

### Epic 5.3: Parent App Unit Tests

**Story 5.3.1: AuthRepository Tests**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 5.3.1.1 | Test phone auth verification | Verification ID returned |
| 5.3.1.2 | Test OTP verification | Sign in success on valid OTP |
| 5.3.1.3 | Test email auth | Sign in success with email/password |
| 5.3.1.4 | Test sign out | User signed out |

**Implementation: auth_repository_test.dart**

```dart
// test/core/repositories/auth_repository_test.dart

import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:parent_app/core/repositories/auth_repository.dart';

class MockFirebaseAuth extends Mock implements FirebaseAuth {}
class MockUser extends Mock implements User {}
class MockUserCredential extends Mock implements UserCredential {}
class MockConfirmationResult extends Mock implements ConfirmationResult {}

void main() {
  late MockFirebaseAuth mockFirebaseAuth;
  late AuthRepository authRepository;

  setUp(() {
    mockFirebaseAuth = MockFirebaseAuth();
    authRepository = AuthRepository(firebaseAuth: mockFirebaseAuth);
  });

  group('AuthRepository', () {
    group('signInWithPhone', () {
      test('sends verification code and returns verification ID', () async {
        // Arrange
        const phoneNumber = '+919876543210';
        const verificationId = 'test-verification-id';

        when(() => mockFirebaseAuth.verifyPhoneNumber(
          phoneNumber: any(named: 'phoneNumber'),
          verificationCompleted: any(named: 'verificationCompleted'),
          verificationFailed: any(named: 'verificationFailed'),
          codeSent: any(named: 'codeSent'),
          codeAutoRetrievalTimeout: any(named: 'codeAutoRetrievalTimeout'),
        )).thenAnswer((invocation) async {
          // Simulate codeSent callback
          final codeSent = invocation.namedArguments[const Symbol('codeSent')]
              as PhoneCodeSent;
          codeSent(verificationId, null);
        });

        // Act
        final result = await authRepository.signInWithPhone(phoneNumber);

        // Assert
        expect(result, equals(verificationId));
        verify(() => mockFirebaseAuth.verifyPhoneNumber(
          phoneNumber: phoneNumber,
          verificationCompleted: any(named: 'verificationCompleted'),
          verificationFailed: any(named: 'verificationFailed'),
          codeSent: any(named: 'codeSent'),
          codeAutoRetrievalTimeout: any(named: 'codeAutoRetrievalTimeout'),
        )).called(1);
      });
    });

    group('verifyOtp', () {
      test('signs in with OTP and returns user', () async {
        // Arrange
        const verificationId = 'test-verification-id';
        const otp = '123456';
        final mockUser = MockUser();
        final mockCredential = MockUserCredential();

        when(() => mockCredential.user).thenReturn(mockUser);
        when(() => mockUser.uid).thenReturn('test-uid');
        when(() => mockFirebaseAuth.signInWithCredential(any()))
            .thenAnswer((_) async => mockCredential);

        // Act
        final result = await authRepository.verifyOtp(verificationId, otp);

        // Assert
        expect(result, isNotNull);
        expect(result!.uid, equals('test-uid'));
      });

      test('returns null on invalid OTP', () async {
        // Arrange
        const verificationId = 'test-verification-id';
        const otp = '000000';

        when(() => mockFirebaseAuth.signInWithCredential(any()))
            .thenThrow(FirebaseAuthException.fromCode('invalid-verification-code'));

        // Act
        final result = await authRepository.verifyOtp(verificationId, otp);

        // Assert
        expect(result, isNull);
      });
    });

    group('signOut', () {
      test('signs out user successfully', () async {
        // Arrange
        when(() => mockFirebaseAuth.signOut()).thenAnswer((_) async {});

        // Act
        await authRepository.signOut();

        // Assert
        verify(() => mockFirebaseAuth.signOut()).called(1);
      });
    });

    group('currentUser', () {
      test('returns current user when logged in', () {
        // Arrange
        final mockUser = MockUser();
        when(() => mockFirebaseAuth.currentUser).thenReturn(mockUser);
        when(() => mockUser.uid).thenReturn('test-uid');

        // Act
        final result = authRepository.currentUser;

        // Assert
        expect(result, isNotNull);
        expect(result!.uid, equals('test-uid'));
      });

      test('returns null when not logged in', () {
        // Arrange
        when(() => mockFirebaseAuth.currentUser).thenReturn(null);

        // Act
        final result = authRepository.currentUser;

        // Assert
        expect(result, isNull);
      });
    });

    group('authStateChanges', () {
      test('emits user on auth state change', () async {
        // Arrange
        final mockUser = MockUser();
        when(() => mockUser.uid).thenReturn('test-uid');
        when(() => mockFirebaseAuth.authStateChanges())
            .thenAnswer((_) => Stream.value(mockUser));

        // Act & Assert
        await expectLater(
          authRepository.authStateChanges,
          emits(isA<User>()),
        );
      });
    });
  });
}

class FirebaseAuthException implements Exception {
  final String code;
  FirebaseAuthException.fromCode(this.code);
}
```

**Story 5.3.2: DeviceRepository Tests**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 5.3.2.1 | Test device creation | Device doc created in Firestore |
| 5.3.2.2 | Test device linking | Device linked to parent |
| 5.3.2.3 | Test device fetch | Returns correct device data |
| 5.3.2.4 | Test device status updates | Status synced correctly |

**Implementation: device_repository_test.dart**

```dart
// test/core/repositories/device_repository_test.dart

import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:parent_app/core/repositories/device_repository.dart';
import 'package:parent_app/core/models/device_model.dart';

class MockFirebaseFirestore extends Mock implements FirebaseFirestore {}
class MockCollectionReference extends Mock
    implements CollectionReference<Map<String, dynamic>> {}
class MockDocumentReference extends Mock
    implements DocumentReference<Map<String, dynamic>> {}
class MockDocumentSnapshot extends Mock
    implements DocumentSnapshot<Map<String, dynamic>> {}
class MockQuerySnapshot extends Mock
    implements QuerySnapshot<Map<String, dynamic>> {}
class MockQuery extends Mock implements Query<Map<String, dynamic>> {}

void main() {
  late MockFirebaseFirestore mockFirestore;
  late MockCollectionReference mockCollection;
  late MockDocumentReference mockDocRef;
  late DeviceRepository deviceRepository;

  setUpAll(() {
    registerFallbackValue(<String, dynamic>{});
  });

  setUp(() {
    mockFirestore = MockFirebaseFirestore();
    mockCollection = MockCollectionReference();
    mockDocRef = MockDocumentReference();

    when(() => mockFirestore.collection('devices')).thenReturn(mockCollection);
    when(() => mockCollection.doc(any())).thenReturn(mockDocRef);

    deviceRepository = DeviceRepository(firestore: mockFirestore);
  });

  group('DeviceRepository', () {
    group('createDevice', () {
      test('creates device document in Firestore', () async {
        // Arrange
        final device = DeviceModel(
          id: 'device-123',
          parentId: 'parent-123',
          childName: 'Rahul',
          childAge: 8,
          protectionLevel: ProtectionLevel.balanced,
          status: DeviceStatus.pending,
        );

        when(() => mockDocRef.set(any())).thenAnswer((_) async {});

        // Act
        await deviceRepository.createDevice(device);

        // Assert
        verify(() => mockDocRef.set(any())).called(1);
      });
    });

    group('getDevice', () {
      test('returns device when exists', () async {
        // Arrange
        const deviceId = 'device-123';
        final mockSnapshot = MockDocumentSnapshot();

        when(() => mockDocRef.get()).thenAnswer((_) async => mockSnapshot);
        when(() => mockSnapshot.exists).thenReturn(true);
        when(() => mockSnapshot.data()).thenReturn({
          'id': deviceId,
          'parentId': 'parent-123',
          'childName': 'Rahul',
          'childAge': 8,
          'protectionLevel': 'balanced',
          'status': 'online',
        });
        when(() => mockSnapshot.id).thenReturn(deviceId);

        // Act
        final result = await deviceRepository.getDevice(deviceId);

        // Assert
        expect(result, isNotNull);
        expect(result!.id, equals(deviceId));
        expect(result.childName, equals('Rahul'));
      });

      test('returns null when device not found', () async {
        // Arrange
        const deviceId = 'nonexistent';
        final mockSnapshot = MockDocumentSnapshot();

        when(() => mockDocRef.get()).thenAnswer((_) async => mockSnapshot);
        when(() => mockSnapshot.exists).thenReturn(false);

        // Act
        final result = await deviceRepository.getDevice(deviceId);

        // Assert
        expect(result, isNull);
      });
    });

    group('updateDeviceStatus', () {
      test('updates device status in Firestore', () async {
        // Arrange
        const deviceId = 'device-123';

        when(() => mockDocRef.update(any())).thenAnswer((_) async {});

        // Act
        await deviceRepository.updateDeviceStatus(
          deviceId,
          DeviceStatus.online,
        );

        // Assert
        verify(() => mockDocRef.update({'status': 'online'})).called(1);
      });
    });

    group('getDevicesForParent', () {
      test('returns all devices for parent', () async {
        // Arrange
        const parentId = 'parent-123';
        final mockQuery = MockQuery();
        final mockQuerySnapshot = MockQuerySnapshot();
        final mockDoc = MockDocumentSnapshot();

        when(() => mockCollection.where('parentId', isEqualTo: parentId))
            .thenReturn(mockQuery);
        when(() => mockQuery.get()).thenAnswer((_) async => mockQuerySnapshot);
        when(() => mockQuerySnapshot.docs).thenReturn([mockDoc]);
        when(() => mockDoc.data()).thenReturn({
          'id': 'device-123',
          'parentId': parentId,
          'childName': 'Rahul',
          'childAge': 8,
          'protectionLevel': 'balanced',
          'status': 'online',
        });
        when(() => mockDoc.id).thenReturn('device-123');

        // Act
        final result = await deviceRepository.getDevicesForParent(parentId);

        // Assert
        expect(result, hasLength(1));
        expect(result.first.parentId, equals(parentId));
      });
    });
  });
}
```

**Story 5.3.3: CommandService Tests**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 5.3.3.1 | Test send toggle mode command | Command written to RTDB |
| 5.3.3.2 | Test send app whitelist update | Command written correctly |
| 5.3.3.3 | Test send screen time limit | Command written correctly |
| 5.3.3.4 | Test command acknowledgement | Ack received from device |

**Implementation: command_service_test.dart**

```dart
// test/core/services/command_service_test.dart

import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:firebase_database/firebase_database.dart';
import 'package:parent_app/core/services/command_service.dart';
import 'package:parent_app/core/models/command_model.dart';

class MockFirebaseDatabase extends Mock implements FirebaseDatabase {}
class MockDatabaseReference extends Mock implements DatabaseReference {}
class MockDatabaseEvent extends Mock implements DatabaseEvent {}
class MockDataSnapshot extends Mock implements DataSnapshot {}

void main() {
  late MockFirebaseDatabase mockDatabase;
  late MockDatabaseReference mockRef;
  late CommandService commandService;

  setUpAll(() {
    registerFallbackValue(<String, dynamic>{});
  });

  setUp(() {
    mockDatabase = MockFirebaseDatabase();
    mockRef = MockDatabaseReference();

    when(() => mockDatabase.ref(any())).thenReturn(mockRef);
    when(() => mockRef.child(any())).thenReturn(mockRef);
    when(() => mockRef.push()).thenReturn(mockRef);

    commandService = CommandService(database: mockDatabase);
  });

  group('CommandService', () {
    group('sendCommand', () {
      test('sends toggle mode command to device', () async {
        // Arrange
        const deviceId = 'device-123';
        final command = Command(
          type: CommandType.toggleMode,
          payload: {'mode': 'kids'},
          timestamp: DateTime.now(),
        );

        when(() => mockRef.set(any())).thenAnswer((_) async {});

        // Act
        await commandService.sendCommand(deviceId, command);

        // Assert
        verify(() => mockDatabase.ref('devices/$deviceId/commands')).called(1);
        verify(() => mockRef.push()).called(1);
        verify(() => mockRef.set(any())).called(1);
      });

      test('sends app whitelist update command', () async {
        // Arrange
        const deviceId = 'device-123';
        final command = Command(
          type: CommandType.updateWhitelist,
          payload: {
            'action': 'add',
            'packageName': 'com.phonepe.app',
          },
          timestamp: DateTime.now(),
        );

        when(() => mockRef.set(any())).thenAnswer((_) async {});

        // Act
        await commandService.sendCommand(deviceId, command);

        // Assert
        verify(() => mockRef.set(argThat(
          predicate<Map<String, dynamic>>((map) =>
            map['type'] == 'updateWhitelist' &&
            map['payload']['packageName'] == 'com.phonepe.app'
          ),
        ))).called(1);
      });

      test('sends screen time limit command', () async {
        // Arrange
        const deviceId = 'device-123';
        final command = Command(
          type: CommandType.setScreenTimeLimit,
          payload: {
            'dailyLimitMinutes': 120,
            'appLimits': {
              'com.spotify.music': 60,
              'com.jio.saavn': 60,
            },
          },
          timestamp: DateTime.now(),
        );

        when(() => mockRef.set(any())).thenAnswer((_) async {});

        // Act
        await commandService.sendCommand(deviceId, command);

        // Assert
        verify(() => mockRef.set(argThat(
          predicate<Map<String, dynamic>>((map) =>
            map['type'] == 'setScreenTimeLimit' &&
            map['payload']['dailyLimitMinutes'] == 120
          ),
        ))).called(1);
      });
    });

    group('listenForAcknowledgements', () {
      test('receives command acknowledgement', () async {
        // Arrange
        const deviceId = 'device-123';
        const commandId = 'cmd-456';
        final mockEvent = MockDatabaseEvent();
        final mockSnapshot = MockDataSnapshot();

        when(() => mockSnapshot.value).thenReturn({
          'commandId': commandId,
          'status': 'completed',
          'timestamp': DateTime.now().millisecondsSinceEpoch,
        });
        when(() => mockEvent.snapshot).thenReturn(mockSnapshot);
        when(() => mockRef.onChildAdded)
            .thenAnswer((_) => Stream.value(mockEvent));

        // Act & Assert
        await expectLater(
          commandService.listenForAcknowledgements(deviceId),
          emits(isA<CommandAck>()),
        );
      });
    });

    group('getCommandStatus', () {
      test('returns pending for unprocessed command', () async {
        // Arrange
        const deviceId = 'device-123';
        const commandId = 'cmd-456';
        final mockSnapshot = MockDataSnapshot();

        when(() => mockRef.get()).thenAnswer((_) async => mockSnapshot);
        when(() => mockSnapshot.exists).thenReturn(true);
        when(() => mockSnapshot.value).thenReturn({
          'status': 'pending',
        });

        // Act
        final result = await commandService.getCommandStatus(
          deviceId,
          commandId,
        );

        // Assert
        expect(result, equals(CommandStatus.pending));
      });
    });
  });
}
```

---

### Epic 5.4: Integration Tests

**Story 5.4.1: Android Integration Tests**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 5.4.1.1 | Test launcher as default home | Launcher receives HOME intent |
| 5.4.1.2 | Test mode switching flow | UI updates on mode switch |
| 5.4.1.3 | Test app grid display | Correct apps shown in grid |
| 5.4.1.4 | Test PIN dialog validation | PIN entry works correctly |

**Implementation: LauncherIntegrationTest.kt**

```kotlin
// app/src/androidTest/java/com/kidsafe/launcher/LauncherIntegrationTest.kt

package com.kidsafe.launcher

import androidx.test.espresso.Espresso.onView
import androidx.test.espresso.action.ViewActions.*
import androidx.test.espresso.assertion.ViewAssertions.matches
import androidx.test.espresso.matcher.ViewMatchers.*
import androidx.test.ext.junit.rules.ActivityScenarioRule
import androidx.test.ext.junit.runners.AndroidJUnit4
import androidx.test.filters.LargeTest
import org.junit.Rule
import org.junit.Test
import org.junit.runner.RunWith

@RunWith(AndroidJUnit4::class)
@LargeTest
class LauncherIntegrationTest {

    @get:Rule
    val activityRule = ActivityScenarioRule(LauncherActivity::class.java)

    @Test
    fun testLauncherDisplaysKidsModeByDefault() {
        // Verify kids mode UI is displayed
        onView(withId(R.id.kids_mode_container))
            .check(matches(isDisplayed()))

        // Verify greeting text
        onView(withId(R.id.greeting_text))
            .check(matches(isDisplayed()))

        // Verify panic button is visible
        onView(withId(R.id.panic_button))
            .check(matches(isDisplayed()))
    }

    @Test
    fun testAppGridDisplaysWhitelistedApps() {
        // Verify app grid is displayed
        onView(withId(R.id.app_grid))
            .check(matches(isDisplayed()))

        // Note: Number of apps depends on whitelist
        // This test verifies the grid exists
    }

    @Test
    fun testModeSwitchRequiresPin() {
        // Click mode switch button (in standard mode area)
        onView(withId(R.id.switch_mode_button))
            .perform(click())

        // Verify PIN dialog appears
        onView(withId(R.id.pin_dialog))
            .check(matches(isDisplayed()))

        // Verify PIN input field exists
        onView(withId(R.id.pin_input))
            .check(matches(isDisplayed()))
    }

    @Test
    fun testCorrectPinSwitchesMode() {
        // Open PIN dialog
        onView(withId(R.id.switch_mode_button))
            .perform(click())

        // Enter correct PIN (assuming 1234 is default test PIN)
        onView(withId(R.id.pin_input))
            .perform(typeText("1234"), closeSoftKeyboard())

        // Submit PIN
        onView(withId(R.id.pin_submit_button))
            .perform(click())

        // Verify mode switched to standard
        onView(withId(R.id.standard_mode_container))
            .check(matches(isDisplayed()))
    }

    @Test
    fun testWrongPinShowsError() {
        // Open PIN dialog
        onView(withId(R.id.switch_mode_button))
            .perform(click())

        // Enter wrong PIN
        onView(withId(R.id.pin_input))
            .perform(typeText("0000"), closeSoftKeyboard())

        // Submit PIN
        onView(withId(R.id.pin_submit_button))
            .perform(click())

        // Verify error message
        onView(withId(R.id.pin_error_text))
            .check(matches(isDisplayed()))
            .check(matches(withText(R.string.wrong_pin_error)))
    }

    @Test
    fun testPanicButtonShowsConfirmation() {
        // Click panic button
        onView(withId(R.id.panic_button))
            .perform(click())

        // Verify confirmation dialog
        onView(withText(R.string.panic_confirm_title))
            .check(matches(isDisplayed()))
    }

    @Test
    fun testAppLaunchFromGrid() {
        // Note: This test needs a whitelisted app installed
        // Click on first app in grid
        onView(withId(R.id.app_grid))
            .perform(actionOnItemAtPosition<AppViewHolder>(0, click()))

        // Activity should attempt to launch app
        // Verification depends on app being available
    }
}
```

**Story 5.4.2: Flutter Integration Tests**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 5.4.2.1 | Test login flow | User can login and see dashboard |
| 5.4.2.2 | Test device pairing flow | QR code generates and scans |
| 5.4.2.3 | Test app management | Can toggle apps on/off |
| 5.4.2.4 | Test screen time settings | Can set and save limits |

**Implementation: integration_test/app_test.dart**

```dart
// integration_test/app_test.dart

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:parent_app/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('Parent App Integration Tests', () {
    testWidgets('Login flow - phone authentication', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Verify login screen is displayed
      expect(find.text('Welcome to KidSafe'), findsOneWidget);

      // Enter phone number
      await tester.enterText(
        find.byKey(const Key('phone_input')),
        '9876543210',
      );

      // Tap continue button
      await tester.tap(find.byKey(const Key('continue_button')));
      await tester.pumpAndSettle();

      // Verify OTP screen is displayed
      expect(find.text('Enter OTP'), findsOneWidget);

      // Note: Cannot test actual OTP verification in integration test
      // Would need mock Firebase
    });

    testWidgets('Dashboard displays device status', (tester) async {
      // Pre-condition: User is logged in (use mock auth)
      app.main();
      await tester.pumpAndSettle();

      // Skip to dashboard (assuming logged in state)
      // This would need proper mock setup

      // Verify dashboard elements
      expect(find.byKey(const Key('device_card')), findsWidgets);
      expect(find.byKey(const Key('quick_actions')), findsOneWidget);
    });

    testWidgets('Device pairing generates QR code', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Navigate to add device
      await tester.tap(find.byKey(const Key('add_device_button')));
      await tester.pumpAndSettle();

      // Fill child details
      await tester.enterText(
        find.byKey(const Key('child_name_input')),
        'Rahul',
      );
      await tester.enterText(
        find.byKey(const Key('child_age_input')),
        '8',
      );

      // Select protection level
      await tester.tap(find.text('Balanced'));
      await tester.pumpAndSettle();

      // Continue to pairing
      await tester.tap(find.byKey(const Key('continue_to_pairing')));
      await tester.pumpAndSettle();

      // Verify QR code is displayed
      expect(find.byKey(const Key('pairing_qr_code')), findsOneWidget);
    });

    testWidgets('App management toggle works', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Navigate to app management
      await tester.tap(find.byKey(const Key('manage_apps_button')));
      await tester.pumpAndSettle();

      // Find Spotify toggle
      final spotifyToggle = find.byKey(const Key('app_toggle_com.spotify.music'));
      expect(spotifyToggle, findsOneWidget);

      // Toggle it
      await tester.tap(spotifyToggle);
      await tester.pumpAndSettle();

      // Verify state changed (would need mock verification)
    });

    testWidgets('Screen time slider updates value', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Navigate to screen time settings
      await tester.tap(find.byKey(const Key('screen_time_button')));
      await tester.pumpAndSettle();

      // Find daily limit slider
      final slider = find.byKey(const Key('daily_limit_slider'));
      expect(slider, findsOneWidget);

      // Drag slider to new value
      await tester.drag(slider, const Offset(100, 0));
      await tester.pumpAndSettle();

      // Verify value label updated
      // The exact verification depends on UI implementation
    });

    testWidgets('Bedtime picker sets times', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Navigate to screen time settings
      await tester.tap(find.byKey(const Key('screen_time_button')));
      await tester.pumpAndSettle();

      // Tap bedtime start picker
      await tester.tap(find.byKey(const Key('bedtime_start_picker')));
      await tester.pumpAndSettle();

      // Select time (time picker interaction)
      // Note: Time picker testing requires specific handling

      // Verify bedtime is set
      expect(find.textContaining('PM'), findsWidgets);
    });

    testWidgets('Kids mode toggle sends command', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Find kids mode toggle on dashboard
      final kidsToggle = find.byKey(const Key('kids_mode_toggle'));
      expect(kidsToggle, findsOneWidget);

      // Toggle kids mode
      await tester.tap(kidsToggle);
      await tester.pumpAndSettle();

      // Verify loading indicator (command being sent)
      // Then verify success feedback
    });
  });
}
```

---

### Epic 5.5: Bug Bash & Fixes

**Story 5.5.1: Internal Bug Bash (Days 57-59)**

| Task | Description | Owner |
|------|-------------|-------|
| 5.5.1.1 | Create bug bash checklist | QA |
| 5.5.1.2 | Execute launcher test scenarios | Android Dev |
| 5.5.1.3 | Execute parent app test scenarios | Flutter Dev |
| 5.5.1.4 | Execute end-to-end scenarios | All |
| 5.5.1.5 | Document all bugs in issue tracker | QA |

**Bug Bash Checklist Template:**

```markdown
# Bug Bash Checklist - KidSafe Device

## Test Device Information
- Phone Model: Samsung Galaxy A05
- Android Version: ___
- Build Version: ___
- Tester Name: ___
- Date: ___

## Launcher Tests

### Kids Mode
- [ ] Launcher loads on boot
- [ ] Greeting shows correct name
- [ ] Time of day greeting is accurate (morning/afternoon/evening)
- [ ] App grid shows only whitelisted apps
- [ ] App icons are clearly visible
- [ ] Apps launch when tapped
- [ ] Panic button is visible
- [ ] Panic button triggers alert
- [ ] Volume controls work
- [ ] Status bar shows battery/signal

### Standard Mode
- [ ] PIN dialog appears when switching
- [ ] Correct PIN allows mode switch
- [ ] Wrong PIN shows error
- [ ] 3 wrong attempts locks for 1 minute
- [ ] Standard grid shows all whitelisted apps
- [ ] Settings accessible (if allowed)
- [ ] Can switch back to Kids mode

### DPC Restrictions
- [ ] Cannot uninstall launcher
- [ ] Cannot uninstall DPC
- [ ] Cannot access Play Store (Kids Mode)
- [ ] Cannot install APKs (Kids Mode)
- [ ] Cannot factory reset without PIN
- [ ] Safe boot disabled

### Remote Commands
- [ ] Mode switch from parent app works
- [ ] App added remotely appears
- [ ] App removed remotely disappears
- [ ] Screen time limit enforced
- [ ] Bedtime mode activates on schedule
- [ ] Lock device command works
- [ ] Unlock device command works

## Parent App Tests

### Authentication
- [ ] Phone login works
- [ ] OTP delivery within 30s
- [ ] OTP verification works
- [ ] Email login works
- [ ] Password reset works
- [ ] Logout works

### Device Pairing
- [ ] QR code generates
- [ ] QR code scannable
- [ ] Pairing completes successfully
- [ ] Device appears in dashboard

### Dashboard
- [ ] Device status shows (online/offline)
- [ ] Battery level shows
- [ ] Current mode shows
- [ ] Now playing shows (if playing)
- [ ] Location shows (if enabled)
- [ ] Quick actions work

### App Management
- [ ] All installed apps list shows
- [ ] Toggle on adds to whitelist
- [ ] Toggle off removes from whitelist
- [ ] Time limits can be set
- [ ] Schedule restrictions work

### Screen Time
- [ ] Daily limit slider works
- [ ] Per-app limits work
- [ ] Bedtime picker works
- [ ] Usage history shows

### Notifications
- [ ] Panic alert received
- [ ] Battery low alert received
- [ ] Device offline alert received
- [ ] Screen time limit reached alert

## Edge Cases
- [ ] Airplane mode behavior
- [ ] Low battery behavior
- [ ] App update handling
- [ ] Multiple parent accounts
- [ ] Device restart persistence

## Performance
- [ ] App launch time < 3s
- [ ] Mode switch time < 2s
- [ ] Command execution < 5s
- [ ] Battery drain acceptable
```

**Story 5.5.2: Bug Triage & Prioritization**

| Priority | Description | SLA |
|----------|-------------|-----|
| P0 - Critical | App crash, data loss, security issue | Fix within 24 hours |
| P1 - High | Feature broken, major UX issue | Fix within 3 days |
| P2 - Medium | Minor feature issue, edge case | Fix within 1 week |
| P3 - Low | Polish, enhancement | Backlog |

**Story 5.5.3: Bug Fix Sprints (Days 60-63)**

| Day | Focus | Deliverable |
|-----|-------|-------------|
| 60 | Fix all P0 bugs | Zero P0 bugs |
| 61 | Fix P1 bugs (Part 1) | P1 count reduced by 50% |
| 62 | Fix P1 bugs (Part 2) | Zero P1 bugs |
| 63 | Fix critical P2 bugs | P2 count reduced by 30% |

---

### Epic 5.6: Device Farm Testing (CRITICAL)

**Story 5.6.1: Firebase Test Lab Setup**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 5.6.1.1 | Configure Firebase Test Lab | Test Lab project configured |
| 5.6.1.2 | Define device matrix | 5+ target devices defined |
| 5.6.1.3 | Create CI workflow | GitHub Actions workflow running |
| 5.6.1.4 | Run initial device tests | Tests pass on all target devices |

**Firebase Test Lab Configuration:**

```yaml
# .github/workflows/device-tests.yml
name: Device Farm Tests

on:
  push:
    branches: [develop, main]
  pull_request:
    branches: [main]

jobs:
  instrumentation-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build APKs
        run: |
          cd android/launcher
          ./gradlew assembleDebug assembleDebugAndroidTest

      - name: Authenticate with Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GOOGLE_CREDENTIALS }}

      - name: Run Firebase Test Lab
        uses: google-github-actions/firebase-test-lab@v1
        with:
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

**Priority Device Matrix:**

| Priority | Model | Android | Reason |
|----------|-------|---------|--------|
| P0 | Samsung Galaxy A05 | 14 | Primary target device |
| P1 | Samsung Galaxy A06 | 14 | Backup target device |
| P2 | Redmi Note 12 | 13 | Popular budget device |
| P3 | Pixel 6 | 14 | Reference device |
| P4 | Samsung Galaxy S21 | 14 | Premium device testing |

---

### Epic 5.7: Security Penetration Testing (CRITICAL)

**Story 5.7.1: OWASP Mobile Top 10 Testing**

| Test Area | Tool/Method | Pass Criteria |
|-----------|-------------|---------------|
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

**Story 5.7.2: DPC-Specific Security Tests**

| Test | Method | Expected Result |
|------|--------|-----------------|
| Bypass Device Owner | ADB commands | Cannot bypass |
| Uninstall DPC | Settings app | Blocked by Device Owner |
| Disable admin | adb shell pm disable | Blocked |
| Factory reset | Recovery mode | Requires PIN (if FRP enabled) |
| Safe mode bypass | Boot safe mode | DPC still enforced |
| Root detection | Magisk, frida | Detected and blocked |
| Debug bridge bypass | ADB shell | Limited functionality |

**Story 5.7.3: Firebase Security Tests**

| Test | Method | Expected Result |
|------|--------|-----------------|
| Unauthorized data access | Direct API calls | Blocked by security rules |
| Cross-user data access | Modified client | Blocked by rules |
| Rate limit bypass | Automated requests | Rate limited |
| Token manipulation | JWT tampering | Rejected |
| Injection attacks | Malformed inputs | Sanitized/rejected |

**Budget: Security Audit**
- Internal testing (Days 60-61): Team time
- External audit (Days 62-63): ₹50,000-1,00,000
- Focus: DPC bypass, data security, Firebase rules

---

### Epic 5.8: Accessibility Testing

**Story 5.8.1: Accessibility Compliance Checklist**

| Check | Tool | Pass Criteria |
|-------|------|---------------|
| TalkBack navigation | Manual testing | All screens navigable |
| Content descriptions | Android Lint | All interactive elements have descriptions |
| Color contrast | Accessibility Scanner | Ratio ≥ 4.5:1 |
| Touch targets | Android Lint | All targets ≥ 48x48dp |
| Font scaling | Device settings | UI works at 200% scale |
| Focus indicators | Manual testing | Visible on all focusable elements |

**Kids Mode Accessibility Requirements:**
- Extra large touch targets (120dp icons)
- High contrast colors
- Simple navigation (no complex gestures)
- Audio feedback for actions (optional)

---

### Epic 5.9: Advanced Parental Control Tests (CRITICAL)

**Story 5.9.1: Web Content Filtering Tests (F19)**

| Test | Steps | Expected Result |
|------|-------|-----------------|
| DNS filter applies | Set filter to STRICT, try to access adult site | Site doesn't resolve |
| Filter level changes | Switch from STRICT to OFF via parent app | Sites now accessible |
| CleanBrowsing integration | Verify DNS is set to family-filter-dns.cleanbrowsing.org | `getprop net.dns1` shows filter DNS |
| Categories blocked | Try accessing gambling, social media sites | All blocked in STRICT mode |
| Safe Search enforced | Search explicit terms on Google | No explicit results |
| Filter survives reboot | Reboot device, check filter is still active | Filter remains active |

**Implementation: Content Filter Test**

```kotlin
@Test
fun `DNS filter blocks adult content when set to STRICT`() = runTest {
    // Arrange
    val contentFilterManager = ContentFilterManager(context, preferences)

    // Act
    contentFilterManager.setFilterLevel(FilterLevel.STRICT)

    // Assert
    val dnsHost = Settings.Global.getString(
        context.contentResolver,
        "private_dns_specifier"
    )
    assertEquals("family-filter-dns.cleanbrowsing.org", dnsHost)
}
```

**Story 5.9.2: Call Control Tests (F20)**

| Test | Steps | Expected Result |
|------|-------|-----------------|
| Block unknown caller | Call from unknown number in APPROVED_ONLY mode | Call rejected |
| Allow approved contact | Call from approved number | Call goes through |
| Emergency bypass | Call from 100, 108, 112 | Call always allowed |
| Parent bypass | Call from parent number | Call always allowed |
| Call logging | Make test call | Call logged to Firebase |
| Safe dialer shows only approved | Open safe dialer | Only approved contacts visible |
| System dialer hidden | Try to open Google Dialer | App not found/hidden |

**Implementation: Call Screening Test**

```kotlin
@Test
fun `CallScreener blocks unapproved numbers`() = runTest {
    // Arrange
    val screener = KidSafeCallScreener()
    val mockCallDetails = mockk<Call.Details>()
    every { mockCallDetails.handle } returns Uri.parse("tel:+919999999999")

    coEvery { approvedContactsRepo.isApproved(any()) } returns false
    coEvery { approvedContactsRepo.isParentNumber(any()) } returns false

    // Act
    screener.onScreenCall(mockCallDetails)

    // Assert
    verify { screener.respondToCall(mockCallDetails, match { !it.disallowCall }) }
}

@Test
fun `Emergency numbers always allowed`() = runTest {
    // Arrange
    val screener = KidSafeCallScreener()
    val emergencyNumbers = listOf("100", "108", "112", "1098")

    emergencyNumbers.forEach { number ->
        val mockCallDetails = mockk<Call.Details>()
        every { mockCallDetails.handle } returns Uri.parse("tel:$number")

        // Act
        screener.onScreenCall(mockCallDetails)

        // Assert
        verify { screener.respondToCall(mockCallDetails, match { !it.disallowCall }) }
    }
}
```

**Story 5.9.3: App Controls Tests (F22)**

| Test | Steps | Expected Result |
|------|-------|-----------------|
| YouTube Restricted Mode | Set restricted mode, search explicit content | Explicit content filtered |
| Chrome Incognito blocked | Try to open incognito tab in Chrome | Option not available |
| Chrome SafeSearch | Search explicit terms in Chrome | Filtered by SafeSearch |
| In-app purchase blocked | Try to purchase in game | Purchase requires password/blocked |
| Settings persist | Reboot device, check app settings | All settings retained |
| Config applied to Chrome | Check Chrome managed policies | Policies show as applied |

**Implementation: Managed Config Test**

```kotlin
@Test
fun `Chrome incognito mode blocked when configured`() = runTest {
    // Arrange
    val configManager = ManagedConfigManager(context)

    // Act
    configManager.configureChromeRestrictions(ChromeConfig(blockIncognito = true))

    // Assert
    val restrictions = dpm.getApplicationRestrictions(
        adminComponent,
        "com.android.chrome"
    )
    assertEquals(1, restrictions.getInt("IncognitoModeAvailability"))
}

@Test
fun `YouTube restricted mode enabled`() = runTest {
    // Arrange
    val configManager = ManagedConfigManager(context)

    // Act
    configManager.configureYouTubeRestrictions(restrictedMode = true)

    // Assert
    val restrictions = dpm.getApplicationRestrictions(
        adminComponent,
        "com.google.android.youtube"
    )
    assertEquals("Strict", restrictions.getString("restrict"))
}
```

**Story 5.9.4: End-to-End Advanced Control Tests**

| Scenario | Steps | Expected Result |
|----------|-------|-----------------|
| Full lockdown mode | Enable STRICT filter + APPROVED_ONLY calls + YouTube restricted | All protections active |
| Parent control flow | Parent sets controls in app, verify on device | Settings sync < 10 sec |
| Child attempts bypass | Child tries VPN, alternate browser, etc. | Controls remain effective |
| Offline behavior | Set controls, go offline, reboot | Controls persist offline |

---

## Sprint 6: Beta Testing & Polish (Days 85-98)

### Epic 6.1: Beta Recruitment

**Story 6.1.1: Recruit Beta Families**

| Task | Description | Target |
|------|-------------|--------|
| 6.1.1.1 | Create beta application form | Google Form ready |
| 6.1.1.2 | Define beta criteria | Clear requirements |
| 6.1.1.3 | Reach out to network | 50+ applications |
| 6.1.1.4 | Select 20 families | Diverse demographics |
| 6.1.1.5 | Schedule beta kickoff | Call scheduled |

**Beta Family Criteria:**
- Child age 5-12 years
- Parent has smartphone (Android or iOS)
- Willing to provide weekly feedback
- Located in metro city (for support)
- No technical background required (preferred)

**Beta Application Form Fields:**
```
1. Parent Name
2. Email
3. Phone Number
4. City
5. Child's Name
6. Child's Age
7. Current device child uses
8. Biggest concern about screen time
9. Would you pay ₹10,999 for this solution?
10. Availability for 2-week beta testing
```

**Story 6.1.2: Configure Beta Devices**

| Task | Description | Deliverable |
|------|-------------|-------------|
| 6.1.2.1 | Procure 20 Samsung Galaxy A05 devices | Devices received |
| 6.1.2.2 | Create device setup script | Script ready |
| 6.1.2.3 | Flash all devices with beta build | 20 devices configured |
| 6.1.2.4 | Create beta user accounts | 20 accounts created |
| 6.1.2.5 | Prepare device packages | Ready for distribution |

**Device Setup Script: setup_beta_device.sh**

```bash
#!/bin/bash

# Beta Device Setup Script
# Usage: ./setup_beta_device.sh <device_serial> <beta_user_email>

DEVICE_SERIAL=$1
BETA_USER_EMAIL=$2

if [ -z "$DEVICE_SERIAL" ] || [ -z "$BETA_USER_EMAIL" ]; then
    echo "Usage: ./setup_beta_device.sh <device_serial> <beta_user_email>"
    exit 1
fi

echo "Setting up device: $DEVICE_SERIAL for user: $BETA_USER_EMAIL"

# 1. Factory reset (if needed)
# adb -s $DEVICE_SERIAL shell am broadcast -a android.intent.action.FACTORY_RESET

# 2. Wait for device to be ready
adb -s $DEVICE_SERIAL wait-for-device

# 3. Install DPC
echo "Installing Device Policy Controller..."
adb -s $DEVICE_SERIAL install -r ./builds/dpc-beta.apk

# 4. Set as Device Owner via QR provisioning helper
echo "Setting up Device Owner mode..."
adb -s $DEVICE_SERIAL shell dpm set-device-owner com.kidsafe.dpc/.KidDeviceAdminReceiver

# 5. Install Launcher
echo "Installing Launcher..."
adb -s $DEVICE_SERIAL install -r ./builds/launcher-beta.apk

# 6. Set launcher as default
echo "Setting launcher as default home..."
adb -s $DEVICE_SERIAL shell cmd package set-home-activity com.kidsafe.launcher/.LauncherActivity

# 7. Install required apps
echo "Installing whitelisted apps..."
adb -s $DEVICE_SERIAL install -r ./apks/spotify.apk
adb -s $DEVICE_SERIAL install -r ./apks/jiosaavn.apk

# 8. Configure beta user ID
echo "Configuring beta user..."
adb -s $DEVICE_SERIAL shell am broadcast \
    -a com.kidsafe.launcher.CONFIGURE \
    --es user_email "$BETA_USER_EMAIL"

# 9. Enable stay awake for setup
adb -s $DEVICE_SERIAL shell settings put global stay_on_while_plugged_in 3

# 10. Verify setup
echo "Verifying setup..."
adb -s $DEVICE_SERIAL shell dumpsys package com.kidsafe.dpc | grep "Device Owner"
adb -s $DEVICE_SERIAL shell pm list packages | grep kidsafe

echo "Setup complete for device: $DEVICE_SERIAL"
echo "Please complete pairing using parent app for: $BETA_USER_EMAIL"
```

---

### Epic 6.2: Beta Feedback Collection

**Story 6.2.1: Feedback Infrastructure**

| Task | Description | Deliverable |
|------|-------------|-------------|
| 6.2.1.1 | Create feedback Google Form | Form URL |
| 6.2.1.2 | Set up WhatsApp group for beta users | Group created |
| 6.2.1.3 | Create daily check-in bot | Bot sends reminders |
| 6.2.1.4 | Set up Crashlytics alerts | Alerts configured |
| 6.2.1.5 | Create beta dashboard | Dashboard showing status |

**Daily Feedback Form:**
```
KidSafe Beta - Daily Feedback

Date: [Auto-filled]
Family Name: [Dropdown]

1. Did your child use the device today?
   - Yes
   - No

2. How many hours did they use it?
   - Less than 1 hour
   - 1-2 hours
   - 2-3 hours
   - More than 3 hours

3. Did you use the parent app today?
   - Yes
   - No

4. Did you encounter any issues?
   - No issues
   - Minor issues (still usable)
   - Major issues (blocked usage)

5. If issues, please describe:
   [Text field]

6. Any features you wish existed?
   [Text field]

7. Overall satisfaction today (1-5):
   [Rating scale]

8. Would you recommend this to friends? (1-10)
   [NPS scale]
```

**Story 6.2.2: Weekly Beta Calls**

| Week | Focus | Agenda |
|------|-------|--------|
| Week 1 | Onboarding | Setup help, initial impressions |
| Week 2 | Usage patterns | How are they using it? Pain points? |

**Weekly Call Agenda Template:**
```markdown
# KidSafe Beta - Week [X] Call

## Attendees
- Families: [List]
- KidSafe Team: [List]

## Agenda (45 minutes)

### Opening (5 min)
- Thank everyone for participating
- Quick summary of week's metrics

### Round Robin Feedback (25 min)
Each family (2-3 min each):
1. How is your child using the device?
2. What's working well?
3. What's frustrating?
4. Any specific bugs to report?

### Feature Discussion (10 min)
- Top requested features from feedback
- Prioritization discussion
- Timeline expectations

### Closing (5 min)
- Preview of next week's fixes/updates
- Next call scheduling
- Thank you

## Action Items
- [ ] Item 1
- [ ] Item 2
```

---

### Epic 6.3: Polish & UX Improvements

**Story 6.3.1: Parent App UI Polish**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 6.3.1.1 | Implement consistent color scheme | All screens use theme colors |
| 6.3.1.2 | Add loading states | All async ops show loading |
| 6.3.1.3 | Add error states | Friendly error messages |
| 6.3.1.4 | Add empty states | Helpful empty state illustrations |
| 6.3.1.5 | Add animations | Smooth transitions |
| 6.3.1.6 | Accessibility audit | WCAG 2.1 AA compliance |

**Implementation: Theme Configuration**

```dart
// lib/core/theme/app_theme.dart

import 'package:flutter/material.dart';

class AppTheme {
  // Colors
  static const Color primaryColor = Color(0xFF6366F1); // Indigo
  static const Color secondaryColor = Color(0xFF10B981); // Green
  static const Color errorColor = Color(0xFFEF4444); // Red
  static const Color warningColor = Color(0xFFF59E0B); // Amber
  static const Color successColor = Color(0xFF22C55E); // Green

  // Background colors
  static const Color backgroundColor = Color(0xFFF9FAFB);
  static const Color surfaceColor = Colors.white;
  static const Color cardColor = Colors.white;

  // Text colors
  static const Color textPrimary = Color(0xFF111827);
  static const Color textSecondary = Color(0xFF6B7280);
  static const Color textTertiary = Color(0xFF9CA3AF);

  // Status colors
  static const Color onlineColor = Color(0xFF22C55E);
  static const Color offlineColor = Color(0xFF9CA3AF);
  static const Color kidsModeColor = Color(0xFF8B5CF6); // Purple
  static const Color standardModeColor = Color(0xFF3B82F6); // Blue

  static ThemeData get lightTheme {
    return ThemeData(
      useMaterial3: true,
      colorScheme: ColorScheme.fromSeed(
        seedColor: primaryColor,
        primary: primaryColor,
        secondary: secondaryColor,
        error: errorColor,
        background: backgroundColor,
        surface: surfaceColor,
      ),
      scaffoldBackgroundColor: backgroundColor,
      appBarTheme: const AppBarTheme(
        elevation: 0,
        backgroundColor: surfaceColor,
        foregroundColor: textPrimary,
        centerTitle: true,
        titleTextStyle: TextStyle(
          fontSize: 18,
          fontWeight: FontWeight.w600,
          color: textPrimary,
        ),
      ),
      cardTheme: CardTheme(
        elevation: 0,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(16),
          side: BorderSide(
            color: Colors.grey.shade200,
          ),
        ),
        color: cardColor,
      ),
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          elevation: 0,
          padding: const EdgeInsets.symmetric(
            horizontal: 24,
            vertical: 16,
          ),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(12),
          ),
          textStyle: const TextStyle(
            fontSize: 16,
            fontWeight: FontWeight.w600,
          ),
        ),
      ),
      inputDecorationTheme: InputDecorationTheme(
        filled: true,
        fillColor: Colors.grey.shade50,
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: BorderSide.none,
        ),
        enabledBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: BorderSide(color: Colors.grey.shade200),
        ),
        focusedBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: const BorderSide(color: primaryColor, width: 2),
        ),
        errorBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: const BorderSide(color: errorColor),
        ),
        contentPadding: const EdgeInsets.symmetric(
          horizontal: 16,
          vertical: 16,
        ),
      ),
      switchTheme: SwitchThemeData(
        thumbColor: MaterialStateProperty.resolveWith((states) {
          if (states.contains(MaterialState.selected)) {
            return primaryColor;
          }
          return Colors.grey.shade400;
        }),
        trackColor: MaterialStateProperty.resolveWith((states) {
          if (states.contains(MaterialState.selected)) {
            return primaryColor.withOpacity(0.3);
          }
          return Colors.grey.shade300;
        }),
      ),
      sliderTheme: SliderThemeData(
        activeTrackColor: primaryColor,
        inactiveTrackColor: primaryColor.withOpacity(0.2),
        thumbColor: primaryColor,
        overlayColor: primaryColor.withOpacity(0.1),
        valueIndicatorColor: primaryColor,
        trackHeight: 8,
        thumbShape: const RoundSliderThumbShape(
          enabledThumbRadius: 14,
        ),
      ),
    );
  }
}
```

**Story 6.3.2: Launcher UI Polish**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 6.3.2.1 | Create kid-friendly icons | Custom icon set |
| 6.3.2.2 | Add greeting animations | Wave animation on load |
| 6.3.2.3 | Add app launch animations | Scale/fade on tap |
| 6.3.2.4 | Add haptic feedback | Vibration on interactions |
| 6.3.2.5 | Sound effects (optional) | Pleasant tap sounds |

**Implementation: Kids Mode Animations**

```kotlin
// app/src/main/java/com/kidsafe/launcher/animation/LauncherAnimations.kt

package com.kidsafe.launcher.animation

import android.animation.AnimatorSet
import android.animation.ObjectAnimator
import android.view.View
import android.view.animation.AccelerateDecelerateInterpolator
import android.view.animation.OvershootInterpolator

object LauncherAnimations {

    /**
     * Greeting wave animation for emoji
     */
    fun animateGreetingWave(view: View) {
        val rotation = ObjectAnimator.ofFloat(view, "rotation", 0f, 15f, -15f, 10f, -10f, 0f)
        rotation.duration = 1000
        rotation.interpolator = AccelerateDecelerateInterpolator()
        rotation.start()
    }

    /**
     * App icon bounce when appearing
     */
    fun animateAppAppear(view: View, delay: Long = 0) {
        view.scaleX = 0f
        view.scaleY = 0f
        view.alpha = 0f

        val scaleX = ObjectAnimator.ofFloat(view, "scaleX", 0f, 1f)
        val scaleY = ObjectAnimator.ofFloat(view, "scaleY", 0f, 1f)
        val alpha = ObjectAnimator.ofFloat(view, "alpha", 0f, 1f)

        AnimatorSet().apply {
            playTogether(scaleX, scaleY, alpha)
            duration = 400
            startDelay = delay
            interpolator = OvershootInterpolator(1.5f)
            start()
        }
    }

    /**
     * App icon press animation
     */
    fun animateAppPress(view: View) {
        val scaleDown = AnimatorSet().apply {
            playTogether(
                ObjectAnimator.ofFloat(view, "scaleX", 1f, 0.9f),
                ObjectAnimator.ofFloat(view, "scaleY", 1f, 0.9f)
            )
            duration = 100
        }

        val scaleUp = AnimatorSet().apply {
            playTogether(
                ObjectAnimator.ofFloat(view, "scaleX", 0.9f, 1f),
                ObjectAnimator.ofFloat(view, "scaleY", 0.9f, 1f)
            )
            duration = 100
        }

        AnimatorSet().apply {
            playSequentially(scaleDown, scaleUp)
            start()
        }
    }

    /**
     * Mode switch transition
     */
    fun animateModeSwitch(outView: View, inView: View, toKidsMode: Boolean) {
        val duration = 300L

        // Fade out current mode
        ObjectAnimator.ofFloat(outView, "alpha", 1f, 0f).apply {
            this.duration = duration
            start()
        }

        // Translate out
        val translationOut = if (toKidsMode) -100f else 100f
        ObjectAnimator.ofFloat(outView, "translationX", 0f, translationOut).apply {
            this.duration = duration
            start()
        }

        // Prepare in view
        inView.alpha = 0f
        inView.translationX = if (toKidsMode) 100f else -100f
        inView.visibility = View.VISIBLE

        // Fade in new mode
        ObjectAnimator.ofFloat(inView, "alpha", 0f, 1f).apply {
            this.duration = duration
            startDelay = 150
            start()
        }

        // Translate in
        ObjectAnimator.ofFloat(inView, "translationX", inView.translationX, 0f).apply {
            this.duration = duration
            startDelay = 150
            start()
        }
    }

    /**
     * Panic button pulse animation
     */
    fun animatePanicButtonPulse(view: View) {
        val scaleX = ObjectAnimator.ofFloat(view, "scaleX", 1f, 1.1f, 1f)
        val scaleY = ObjectAnimator.ofFloat(view, "scaleY", 1f, 1.1f, 1f)

        AnimatorSet().apply {
            playTogether(scaleX, scaleY)
            duration = 1000
            repeatCount = ObjectAnimator.INFINITE
            start()
        }
    }

    /**
     * Lock screen slide down
     */
    fun animateLockScreen(view: View, show: Boolean) {
        val startY = if (show) -view.height.toFloat() else 0f
        val endY = if (show) 0f else -view.height.toFloat()

        ObjectAnimator.ofFloat(view, "translationY", startY, endY).apply {
            duration = 300
            interpolator = AccelerateDecelerateInterpolator()
            start()
        }
    }
}
```

**Story 6.3.3: Error Handling & Empty States**

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| 6.3.3.1 | Design error state illustrations | 3 error illustrations |
| 6.3.3.2 | Design empty state illustrations | 3 empty state illustrations |
| 6.3.3.3 | Implement error boundary widget | Catches all errors gracefully |
| 6.3.3.4 | Implement retry mechanisms | User can retry failed operations |

**Implementation: Error Handling Widget**

```dart
// lib/core/widgets/error_state_widget.dart

import 'package:flutter/material.dart';
import 'package:flutter_svg/flutter_svg.dart';

class ErrorStateWidget extends StatelessWidget {
  final String title;
  final String message;
  final VoidCallback? onRetry;
  final ErrorType type;

  const ErrorStateWidget({
    super.key,
    required this.title,
    required this.message,
    this.onRetry,
    this.type = ErrorType.general,
  });

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(32),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Error illustration
            SvgPicture.asset(
              _getIllustrationPath(),
              width: 200,
              height: 200,
            ),
            const SizedBox(height: 24),

            // Title
            Text(
              title,
              style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                fontWeight: FontWeight.bold,
              ),
              textAlign: TextAlign.center,
            ),
            const SizedBox(height: 8),

            // Message
            Text(
              message,
              style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                color: Colors.grey.shade600,
              ),
              textAlign: TextAlign.center,
            ),
            const SizedBox(height: 24),

            // Retry button
            if (onRetry != null)
              ElevatedButton.icon(
                onPressed: onRetry,
                icon: const Icon(Icons.refresh),
                label: const Text('Try Again'),
              ),
          ],
        ),
      ),
    );
  }

  String _getIllustrationPath() {
    switch (type) {
      case ErrorType.network:
        return 'assets/illustrations/no_connection.svg';
      case ErrorType.notFound:
        return 'assets/illustrations/not_found.svg';
      case ErrorType.timeout:
        return 'assets/illustrations/timeout.svg';
      case ErrorType.general:
      default:
        return 'assets/illustrations/error.svg';
    }
  }
}

enum ErrorType {
  general,
  network,
  notFound,
  timeout,
}

// Empty state widget
class EmptyStateWidget extends StatelessWidget {
  final String title;
  final String message;
  final Widget? action;
  final EmptyStateType type;

  const EmptyStateWidget({
    super.key,
    required this.title,
    required this.message,
    this.action,
    this.type = EmptyStateType.general,
  });

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(32),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Illustration
            SvgPicture.asset(
              _getIllustrationPath(),
              width: 200,
              height: 200,
            ),
            const SizedBox(height: 24),

            // Title
            Text(
              title,
              style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                fontWeight: FontWeight.bold,
              ),
              textAlign: TextAlign.center,
            ),
            const SizedBox(height: 8),

            // Message
            Text(
              message,
              style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                color: Colors.grey.shade600,
              ),
              textAlign: TextAlign.center,
            ),

            // Action
            if (action != null) ...[
              const SizedBox(height: 24),
              action!,
            ],
          ],
        ),
      ),
    );
  }

  String _getIllustrationPath() {
    switch (type) {
      case EmptyStateType.noDevices:
        return 'assets/illustrations/no_devices.svg';
      case EmptyStateType.noApps:
        return 'assets/illustrations/no_apps.svg';
      case EmptyStateType.noActivity:
        return 'assets/illustrations/no_activity.svg';
      case EmptyStateType.general:
      default:
        return 'assets/illustrations/empty.svg';
    }
  }
}

enum EmptyStateType {
  general,
  noDevices,
  noApps,
  noActivity,
}
```

---

### Epic 6.4: Performance Optimization

**Story 6.4.1: Android Performance**

| Task | Description | Target |
|------|-------------|--------|
| 6.4.1.1 | Profile launcher startup time | < 500ms cold start |
| 6.4.1.2 | Optimize app grid rendering | < 16ms frame time |
| 6.4.1.3 | Reduce memory footprint | < 50MB RAM |
| 6.4.1.4 | Battery optimization | < 2% daily drain |
| 6.4.1.5 | Firebase sync optimization | Batch writes |

**Story 6.4.2: Flutter Performance**

| Task | Description | Target |
|------|-------------|--------|
| 6.4.2.1 | Profile app startup | < 2s cold start |
| 6.4.2.2 | Optimize list rendering | Lazy loading |
| 6.4.2.3 | Image caching | Cache app icons |
| 6.4.2.4 | Reduce app size | < 30MB APK |
| 6.4.2.5 | Network optimization | Request batching |

**Implementation: Flutter Performance Optimizations**

```dart
// lib/core/utils/performance_utils.dart

import 'package:flutter/foundation.dart';
import 'package:flutter/scheduler.dart';

class PerformanceUtils {
  /// Log frame timings in debug mode
  static void enableFrameTimingLogging() {
    if (kDebugMode) {
      SchedulerBinding.instance.addTimingsCallback((timings) {
        for (final timing in timings) {
          final buildDuration = timing.buildDuration.inMicroseconds;
          final rasterDuration = timing.rasterDuration.inMicroseconds;

          if (buildDuration > 16000 || rasterDuration > 16000) {
            debugPrint(
              'Frame timing: build=${buildDuration}μs, '
              'raster=${rasterDuration}μs',
            );
          }
        }
      });
    }
  }

  /// Defer work until after frame
  static void deferWork(VoidCallback work) {
    SchedulerBinding.instance.addPostFrameCallback((_) {
      work();
    });
  }

  /// Run expensive work in isolate
  static Future<T> computeExpensive<T>(
    ComputeCallback<dynamic, T> callback,
    dynamic message,
  ) async {
    return compute(callback, message);
  }
}

// Image caching configuration
class ImageCacheConfig {
  static void configure() {
    // Increase image cache size
    PaintingBinding.instance.imageCache.maximumSize = 200;
    PaintingBinding.instance.imageCache.maximumSizeBytes = 100 << 20; // 100 MB
  }
}

// Network request batching
class RequestBatcher {
  final Duration batchWindow;
  final int maxBatchSize;

  final List<_BatchedRequest> _pendingRequests = [];
  Timer? _batchTimer;

  RequestBatcher({
    this.batchWindow = const Duration(milliseconds: 100),
    this.maxBatchSize = 10,
  });

  Future<T> addRequest<T>(Future<T> Function() request) {
    final completer = Completer<T>();

    _pendingRequests.add(_BatchedRequest(
      execute: () async {
        try {
          final result = await request();
          completer.complete(result);
        } catch (e) {
          completer.completeError(e);
        }
      },
    ));

    if (_pendingRequests.length >= maxBatchSize) {
      _executeBatch();
    } else {
      _batchTimer?.cancel();
      _batchTimer = Timer(batchWindow, _executeBatch);
    }

    return completer.future;
  }

  void _executeBatch() {
    final requests = List<_BatchedRequest>.from(_pendingRequests);
    _pendingRequests.clear();

    for (final request in requests) {
      request.execute();
    }
  }
}

class _BatchedRequest {
  final Future<void> Function() execute;
  _BatchedRequest({required this.execute});
}
```

---

## Sprint 5 & 6 Exit Criteria

### Sprint 5 Exit Criteria (Day 63)

| Criteria | Target | Actual |
|----------|--------|--------|
| Unit test coverage | > 70% | ___ |
| Integration test pass rate | 100% | ___ |
| P0 bugs | 0 | ___ |
| P1 bugs | 0 | ___ |
| P2 bugs | < 10 | ___ |
| Code review complete | Yes | ___ |
| Security audit pass | Yes | ___ |

### Sprint 6 Exit Criteria (Day 77)

| Criteria | Target | Actual |
|----------|--------|--------|
| Beta families active | 20 | ___ |
| Beta NPS score | > 40 | ___ |
| Beta satisfaction (1-5) | > 4.0 | ___ |
| Critical beta bugs fixed | 100% | ___ |
| UI polish complete | Yes | ___ |
| Performance targets met | Yes | ___ |
| Documentation complete | Yes | ___ |

---

## Files Created/Modified in Sprint 5 & 6

| File | Type | Purpose |
|------|------|---------|
| `app/build.gradle.kts` | Modified | Add test dependencies |
| `app/src/test/**/*Test.kt` | Created | Unit tests |
| `app/src/androidTest/**/*Test.kt` | Created | Integration tests |
| `pubspec.yaml` | Modified | Add test dependencies |
| `test/**/*_test.dart` | Created | Unit tests |
| `integration_test/**/*.dart` | Created | Integration tests |
| `lib/core/theme/app_theme.dart` | Created | Theme configuration |
| `lib/core/widgets/error_state_widget.dart` | Created | Error/empty states |
| `lib/core/utils/performance_utils.dart` | Created | Performance utilities |
| `setup_beta_device.sh` | Created | Device setup script |

---

## Next: Sprint 7 - Production Readiness

Continue to [07_sprint_7_production.md](./07_sprint_7_production.md) for production deployment preparation.
