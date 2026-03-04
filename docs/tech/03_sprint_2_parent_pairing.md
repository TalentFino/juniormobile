# Sprint 2: Parent App & Device Pairing (Days 18-27)

## Sprint Goal
Build the parent companion app with authentication and implement device pairing between parent app and child device.

---

## Epic E04: Parent App Foundation

### Story E04-S01: Flutter Project Setup
**Points**: 3
**Owner**: Flutter Developer
**Priority**: P0

#### Description
Initialize Flutter project with proper architecture and dependencies.

#### Acceptance Criteria
- [ ] Project compiles for Android and iOS
- [ ] All dependencies installed
- [ ] Firebase connected
- [ ] Basic navigation working

#### Tasks

##### Task E04-S01-T01: Create Flutter Project
**Time**: 30 minutes

```bash
cd ~/projects/kidtunes/flutter
flutter create parent_app --org com.kidtunes
cd parent_app
```

##### Task E04-S01-T02: Configure pubspec.yaml
**Time**: 30 minutes

Update `pubspec.yaml`:

```yaml
name: kidtunes_parent
description: Parent companion app for KidTunes kid-safe phone.
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: '>=3.3.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter

  # Firebase
  firebase_core: ^2.27.0
  firebase_auth: ^4.17.0
  cloud_firestore: ^4.15.0
  firebase_database: ^10.4.0
  firebase_messaging: ^14.7.0
  firebase_crashlytics: ^3.4.0
  firebase_analytics: ^10.8.0

  # State Management
  flutter_riverpod: ^2.5.0
  riverpod_annotation: ^2.3.4

  # Navigation
  go_router: ^13.2.0

  # UI
  cupertino_icons: ^1.0.6
  flutter_svg: ^2.0.10
  cached_network_image: ^3.3.1
  shimmer: ^3.0.0
  fl_chart: ^0.66.0
  google_maps_flutter: ^2.5.3

  # Utilities
  mobile_scanner: ^4.0.1
  geolocator: ^11.0.0
  geocoding: ^2.2.0
  intl: ^0.19.0
  timeago: ^3.6.1
  uuid: ^4.3.3

  # Storage
  shared_preferences: ^2.2.2
  flutter_secure_storage: ^9.0.0

  # Notifications
  flutter_local_notifications: ^17.0.0

  # Permissions
  permission_handler: ^11.3.0

  # Fonts
  google_fonts: ^6.1.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.1
  riverpod_generator: ^2.3.11
  build_runner: ^2.4.8
  json_serializable: ^6.7.1
  freezed: ^2.4.7
  freezed_annotation: ^2.4.1

flutter:
  uses-material-design: true

  assets:
    - assets/images/
    - assets/icons/

  fonts:
    - family: Roboto
      fonts:
        - asset: assets/fonts/Roboto-Regular.ttf
        - asset: assets/fonts/Roboto-Medium.ttf
          weight: 500
        - asset: assets/fonts/Roboto-Bold.ttf
          weight: 700
```

Run `flutter pub get`.

##### Task E04-S01-T03: Configure Firebase for Flutter
**Time**: 45 minutes

```bash
# Install FlutterFire CLI if not installed
dart pub global activate flutterfire_cli

# Configure Firebase
flutterfire configure --project=kidtunes-prod

# This will:
# - Create lib/firebase_options.dart
# - Update android/app/google-services.json
# - Update ios/Runner/GoogleService-Info.plist
```

##### Task E04-S01-T04: Create Project Structure
**Time**: 30 minutes

```bash
cd lib
mkdir -p app
mkdir -p core/{constants,theme,extensions,utils}
mkdir -p data/{models,repositories,services}
mkdir -p presentation/{providers,screens,widgets}
mkdir -p l10n

# Create placeholder files
touch app/app.dart
touch app/router.dart
touch core/constants/app_constants.dart
touch core/theme/app_theme.dart
touch main.dart
```

##### Task E04-S01-T05: Create Main Entry Point
**Time**: 30 minutes

Create `lib/main.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_crashlytics/firebase_crashlytics.dart';

import 'firebase_options.dart';
import 'app/app.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Initialize Firebase
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );

  // Configure Crashlytics
  FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterFatalError;

  // Set preferred orientations
  await SystemChrome.setPreferredOrientations([
    DeviceOrientation.portraitUp,
    DeviceOrientation.portraitDown,
  ]);

  // Set system UI overlay style
  SystemChrome.setSystemUIOverlayStyle(
    const SystemUiOverlayStyle(
      statusBarColor: Colors.transparent,
      statusBarIconBrightness: Brightness.dark,
    ),
  );

  runApp(
    const ProviderScope(
      child: KidTunesParentApp(),
    ),
  );
}
```

##### Task E04-S01-T06: Create App Widget
**Time**: 30 minutes

Create `lib/app/app.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../core/theme/app_theme.dart';
import 'router.dart';

class KidTunesParentApp extends ConsumerWidget {
  const KidTunesParentApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);

    return MaterialApp.router(
      title: 'KidTunes Parent',
      debugShowCheckedModeBanner: false,
      theme: AppTheme.lightTheme,
      darkTheme: AppTheme.darkTheme,
      themeMode: ThemeMode.system,
      routerConfig: router,
    );
  }
}
```

##### Task E04-S01-T07: Create App Theme
**Time**: 45 minutes

Create `lib/core/theme/app_theme.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';

class AppTheme {
  static const Color primaryColor = Color(0xFF4A90E2);
  static const Color primaryDark = Color(0xFF357ABD);
  static const Color primaryLight = Color(0xFF7BB8F5);

  static const Color successGreen = Color(0xFF4CAF50);
  static const Color warningOrange = Color(0xFFFF9800);
  static const Color errorRed = Color(0xFFFF4444);

  static const Color textPrimary = Color(0xFF333333);
  static const Color textSecondary = Color(0xFF666666);
  static const Color textHint = Color(0xFF999999);

  static const Color backgroundLight = Color(0xFFF5F5F5);
  static const Color surfaceLight = Color(0xFFFFFFFF);

  static const Color backgroundDark = Color(0xFF121212);
  static const Color surfaceDark = Color(0xFF1E1E1E);

  static ThemeData get lightTheme {
    return ThemeData(
      useMaterial3: true,
      brightness: Brightness.light,
      primaryColor: primaryColor,
      scaffoldBackgroundColor: backgroundLight,
      colorScheme: ColorScheme.light(
        primary: primaryColor,
        secondary: primaryLight,
        surface: surfaceLight,
        background: backgroundLight,
        error: errorRed,
      ),
      textTheme: GoogleFonts.robotoTextTheme().apply(
        bodyColor: textPrimary,
        displayColor: textPrimary,
      ),
      appBarTheme: AppBarTheme(
        elevation: 0,
        centerTitle: true,
        backgroundColor: surfaceLight,
        foregroundColor: textPrimary,
        titleTextStyle: GoogleFonts.roboto(
          fontSize: 18,
          fontWeight: FontWeight.w600,
          color: textPrimary,
        ),
      ),
      cardTheme: CardTheme(
        elevation: 2,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(16),
        ),
      ),
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: primaryColor,
          foregroundColor: Colors.white,
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 14),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(12),
          ),
          textStyle: GoogleFonts.roboto(
            fontSize: 16,
            fontWeight: FontWeight.w600,
          ),
        ),
      ),
      inputDecorationTheme: InputDecorationTheme(
        filled: true,
        fillColor: Colors.white,
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: BorderSide.none,
        ),
        enabledBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: BorderSide(color: Colors.grey.shade300),
        ),
        focusedBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: const BorderSide(color: primaryColor, width: 2),
        ),
        contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 14),
      ),
    );
  }

  static ThemeData get darkTheme {
    return ThemeData(
      useMaterial3: true,
      brightness: Brightness.dark,
      primaryColor: primaryColor,
      scaffoldBackgroundColor: backgroundDark,
      colorScheme: ColorScheme.dark(
        primary: primaryColor,
        secondary: primaryLight,
        surface: surfaceDark,
        background: backgroundDark,
        error: errorRed,
      ),
      textTheme: GoogleFonts.robotoTextTheme(ThemeData.dark().textTheme),
    );
  }
}
```

