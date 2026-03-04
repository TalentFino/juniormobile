# Sprint 3: Core Features (Days 28-37)

## Sprint Goal
Implement core parental control features: remote commands, app management, and screen time controls.

---

## Epic E06: Remote Controls

### Story E06-S01: Firebase Command System
**Points**: 8
**Owner**: Both
**Priority**: P0

#### Description
Implement the command system for parent app to control child device in real-time.

#### Acceptance Criteria
- [ ] Parent can send commands via Firebase
- [ ] Child device receives and executes commands
- [ ] Commands work within 5 seconds
- [ ] Offline commands queue and execute when online

#### Tasks

##### Task E06-S01-T01: Define Command Schema
**Time**: 30 minutes

Commands are stored in Firebase Realtime Database for low latency:

```
devices/{deviceId}/commands/{commandId}
{
  "command": "TOGGLE_KIDS_MODE",
  "payload": {
    "enabled": true
  },
  "timestamp": 1709550000000,
  "status": "pending"  // pending, executing, completed, failed
}
```

Command types:
```typescript
type Command =
  | { command: 'TOGGLE_KIDS_MODE', payload: { enabled: boolean } }
  | { command: 'LOCK_DEVICE', payload: {} }
  | { command: 'UNLOCK_DEVICE', payload: {} }
  | { command: 'SET_VOLUME_LIMIT', payload: { percent: number } }
  | { command: 'UPDATE_WHITELIST', payload: { packages: string[] } }
  | { command: 'SET_SCREEN_TIME', payload: { dailyMinutes: number } }
  | { command: 'SET_BEDTIME', payload: { start: string, end: string } }
  | { command: 'REQUEST_LOCATION', payload: {} }
  | { command: 'PLAY_SOUND', payload: {} }
  | { command: 'INSTALL_APP', payload: { packageName: string } }
  | { command: 'REMOVE_APP', payload: { packageName: string } }
  | { command: 'SYNC_SETTINGS', payload: {} };
```

##### Task E06-S01-T02: Create Command Service (Parent App)
**Time**: 1 hour

Create `lib/data/services/command_service.dart`:

```dart
import 'package:firebase_database/firebase_database.dart';
import 'package:uuid/uuid.dart';

class CommandService {
  final FirebaseDatabase _rtdb = FirebaseDatabase.instance;

  /// Send a command to a device
  Future<String> sendCommand({
    required String deviceId,
    required String command,
    required Map<String, dynamic> payload,
  }) async {
    final commandId = const Uuid().v4();

    await _rtdb.ref('devices/$deviceId/commands/$commandId').set({
      'command': command,
      'payload': payload,
      'timestamp': ServerValue.timestamp,
      'status': 'pending',
    });

    return commandId;
  }

  /// Watch command status
  Stream<CommandStatus> watchCommandStatus(String deviceId, String commandId) {
    return _rtdb
        .ref('devices/$deviceId/commands/$commandId/status')
        .onValue
        .map((event) {
      final status = event.snapshot.value as String?;
      return CommandStatus.values.firstWhere(
        (e) => e.name == status,
        orElse: () => CommandStatus.pending,
      );
    });
  }

  // Convenience methods
  Future<void> toggleKidsMode(String deviceId, bool enabled) async {
    await sendCommand(
      deviceId: deviceId,
      command: 'TOGGLE_KIDS_MODE',
      payload: {'enabled': enabled},
    );
  }

  Future<void> lockDevice(String deviceId) async {
    await sendCommand(
      deviceId: deviceId,
      command: 'LOCK_DEVICE',
      payload: {},
    );
  }

  Future<void> setVolumeLimit(String deviceId, int percent) async {
    await sendCommand(
      deviceId: deviceId,
      command: 'SET_VOLUME_LIMIT',
      payload: {'percent': percent},
    );
  }

  Future<void> updateWhitelist(String deviceId, List<String> packages) async {
    await sendCommand(
      deviceId: deviceId,
      command: 'UPDATE_WHITELIST',
      payload: {'packages': packages},
    );
  }

  Future<void> setScreenTime(String deviceId, int dailyMinutes) async {
    await sendCommand(
      deviceId: deviceId,
      command: 'SET_SCREEN_TIME',
      payload: {'dailyMinutes': dailyMinutes},
    );
  }

  Future<void> setBedtime(String deviceId, String start, String end) async {
    await sendCommand(
      deviceId: deviceId,
      command: 'SET_BEDTIME',
      payload: {'start': start, 'end': end},
    );
  }

  Future<void> requestLocation(String deviceId) async {
    await sendCommand(
      deviceId: deviceId,
      command: 'REQUEST_LOCATION',
      payload: {},
    );
  }

  Future<void> playSound(String deviceId) async {
    await sendCommand(
      deviceId: deviceId,
      command: 'PLAY_SOUND',
      payload: {},
    );
  }

  Future<void> installApp(String deviceId, String packageName) async {
    await sendCommand(
      deviceId: deviceId,
      command: 'INSTALL_APP',
      payload: {'packageName': packageName},
    );
  }

  Future<void> removeApp(String deviceId, String packageName) async {
    await sendCommand(
      deviceId: deviceId,
      command: 'REMOVE_APP',
      payload: {'packageName': packageName},
    );
  }
}

enum CommandStatus {
  pending,
  executing,
  completed,
  failed,
}
```

##### Task E06-S01-T03: Create FCM Service (Child Device)
**Time**: 1.5 hours

Create `app/src/main/java/com/kidtunes/launcher/services/FCMService.kt`:

```kotlin
package com.kidtunes.launcher.services

import android.util.Log
import com.google.firebase.database.ChildEventListener
import com.google.firebase.database.DataSnapshot
import com.google.firebase.database.DatabaseError
import com.google.firebase.database.FirebaseDatabase
import com.google.firebase.messaging.FirebaseMessagingService
import com.google.firebase.messaging.RemoteMessage
import com.kidtunes.launcher.data.local.LauncherPreferences
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.launch
import javax.inject.Inject

@AndroidEntryPoint
class FCMService : FirebaseMessagingService() {

    @Inject
    lateinit var preferences: LauncherPreferences

    @Inject
    lateinit var commandExecutor: CommandExecutor

    private val serviceScope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private val rtdb = FirebaseDatabase.getInstance()

    companion object {
        private const val TAG = "FCMService"
    }

    override fun onCreate() {
        super.onCreate()
        startListeningForCommands()
    }

    override fun onMessageReceived(message: RemoteMessage) {
        Log.d(TAG, "FCM message received: ${message.data}")

        // Handle high-priority commands via FCM
        message.data["command"]?.let { command ->
            val payload = message.data["payload"] ?: "{}"
            serviceScope.launch {
                commandExecutor.execute(command, payload)
            }
        }
    }

    override fun onNewToken(token: String) {
        Log.d(TAG, "New FCM token: $token")
        // Update token in Firebase
        val deviceId = preferences.getDeviceId() ?: return
        rtdb.getReference("devices/$deviceId/fcmToken").setValue(token)
    }

    private fun startListeningForCommands() {
        val deviceId = preferences.getDeviceId() ?: return

        rtdb.getReference("devices/$deviceId/commands")
            .addChildEventListener(object : ChildEventListener {
                override fun onChildAdded(snapshot: DataSnapshot, previousChildName: String?) {
                    processCommand(snapshot)
                }

                override fun onChildChanged(snapshot: DataSnapshot, previousChildName: String?) {}
                override fun onChildRemoved(snapshot: DataSnapshot) {}
                override fun onChildMoved(snapshot: DataSnapshot, previousChildName: String?) {}
                override fun onCancelled(error: DatabaseError) {
                    Log.e(TAG, "Command listener cancelled", error.toException())
                }
            })
    }

    private fun processCommand(snapshot: DataSnapshot) {
        val commandId = snapshot.key ?: return
        val command = snapshot.child("command").getValue(String::class.java) ?: return
        val payload = snapshot.child("payload").value?.toString() ?: "{}"
        val status = snapshot.child("status").getValue(String::class.java)

        // Only process pending commands
        if (status != "pending") return

        serviceScope.launch {
            val deviceId = preferences.getDeviceId() ?: return@launch

            // Mark as executing
            rtdb.getReference("devices/$deviceId/commands/$commandId/status")
                .setValue("executing")

            try {
                commandExecutor.execute(command, payload)

                // Mark as completed
                rtdb.getReference("devices/$deviceId/commands/$commandId/status")
                    .setValue("completed")

                // Delete command after 1 minute
                rtdb.getReference("devices/$deviceId/commands/$commandId")
                    .removeValue()

            } catch (e: Exception) {
                Log.e(TAG, "Command execution failed: $command", e)
                rtdb.getReference("devices/$deviceId/commands/$commandId/status")
                    .setValue("failed")
            }
        }
    }
}
```

##### Task E06-S01-T04: Create Command Executor
**Time**: 2 hours

Create `app/src/main/java/com/kidtunes/launcher/services/CommandExecutor.kt`:

```kotlin
package com.kidtunes.launcher.services

import android.app.admin.DevicePolicyManager
import android.content.ComponentName
import android.content.Context
import android.content.Intent
import android.media.AudioManager
import android.media.MediaPlayer
import android.media.RingtoneManager
import android.os.Build
import com.google.firebase.firestore.FirebaseFirestore
import com.kidtunes.launcher.data.local.AppWhitelistManager
import com.kidtunes.launcher.data.local.LauncherPreferences
import com.kidtunes.launcher.ui.LauncherActivity
import dagger.hilt.android.qualifiers.ApplicationContext
import org.json.JSONArray
import org.json.JSONObject
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class CommandExecutor @Inject constructor(
    @ApplicationContext private val context: Context,
    private val preferences: LauncherPreferences,
    private val whitelistManager: AppWhitelistManager,
    private val screenTimeService: ScreenTimeService,
    private val locationService: LocationService,
) {
    private val firestore = FirebaseFirestore.getInstance()

    suspend fun execute(command: String, payloadJson: String) {
        val payload = try {
            JSONObject(payloadJson)
        } catch (e: Exception) {
            JSONObject()
        }

        when (command) {
            "TOGGLE_KIDS_MODE" -> toggleKidsMode(payload.optBoolean("enabled", true))
            "LOCK_DEVICE" -> lockDevice()
            "UNLOCK_DEVICE" -> unlockDevice()
            "SET_VOLUME_LIMIT" -> setVolumeLimit(payload.optInt("percent", 70))
            "UPDATE_WHITELIST" -> updateWhitelist(payload.optJSONArray("packages"))
            "SET_SCREEN_TIME" -> setScreenTime(payload.optInt("dailyMinutes", 120))
            "SET_BEDTIME" -> setBedtime(
                payload.optString("start", "20:30"),
                payload.optString("end", "07:00")
            )
            "REQUEST_LOCATION" -> requestLocation()
            "PLAY_SOUND" -> playSound()
            "INSTALL_APP" -> installApp(payload.optString("packageName", ""))
            "REMOVE_APP" -> removeApp(payload.optString("packageName", ""))
            "SYNC_SETTINGS" -> syncSettings()
        }
    }

    private fun toggleKidsMode(enabled: Boolean) {
        preferences.setKidsMode(enabled)

        // Broadcast to LauncherActivity to refresh
        val intent = Intent("com.kidtunes.launcher.MODE_CHANGED").apply {
            putExtra("kidsMode", enabled)
        }
        context.sendBroadcast(intent)

        // Update status in Firebase
        updateDeviceStatus("kidsMode", enabled)
    }

    private fun lockDevice() {
        val dpm = context.getSystemService(Context.DEVICE_POLICY_SERVICE)
            as DevicePolicyManager
        dpm.lockNow()
    }

    private fun unlockDevice() {
        // Send intent to show unlock screen
        // Note: Cannot programmatically unlock without user interaction
        // Instead, we can disable keyguard temporarily if Device Owner
    }

    private fun setVolumeLimit(percent: Int) {
        preferences.setVolumeLimit(percent)

        // Apply immediately
        val audioManager = context.getSystemService(Context.AUDIO_SERVICE) as AudioManager
        val maxVolume = audioManager.getStreamMaxVolume(AudioManager.STREAM_MUSIC)
        val targetVolume = (maxVolume * percent / 100)
        val currentVolume = audioManager.getStreamVolume(AudioManager.STREAM_MUSIC)

        if (currentVolume > targetVolume) {
            audioManager.setStreamVolume(AudioManager.STREAM_MUSIC, targetVolume, 0)
        }

        updateDeviceStatus("volumeLimit", percent)
    }

    private suspend fun updateWhitelist(packages: JSONArray?) {
        if (packages == null) return

        val packageList = mutableSetOf<String>()
        for (i in 0 until packages.length()) {
            packageList.add(packages.getString(i))
        }

        whitelistManager.setWhitelist(packageList)
        whitelistManager.refreshInstalledApps()

        // Also update in Firestore
        val deviceId = preferences.getDeviceId() ?: return
        firestore.collection("devices").document(deviceId)
            .update("whitelist", packageList.toList())
    }

    private fun setScreenTime(dailyMinutes: Int) {
        preferences.setDailyLimitMinutes(dailyMinutes)
        screenTimeService.updateDailyLimit(dailyMinutes)
        updateDeviceStatus("dailyLimitMinutes", dailyMinutes)
    }

    private fun setBedtime(start: String, end: String) {
        preferences.setBedtimeStart(start)
        preferences.setBedtimeEnd(end)
        screenTimeService.updateBedtime(start, end)
    }

    private suspend fun requestLocation() {
        val location = locationService.getCurrentLocation()
        location?.let {
            val deviceId = preferences.getDeviceId() ?: return@let
            firestore.collection("devices").document(deviceId)
                .update(mapOf(
                    "lastLocation" to mapOf(
                        "latitude" to it.latitude,
                        "longitude" to it.longitude,
                        "accuracy" to it.accuracy,
                        "timestamp" to com.google.firebase.Timestamp.now()
                    )
                ))
        }
    }

    private fun playSound() {
        try {
            val notification = RingtoneManager.getDefaultUri(RingtoneManager.TYPE_ALARM)
            val mediaPlayer = MediaPlayer.create(context, notification)
            mediaPlayer?.apply {
                setVolume(1.0f, 1.0f)
                start()
                setOnCompletionListener { release() }
            }
        } catch (e: Exception) {
            // Fallback to system sound
        }
    }

    private fun installApp(packageName: String) {
        if (packageName.isEmpty()) return

        // For managed Google Play, we would use:
        // val dpm = context.getSystemService(Context.DEVICE_POLICY_SERVICE) as DevicePolicyManager
        // dpm.installExistingPackage(componentName, packageName)

        // For now, add to whitelist and let user install from Play Store
        whitelistManager.addToWhitelist(packageName)

        // Open Play Store to the app
        try {
            val intent = Intent(Intent.ACTION_VIEW).apply {
                data = android.net.Uri.parse("market://details?id=$packageName")
                addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            }
            context.startActivity(intent)
        } catch (e: Exception) {
            // Play Store not available
        }
    }

    private fun removeApp(packageName: String) {
        if (packageName.isEmpty()) return

        whitelistManager.removeFromWhitelist(packageName)

        // Optionally uninstall the app (requires Device Owner)
        // val dpm = context.getSystemService(Context.DEVICE_POLICY_SERVICE) as DevicePolicyManager
        // dpm.clearPackagePersistentPreferredActivities(componentName, packageName)
    }

    private suspend fun syncSettings() {
        val deviceId = preferences.getDeviceId() ?: return

        // Fetch settings from Firestore and apply
        firestore.collection("devices").document(deviceId)
            .get()
            .addOnSuccessListener { doc ->
                val settings = doc.get("settings") as? Map<*, *> ?: return@addOnSuccessListener

                settings["kidsMode"]?.let { toggleKidsMode(it as Boolean) }
                settings["volumeLimit"]?.let { setVolumeLimit((it as Long).toInt()) }
                settings["dailyLimitMinutes"]?.let { setScreenTime((it as Long).toInt()) }

                val bedtimeStart = settings["bedtimeStart"] as? String
                val bedtimeEnd = settings["bedtimeEnd"] as? String
                if (bedtimeStart != null && bedtimeEnd != null) {
                    setBedtime(bedtimeStart, bedtimeEnd)
                }

                val whitelist = doc.get("whitelist") as? List<*>
                whitelist?.let {
                    val packages = it.filterIsInstance<String>().toSet()
                    whitelistManager.setWhitelist(packages)
                }
            }
    }

    private fun updateDeviceStatus(key: String, value: Any) {
        val deviceId = preferences.getDeviceId() ?: return
        firestore.collection("devices").document(deviceId)
            .update("settings.$key", value)
    }
}
```

---

## Epic E07: App Management

### Story E07-S01: App Management UI (Parent)
**Points**: 5
**Owner**: Flutter Developer
**Priority**: P0

#### Description
Create the app management screen where parents can see and control installed apps.

#### Acceptance Criteria
- [ ] Shows all installed apps on child device
- [ ] Toggle to allow/block each app
- [ ] Per-app time limits
- [ ] Search and filter apps
- [ ] Category grouping

#### Tasks

##### Task E07-S01-T01: Create App Repository
**Time**: 45 minutes

Create `lib/data/repositories/app_repository.dart`:

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import '../models/app_info.dart';
import '../services/command_service.dart';

class AppRepository {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;
  final CommandService _commandService = CommandService();

  /// Get all apps installed on device
  Stream<List<AppInfo>> getInstalledApps(String deviceId) {
    return _firestore
        .collection('devices')
        .doc(deviceId)
        .collection('installedApps')
        .orderBy('label')
        .snapshots()
        .map((snapshot) {
      return snapshot.docs.map((doc) => AppInfo.fromFirestore(doc)).toList();
    });
  }

  /// Get whitelist
  Stream<List<String>> getWhitelist(String deviceId) {
    return _firestore
        .collection('devices')
        .doc(deviceId)
        .snapshots()
        .map((doc) {
      final data = doc.data();
      return List<String>.from(data?['whitelist'] ?? []);
    });
  }

  /// Update whitelist
  Future<void> updateWhitelist(String deviceId, List<String> packages) async {
    await _firestore.collection('devices').doc(deviceId).update({
      'whitelist': packages,
    });

    // Send command to device
    await _commandService.updateWhitelist(deviceId, packages);
  }

  /// Add app to whitelist
  Future<void> allowApp(String deviceId, String packageName) async {
    await _firestore.collection('devices').doc(deviceId).update({
      'whitelist': FieldValue.arrayUnion([packageName]),
    });

    final whitelist = await _getWhitelist(deviceId);
    await _commandService.updateWhitelist(deviceId, whitelist);
  }