##### Task E04-S01-T08: Create Router
**Time**: 45 minutes

Create `lib/app/router.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

import '../presentation/screens/splash/splash_screen.dart';
import '../presentation/screens/auth/login_screen.dart';
import '../presentation/screens/auth/register_screen.dart';
import '../presentation/screens/auth/verify_otp_screen.dart';
import '../presentation/screens/onboarding/onboarding_screen.dart';
import '../presentation/screens/onboarding/add_child_screen.dart';
import '../presentation/screens/onboarding/pair_device_screen.dart';
import '../presentation/screens/dashboard/dashboard_screen.dart';
import '../presentation/screens/device_detail/device_detail_screen.dart';
import '../presentation/screens/app_management/app_management_screen.dart';
import '../presentation/screens/screen_time/screen_time_screen.dart';
import '../presentation/screens/location/location_screen.dart';
import '../presentation/screens/usage_reports/usage_reports_screen.dart';
import '../presentation/screens/settings/settings_screen.dart';
import '../presentation/providers/auth_provider.dart';

final routerProvider = Provider<GoRouter>((ref) {
  final authState = ref.watch(authStateProvider);

  return GoRouter(
    initialLocation: '/splash',
    debugLogDiagnostics: true,
    redirect: (context, state) {
      // CRITICAL FIX: Handle loading state properly
      // Without this, authState.valueOrNull returns null during loading,
      // causing an unwanted redirect to login screen
      final isLoading = authState.isLoading;
      final hasError = authState.hasError;
      final isLoggedIn = authState.valueOrNull != null;

      final currentPath = state.matchedLocation;
      final isOnAuthRoute = currentPath.startsWith('/auth');
      final isOnSplash = currentPath == '/splash';

      // Stay on splash while auth state is loading
      if (isLoading) {
        return isOnSplash ? null : '/splash';
      }

      // On error, redirect to splash (splash will handle showing error)
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
      GoRoute(
        path: '/splash',
        builder: (context, state) => const SplashScreen(),
      ),

      // Auth routes
      GoRoute(
        path: '/auth/login',
        builder: (context, state) => const LoginScreen(),
      ),
      GoRoute(
        path: '/auth/register',
        builder: (context, state) => const RegisterScreen(),
      ),
      GoRoute(
        path: '/auth/verify-otp',
        builder: (context, state) {
          final extra = state.extra as Map<String, dynamic>?;
          return VerifyOtpScreen(
            phoneNumber: extra?['phoneNumber'] ?? '',
            verificationId: extra?['verificationId'] ?? '',
          );
        },
      ),

      // Onboarding routes
      GoRoute(
        path: '/onboarding',
        builder: (context, state) => const OnboardingScreen(),
      ),
      GoRoute(
        path: '/onboarding/add-child',
        builder: (context, state) => const AddChildScreen(),
      ),
      GoRoute(
        path: '/onboarding/pair-device',
        builder: (context, state) {
          final extra = state.extra as Map<String, dynamic>?;
          return PairDeviceScreen(childId: extra?['childId'] ?? '');
        },
      ),

      // Main app routes
      GoRoute(
        path: '/dashboard',
        builder: (context, state) => const DashboardScreen(),
      ),
      GoRoute(
        path: '/device/:deviceId',
        builder: (context, state) {
          return DeviceDetailScreen(deviceId: state.pathParameters['deviceId']!);
        },
      ),
      GoRoute(
        path: '/device/:deviceId/apps',
        builder: (context, state) {
          return AppManagementScreen(deviceId: state.pathParameters['deviceId']!);
        },
      ),
      GoRoute(
        path: '/device/:deviceId/screen-time',
        builder: (context, state) {
          return ScreenTimeScreen(deviceId: state.pathParameters['deviceId']!);
        },
      ),
      GoRoute(
        path: '/device/:deviceId/location',
        builder: (context, state) {
          return LocationScreen(deviceId: state.pathParameters['deviceId']!);
        },
      ),
      GoRoute(
        path: '/device/:deviceId/reports',
        builder: (context, state) {
          return UsageReportsScreen(deviceId: state.pathParameters['deviceId']!);
        },
      ),
      GoRoute(
        path: '/settings',
        builder: (context, state) => const SettingsScreen(),
      ),
    ],
  );
});
```

---

### Story E04-S02: Authentication Screens
**Points**: 5
**Owner**: Flutter Developer
**Priority**: P0

#### Description
Create login, registration, and OTP verification screens.

#### Acceptance Criteria
- [ ] Phone number login with OTP
- [ ] Email/password login
- [ ] Registration flow
- [ ] Error handling
- [ ] Loading states

#### Tasks

##### Task E04-S02-T01: Create Auth Repository
**Time**: 1 hour

Create `lib/data/repositories/auth_repository.dart`:

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import '../models/user_model.dart';

class AuthRepository {
  final FirebaseAuth _auth = FirebaseAuth.instance;
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  User? get currentUser => _auth.currentUser;
  Stream<User?> get authStateChanges => _auth.authStateChanges();

  // Phone Authentication
  Future<void> verifyPhoneNumber({
    required String phoneNumber,
    required Function(String verificationId) onCodeSent,
    required Function(PhoneAuthCredential credential) onVerificationCompleted,
    required Function(FirebaseAuthException e) onVerificationFailed,
    int? resendToken,
  }) async {
    await _auth.verifyPhoneNumber(
      phoneNumber: phoneNumber,
      timeout: const Duration(seconds: 60),
      verificationCompleted: onVerificationCompleted,
      verificationFailed: onVerificationFailed,
      codeSent: (verificationId, resendToken) {
        onCodeSent(verificationId);
      },
      codeAutoRetrievalTimeout: (verificationId) {},
      forceResendingToken: resendToken,
    );
  }

  Future<UserCredential> signInWithOTP({
    required String verificationId,
    required String otp,
  }) async {
    final credential = PhoneAuthProvider.credential(
      verificationId: verificationId,
      smsCode: otp,
    );
    return await _auth.signInWithCredential(credential);
  }

  // Email Authentication
  Future<UserCredential> signInWithEmail({
    required String email,
    required String password,
  }) async {
    return await _auth.signInWithEmailAndPassword(
      email: email,
      password: password,
    );
  }

  Future<UserCredential> registerWithEmail({
    required String email,
    required String password,
    required String name,
  }) async {
    final credential = await _auth.createUserWithEmailAndPassword(
      email: email,
      password: password,
    );

    await credential.user?.updateDisplayName(name);

    // Create user document in Firestore
    await _createUserDocument(
      uid: credential.user!.uid,
      email: email,
      name: name,
    );

    return credential;
  }

  Future<void> _createUserDocument({
    required String uid,
    required String email,
    String? name,
    String? phone,
  }) async {
    final userDoc = _firestore.collection('users').doc(uid);

    await userDoc.set({
      'email': email,
      'name': name,
      'phone': phone,
      'createdAt': FieldValue.serverTimestamp(),
      'subscription': {
        'plan': 'free',
        'expiresAt': null,
      },
    });
  }

  Future<void> signOut() async {
    await _auth.signOut();
  }

  Future<void> resetPassword(String email) async {
    await _auth.sendPasswordResetEmail(email: email);
  }

  Future<UserModel?> getUserProfile() async {
    final user = currentUser;
    if (user == null) return null;

    final doc = await _firestore.collection('users').doc(user.uid).get();
    if (!doc.exists) return null;

    return UserModel.fromFirestore(doc);
  }
}
```

##### Task E04-S02-T02: Create Auth Provider
**Time**: 45 minutes

Create `lib/presentation/providers/auth_provider.dart`:

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../data/repositories/auth_repository.dart';

final authRepositoryProvider = Provider<AuthRepository>((ref) {
  return AuthRepository();
});

final authStateProvider = StreamProvider<User?>((ref) {
  return ref.watch(authRepositoryProvider).authStateChanges;
});

final currentUserProvider = Provider<User?>((ref) {
  return ref.watch(authStateProvider).valueOrNull;
});

// Phone Auth State
class PhoneAuthState {
  final bool isLoading;
  final String? verificationId;
  final String? error;
  final int? resendToken;

  const PhoneAuthState({
    this.isLoading = false,
    this.verificationId,
    this.error,
    this.resendToken,
  });

  PhoneAuthState copyWith({
    bool? isLoading,
    String? verificationId,
    String? error,
    int? resendToken,
  }) {
    return PhoneAuthState(
      isLoading: isLoading ?? this.isLoading,
      verificationId: verificationId ?? this.verificationId,
      error: error,
      resendToken: resendToken ?? this.resendToken,
    );
  }
}

class PhoneAuthNotifier extends StateNotifier<PhoneAuthState> {
  final AuthRepository _authRepository;

  PhoneAuthNotifier(this._authRepository) : super(const PhoneAuthState());

  Future<void> sendOTP(String phoneNumber) async {
    state = state.copyWith(isLoading: true, error: null);

    await _authRepository.verifyPhoneNumber(
      phoneNumber: phoneNumber,
      onCodeSent: (verificationId) {
        state = state.copyWith(
          isLoading: false,
          verificationId: verificationId,
        );
      },
      onVerificationCompleted: (credential) async {
        // Auto-verification (rare on most devices)
        await _authRepository._auth.signInWithCredential(credential);
        state = state.copyWith(isLoading: false);
      },
      onVerificationFailed: (e) {
        state = state.copyWith(
          isLoading: false,
          error: e.message ?? 'Verification failed',
        );
      },
      resendToken: state.resendToken,
    );
  }

  Future<void> verifyOTP(String otp) async {
    if (state.verificationId == null) {
      state = state.copyWith(error: 'Please request OTP first');
      return;
    }

    state = state.copyWith(isLoading: true, error: null);

    try {
      await _authRepository.signInWithOTP(
        verificationId: state.verificationId!,
        otp: otp,
      );
      state = state.copyWith(isLoading: false);
    } on FirebaseAuthException catch (e) {
      state = state.copyWith(
        isLoading: false,
        error: e.message ?? 'Invalid OTP',
      );
    }
  }

  void reset() {
    state = const PhoneAuthState();
  }
}

final phoneAuthProvider =
    StateNotifierProvider<PhoneAuthNotifier, PhoneAuthState>((ref) {
  return PhoneAuthNotifier(ref.watch(authRepositoryProvider));
});
```

##### Task E04-S02-T03: Create Login Screen
**Time**: 1.5 hours

Create `lib/presentation/screens/auth/login_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

import '../../../core/theme/app_theme.dart';
import '../../providers/auth_provider.dart';
import '../../widgets/loading_button.dart';

class LoginScreen extends ConsumerStatefulWidget {
  const LoginScreen({super.key});

  @override
  ConsumerState<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends ConsumerState<LoginScreen> {
  final _phoneController = TextEditingController();
  final _formKey = GlobalKey<FormState>();
  bool _isPhoneMode = true;

  @override
  void dispose() {
    _phoneController.dispose();
    super.dispose();
  }

  void _submitPhone() {
    if (!_formKey.currentState!.validate()) return;

    final phone = '+91${_phoneController.text.trim()}';
    ref.read(phoneAuthProvider.notifier).sendOTP(phone);
  }

  @override
  Widget build(BuildContext context) {
    final phoneAuthState = ref.watch(phoneAuthProvider);

    ref.listen<PhoneAuthState>(phoneAuthProvider, (previous, next) {
      if (next.verificationId != null && previous?.verificationId == null) {
        context.push('/auth/verify-otp', extra: {
          'phoneNumber': '+91${_phoneController.text.trim()}',
          'verificationId': next.verificationId,
        });
      }

      if (next.error != null) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(next.error!), backgroundColor: Colors.red),
        );
      }
    });

    return Scaffold(
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(24),
          child: Form(
            key: _formKey,
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                const SizedBox(height: 60),

                // Logo
                Center(
                  child: Container(
                    width: 100,
                    height: 100,
                    decoration: BoxDecoration(
                      color: AppTheme.primaryColor,
                      borderRadius: BorderRadius.circular(24),
                    ),
                    child: const Icon(
                      Icons.child_care,
                      size: 60,
                      color: Colors.white,
                    ),
                  ),
                ),

                const SizedBox(height: 32),

                // Title
                Text(
                  'Welcome to KidTunes',
                  style: Theme.of(context).textTheme.headlineMedium?.copyWith(
                    fontWeight: FontWeight.bold,
                  ),
                  textAlign: TextAlign.center,
                ),

                const SizedBox(height: 8),

                Text(
                  'Sign in to manage your child\'s device',
                  style: Theme.of(context).textTheme.bodyLarge?.copyWith(
                    color: AppTheme.textSecondary,
                  ),
                  textAlign: TextAlign.center,
                ),

                const SizedBox(height: 48),

                // Phone Input
                TextFormField(
                  controller: _phoneController,
                  keyboardType: TextInputType.phone,
                  inputFormatters: [
                    FilteringTextInputFormatter.digitsOnly,
                    LengthLimitingTextInputFormatter(10),
                  ],
                  decoration: InputDecoration(
                    labelText: 'Phone Number',
                    hintText: '9876543210',
                    prefixIcon: const Icon(Icons.phone),
                    prefixText: '+91 ',
                  ),
                  validator: (value) {
                    if (value == null || value.length != 10) {
                      return 'Enter valid 10-digit phone number';
                    }
                    return null;
                  },
                ),

                const SizedBox(height: 24),

                // Submit Button
                LoadingButton(
                  onPressed: _submitPhone,
                  isLoading: phoneAuthState.isLoading,
                  label: 'Send OTP',
                ),

                const SizedBox(height: 24),

                // Divider
                Row(
                  children: [
                    const Expanded(child: Divider()),
                    Padding(
                      padding: const EdgeInsets.symmetric(horizontal: 16),
                      child: Text(
                        'OR',
                        style: TextStyle(color: AppTheme.textSecondary),
                      ),
                    ),
                    const Expanded(child: Divider()),
                  ],
                ),

                const SizedBox(height: 24),

                // Email Login
                OutlinedButton.icon(
                  onPressed: () => context.push('/auth/register'),
                  icon: const Icon(Icons.email),
                  label: const Text('Continue with Email'),
                  style: OutlinedButton.styleFrom(
                    padding: const EdgeInsets.symmetric(vertical: 14),
                  ),
                ),

                const SizedBox(height: 32),

                // Terms
                Text(
                  'By continuing, you agree to our Terms of Service and Privacy Policy',
                  style: Theme.of(context).textTheme.bodySmall?.copyWith(
                    color: AppTheme.textHint,
                  ),
                  textAlign: TextAlign.center,
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

##### Task E04-S02-T04: Create OTP Verification Screen
**Time**: 1 hour

Create `lib/presentation/screens/auth/verify_otp_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

import '../../../core/theme/app_theme.dart';
import '../../providers/auth_provider.dart';
import '../../widgets/loading_button.dart';

class VerifyOtpScreen extends ConsumerStatefulWidget {
  final String phoneNumber;
  final String verificationId;

  const VerifyOtpScreen({
    super.key,
    required this.phoneNumber,
    required this.verificationId,
  });