  /// Remove app from whitelist
  Future<void> blockApp(String deviceId, String packageName) async {
    await _firestore.collection('devices').doc(deviceId).update({
      'whitelist': FieldValue.arrayRemove([packageName]),
    });

    final whitelist = await _getWhitelist(deviceId);
    await _commandService.updateWhitelist(deviceId, whitelist);
  }

  /// Set per-app time limit
  Future<void> setAppTimeLimit(
    String deviceId,
    String packageName,
    int? minutes,
  ) async {
    if (minutes == null) {
      await _firestore
          .collection('devices')
          .doc(deviceId)
          .collection('appLimits')
          .doc(packageName)
          .delete();
    } else {
      await _firestore
          .collection('devices')
          .doc(deviceId)
          .collection('appLimits')
          .doc(packageName)
          .set({'minutes': minutes});
    }
  }

  Future<List<String>> _getWhitelist(String deviceId) async {
    final doc = await _firestore.collection('devices').doc(deviceId).get();
    return List<String>.from(doc.data()?['whitelist'] ?? []);
  }

  /// Request app installation
  Future<void> installApp(String deviceId, String packageName) async {
    await _commandService.installApp(deviceId, packageName);
  }

  /// Request app removal
  Future<void> removeApp(String deviceId, String packageName) async {
    await _commandService.removeApp(deviceId, packageName);
  }
}
```

##### Task E07-S01-T02: Create App Info Model
**Time**: 30 minutes

Create `lib/data/models/app_info.dart`:

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

enum AppCategory {
  music,
  education,
  games,
  communication,
  finance,
  utility,
  social,
  other,
}

class AppInfo {
  final String packageName;
  final String label;
  final String? iconUrl;
  final AppCategory category;
  final bool isSystemApp;
  final DateTime? lastUsed;
  final int usageMinutesToday;

  const AppInfo({
    required this.packageName,
    required this.label,
    this.iconUrl,
    this.category = AppCategory.other,
    this.isSystemApp = false,
    this.lastUsed,
    this.usageMinutesToday = 0,
  });

  factory AppInfo.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return AppInfo(
      packageName: doc.id,
      label: data['label'] ?? '',
      iconUrl: data['iconUrl'],
      category: AppCategory.values.firstWhere(
        (e) => e.name == data['category'],
        orElse: () => AppCategory.other,
      ),
      isSystemApp: data['isSystemApp'] ?? false,
      lastUsed: (data['lastUsed'] as Timestamp?)?.toDate(),
      usageMinutesToday: data['usageMinutesToday'] ?? 0,
    );
  }

  String get categoryLabel {
    switch (category) {
      case AppCategory.music:
        return '🎵 Music';
      case AppCategory.education:
        return '📚 Education';
      case AppCategory.games:
        return '🎮 Games';
      case AppCategory.communication:
        return '💬 Communication';
      case AppCategory.finance:
        return '💳 Finance';
      case AppCategory.utility:
        return '🔧 Utility';
      case AppCategory.social:
        return '👥 Social';
      case AppCategory.other:
        return '📱 Other';
    }
  }
}
```

##### Task E07-S01-T03: Create App Management Screen
**Time**: 2 hours

Create `lib/presentation/screens/app_management/app_management_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:cached_network_image/cached_network_image.dart';

import '../../../core/theme/app_theme.dart';
import '../../../data/models/app_info.dart';
import '../../../data/repositories/app_repository.dart';

class AppManagementScreen extends ConsumerStatefulWidget {
  final String deviceId;

  const AppManagementScreen({super.key, required this.deviceId});

  @override
  ConsumerState<AppManagementScreen> createState() => _AppManagementScreenState();
}

class _AppManagementScreenState extends ConsumerState<AppManagementScreen> {
  final AppRepository _appRepository = AppRepository();
  final TextEditingController _searchController = TextEditingController();
  String _searchQuery = '';
  AppCategory? _selectedCategory;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Manage Apps'),
        actions: [
          IconButton(
            icon: const Icon(Icons.add),
            onPressed: _showAddAppDialog,
            tooltip: 'Install App',
          ),
        ],
      ),
      body: Column(
        children: [
          // Search Bar
          Padding(
            padding: const EdgeInsets.all(16),
            child: TextField(
              controller: _searchController,
              decoration: InputDecoration(
                hintText: 'Search apps...',
                prefixIcon: const Icon(Icons.search),
                suffixIcon: _searchQuery.isNotEmpty
                    ? IconButton(
                        icon: const Icon(Icons.clear),
                        onPressed: () {
                          _searchController.clear();
                          setState(() => _searchQuery = '');
                        },
                      )
                    : null,
              ),
              onChanged: (value) => setState(() => _searchQuery = value),
            ),
          ),

          // Category Filter
          SingleChildScrollView(
            scrollDirection: Axis.horizontal,
            padding: const EdgeInsets.symmetric(horizontal: 16),
            child: Row(
              children: [
                FilterChip(
                  label: const Text('All'),
                  selected: _selectedCategory == null,
                  onSelected: (_) => setState(() => _selectedCategory = null),
                ),
                const SizedBox(width: 8),
                ...AppCategory.values.map((category) {
                  return Padding(
                    padding: const EdgeInsets.only(right: 8),
                    child: FilterChip(
                      label: Text(_getCategoryLabel(category)),
                      selected: _selectedCategory == category,
                      onSelected: (_) => setState(() => _selectedCategory = category),
                    ),
                  );
                }),
              ],
            ),
          ),

          const SizedBox(height: 16),

          // App List
          Expanded(
            child: StreamBuilder<List<AppInfo>>(
              stream: _appRepository.getInstalledApps(widget.deviceId),
              builder: (context, appsSnapshot) {
                if (!appsSnapshot.hasData) {
                  return const Center(child: CircularProgressIndicator());
                }

                return StreamBuilder<List<String>>(
                  stream: _appRepository.getWhitelist(widget.deviceId),
                  builder: (context, whitelistSnapshot) {
                    final whitelist = whitelistSnapshot.data ?? [];
                    var apps = appsSnapshot.data!;

                    // Apply filters
                    if (_searchQuery.isNotEmpty) {
                      apps = apps.where((app) =>
                        app.label.toLowerCase().contains(_searchQuery.toLowerCase())
                      ).toList();
                    }

                    if (_selectedCategory != null) {
                      apps = apps.where((app) =>
                        app.category == _selectedCategory
                      ).toList();
                    }

                    if (apps.isEmpty) {
                      return const Center(
                        child: Text('No apps found'),
                      );
                    }

                    return ListView.builder(
                      padding: const EdgeInsets.symmetric(horizontal: 16),
                      itemCount: apps.length,
                      itemBuilder: (context, index) {
                        final app = apps[index];
                        final isAllowed = whitelist.contains(app.packageName);

                        return _AppListItem(
                          app: app,
                          isAllowed: isAllowed,
                          onToggle: () => _toggleApp(app.packageName, isAllowed),
                          onTap: () => _showAppDetails(app, isAllowed),
                        );
                      },
                    );
                  },
                );
              },
            ),
          ),
        ],
      ),
    );
  }

  void _toggleApp(String packageName, bool currentlyAllowed) async {
    try {
      if (currentlyAllowed) {
        await _appRepository.blockApp(widget.deviceId, packageName);
      } else {
        await _appRepository.allowApp(widget.deviceId, packageName);
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Error: $e')),
        );
      }
    }
  }

  void _showAppDetails(AppInfo app, bool isAllowed) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      builder: (context) => _AppDetailsSheet(
        app: app,
        deviceId: widget.deviceId,
        isAllowed: isAllowed,
      ),
    );
  }

  void _showAddAppDialog() {
    showDialog(
      context: context,
      builder: (context) => _AddAppDialog(deviceId: widget.deviceId),
    );
  }

  String _getCategoryLabel(AppCategory category) {
    switch (category) {
      case AppCategory.music: return '🎵';
      case AppCategory.education: return '📚';
      case AppCategory.games: return '🎮';
      case AppCategory.communication: return '💬';
      case AppCategory.finance: return '💳';
      case AppCategory.utility: return '🔧';
      case AppCategory.social: return '👥';
      case AppCategory.other: return '📱';
    }
  }
}

class _AppListItem extends StatelessWidget {
  final AppInfo app;
  final bool isAllowed;
  final VoidCallback onToggle;
  final VoidCallback onTap;

  const _AppListItem({
    required this.app,
    required this.isAllowed,
    required this.onToggle,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: ListTile(
        leading: app.iconUrl != null
            ? CachedNetworkImage(
                imageUrl: app.iconUrl!,
                width: 48,
                height: 48,
                placeholder: (_, __) => const Icon(Icons.android, size: 48),
                errorWidget: (_, __, ___) => const Icon(Icons.android, size: 48),
              )
            : const Icon(Icons.android, size: 48),
        title: Text(app.label),
        subtitle: Text(
          app.categoryLabel,
          style: TextStyle(color: AppTheme.textSecondary, fontSize: 12),
        ),
        trailing: Switch(
          value: isAllowed,
          onChanged: (_) => onToggle(),
          activeColor: AppTheme.successGreen,
        ),
        onTap: onTap,
      ),
    );
  }
}

class _AppDetailsSheet extends StatefulWidget {
  final AppInfo app;
  final String deviceId;
  final bool isAllowed;

  const _AppDetailsSheet({
    required this.app,
    required this.deviceId,
    required this.isAllowed,
  });

  @override
  State<_AppDetailsSheet> createState() => _AppDetailsSheetState();
}

class _AppDetailsSheetState extends State<_AppDetailsSheet> {
  final AppRepository _appRepository = AppRepository();
  int? _timeLimit;
  bool _isAllowed = false;

  @override
  void initState() {
    super.initState();
    _isAllowed = widget.isAllowed;
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(24),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(
            children: [
              widget.app.iconUrl != null
                  ? CachedNetworkImage(
                      imageUrl: widget.app.iconUrl!,
                      width: 64,
                      height: 64,
                    )
                  : const Icon(Icons.android, size: 64),
              const SizedBox(width: 16),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      widget.app.label,
                      style: Theme.of(context).textTheme.titleLarge,
                    ),
                    Text(
                      widget.app.categoryLabel,
                      style: TextStyle(color: AppTheme.textSecondary),
                    ),
                  ],
                ),
              ),
            ],
          ),

          const SizedBox(height: 24),

          // Allow/Block Toggle
          SwitchListTile(
            title: const Text('Allow this app'),
            subtitle: Text(_isAllowed ? 'App is visible on device' : 'App is hidden'),
            value: _isAllowed,
            onChanged: (value) async {
              setState(() => _isAllowed = value);
              if (value) {
                await _appRepository.allowApp(widget.deviceId, widget.app.packageName);
              } else {
                await _appRepository.blockApp(widget.deviceId, widget.app.packageName);
              }
            },
          ),

          const Divider(),

          // Time Limit
          ListTile(
            title: const Text('Daily time limit'),
            subtitle: Text(_timeLimit == null ? 'No limit' : '$_timeLimit minutes'),
            trailing: const Icon(Icons.chevron_right),
            onTap: _showTimeLimitPicker,
          ),

          // Usage today
          ListTile(
            title: const Text('Usage today'),
            subtitle: Text('${widget.app.usageMinutesToday} minutes'),
            leading: const Icon(Icons.timer),
          ),

          const SizedBox(height: 24),

          // Remove App Button
          if (!widget.app.isSystemApp)
            SizedBox(
              width: double.infinity,
              child: OutlinedButton.icon(
                onPressed: _confirmRemoveApp,
                icon: const Icon(Icons.delete, color: Colors.red),
                label: const Text('Remove App', style: TextStyle(color: Colors.red)),
              ),
            ),

          const SizedBox(height: 16),
        ],
      ),
    );
  }

  void _showTimeLimitPicker() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Set Daily Limit'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            _buildTimeLimitOption(null, 'No limit'),
            _buildTimeLimitOption(30, '30 minutes'),
            _buildTimeLimitOption(60, '1 hour'),
            _buildTimeLimitOption(90, '1.5 hours'),
            _buildTimeLimitOption(120, '2 hours'),
          ],
        ),
      ),
    );
  }

  Widget _buildTimeLimitOption(int? minutes, String label) {
    return ListTile(
      title: Text(label),
      trailing: _timeLimit == minutes ? const Icon(Icons.check) : null,
      onTap: () async {
        setState(() => _timeLimit = minutes);
        await _appRepository.setAppTimeLimit(
          widget.deviceId,
          widget.app.packageName,
          minutes,
        );
        if (mounted) Navigator.pop(context);
      },
    );
  }

  void _confirmRemoveApp() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Remove App?'),
        content: Text('This will remove ${widget.app.label} from the device.'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('Cancel'),
          ),
          TextButton(
            onPressed: () async {
              Navigator.pop(context);
              Navigator.pop(context);
              await _appRepository.removeApp(widget.deviceId, widget.app.packageName);
            },
            child: const Text('Remove', style: TextStyle(color: Colors.red)),
          ),
        ],
      ),
    );
  }
}

class _AddAppDialog extends StatefulWidget {
  final String deviceId;

  const _AddAppDialog({required this.deviceId});

  @override
  State<_AddAppDialog> createState() => _AddAppDialogState();
}

class _AddAppDialogState extends State<_AddAppDialog> {
  final TextEditingController _controller = TextEditingController();
  final AppRepository _appRepository = AppRepository();

  // Popular apps for quick selection
  final List<Map<String, String>> _popularApps = [
    {'name': 'Spotify', 'package': 'com.spotify.music'},
    {'name': 'JioSaavn', 'package': 'com.jio.saavn'},
    {'name': 'WhatsApp', 'package': 'com.whatsapp'},
    {'name': 'Google Pay', 'package': 'com.google.android.apps.nbu.paisa.user'},
    {'name': 'PhonePe', 'package': 'com.phonepe.app'},
    {'name': 'Duolingo', 'package': 'com.duolingo'},
  ];

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: const Text('Add App'),
      content: Column(
        mainAxisSize: MainAxisSize.min,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          TextField(
            controller: _controller,
            decoration: const InputDecoration(
              labelText: 'Package Name',
              hintText: 'com.example.app',
            ),
          ),
          const SizedBox(height: 16),
          const Text('Popular Apps:', style: TextStyle(fontWeight: FontWeight.bold)),
          const SizedBox(height: 8),
          Wrap(
            spacing: 8,
            runSpacing: 8,
            children: _popularApps.map((app) {
              return ActionChip(
                label: Text(app['name']!),
                onPressed: () {
                  _controller.text = app['package']!;
                },
              );
            }).toList(),
          ),
        ],
      ),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context),
          child: const Text('Cancel'),
        ),
        ElevatedButton(
          onPressed: () async {
            final packageName = _controller.text.trim();
            if (packageName.isNotEmpty) {
              await _appRepository.installApp(widget.deviceId, packageName);
              if (context.mounted) Navigator.pop(context);
            }
          },
          child: const Text('Install'),
        ),
      ],
    );
  }
}
```