  @override
  ConsumerState<VerifyOtpScreen> createState() => _VerifyOtpScreenState();
}

class _VerifyOtpScreenState extends ConsumerState<VerifyOtpScreen> {
  final List<TextEditingController> _otpControllers = List.generate(
    6,
    (index) => TextEditingController(),
  );
  final List<FocusNode> _focusNodes = List.generate(6, (index) => FocusNode());

  @override
  void dispose() {
    for (final controller in _otpControllers) {
      controller.dispose();
    }
    for (final node in _focusNodes) {
      node.dispose();
    }
    super.dispose();
  }

  String get _otp => _otpControllers.map((c) => c.text).join();

  void _onOtpChanged(int index, String value) {
    if (value.isNotEmpty && index < 5) {
      _focusNodes[index + 1].requestFocus();
    }

    if (_otp.length == 6) {
      _verifyOtp();
    }
  }

  void _verifyOtp() {
    ref.read(phoneAuthProvider.notifier).verifyOTP(_otp);
  }

  @override
  Widget build(BuildContext context) {
    final authState = ref.watch(phoneAuthProvider);

    ref.listen(authStateProvider, (previous, next) {
      if (next.valueOrNull != null) {
        context.go('/onboarding');
      }
    });

    return Scaffold(
      appBar: AppBar(
        title: const Text('Verify OTP'),
      ),
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              Text(
                'Enter verification code',
                style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                  fontWeight: FontWeight.bold,
                ),
              ),

              const SizedBox(height: 8),

              Text(
                'We sent a 6-digit code to ${widget.phoneNumber}',
                style: TextStyle(color: AppTheme.textSecondary),
              ),

              const SizedBox(height: 32),

              // OTP Input Fields
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                children: List.generate(6, (index) {
                  return SizedBox(
                    width: 48,
                    height: 56,
                    child: TextField(
                      controller: _otpControllers[index],
                      focusNode: _focusNodes[index],
                      keyboardType: TextInputType.number,
                      textAlign: TextAlign.center,
                      maxLength: 1,
                      style: const TextStyle(
                        fontSize: 24,
                        fontWeight: FontWeight.bold,
                      ),
                      decoration: InputDecoration(
                        counterText: '',
                        border: OutlineInputBorder(
                          borderRadius: BorderRadius.circular(12),
                        ),
                      ),
                      inputFormatters: [
                        FilteringTextInputFormatter.digitsOnly,
                      ],
                      onChanged: (value) => _onOtpChanged(index, value),
                    ),
                  );
                }),
              ),

              const SizedBox(height: 32),

              LoadingButton(
                onPressed: _verifyOtp,
                isLoading: authState.isLoading,
                label: 'Verify',
              ),

              const SizedBox(height: 24),

              // Resend OTP
              TextButton(
                onPressed: () {
                  ref.read(phoneAuthProvider.notifier).sendOTP(widget.phoneNumber);
                },
                child: const Text('Didn\'t receive code? Resend'),
              ),

              if (authState.error != null) ...[
                const SizedBox(height: 16),
                Text(
                  authState.error!,
                  style: const TextStyle(color: Colors.red),
                  textAlign: TextAlign.center,
                ),
              ],
            ],
          ),
        ),
      ),
    );
  }
}
```

##### Task E04-S02-T05: Create Loading Button Widget
**Time**: 20 minutes

Create `lib/presentation/widgets/loading_button.dart`:

```dart
import 'package:flutter/material.dart';

class LoadingButton extends StatelessWidget {
  final VoidCallback onPressed;
  final bool isLoading;
  final String label;
  final IconData? icon;

  const LoadingButton({
    super.key,
    required this.onPressed,
    required this.isLoading,
    required this.label,
    this.icon,
  });

  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: isLoading ? null : onPressed,
      child: isLoading
          ? const SizedBox(
              height: 20,
              width: 20,
              child: CircularProgressIndicator(
                strokeWidth: 2,
                valueColor: AlwaysStoppedAnimation<Color>(Colors.white),
              ),
            )
          : Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                if (icon != null) ...[
                  Icon(icon),
                  const SizedBox(width: 8),
                ],
                Text(label),
              ],
            ),
    );
  }
}
```

---

### Story E04-S03: Onboarding Flow
**Points**: 5
**Owner**: Flutter Developer
**Priority**: P0

#### Description
Create the onboarding screens for adding child profile and selecting protection level.

#### Acceptance Criteria
- [ ] Welcome screen after login
- [ ] Add child profile (name, age, avatar)
- [ ] Protection level selection
- [ ] Save to Firestore

#### Tasks

##### Task E04-S03-T01: Create Data Models
**Time**: 1 hour

Create `lib/data/models/child_profile.dart`:

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

enum ProtectionLevel {
  maximum,  // Ages 5-8
  balanced, // Ages 9-12
  trust,    // Ages 13+
}

class ChildProfile {
  final String id;
  final String name;
  final int age;
  final String avatar;
  final ProtectionLevel protectionLevel;
  final String? deviceId;
  final DateTime createdAt;

  const ChildProfile({
    required this.id,
    required this.name,
    required this.age,
    required this.avatar,
    required this.protectionLevel,
    this.deviceId,
    required this.createdAt,
  });

  factory ChildProfile.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return ChildProfile(
      id: doc.id,
      name: data['name'] ?? '',
      age: data['age'] ?? 0,
      avatar: data['avatar'] ?? 'default',
      protectionLevel: ProtectionLevel.values.firstWhere(
        (e) => e.name == data['protectionLevel'],
        orElse: () => ProtectionLevel.balanced,
      ),
      deviceId: data['deviceId'],
      createdAt: (data['createdAt'] as Timestamp?)?.toDate() ?? DateTime.now(),
    );
  }

  Map<String, dynamic> toFirestore() {
    return {
      'name': name,
      'age': age,
      'avatar': avatar,
      'protectionLevel': protectionLevel.name,
      'deviceId': deviceId,
      'createdAt': Timestamp.fromDate(createdAt),
    };
  }

  ChildProfile copyWith({
    String? id,
    String? name,
    int? age,
    String? avatar,
    ProtectionLevel? protectionLevel,
    String? deviceId,
    DateTime? createdAt,
  }) {
    return ChildProfile(
      id: id ?? this.id,
      name: name ?? this.name,
      age: age ?? this.age,
      avatar: avatar ?? this.avatar,
      protectionLevel: protectionLevel ?? this.protectionLevel,
      deviceId: deviceId ?? this.deviceId,
      createdAt: createdAt ?? this.createdAt,
    );
  }
}
```

Create `lib/data/models/device.dart`:

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

class Device {
  final String id;
  final String ownerId;
  final String childId;
  final String model;
  final String androidVersion;
  final String appVersion;
  final DateTime registeredAt;
  final DeviceSettings settings;
  final List<String> whitelist;

  const Device({
    required this.id,
    required this.ownerId,
    required this.childId,
    required this.model,
    required this.androidVersion,
    required this.appVersion,
    required this.registeredAt,
    required this.settings,
    required this.whitelist,
  });

  factory Device.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Device(
      id: doc.id,
      ownerId: data['ownerId'] ?? '',
      childId: data['childId'] ?? '',
      model: data['model'] ?? '',
      androidVersion: data['androidVersion'] ?? '',
      appVersion: data['appVersion'] ?? '',
      registeredAt: (data['registeredAt'] as Timestamp?)?.toDate() ?? DateTime.now(),
      settings: DeviceSettings.fromMap(data['settings'] ?? {}),
      whitelist: List<String>.from(data['whitelist'] ?? []),
    );
  }

  Map<String, dynamic> toFirestore() {
    return {
      'ownerId': ownerId,
      'childId': childId,
      'model': model,
      'androidVersion': androidVersion,
      'appVersion': appVersion,
      'registeredAt': Timestamp.fromDate(registeredAt),
      'settings': settings.toMap(),
      'whitelist': whitelist,
    };
  }
}

class DeviceSettings {
  final bool kidsMode;
  final int volumeLimit;
  final int dailyLimitMinutes;
  final String? bedtimeStart;
  final String? bedtimeEnd;
  final bool locationEnabled;

  const DeviceSettings({
    this.kidsMode = true,
    this.volumeLimit = 70,
    this.dailyLimitMinutes = 120,
    this.bedtimeStart,
    this.bedtimeEnd,
    this.locationEnabled = false,
  });

  factory DeviceSettings.fromMap(Map<String, dynamic> map) {
    return DeviceSettings(
      kidsMode: map['kidsMode'] ?? true,
      volumeLimit: map['volumeLimit'] ?? 70,
      dailyLimitMinutes: map['dailyLimitMinutes'] ?? 120,
      bedtimeStart: map['bedtimeStart'],
      bedtimeEnd: map['bedtimeEnd'],
      locationEnabled: map['locationEnabled'] ?? false,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'kidsMode': kidsMode,
      'volumeLimit': volumeLimit,
      'dailyLimitMinutes': dailyLimitMinutes,
      'bedtimeStart': bedtimeStart,
      'bedtimeEnd': bedtimeEnd,
      'locationEnabled': locationEnabled,
    };
  }
}
```

##### Task E04-S03-T02: Create Add Child Screen
**Time**: 1.5 hours

Create `lib/presentation/screens/onboarding/add_child_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

import '../../../core/theme/app_theme.dart';
import '../../../data/models/child_profile.dart';
import '../../providers/child_provider.dart';
import '../../widgets/loading_button.dart';

class AddChildScreen extends ConsumerStatefulWidget {
  const AddChildScreen({super.key});

  @override
  ConsumerState<AddChildScreen> createState() => _AddChildScreenState();
}

class _AddChildScreenState extends ConsumerState<AddChildScreen> {
  final _nameController = TextEditingController();
  int _selectedAge = 8;
  String _selectedAvatar = 'boy_1';
  ProtectionLevel _protectionLevel = ProtectionLevel.balanced;
  bool _isLoading = false;

  final List<String> _avatars = [
    'boy_1', 'girl_1', 'boy_2', 'girl_2', 'boy_3', 'girl_3',
  ];

  @override
  void dispose() {
    _nameController.dispose();
    super.dispose();
  }

  Future<void> _saveChild() async {
    if (_nameController.text.trim().isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Please enter child\'s name')),
      );
      return;
    }

    setState(() => _isLoading = true);

    try {
      final childId = await ref.read(childRepositoryProvider).createChild(
        name: _nameController.text.trim(),
        age: _selectedAge,
        avatar: _selectedAvatar,
        protectionLevel: _protectionLevel,
      );

      if (mounted) {
        context.push('/onboarding/pair-device', extra: {'childId': childId});
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Error: $e')),
        );
      }
    } finally {
      if (mounted) {
        setState(() => _isLoading = false);
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Add Child'),
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(24),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // Progress indicator
            LinearProgressIndicator(
              value: 0.33,
              backgroundColor: Colors.grey.shade200,
              valueColor: const AlwaysStoppedAnimation(AppTheme.primaryColor),
            ),

            const SizedBox(height: 32),

            Text(
              'Who is this device for?',
              style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                fontWeight: FontWeight.bold,
              ),
            ),

            const SizedBox(height: 32),

            // Name Input
            TextField(
              controller: _nameController,
              decoration: const InputDecoration(
                labelText: 'Child\'s Name',
                hintText: 'Enter name',
                prefixIcon: Icon(Icons.person),
              ),
              textCapitalization: TextCapitalization.words,
            ),

            const SizedBox(height: 24),

            // Age Selector
            Text(
              'Age',
              style: Theme.of(context).textTheme.titleMedium,
            ),
            const SizedBox(height: 8),
            SingleChildScrollView(
              scrollDirection: Axis.horizontal,
              child: Row(
                children: List.generate(12, (index) {
                  final age = index + 5; // 5-16
                  final isSelected = age == _selectedAge;
                  return Padding(
                    padding: const EdgeInsets.only(right: 8),
                    child: ChoiceChip(
                      label: Text('$age'),
                      selected: isSelected,
                      onSelected: (_) => setState(() => _selectedAge = age),
                    ),
                  );
                }),
              ),
            ),

            const SizedBox(height: 24),

            // Avatar Selector
            Text(
              'Choose an avatar',
              style: Theme.of(context).textTheme.titleMedium,
            ),
            const SizedBox(height: 8),
            Wrap(
              spacing: 16,
              runSpacing: 16,
              children: _avatars.map((avatar) {
                final isSelected = avatar == _selectedAvatar;
                return GestureDetector(
                  onTap: () => setState(() => _selectedAvatar = avatar),
                  child: Container(
                    width: 64,
                    height: 64,
                    decoration: BoxDecoration(
                      shape: BoxShape.circle,
                      border: Border.all(
                        color: isSelected ? AppTheme.primaryColor : Colors.grey.shade300,
                        width: isSelected ? 3 : 1,
                      ),
                    ),
                    child: Center(
                      child: Text(
                        _getAvatarEmoji(avatar),
                        style: const TextStyle(fontSize: 32),
                      ),
                    ),
                  ),
                );
              }).toList(),
            ),

            const SizedBox(height: 32),

            // Protection Level
            Text(
              'Protection Level',
              style: Theme.of(context).textTheme.titleMedium,
            ),
            const SizedBox(height: 16),

            _buildProtectionOption(
              level: ProtectionLevel.maximum,
              title: '🔒 Maximum Safety',
              subtitle: 'Best for ages 5-8',
              description: 'Only pre-approved apps, strict limits',
            ),

            const SizedBox(height: 12),

            _buildProtectionOption(
              level: ProtectionLevel.balanced,
              title: '🛡️ Balanced',
              subtitle: 'Best for ages 9-12 (Recommended)',
              description: 'Core apps + parent-approved extras',
            ),

            const SizedBox(height: 12),

            _buildProtectionOption(
              level: ProtectionLevel.trust,
              title: '🔓 Trust Mode',
              subtitle: 'Best for ages 13+',
              description: 'Most apps allowed, monitoring only',
            ),

            const SizedBox(height: 32),

            LoadingButton(
              onPressed: _saveChild,
              isLoading: _isLoading,
              label: 'Continue',
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildProtectionOption({
    required ProtectionLevel level,
    required String title,
    required String subtitle,
    required String description,
  }) {
    final isSelected = _protectionLevel == level;
    return GestureDetector(
      onTap: () => setState(() => _protectionLevel = level),
      child: Container(
        padding: const EdgeInsets.all(16),
        decoration: BoxDecoration(
          border: Border.all(
            color: isSelected ? AppTheme.primaryColor : Colors.grey.shade300,
            width: isSelected ? 2 : 1,
          ),
          borderRadius: BorderRadius.circular(12),
          color: isSelected ? AppTheme.primaryColor.withOpacity(0.05) : null,
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Expanded(
                  child: Text(
                    title,
                    style: const TextStyle(
                      fontWeight: FontWeight.bold,
                      fontSize: 16,
                    ),
                  ),
                ),
                if (isSelected)
                  const Icon(Icons.check_circle, color: AppTheme.primaryColor),
              ],
            ),
            Text(
              subtitle,
              style: TextStyle(
                color: AppTheme.textSecondary,
                fontSize: 12,
              ),
            ),
            const SizedBox(height: 4),
            Text(
              description,
              style: TextStyle(
                color: AppTheme.textHint,
                fontSize: 12,
              ),
            ),
          ],
        ),
      ),
    );
  }

  String _getAvatarEmoji(String avatar) {
    switch (avatar) {
      case 'boy_1': return '👦';
      case 'girl_1': return '👧';
      case 'boy_2': return '👦🏽';
      case 'girl_2': return '👧🏻';
      case 'boy_3': return '🧒';
      case 'girl_3': return '👩';
      default: return '👤';
    }
  }
}
```

---

## Epic E05: Device Pairing

### Story E05-S01: QR Code Pairing
**Points**: 8
**Owner**: Both
**Priority**: P0

#### Description
Implement QR code-based device pairing between parent app and child device.

#### Acceptance Criteria
- [ ] Parent app generates unique QR code
- [ ] Child device can scan QR code
- [ ] Pairing token exchanged securely
- [ ] Devices linked in Firebase
- [ ] Real-time confirmation

#### Tasks

##### Task E05-S01-T00: Create Splash Screen with Loading Handling
**Time**: 30 minutes

Create `lib/presentation/screens/splash/splash_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