---

## Epic E08: Screen Time

### Story E08-S01: Screen Time Tracking (Child)
**Points**: 8
**Owner**: Android Developer
**Priority**: P0

#### Description
Track app usage on child device and enforce screen time limits.

#### Tasks

##### Task E08-S01-T01: Create Screen Time Service
**Time**: 2 hours

**IMPORTANT: UsageStatsManager Permission Required**

The `UsageStatsManager` API requires a special permission that users must manually grant in Settings. This cannot be granted automatically. However, **Device Owner** apps CAN grant this permission programmatically.

Create `app/src/main/java/com/kidtunes/launcher/util/UsageStatsPermissionChecker.kt`:

```kotlin
package com.kidtunes.launcher.util

import android.app.Activity
import android.app.AppOpsManager
import android.content.Context
import android.content.Intent
import android.os.Build
import android.provider.Settings
import android.net.Uri

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

        // Try to deep link to our app's setting (API 29+)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            intent.data = Uri.fromParts("package", activity.packageName, null)
        }

        activity.startActivity(intent)
    }
}
```

**Grant permission via Device Owner (in ProvisioningActivity):**

```kotlin
// In ProvisioningActivity - grant UsageStats permission via Device Owner
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

Create `app/src/main/java/com/kidtunes/launcher/services/ScreenTimeService.kt`:

```kotlin
package com.kidtunes.launcher.services

import android.app.Service
import android.app.usage.UsageStats
import android.app.usage.UsageStatsManager
import android.content.Context
import android.content.Intent
import android.os.IBinder
import androidx.core.app.NotificationCompat
import com.google.firebase.firestore.FirebaseFirestore
import com.kidtunes.launcher.R
import com.kidtunes.launcher.data.local.LauncherPreferences
import com.kidtunes.launcher.ui.lockscreen.ScreenTimeLockActivity
import com.kidtunes.launcher.util.UsageStatsPermissionChecker
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.*
import java.util.*
import javax.inject.Inject

@AndroidEntryPoint
class ScreenTimeService : Service() {

    @Inject
    lateinit var preferences: LauncherPreferences

    private val serviceScope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private val firestore = FirebaseFirestore.getInstance()

    private var usageStatsManager: UsageStatsManager? = null
    private var trackingJob: Job? = null

    private var todayUsageMinutes: Int = 0
    private var dailyLimitMinutes: Int = 120
    private var bedtimeStart: String? = null
    private var bedtimeEnd: String? = null

    override fun onCreate() {
        super.onCreate()
        usageStatsManager = getSystemService(Context.USAGE_STATS_SERVICE) as UsageStatsManager

        loadSettings()
        startTracking()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        startForeground()
        return START_STICKY
    }

    override fun onBind(intent: Intent?): IBinder? = null

    private fun startForeground() {
        val notification = NotificationCompat.Builder(this, "screen_time_channel")
            .setContentTitle("Screen Time Active")
            .setContentText("Tracking usage")
            .setSmallIcon(R.drawable.ic_notification)
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .build()

        startForeground(NOTIFICATION_ID, notification)
    }

    private fun loadSettings() {
        dailyLimitMinutes = preferences.getDailyLimitMinutes()
        bedtimeStart = preferences.getBedtimeStart()
        bedtimeEnd = preferences.getBedtimeEnd()
    }

    private fun startTracking() {
        trackingJob = serviceScope.launch {
            while (isActive) {
                checkScreenTime()
                checkBedtime()
                syncUsageToFirebase()

                delay(60_000) // Check every minute
            }
        }
    }