import '../../../core/theme/app_theme.dart';
import '../../providers/auth_provider.dart';

class SplashScreen extends ConsumerWidget {
  const SplashScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final authState = ref.watch(authStateProvider);

    // Listen for auth state changes and navigate accordingly
    ref.listen(authStateProvider, (previous, next) {
      next.when(
        data: (user) {
          if (user != null) {
            // User is logged in - check if onboarding complete
            _checkOnboardingStatus(context, ref);
          } else {
            // User is not logged in
            context.go('/auth/login');
          }
        },
        loading: () {
          // Stay on splash - do nothing
        },
        error: (error, stack) {
          // Log error and redirect to login
          debugPrint('Auth error: $error');
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
              child: const Icon(
                Icons.child_care,
                size: 80,
                color: Colors.white,
              ),
            ),
            const SizedBox(height: 24),

            // Loading indicator
            const CircularProgressIndicator(),

            const SizedBox(height: 16),

            // Status text
            if (authState.isLoading)
              const Text('Loading...')
            else if (authState.hasError)
              Text(
                'Connection error. Retrying...',
                style: TextStyle(color: AppTheme.errorRed),
              ),
          ],
        ),
      ),
    );
  }

  Future<void> _checkOnboardingStatus(BuildContext context, WidgetRef ref) async {
    // Check if user has any children profiles
    // If not, redirect to onboarding
    // Otherwise, go to dashboard
    context.go('/dashboard');
  }
}
```

##### Task E05-S01-T01: Create Pairing Repository
**Time**: 1 hour

Create `lib/data/repositories/pairing_repository.dart`:

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_database/firebase_database.dart';
import 'package:uuid/uuid.dart';

class PairingRepository {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;
  final FirebaseDatabase _rtdb = FirebaseDatabase.instance;

  /// Generate a unique pairing token
  Future<PairingToken> generatePairingToken({
    required String userId,
    required String childId,
  }) async {
    final token = const Uuid().v4();
    final expiresAt = DateTime.now().add(const Duration(minutes: 10));

    // Store token in Firestore
    await _firestore.collection('pairing_tokens').doc(token).set({
      'userId': userId,
      'childId': childId,
      'createdAt': FieldValue.serverTimestamp(),
      'expiresAt': Timestamp.fromDate(expiresAt),
      'used': false,
    });

    return PairingToken(
      token: token,
      userId: userId,
      childId: childId,
      expiresAt: expiresAt,
    );
  }

  /// Listen for pairing completion
  Stream<PairingStatus> watchPairingStatus(String token) {
    return _firestore
        .collection('pairing_tokens')
        .doc(token)
        .snapshots()
        .map((doc) {
      if (!doc.exists) {
        return PairingStatus.expired;
      }

      final data = doc.data()!;
      if (data['deviceId'] != null) {
        return PairingStatus.completed;
      }
      if (data['used'] == true) {
        return PairingStatus.inProgress;
      }

      final expiresAt = (data['expiresAt'] as Timestamp).toDate();
      if (DateTime.now().isAfter(expiresAt)) {
        return PairingStatus.expired;
      }

      return PairingStatus.waiting;
    });
  }

  /// Get device ID after pairing
  Future<String?> getDeviceIdFromToken(String token) async {
    final doc = await _firestore.collection('pairing_tokens').doc(token).get();
    return doc.data()?['deviceId'] as String?;
  }

  /// Cancel/delete pairing token
  Future<void> cancelPairing(String token) async {
    await _firestore.collection('pairing_tokens').doc(token).delete();
  }
}

class PairingToken {
  final String token;
  final String userId;
  final String childId;
  final DateTime expiresAt;

  const PairingToken({
    required this.token,
    required this.userId,
    required this.childId,
    required this.expiresAt,
  });

  Map<String, dynamic> toQRData() {
    return {
      'token': token,
      'userId': userId,
      'childId': childId,
      'expiresAt': expiresAt.toIso8601String(),
    };
  }
}

enum PairingStatus {
  waiting,
  inProgress,
  completed,
  expired,
}
```

##### Task E05-S01-T01B: Create Rate-Limited Pairing Cloud Function (SECURITY)
**Time**: 1.5 hours

Create `firebase/functions/src/pairing.ts`:

```typescript
import * as functions from 'firebase-functions';
import * as admin from 'firebase-admin';
import * as crypto from 'crypto';

// In-memory rate limiter (use Redis for production at scale)
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
    const { childId } = data;

    if (!childId) {
      throw new functions.https.HttpsError('invalid-argument', 'childId is required');
    }

    const now = Date.now();

    // Rate limiting check
    const userLimit = rateLimiter.get(userId);
    if (userLimit) {
      if (now < userLimit.resetTime) {
        if (userLimit.count >= RATE_LIMIT.MAX_REQUESTS) {
          const waitMinutes = Math.ceil((userLimit.resetTime - now) / 60000);
          throw new functions.https.HttpsError(
            'resource-exhausted',
            `Too many pairing attempts. Try again in ${waitMinutes} minutes`
          );
        }
        userLimit.count++;
      } else {
        rateLimiter.set(userId, { count: 1, resetTime: now + RATE_LIMIT.WINDOW_MS });
      }
    } else {
      rateLimiter.set(userId, { count: 1, resetTime: now + RATE_LIMIT.WINDOW_MS });
    }

    // Generate secure token (6 chars, no confusing characters)
    const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789'; // No O/0/I/1/L
    let token = '';
    const randomBytes = crypto.randomBytes(6);
    for (let i = 0; i < 6; i++) {
      token += chars[randomBytes[i] % chars.length];
    }

    const expiresAt = new Date(now + 10 * 60 * 1000); // 10 minutes

    // Store token in Firestore
    await admin.firestore().collection('pairing_tokens').doc(token).set({
      userId,
      childId,
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
    const { token, deviceId, deviceModel, androidVersion } = data;

    if (!token || !deviceId) {
      throw new functions.https.HttpsError('invalid-argument', 'token and deviceId required');
    }

    const tokenRef = admin.firestore().collection('pairing_tokens').doc(token);
    const tokenDoc = await tokenRef.get();

    if (!tokenDoc.exists) {
      throw new functions.https.HttpsError('not-found', 'Invalid pairing code');
    }

    const tokenData = tokenDoc.data()!;

    // Check if already used
    if (tokenData.used) {
      throw new functions.https.HttpsError('failed-precondition', 'Code already used');
    }

    // Check expiry
    if (tokenData.expiresAt.toDate() < new Date()) {
      throw new functions.https.HttpsError('failed-precondition', 'Code expired');
    }

    // Rate limit validation attempts (prevent brute force)
    if (tokenData.attempts >= 3) {
      throw new functions.https.HttpsError(
        'resource-exhausted',
        'Too many validation attempts for this code'
      );
    }

    // Increment attempts
    await tokenRef.update({
      attempts: admin.firestore.FieldValue.increment(1),
    });

    // Success - mark as used and create device
    const batch = admin.firestore().batch();

    // Mark token as used
    batch.update(tokenRef, {
      used: true,
      deviceId,
      completedAt: admin.firestore.FieldValue.serverTimestamp(),
    });

    // Create device document
    const deviceRef = admin.firestore().collection('devices').doc(deviceId);
    batch.set(deviceRef, {
      ownerId: tokenData.userId,
      childId: tokenData.childId,
      model: deviceModel || 'Unknown',
      androidVersion: androidVersion || 'Unknown',
      appVersion: '1.0.0',
      registeredAt: admin.firestore.FieldValue.serverTimestamp(),
      settings: {
        kidsMode: true,
        volumeLimit: 70,
        dailyLimitMinutes: 120,
        locationEnabled: false,
      },
      whitelist: [
        'com.spotify.music',
        'com.jio.saavn',
        'com.amazon.mp3',
        'com.android.deskclock',
      ],
    });

    // Update child profile with device ID
    const childRef = admin.firestore()
      .collection('users').doc(tokenData.userId)
      .collection('children').doc(tokenData.childId);
    batch.update(childRef, { deviceId });

    await batch.commit();

    return {
      success: true,
      userId: tokenData.userId,
      childId: tokenData.childId,
    };
  });
```