    private fun checkScreenTime() {
        todayUsageMinutes = getTodayUsageMinutes()

        if (todayUsageMinutes >= dailyLimitMinutes) {
            showLimitReachedScreen()
        } else if (todayUsageMinutes >= dailyLimitMinutes - 15) {
            // 15 minute warning
            showWarningNotification(dailyLimitMinutes - todayUsageMinutes)
        }
    }

    private fun checkBedtime() {
        if (bedtimeStart == null || bedtimeEnd == null) return

        if (isInBedtimeWindow()) {
            showBedtimeScreen()
        }
    }

    private fun isInBedtimeWindow(): Boolean {
        val now = Calendar.getInstance()
        val currentMinutes = now.get(Calendar.HOUR_OF_DAY) * 60 + now.get(Calendar.MINUTE)

        val startParts = bedtimeStart!!.split(":")
        val startMinutes = startParts[0].toInt() * 60 + startParts[1].toInt()

        val endParts = bedtimeEnd!!.split(":")
        val endMinutes = endParts[0].toInt() * 60 + endParts[1].toInt()

        return if (startMinutes > endMinutes) {
            // Crosses midnight (e.g., 20:30 - 07:00)
            currentMinutes >= startMinutes || currentMinutes < endMinutes
        } else {
            currentMinutes in startMinutes until endMinutes
        }
    }

    private fun getTodayUsageMinutes(): Int {
        // CRITICAL: Check permission before querying
        if (!UsageStatsPermissionChecker.hasPermission(this)) {
            Log.w(TAG, "Usage stats permission not granted")
            // Permission should have been granted during provisioning
            // If not, show notification to admin
            return 0
        }

        val calendar = Calendar.getInstance()
        calendar.set(Calendar.HOUR_OF_DAY, 0)
        calendar.set(Calendar.MINUTE, 0)
        calendar.set(Calendar.SECOND, 0)

        val startTime = calendar.timeInMillis
        val endTime = System.currentTimeMillis()

        val usageStats = usageStatsManager?.queryUsageStats(
            UsageStatsManager.INTERVAL_DAILY,
            startTime,
            endTime
        ) ?: return 0

        var totalTime = 0L
        for (stats in usageStats) {
            totalTime += stats.totalTimeInForeground
        }

        return (totalTime / 60000).toInt() // Convert to minutes
    }

    fun getAppUsageToday(): Map<String, Int> {
        val calendar = Calendar.getInstance()
        calendar.set(Calendar.HOUR_OF_DAY, 0)
        calendar.set(Calendar.MINUTE, 0)

        val usageStats = usageStatsManager?.queryUsageStats(
            UsageStatsManager.INTERVAL_DAILY,
            calendar.timeInMillis,
            System.currentTimeMillis()
        ) ?: return emptyMap()

        return usageStats.associate {
            it.packageName to (it.totalTimeInForeground / 60000).toInt()
        }
    }

    private fun showLimitReachedScreen() {
        val intent = Intent(this, ScreenTimeLockActivity::class.java).apply {
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            putExtra("reason", "limit_reached")
        }
        startActivity(intent)
    }

    private fun showBedtimeScreen() {
        val intent = Intent(this, com.kidtunes.launcher.ui.lockscreen.BedtimeLockActivity::class.java).apply {
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            putExtra("bedtimeEnd", bedtimeEnd)
        }
        startActivity(intent)
    }

    private fun showWarningNotification(minutesLeft: Int) {
        val notification = NotificationCompat.Builder(this, "screen_time_channel")
            .setContentTitle("Screen Time Warning")
            .setContentText("$minutesLeft minutes left for today")
            .setSmallIcon(R.drawable.ic_warning)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .build()

        val notificationManager = getSystemService(Context.NOTIFICATION_SERVICE)
            as android.app.NotificationManager
        notificationManager.notify(WARNING_NOTIFICATION_ID, notification)
    }

    private suspend fun syncUsageToFirebase() {
        val deviceId = preferences.getDeviceId() ?: return
        val today = java.text.SimpleDateFormat("yyyy-MM-dd", Locale.US).format(Date())

        val usageData = mapOf(
            "date" to today,
            "totalMinutes" to todayUsageMinutes,
            "apps" to getAppUsageToday(),
            "updatedAt" to com.google.firebase.Timestamp.now()
        )

        firestore.collection("devices")
            .document(deviceId)
            .collection("usage")
            .document(today)
            .set(usageData, com.google.firebase.firestore.SetOptions.merge())
    }

    fun updateDailyLimit(minutes: Int) {
        dailyLimitMinutes = minutes
        checkScreenTime()
    }

    fun updateBedtime(start: String, end: String) {
        bedtimeStart = start
        bedtimeEnd = end
        preferences.setBedtimeStart(start)
        preferences.setBedtimeEnd(end)
    }

    override fun onDestroy() {
        super.onDestroy()
        trackingJob?.cancel()
        serviceScope.cancel()
    }

    companion object {
        private const val TAG = "ScreenTimeService"
        private const val NOTIFICATION_ID = 1001
        private const val WARNING_NOTIFICATION_ID = 1002
    }
}
```

---

##### Task E08-S01-T01B: WorkManager Alternative for Battery Optimization (RECOMMENDED)
**Time**: 1.5 hours

**Problem**: Foreground services can be killed by aggressive battery optimization on some devices (Samsung, Xiaomi, Huawei). WorkManager is more battery-efficient and has system-level guarantees.

**Solution**: Use WorkManager for periodic checks, combined with AlarmManager for precise bedtime scheduling.

Create `app/src/main/java/com/kidtunes/launcher/workers/ScreenTimeWorker.kt`:

```kotlin
package com.kidtunes.launcher.workers

import android.content.Context
import android.content.Intent
import android.util.Log
import androidx.hilt.work.HiltWorker
import androidx.work.*
import com.google.firebase.firestore.FirebaseFirestore
import com.kidtunes.launcher.data.local.LauncherPreferences
import com.kidtunes.launcher.ui.lockscreen.ScreenTimeLockActivity
import com.kidtunes.launcher.util.UsageStatsPermissionChecker
import dagger.assisted.Assisted
import dagger.assisted.AssistedInject
import java.text.SimpleDateFormat
import java.util.*
import java.util.concurrent.TimeUnit

@HiltWorker
class ScreenTimeWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val preferences: LauncherPreferences
) : CoroutineWorker(context, params) {

    private val firestore = FirebaseFirestore.getInstance()

    override suspend fun doWork(): Result {
        return try {
            // Check screen time
            if (!UsageStatsPermissionChecker.hasPermission(applicationContext)) {
                Log.w(TAG, "Usage stats permission not granted")
                return Result.success()
            }

            val usageTracker = UsageTracker(applicationContext)
            val todayUsage = usageTracker.getTodayUsageMinutes()
            val dailyLimit = preferences.getDailyLimitMinutes()

            // Check if limit reached
            if (todayUsage >= dailyLimit) {
                showLimitReachedScreen()
            } else if (todayUsage >= dailyLimit - 10) {
                showWarningNotification(dailyLimit - todayUsage)
            }

            // Check bedtime
            val bedtimeStart = preferences.getBedtimeStart()
            val bedtimeEnd = preferences.getBedtimeEnd()
            if (bedtimeStart != null && bedtimeEnd != null) {
                if (isInBedtimeWindow(bedtimeStart, bedtimeEnd)) {
                    showBedtimeScreen()
                }
            }

            // Sync usage to Firebase
            syncUsageToFirebase(usageTracker)

            Result.success()
        } catch (e: Exception) {
            Log.e(TAG, "ScreenTimeWorker failed", e)
            Result.retry()
        }
    }

    private fun showLimitReachedScreen() {
        val intent = Intent(applicationContext, ScreenTimeLockActivity::class.java).apply {
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            putExtra("reason", "limit_reached")
        }
        applicationContext.startActivity(intent)
    }

    private suspend fun syncUsageToFirebase(usageTracker: UsageTracker) {
        val deviceId = preferences.getDeviceId() ?: return
        val today = SimpleDateFormat("yyyy-MM-dd", Locale.US).format(Date())

        val usageData = mapOf(
            "date" to today,
            "totalMinutes" to usageTracker.getTodayUsageMinutes(),
            "apps" to usageTracker.getAppUsageToday(),
            "updatedAt" to com.google.firebase.Timestamp.now()
        )

        firestore.collection("devices")
            .document(deviceId)
            .collection("usage")
            .document(today)
            .set(usageData, com.google.firebase.firestore.SetOptions.merge())
    }

    companion object {
        private const val TAG = "ScreenTimeWorker"
        private const val WORK_NAME = "screen_time_check"

        fun schedule(context: Context) {
            val constraints = Constraints.Builder()
                .setRequiresBatteryNotLow(false) // Run even on low battery
                .build()

            // Android enforces minimum 15 minute interval for periodic work
            val request = PeriodicWorkRequestBuilder<ScreenTimeWorker>(
                15, TimeUnit.MINUTES
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

        // For immediate check when needed
        fun checkNow(context: Context) {
            val request = OneTimeWorkRequestBuilder<ScreenTimeWorker>()
                .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
                .build()

            WorkManager.getInstance(context).enqueue(request)
        }

        fun cancel(context: Context) {
            WorkManager.getInstance(context).cancelUniqueWork(WORK_NAME)
        }
    }
}
```

**For precise bedtime enforcement, use AlarmManager:**

```kotlin
// BedtimeAlarmReceiver.kt
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

            // Use setExactAndAllowWhileIdle for reliability on Doze mode
            alarmManager.setExactAndAllowWhileIdle(
                AlarmManager.RTC_WAKEUP,
                startCalendar.timeInMillis,
                startPendingIntent
            )

            // Similar for end time...
        }
    }
}
```

**Request Battery Optimization Exemption:**

```kotlin
// Call during device setup
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

**Recommendation:** Use the WorkManager approach (this task) instead of the foreground service approach to reduce battery drain and improve reliability across different device manufacturers.

---

##### Task E08-S01-T02: Create Screen Time Lock Activity
**Time**: 1 hour

Create `app/src/main/java/com/kidtunes/launcher/ui/lockscreen/ScreenTimeLockActivity.kt`:

```kotlin
package com.kidtunes.launcher.ui.lockscreen

import android.os.Bundle
import android.view.WindowManager
import androidx.appcompat.app.AppCompatActivity
import com.kidtunes.launcher.databinding.ActivityScreenTimeLockBinding

class ScreenTimeLockActivity : AppCompatActivity() {

    private lateinit var binding: ActivityScreenTimeLockBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityScreenTimeLockBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // Make it hard to dismiss
        window.addFlags(
            WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON or
            WindowManager.LayoutParams.FLAG_DISMISS_KEYGUARD or
            WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED
        )

        setupUI()
    }

    private fun setupUI() {
        binding.titleText.text = "Time's up for today!"
        binding.messageText.text = "You've used all your screen time.\nCome back tomorrow!"

        binding.askMoreTimeButton.setOnClickListener {
            // Send request to parent
            requestMoreTime()
        }

        binding.emergencyButton.setOnClickListener {
            // Allow emergency access
            finish()
        }
    }

    private fun requestMoreTime() {
        // Send notification to parent app
        // This will be implemented with FCM
    }

    override fun onBackPressed() {
        // Don't allow back press
    }
}
```

---

## Sprint 3 Checklist

### Days 28-30: Command System
- [ ] Command schema defined
- [ ] CommandService (Parent) created
- [ ] FCMService (Child) created
- [ ] CommandExecutor created
- [ ] Commands execute in < 5 seconds

### Days 31-33: App Management
- [ ] AppRepository created
- [ ] App list displays
- [ ] Allow/Block toggle works
- [ ] Per-app time limits
- [ ] App installation command

### Days 34-37: Screen Time
- [ ] ScreenTimeService created
- [ ] Usage tracking works
- [ ] Daily limits enforced
- [ ] Bedtime mode works
- [ ] Usage syncs to Firebase
- [ ] **UsageStats permission granted via Device Owner** (critical)
- [ ] **WorkManager scheduled for battery efficiency** (optional but recommended)
- [ ] **Battery optimization exemption requested**

---

## Sprint 3 Deliverables

1. ✅ Real-time command system
2. ✅ App whitelist management
3. ✅ Remote app installation
4. ✅ Screen time tracking
5. ✅ Daily limit enforcement
6. ✅ Bedtime mode
7. ✅ Usage data in Firebase