Update `firebase/functions/src/index.ts`:
```typescript
export { generatePairingToken, validatePairingToken } from './pairing';
```

Update parent app to use Cloud Function instead of direct Firestore:

```dart
// In PairingRepository - use Cloud Functions for rate limiting
import 'package:cloud_functions/cloud_functions.dart';

class PairingRepository {
  final FirebaseFunctions _functions = FirebaseFunctions.instanceFor(region: 'asia-south1');
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  /// Generate a unique pairing token (rate-limited server-side)
  Future<PairingToken> generatePairingToken({
    required String userId,
    required String childId,
  }) async {
    try {
      final callable = _functions.httpsCallable('generatePairingToken');
      final result = await callable.call({
        'childId': childId,
      });

      final data = result.data as Map<String, dynamic>;
      return PairingToken(
        token: data['token'],
        userId: userId,
        childId: childId,
        expiresAt: DateTime.parse(data['expiresAt']),
      );
    } on FirebaseFunctionsException catch (e) {
      if (e.code == 'resource-exhausted') {
        throw PairingException('Too many attempts. Please wait and try again.');
      }
      rethrow;
    }
  }

  // ... rest of repository
}

class PairingException implements Exception {
  final String message;
  PairingException(this.message);
  @override
  String toString() => message;
}
```

##### Task E05-S01-T02: Create Pair Device Screen
**Time**: 1.5 hours

Create `lib/presentation/screens/onboarding/pair_device_screen.dart`:

```dart
import 'dart:convert';

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:qr_flutter/qr_flutter.dart';

import '../../../core/theme/app_theme.dart';
import '../../../data/repositories/pairing_repository.dart';
import '../../providers/auth_provider.dart';

class PairDeviceScreen extends ConsumerStatefulWidget {
  final String childId;

  const PairDeviceScreen({super.key, required this.childId});

  @override
  ConsumerState<PairDeviceScreen> createState() => _PairDeviceScreenState();
}

class _PairDeviceScreenState extends ConsumerState<PairDeviceScreen> {
  final PairingRepository _pairingRepository = PairingRepository();
  PairingToken? _pairingToken;
  PairingStatus _status = PairingStatus.waiting;
  int _remainingSeconds = 600; // 10 minutes

  @override
  void initState() {
    super.initState();
    _generatePairingToken();
  }

  Future<void> _generatePairingToken() async {
    final userId = ref.read(currentUserProvider)?.uid;
    if (userId == null) return;

    final token = await _pairingRepository.generatePairingToken(
      userId: userId,
      childId: widget.childId,
    );

    setState(() {
      _pairingToken = token;
    });

    // Start countdown
    _startCountdown();

    // Watch for pairing completion
    _watchPairingStatus(token.token);
  }

  void _startCountdown() {
    Future.doWhile(() async {
      await Future.delayed(const Duration(seconds: 1));
      if (!mounted) return false;

      setState(() {
        _remainingSeconds--;
      });

      if (_remainingSeconds <= 0) {
        _refreshToken();
        return false;
      }

      return _status == PairingStatus.waiting;
    });
  }

  void _watchPairingStatus(String token) {
    _pairingRepository.watchPairingStatus(token).listen((status) {
      if (!mounted) return;

      setState(() {
        _status = status;
      });

      if (status == PairingStatus.completed) {
        _onPairingComplete();
      }
    });
  }

  void _onPairingComplete() async {
    // Show success and navigate to dashboard
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(
        content: Text('Device paired successfully!'),
        backgroundColor: AppTheme.successGreen,
      ),
    );

    await Future.delayed(const Duration(seconds: 1));

    if (mounted) {
      context.go('/dashboard');
    }
  }

  void _refreshToken() {
    setState(() {
      _pairingToken = null;
      _remainingSeconds = 600;
    });
    _generatePairingToken();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Pair Device'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          children: [
            // Progress
            LinearProgressIndicator(
              value: 0.66,
              backgroundColor: Colors.grey.shade200,
              valueColor: const AlwaysStoppedAnimation(AppTheme.primaryColor),
            ),

            const SizedBox(height: 32),

            Text(
              'Scan this code with the child\'s device',
              style: Theme.of(context).textTheme.titleLarge?.copyWith(
                fontWeight: FontWeight.bold,
              ),
              textAlign: TextAlign.center,
            ),

            const SizedBox(height: 32),

            // QR Code
            if (_pairingToken != null) ...[
              Container(
                padding: const EdgeInsets.all(16),
                decoration: BoxDecoration(
                  color: Colors.white,
                  borderRadius: BorderRadius.circular(16),
                  boxShadow: [
                    BoxShadow(
                      color: Colors.black.withOpacity(0.1),
                      blurRadius: 10,
                      offset: const Offset(0, 4),
                    ),
                  ],
                ),
                child: QrImageView(
                  data: jsonEncode(_pairingToken!.toQRData()),
                  version: QrVersions.auto,
                  size: 250,
                  errorCorrectionLevel: QrErrorCorrectLevel.M,
                ),
              ),

              const SizedBox(height: 16),

              // Timer
              Text(
                'Code expires in ${_formatTime(_remainingSeconds)}',
                style: TextStyle(
                  color: _remainingSeconds < 60
                      ? AppTheme.errorRed
                      : AppTheme.textSecondary,
                ),
              ),

              if (_remainingSeconds < 60)
                TextButton(
                  onPressed: _refreshToken,
                  child: const Text('Refresh Code'),
                ),
            ] else
              const CircularProgressIndicator(),

            const Spacer(),

            // Status indicator
            _buildStatusIndicator(),

            const SizedBox(height: 24),

            // Instructions
            Container(
              padding: const EdgeInsets.all(16),
              decoration: BoxDecoration(
                color: Colors.blue.shade50,
                borderRadius: BorderRadius.circular(12),
              ),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const Text(
                    'How to scan:',
                    style: TextStyle(fontWeight: FontWeight.bold),
                  ),
                  const SizedBox(height: 8),
                  _buildStep('1', 'Turn on the child\'s phone'),
                  _buildStep('2', 'Open the KidTunes app'),
                  _buildStep('3', 'Tap "Scan Parent Code"'),
                  _buildStep('4', 'Point camera at this QR code'),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildStatusIndicator() {
    switch (_status) {
      case PairingStatus.waiting:
        return Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Container(
              width: 12,
              height: 12,
              decoration: BoxDecoration(
                shape: BoxShape.circle,
                color: AppTheme.warningOrange,
              ),
            ),
            const SizedBox(width: 8),
            const Text('Waiting for device...'),
          ],
        );
      case PairingStatus.inProgress:
        return Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const SizedBox(
              width: 16,
              height: 16,
              child: CircularProgressIndicator(strokeWidth: 2),
            ),
            const SizedBox(width: 8),
            const Text('Connecting...'),
          ],
        );
      case PairingStatus.completed:
        return Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(Icons.check_circle, color: AppTheme.successGreen),
            const SizedBox(width: 8),
            const Text('Device paired!'),
          ],
        );
      case PairingStatus.expired:
        return Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(Icons.error, color: AppTheme.errorRed),
            const SizedBox(width: 8),
            const Text('Code expired'),
          ],
        );
    }
  }

  Widget _buildStep(String number, String text) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        children: [
          Container(
            width: 24,
            height: 24,
            decoration: BoxDecoration(
              shape: BoxShape.circle,
              color: AppTheme.primaryColor,
            ),
            child: Center(
              child: Text(
                number,
                style: const TextStyle(color: Colors.white, fontSize: 12),
              ),
            ),
          ),
          const SizedBox(width: 12),
          Expanded(child: Text(text)),
        ],
      ),
    );
  }

  String _formatTime(int seconds) {
    final minutes = seconds ~/ 60;
    final secs = seconds % 60;
    return '${minutes.toString().padLeft(2, '0')}:${secs.toString().padLeft(2, '0')}';
  }
}
```

Add qr_flutter dependency to pubspec.yaml:
```yaml
dependencies:
  qr_flutter: ^4.1.0
```

##### Task E05-S01-T03: Implement QR Scanning on Child Device (Android)
**Time**: 2 hours

Create `app/src/main/java/com/kidtunes/launcher/ui/pairing/QRScanActivity.kt`:

```kotlin
package com.kidtunes.launcher.ui.pairing

import android.Manifest
import android.content.pm.PackageManager
import android.os.Bundle
import android.util.Log
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.core.content.ContextCompat
import com.google.firebase.firestore.FirebaseFirestore
import com.google.mlkit.vision.barcode.BarcodeScanning
import com.google.mlkit.vision.common.InputImage
import com.kidtunes.launcher.databinding.ActivityQrScanBinding
import com.kidtunes.launcher.data.local.LauncherPreferences
import dagger.hilt.android.AndroidEntryPoint
import org.json.JSONObject
import java.util.UUID
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors
import javax.inject.Inject

@AndroidEntryPoint
class QRScanActivity : AppCompatActivity() {

    private lateinit var binding: ActivityQrScanBinding
    private lateinit var cameraExecutor: ExecutorService

    @Inject
    lateinit var preferences: LauncherPreferences

    private val firestore = FirebaseFirestore.getInstance()
    private var isProcessing = false

    private val permissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            startCamera()
        } else {
            finish()
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityQrScanBinding.inflate(layoutInflater)
        setContentView(binding.root)

        cameraExecutor = Executors.newSingleThreadExecutor()

        if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA)
            == PackageManager.PERMISSION_GRANTED) {
            startCamera()
        } else {
            permissionLauncher.launch(Manifest.permission.CAMERA)
        }
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            val preview = Preview.Builder().build().also {
                it.setSurfaceProvider(binding.previewView.surfaceProvider)
            }

            val imageAnalyzer = ImageAnalysis.Builder()
                .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
                .build()
                .also {
                    it.setAnalyzer(cameraExecutor, BarcodeAnalyzer { qrData ->
                        processPairingData(qrData)
                    })
                }

            val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(
                    this, cameraSelector, preview, imageAnalyzer
                )
            } catch (e: Exception) {
                Log.e("QRScan", "Camera binding failed", e)
            }

        }, ContextCompat.getMainExecutor(this))
    }

    private fun processPairingData(qrData: String) {
        if (isProcessing) return
        isProcessing = true

        try {
            val json = JSONObject(qrData)
            val token = json.getString("token")
            val userId = json.getString("userId")
            val childId = json.getString("childId")

            // Generate device ID
            val deviceId = UUID.randomUUID().toString()

            // Update pairing token with device info
            firestore.collection("pairing_tokens").document(token)
                .update(mapOf(
                    "deviceId" to deviceId,
                    "used" to true,
                    "deviceModel" to android.os.Build.MODEL,
                    "androidVersion" to android.os.Build.VERSION.RELEASE,
                ))
                .addOnSuccessListener {
                    // Create device document
                    createDeviceDocument(deviceId, userId, childId)
                }
                .addOnFailureListener { e ->
                    Log.e("QRScan", "Failed to update pairing token", e)
                    isProcessing = false
                }

        } catch (e: Exception) {
            Log.e("QRScan", "Invalid QR data", e)
            isProcessing = false
        }
    }

    private fun createDeviceDocument(deviceId: String, userId: String, childId: String) {
        val deviceData = mapOf(
            "ownerId" to userId,
            "childId" to childId,
            "model" to android.os.Build.MODEL,
            "androidVersion" to android.os.Build.VERSION.RELEASE,
            "appVersion" to "1.0.0", // Get from BuildConfig
            "registeredAt" to com.google.firebase.Timestamp.now(),
            "settings" to mapOf(
                "kidsMode" to true,
                "volumeLimit" to 70,
                "dailyLimitMinutes" to 120,
                "locationEnabled" to false,
            ),
            "whitelist" to listOf(
                "com.spotify.music",
                "com.jio.saavn",
                "com.amazon.mp3",
                "com.android.deskclock",
            )
        )

        firestore.collection("devices").document(deviceId)
            .set(deviceData)
            .addOnSuccessListener {
                // Save locally
                preferences.setDeviceId(deviceId)
                preferences.setPaired(true)

                // Update child profile with device ID
                firestore.collection("users").document(userId)
                    .collection("children").document(childId)
                    .update("deviceId", deviceId)

                // Navigate to main screen
                finish()
            }
            .addOnFailureListener { e ->
                Log.e("QRScan", "Failed to create device", e)
                isProcessing = false
            }
    }

    override fun onDestroy() {
        super.onDestroy()
        cameraExecutor.shutdown()
    }
}

private class BarcodeAnalyzer(
    private val onBarcodeDetected: (String) -> Unit
) : ImageAnalysis.Analyzer {

    private val scanner = BarcodeScanning.getClient()

    @androidx.camera.core.ExperimentalGetImage
    override fun analyze(imageProxy: ImageProxy) {
        val mediaImage = imageProxy.image ?: run {
            imageProxy.close()
            return
        }

        val image = InputImage.fromMediaImage(mediaImage, imageProxy.imageInfo.rotationDegrees)

        scanner.process(image)
            .addOnSuccessListener { barcodes ->
                barcodes.firstOrNull()?.rawValue?.let { value ->
                    onBarcodeDetected(value)
                }
            }
            .addOnCompleteListener {
                imageProxy.close()
            }
    }
}
```

---

## Sprint 2 Checklist

### Days 18-20: Flutter Setup
- [ ] Flutter project created
- [ ] Dependencies configured
- [ ] Firebase connected
- [ ] Project structure established
- [ ] Router configured

### Days 21-22: Authentication
- [ ] Auth repository created
- [ ] Login screen with phone OTP
- [ ] OTP verification screen
- [ ] Email login option
- [ ] Error handling

### Days 23-24: Onboarding
- [ ] Data models created
- [ ] Add child screen
- [ ] Protection level selector
- [ ] Save to Firestore

### Days 25-27: Device Pairing
- [ ] QR code generation (Parent)
- [ ] QR scanning (Child)
- [ ] Pairing token exchange
- [ ] Firebase linking
- [ ] Success confirmation
- [ ] **Rate limiting Cloud Function deployed** (security)
- [ ] Splash screen with proper loading state handling

---

## Sprint 2 Deliverables

1. ✅ Flutter parent app foundation
2. ✅ Phone and email authentication
3. ✅ Child profile creation
4. ✅ QR code pairing system
5. ✅ Device linked in Firebase
6. ✅ Ready for remote controls in Sprint 3
