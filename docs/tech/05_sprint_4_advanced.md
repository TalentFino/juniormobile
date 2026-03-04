# Sprint 4: Advanced Features (Days 50-70)

## Sprint Goal
Implement location tracking, advanced parental controls (web filtering, call control, app-specific settings), usage analytics, and remote app installation.

> **Note**: Sprint timeline updated to Days 50-70 (3 weeks) to accommodate advanced parental control features.

---

## Epic E09: Location & Safety

### Story E09-S01: Location Tracking
**Points**: 8
**Owner**: Both
**Priority**: P1

#### Description
Implement real-time location tracking with safe zone alerts.

#### Tasks

##### Task E09-S01-T01: Create Location Service (Child)
**Time**: 2 hours

Create `app/src/main/java/com/kidtunes/launcher/services/LocationService.kt`:

```kotlin
package com.kidtunes.launcher.services

import android.Manifest
import android.annotation.SuppressLint
import android.app.Service
import android.content.Intent
import android.content.pm.PackageManager
import android.location.Location
import android.os.IBinder
import android.os.Looper
import androidx.core.app.ActivityCompat
import androidx.core.app.NotificationCompat
import com.google.android.gms.location.*
import com.google.firebase.database.FirebaseDatabase
import com.google.firebase.firestore.FirebaseFirestore
import com.kidtunes.launcher.R
import com.kidtunes.launcher.data.local.LauncherPreferences
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.*
import javax.inject.Inject

@AndroidEntryPoint
class LocationService : Service() {

    @Inject
    lateinit var preferences: LauncherPreferences

    private lateinit var fusedLocationClient: FusedLocationProviderClient
    private lateinit var locationCallback: LocationCallback

    private val serviceScope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private val firestore = FirebaseFirestore.getInstance()
    private val rtdb = FirebaseDatabase.getInstance()

    private var isTracking = false
    private var updateInterval = NORMAL_INTERVAL

    companion object {
        const val NORMAL_INTERVAL = 15 * 60 * 1000L  // 15 minutes
        const val ACTIVE_INTERVAL = 1 * 60 * 1000L   // 1 minute
        const val PANIC_INTERVAL = 30 * 1000L        // 30 seconds
        private const val NOTIFICATION_ID = 2001
    }

    override fun onCreate() {
        super.onCreate()
        fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)
        setupLocationCallback()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            "START_TRACKING" -> startTracking()
            "STOP_TRACKING" -> stopTracking()
            "PANIC_MODE" -> enablePanicMode()
            "REQUEST_LOCATION" -> requestSingleLocation()
        }
        return START_STICKY
    }

    override fun onBind(intent: Intent?): IBinder? = null

    private fun setupLocationCallback() {
        locationCallback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                result.lastLocation?.let { location ->
                    serviceScope.launch {
                        updateLocation(location)
                    }
                }
            }
        }
    }

    @SuppressLint("MissingPermission")
    private fun startTracking() {
        if (isTracking) return
        if (!hasLocationPermission()) return

        isTracking = true
        startForeground()

        val locationRequest = LocationRequest.Builder(
            Priority.PRIORITY_BALANCED_POWER_ACCURACY,
            updateInterval
        ).apply {
            setMinUpdateIntervalMillis(updateInterval / 2)
            setWaitForAccurateLocation(false)
        }.build()

        fusedLocationClient.requestLocationUpdates(
            locationRequest,
            locationCallback,
            Looper.getMainLooper()
        )
    }

    private fun stopTracking() {
        isTracking = false
        fusedLocationClient.removeLocationUpdates(locationCallback)
        stopForeground(STOP_FOREGROUND_REMOVE)
    }

    @SuppressLint("MissingPermission")
    private fun enablePanicMode() {
        updateInterval = PANIC_INTERVAL

        // Restart with faster updates
        if (isTracking) {
            fusedLocationClient.removeLocationUpdates(locationCallback)
        }

        val locationRequest = LocationRequest.Builder(
            Priority.PRIORITY_HIGH_ACCURACY,
            PANIC_INTERVAL
        ).apply {
            setMinUpdateIntervalMillis(PANIC_INTERVAL / 2)
        }.build()

        fusedLocationClient.requestLocationUpdates(
            locationRequest,
            locationCallback,
            Looper.getMainLooper()
        )
    }

    @SuppressLint("MissingPermission")
    suspend fun getCurrentLocation(): Location? = suspendCancellableCoroutine { continuation ->
        if (!hasLocationPermission()) {
            continuation.resume(null) {}
            return@suspendCancellableCoroutine
        }

        fusedLocationClient.getCurrentLocation(
            Priority.PRIORITY_HIGH_ACCURACY,
            null
        ).addOnSuccessListener { location ->
            continuation.resume(location) {}
        }.addOnFailureListener {
            continuation.resume(null) {}
        }
    }

    @SuppressLint("MissingPermission")
    private fun requestSingleLocation() {
        if (!hasLocationPermission()) return

        fusedLocationClient.getCurrentLocation(
            Priority.PRIORITY_HIGH_ACCURACY,
            null
        ).addOnSuccessListener { location ->
            location?.let {
                serviceScope.launch {
                    updateLocation(it)
                }
            }
        }
    }

    private suspend fun updateLocation(location: Location) {
        val deviceId = preferences.getDeviceId() ?: return

        val locationData = mapOf(
            "latitude" to location.latitude,
            "longitude" to location.longitude,
            "accuracy" to location.accuracy,
            "altitude" to location.altitude,
            "speed" to location.speed,
            "bearing" to location.bearing,
            "timestamp" to com.google.firebase.Timestamp.now(),
            "provider" to location.provider
        )

        // Update in Realtime Database for live tracking
        rtdb.getReference("devices/$deviceId/status/location")
            .setValue(locationData)

        // Also store in Firestore for history
        firestore.collection("devices")
            .document(deviceId)
            .update("lastLocation", locationData)

        // Check safe zones
        checkSafeZones(location)
    }

    private suspend fun checkSafeZones(location: Location) {
        val deviceId = preferences.getDeviceId() ?: return

        // Get safe zones from Firestore
        val safeZonesDoc = firestore.collection("devices")
            .document(deviceId)
            .collection("safeZones")
            .get()
            .await()

        for (doc in safeZonesDoc.documents) {
            val lat = doc.getDouble("latitude") ?: continue
            val lng = doc.getDouble("longitude") ?: continue
            val radius = doc.getDouble("radius") ?: 500.0
            val name = doc.getString("name") ?: "Safe Zone"

            val zoneLocation = Location("").apply {
                latitude = lat
                longitude = lng
            }

            val distance = location.distanceTo(zoneLocation)

            if (distance > radius) {
                // Child left safe zone - notify parent
                notifyLeftSafeZone(name)
            }
        }
    }

    private fun notifyLeftSafeZone(zoneName: String) {
        val deviceId = preferences.getDeviceId() ?: return

        // Send alert to parent via Firebase
        firestore.collection("devices")
            .document(deviceId)
            .collection("alerts")
            .add(mapOf(
                "type" to "LEFT_SAFE_ZONE",
                "zone" to zoneName,
                "timestamp" to com.google.firebase.Timestamp.now(),
                "acknowledged" to false
            ))
    }

    private fun hasLocationPermission(): Boolean {
        return ActivityCompat.checkSelfPermission(
            this,
            Manifest.permission.ACCESS_FINE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED
    }

    private fun startForeground() {
        val notification = NotificationCompat.Builder(this, "location_channel")
            .setContentTitle("Location Active")
            .setContentText("Your location is being shared with your parent")
            .setSmallIcon(R.drawable.ic_location)
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .setOngoing(true)
            .build()

        startForeground(NOTIFICATION_ID, notification)
    }

    override fun onDestroy() {
        super.onDestroy()
        stopTracking()
        serviceScope.cancel()
    }
}

private suspend fun <T> com.google.android.gms.tasks.Task<T>.await(): T =
    suspendCancellableCoroutine { continuation ->
        addOnSuccessListener { continuation.resume(it) {} }
        addOnFailureListener { continuation.resumeWithException(it) }
    }
```

##### Task E09-S01-T02: Create Location Screen (Parent)
**Time**: 2 hours

Create `lib/presentation/screens/location/location_screen.dart`:

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:google_maps_flutter/google_maps_flutter.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_database/firebase_database.dart';
import 'package:timeago/timeago.dart' as timeago;

import '../../../core/theme/app_theme.dart';
import '../../../data/services/command_service.dart';

class LocationScreen extends ConsumerStatefulWidget {
  final String deviceId;

  const LocationScreen({super.key, required this.deviceId});

  @override
  ConsumerState<LocationScreen> createState() => _LocationScreenState();
}

class _LocationScreenState extends ConsumerState<LocationScreen> {
  final CommandService _commandService = CommandService();
  GoogleMapController? _mapController;

  LatLng? _currentLocation;
  DateTime? _lastUpdated;
  Set<Marker> _markers = {};
  Set<Circle> _safeZones = {};

  StreamSubscription? _locationSubscription;

  @override
  void initState() {
    super.initState();
    _listenToLocation();
    _loadSafeZones();
  }

  void _listenToLocation() {
    final ref = FirebaseDatabase.instance
        .ref('devices/${widget.deviceId}/status/location');

    _locationSubscription = ref.onValue.listen((event) {
      if (event.snapshot.value == null) return;

      final data = event.snapshot.value as Map;
      final lat = data['latitude'] as double?;
      final lng = data['longitude'] as double?;
      final timestamp = data['timestamp'];

      if (lat != null && lng != null) {
        setState(() {
          _currentLocation = LatLng(lat, lng);
          if (timestamp != null) {
            _lastUpdated = DateTime.fromMillisecondsSinceEpoch(
              (timestamp['_seconds'] as int) * 1000,
            );
          }
          _updateMarker();
        });

        _mapController?.animateCamera(
          CameraUpdate.newLatLng(_currentLocation!),
        );
      }
    });
  }

  void _loadSafeZones() async {
    final snapshot = await FirebaseFirestore.instance
        .collection('devices')
        .doc(widget.deviceId)
        .collection('safeZones')
        .get();

    final zones = <Circle>{};
    for (final doc in snapshot.docs) {
      final data = doc.data();
      zones.add(Circle(
        circleId: CircleId(doc.id),
        center: LatLng(data['latitude'], data['longitude']),
        radius: data['radius']?.toDouble() ?? 500,
        fillColor: AppTheme.primaryColor.withOpacity(0.2),
        strokeColor: AppTheme.primaryColor,
        strokeWidth: 2,
      ));
    }

    setState(() {
      _safeZones = zones;
    });
  }

  void _updateMarker() {
    if (_currentLocation == null) return;

    _markers = {
      Marker(
        markerId: const MarkerId('child'),
        position: _currentLocation!,
        icon: BitmapDescriptor.defaultMarkerWithHue(BitmapDescriptor.hueAzure),
        infoWindow: InfoWindow(
          title: 'Current Location',
          snippet: _lastUpdated != null
              ? 'Updated ${timeago.format(_lastUpdated!)}'
              : null,
        ),
      ),
    };
  }

  void _refreshLocation() {
    _commandService.requestLocation(widget.deviceId);
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('Requesting location...')),
    );
  }

  void _playSound() {
    _commandService.playSound(widget.deviceId);
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('Playing sound on device...')),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Location'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: _refreshLocation,
            tooltip: 'Refresh Location',
          ),
          IconButton(
            icon: const Icon(Icons.volume_up),
            onPressed: _playSound,
            tooltip: 'Play Sound',
          ),
        ],
      ),
      body: Column(
        children: [
          // Map
          Expanded(
            flex: 3,
            child: _currentLocation == null
                ? const Center(child: CircularProgressIndicator())
                : GoogleMap(
                    initialCameraPosition: CameraPosition(
                      target: _currentLocation!,
                      zoom: 15,
                    ),
                    markers: _markers,
                    circles: _safeZones,
                    onMapCreated: (controller) {
                      _mapController = controller;
                    },
                    myLocationEnabled: false,
                    zoomControlsEnabled: false,
                  ),
          ),

          // Location Info
          Container(
            padding: const EdgeInsets.all(16),
            decoration: BoxDecoration(
              color: Colors.white,
              boxShadow: [
                BoxShadow(
                  color: Colors.black.withOpacity(0.1),
                  blurRadius: 10,
                  offset: const Offset(0, -4),
                ),
              ],
            ),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Row(
                  children: [
                    Icon(Icons.location_on, color: AppTheme.primaryColor),
                    const SizedBox(width: 8),
                    Text(
                      'Current Location',
                      style: Theme.of(context).textTheme.titleMedium?.copyWith(
                            fontWeight: FontWeight.bold,
                          ),
                    ),
                  ],
                ),
                const SizedBox(height: 8),
                if (_lastUpdated != null)
                  Text(
                    'Last updated ${timeago.format(_lastUpdated!)}',
                    style: TextStyle(color: AppTheme.textSecondary),
                  ),
                const SizedBox(height: 16),
                Row(
                  children: [
                    Expanded(
                      child: OutlinedButton.icon(
                        onPressed: () => _showAddSafeZoneDialog(),
                        icon: const Icon(Icons.add_location_alt),
                        label: const Text('Add Safe Zone'),
                      ),
                    ),
                    const SizedBox(width: 16),
                    Expanded(
                      child: OutlinedButton.icon(
                        onPressed: () => _showLocationHistory(),
                        icon: const Icon(Icons.history),
                        label: const Text('History'),
                      ),
                    ),
                  ],
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }

  void _showAddSafeZoneDialog() {
    final nameController = TextEditingController();
    double radius = 500;

    showDialog(
      context: context,
      builder: (context) => StatefulBuilder(
        builder: (context, setState) => AlertDialog(
          title: const Text('Add Safe Zone'),
          content: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              TextField(
                controller: nameController,
                decoration: const InputDecoration(
                  labelText: 'Zone Name',
                  hintText: 'e.g., Home, School',
                ),
              ),
              const SizedBox(height: 16),
              Text('Radius: ${radius.toInt()} meters'),
              Slider(
                value: radius,
                min: 100,
                max: 2000,
                divisions: 19,
                onChanged: (value) => setState(() => radius = value),
              ),
              const SizedBox(height: 8),
              const Text(
                'Safe zone will be set at the current device location',
                style: TextStyle(fontSize: 12, color: Colors.grey),
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
                if (nameController.text.isEmpty || _currentLocation == null) return;

                await FirebaseFirestore.instance
                    .collection('devices')
                    .doc(widget.deviceId)
                    .collection('safeZones')
                    .add({
                  'name': nameController.text,
                  'latitude': _currentLocation!.latitude,
                  'longitude': _currentLocation!.longitude,
                  'radius': radius,
                  'createdAt': FieldValue.serverTimestamp(),
                });

                if (mounted) {
                  Navigator.pop(context);
                  _loadSafeZones();
                }
              },
              child: const Text('Add'),
            ),
          ],
        ),
      ),
    );
  }

  void _showLocationHistory() {
    // Navigate to location history screen
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      builder: (context) => DraggableScrollableSheet(
        initialChildSize: 0.6,
        builder: (context, scrollController) => Container(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                'Location History',
                style: Theme.of(context).textTheme.titleLarge,
              ),
              const SizedBox(height: 16),
              Expanded(
                child: StreamBuilder<QuerySnapshot>(
                  stream: FirebaseFirestore.instance
                      .collection('devices')
                      .doc(widget.deviceId)
                      .collection('locationHistory')
                      .orderBy('timestamp', descending: true)
                      .limit(50)
                      .snapshots(),
                  builder: (context, snapshot) {
                    if (!snapshot.hasData) {
                      return const Center(child: CircularProgressIndicator());
                    }

                    final docs = snapshot.data!.docs;
                    if (docs.isEmpty) {
                      return const Center(child: Text('No history yet'));
                    }

                    return ListView.builder(
                      controller: scrollController,
                      itemCount: docs.length,
                      itemBuilder: (context, index) {
                        final data = docs[index].data() as Map<String, dynamic>;
                        final timestamp = data['timestamp'] as Timestamp?;

                        return ListTile(
                          leading: const Icon(Icons.location_on),
                          title: Text(data['address'] ?? 'Unknown location'),
                          subtitle: timestamp != null
                              ? Text(timeago.format(timestamp.toDate()))
                              : null,
                        );
                      },
                    );
                  },
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }

  @override
  void dispose() {
    _locationSubscription?.cancel();
    _mapController?.dispose();
    super.dispose();
  }
}
```

---

## Epic E10: Usage Analytics

### Story E10-S01: Dashboard with Analytics
**Points**: 5
**Owner**: Flutter Developer
**Priority**: P1

#### Tasks

##### Task E10-S01-T01: Create Dashboard Screen
**Time**: 2.5 hours

Create `lib/presentation/screens/dashboard/dashboard_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:firebase_database/firebase_database.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:go_router/go_router.dart';
import 'package:timeago/timeago.dart' as timeago;

import '../../../core/theme/app_theme.dart';
import '../../../data/models/device.dart';
import '../../../data/models/child_profile.dart';
import '../../../data/services/command_service.dart';
import '../../providers/auth_provider.dart';
import '../../widgets/usage_chart.dart';

class DashboardScreen extends ConsumerStatefulWidget {
  const DashboardScreen({super.key});

  @override
  ConsumerState<DashboardScreen> createState() => _DashboardScreenState();
}

class _DashboardScreenState extends ConsumerState<DashboardScreen> {
  final CommandService _commandService = CommandService();

  @override
  Widget build(BuildContext context) {
    final user = ref.watch(currentUserProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('KidTunes'),
        actions: [
          IconButton(
            icon: const Icon(Icons.settings),
            onPressed: () => context.push('/settings'),
          ),
        ],
      ),
      body: StreamBuilder<QuerySnapshot>(
        stream: FirebaseFirestore.instance
            .collection('users')
            .doc(user?.uid)
            .collection('children')
            .snapshots(),
        builder: (context, childSnapshot) {
          if (!childSnapshot.hasData) {
            return const Center(child: CircularProgressIndicator());
          }

          final children = childSnapshot.data!.docs
              .map((doc) => ChildProfile.fromFirestore(doc))
              .toList();

          if (children.isEmpty) {
            return _buildEmptyState();
          }

          return RefreshIndicator(
            onRefresh: () async {
              // Force refresh
              setState(() {});
            },
            child: ListView.builder(
              padding: const EdgeInsets.all(16),
              itemCount: children.length + 1,
              itemBuilder: (context, index) {
                if (index == children.length) {
                  return _buildAddChildButton();
                }
                return _buildChildCard(children[index]);
              },
            ),
          );
        },
      ),
    );
  }

  Widget _buildEmptyState() {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(
            Icons.child_care,
            size: 80,
            color: AppTheme.primaryColor.withOpacity(0.5),
          ),
          const SizedBox(height: 24),
          Text(
            'No children added yet',
            style: Theme.of(context).textTheme.titleLarge,
          ),
          const SizedBox(height: 8),
          const Text('Add a child to get started'),
          const SizedBox(height: 24),
          ElevatedButton.icon(
            onPressed: () => context.push('/onboarding/add-child'),
            icon: const Icon(Icons.add),
            label: const Text('Add Child'),
          ),
        ],
      ),
    );
  }

  Widget _buildChildCard(ChildProfile child) {
    if (child.deviceId == null) {
      return _buildUnpairedCard(child);
    }

    return StreamBuilder<DatabaseEvent>(
      stream: FirebaseDatabase.instance
          .ref('devices/${child.deviceId}/status')
          .onValue,
      builder: (context, statusSnapshot) {
        final status = statusSnapshot.data?.snapshot.value as Map?;
        final isOnline = status?['online'] == true;
        final battery = status?['battery'] as int? ?? 0;
        final kidsMode = status?['kidsMode'] as bool? ?? true;

        return Card(
          margin: const EdgeInsets.only(bottom: 16),
          child: InkWell(
            onTap: () => context.push('/device/${child.deviceId}'),
            borderRadius: BorderRadius.circular(16),
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  // Header
                  Row(
                    children: [
                      CircleAvatar(
                        radius: 24,
                        backgroundColor: AppTheme.primaryColor.withOpacity(0.2),
                        child: Text(
                          _getAvatarEmoji(child.avatar),
                          style: const TextStyle(fontSize: 24),
                        ),
                      ),
                      const SizedBox(width: 12),
                      Expanded(
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            Text(
                              "${child.name}'s Device",
                              style: const TextStyle(
                                fontWeight: FontWeight.bold,
                                fontSize: 16,
                              ),
                            ),
                            Row(
                              children: [
                                Container(
                                  width: 8,
                                  height: 8,
                                  decoration: BoxDecoration(
                                    shape: BoxShape.circle,
                                    color: isOnline
                                        ? AppTheme.successGreen
                                        : AppTheme.textHint,
                                  ),
                                ),
                                const SizedBox(width: 4),
                                Text(
                                  isOnline ? 'Online' : 'Offline',
                                  style: TextStyle(
                                    color: AppTheme.textSecondary,
                                    fontSize: 12,
                                  ),
                                ),
                              ],
                            ),
                          ],
                        ),
                      ),
                      // Battery
                      Row(
                        children: [
                          Icon(
                            _getBatteryIcon(battery),
                            size: 20,
                            color: _getBatteryColor(battery),
                          ),
                          const SizedBox(width: 4),
                          Text('$battery%'),
                        ],
                      ),
                    ],
                  ),

                  const SizedBox(height: 16),

                  // Now Playing (if available)
                  _buildNowPlaying(child.deviceId!),

                  const SizedBox(height: 16),

                  // Screen Time Progress
                  _buildScreenTimeProgress(child.deviceId!),

                  const SizedBox(height: 16),

                  // Quick Actions
                  Row(
                    mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                    children: [
                      _buildQuickAction(
                        icon: Icons.lock,
                        label: 'Lock',
                        onTap: () => _commandService.lockDevice(child.deviceId!),
                      ),
                      _buildQuickAction(
                        icon: kidsMode ? Icons.child_care : Icons.person,
                        label: kidsMode ? 'Kids' : 'Standard',
                        onTap: () => _commandService.toggleKidsMode(
                          child.deviceId!,
                          !kidsMode,
                        ),
                      ),
                      _buildQuickAction(
                        icon: Icons.location_on,
                        label: 'Locate',
                        onTap: () => context.push('/device/${child.deviceId}/location'),
                      ),
                      _buildQuickAction(
                        icon: Icons.apps,
                        label: 'Apps',
                        onTap: () => context.push('/device/${child.deviceId}/apps'),
                      ),
                    ],
                  ),
                ],
              ),
            ),
          ),
        );
      },
    );
  }

  Widget _buildUnpairedCard(ChildProfile child) {
    return Card(
      margin: const EdgeInsets.only(bottom: 16),
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            Icon(
              Icons.link_off,
              size: 48,
              color: AppTheme.warningOrange,
            ),
            const SizedBox(height: 12),
            Text(
              '${child.name} - No device paired',
              style: const TextStyle(fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            ElevatedButton(
              onPressed: () => context.push(
                '/onboarding/pair-device',
                extra: {'childId': child.id},
              ),
              child: const Text('Pair Device'),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildNowPlaying(String deviceId) {
    return StreamBuilder<DatabaseEvent>(
      stream: FirebaseDatabase.instance
          .ref('devices/$deviceId/nowPlaying')
          .onValue,
      builder: (context, snapshot) {
        final data = snapshot.data?.snapshot.value as Map?;
        if (data == null || data['track'] == null) {
          return const SizedBox.shrink();
        }

        return Container(
          padding: const EdgeInsets.all(12),
          decoration: BoxDecoration(
            color: AppTheme.primaryColor.withOpacity(0.1),
            borderRadius: BorderRadius.circular(12),
          ),
          child: Row(
            children: [
              const Icon(Icons.music_note, color: AppTheme.primaryColor),
              const SizedBox(width: 12),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      data['track'] ?? 'Unknown',
                      style: const TextStyle(fontWeight: FontWeight.bold),
                      maxLines: 1,
                      overflow: TextOverflow.ellipsis,
                    ),
                    Text(
                      data['artist'] ?? '',
                      style: TextStyle(color: AppTheme.textSecondary, fontSize: 12),
                      maxLines: 1,
                    ),
                  ],
                ),
              ),
            ],
          ),
        );
      },
    );
  }

  Widget _buildScreenTimeProgress(String deviceId) {
    return StreamBuilder<DocumentSnapshot>(
      stream: FirebaseFirestore.instance
          .collection('devices')
          .doc(deviceId)
          .snapshots(),
      builder: (context, snapshot) {
        final data = snapshot.data?.data() as Map<String, dynamic>?;
        final settings = data?['settings'] as Map<String, dynamic>?;
        final dailyLimit = settings?['dailyLimitMinutes'] as int? ?? 120;

        // Get today's usage
        final today = DateTime.now().toIso8601String().substring(0, 10);

        return FutureBuilder<DocumentSnapshot>(
          future: FirebaseFirestore.instance
              .collection('devices')
              .doc(deviceId)
              .collection('usage')
              .doc(today)
              .get(),
          builder: (context, usageSnapshot) {
            final usageData = usageSnapshot.data?.data() as Map<String, dynamic>?;
            final usedMinutes = usageData?['totalMinutes'] as int? ?? 0;
            final progress = (usedMinutes / dailyLimit).clamp(0.0, 1.0);

            return Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    const Text('Screen Time Today'),
                    Text(
                      '${usedMinutes}m / ${dailyLimit}m',
                      style: TextStyle(color: AppTheme.textSecondary),
                    ),
                  ],
                ),
                const SizedBox(height: 8),
                ClipRRect(
                  borderRadius: BorderRadius.circular(4),
                  child: LinearProgressIndicator(
                    value: progress,
                    backgroundColor: Colors.grey.shade200,
                    valueColor: AlwaysStoppedAnimation(
                      progress > 0.9
                          ? AppTheme.errorRed
                          : progress > 0.7
                              ? AppTheme.warningOrange
                              : AppTheme.successGreen,
                    ),
                    minHeight: 8,
                  ),
                ),
              ],
            );
          },
        );
      },
    );
  }

  Widget _buildQuickAction({
    required IconData icon,
    required String label,
    required VoidCallback onTap,
  }) {
    return InkWell(
      onTap: onTap,
      borderRadius: BorderRadius.circular(8),
      child: Padding(
        padding: const EdgeInsets.all(8),
        child: Column(
          children: [
            Icon(icon, color: AppTheme.primaryColor),
            const SizedBox(height: 4),
            Text(
              label,
              style: TextStyle(
                fontSize: 12,
                color: AppTheme.textSecondary,
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildAddChildButton() {
    return OutlinedButton.icon(
      onPressed: () => context.push('/onboarding/add-child'),
      icon: const Icon(Icons.add),
      label: const Text('Add Another Child'),
      style: OutlinedButton.styleFrom(
        padding: const EdgeInsets.all(16),
      ),
    );
  }

  String _getAvatarEmoji(String avatar) {
    switch (avatar) {
      case 'boy_1': return '👦';
      case 'girl_1': return '👧';
      case 'boy_2': return '👦🏽';
      case 'girl_2': return '👧🏻';
      default: return '👤';
    }
  }

  IconData _getBatteryIcon(int level) {
    if (level > 80) return Icons.battery_full;
    if (level > 50) return Icons.battery_5_bar;
    if (level > 20) return Icons.battery_3_bar;
    return Icons.battery_1_bar;
  }

  Color _getBatteryColor(int level) {
    if (level > 50) return AppTheme.successGreen;
    if (level > 20) return AppTheme.warningOrange;
    return AppTheme.errorRed;
  }
}
```

---

## Epic E11: Remote App Installation

### Story E11-S01: Managed Play Store
**Points**: 8
**Owner**: Android Developer
**Priority**: P2

#### Description
Enable remote app installation via managed Google Play (requires Device Owner).

#### Tasks

##### Task E11-S01-T01: Create Managed Play Manager
**Time**: 2 hours

Create `app/src/main/java/com/kidtunes/dpc/apps/ManagedPlayManager.kt`:

```kotlin
package com.kidtunes.dpc.apps

import android.app.admin.DevicePolicyManager
import android.content.ComponentName
import android.content.Context
import android.content.Intent
import android.net.Uri
import com.kidtunes.dpc.receivers.KidDeviceAdminReceiver

class ManagedPlayManager(private val context: Context) {

    private val dpm = context.getSystemService(Context.DEVICE_POLICY_SERVICE)
        as DevicePolicyManager
    private val componentName = KidDeviceAdminReceiver.getComponentName(context)

    /**
     * Check if we can manage apps (requires Device Owner)
     */
    fun canManageApps(): Boolean {
        return dpm.isDeviceOwnerApp(context.packageName)
    }

    /**
     * Enable system apps that might be disabled
     */
    fun enableSystemApp(packageName: String): Boolean {
        return try {
            dpm.enableSystemApp(componentName, packageName)
            true
        } catch (e: Exception) {
            false
        }
    }

    /**
     * Install app silently via managed Google Play
     * Note: This requires the device to be enrolled in managed Google Play
     */
    fun installApp(packageName: String): Boolean {
        if (!canManageApps()) return false

        return try {
            // Enable installation temporarily
            dpm.clearUserRestriction(
                componentName,
                android.os.UserManager.DISALLOW_INSTALL_APPS
            )

            // Open Play Store to install
            val intent = Intent(Intent.ACTION_VIEW).apply {
                data = Uri.parse("market://details?id=$packageName")
                addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            }
            context.startActivity(intent)

            // Note: For truly silent installation, you need:
            // 1. Device to be managed by Android Enterprise
            // 2. App to be approved in managed Google Play
            // 3. Use DevicePolicyManager.installExistingPackage()

            true
        } catch (e: Exception) {
            false
        }
    }

    /**
     * Uninstall an app
     */
    fun uninstallApp(packageName: String): Boolean {
        if (!canManageApps()) return false

        return try {
            // First unblock it
            dpm.setUninstallBlocked(componentName, packageName, false)

            // Then request uninstall
            val intent = Intent(Intent.ACTION_DELETE).apply {
                data = Uri.parse("package:$packageName")
                addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            }
            context.startActivity(intent)
            true
        } catch (e: Exception) {
            false
        }
    }

    /**
     * Hide an app (makes it invisible but keeps it installed)
     */
    fun hideApp(packageName: String, hidden: Boolean): Boolean {
        if (!canManageApps()) return false

        return try {
            dpm.setApplicationHidden(componentName, packageName, hidden)
            true
        } catch (e: Exception) {
            false
        }
    }

    /**
     * Suspend an app (makes it temporarily unusable)
     */
    fun suspendApp(packageName: String, suspended: Boolean): Boolean {
        if (!canManageApps()) return false

        return try {
            dpm.setPackagesSuspended(
                componentName,
                arrayOf(packageName),
                suspended
            )
            true
        } catch (e: Exception) {
            false
        }
    }

    /**
     * Get list of installed apps
     */
    fun getInstalledApps(): List<AppInfo> {
        val pm = context.packageManager
        val intent = Intent(Intent.ACTION_MAIN).apply {
            addCategory(Intent.CATEGORY_LAUNCHER)
        }

        return pm.queryIntentActivities(intent, 0).map { resolveInfo ->
            val appInfo = pm.getApplicationInfo(resolveInfo.activityInfo.packageName, 0)
            AppInfo(
                packageName = appInfo.packageName,
                label = pm.getApplicationLabel(appInfo).toString(),
                isSystemApp = (appInfo.flags and android.content.pm.ApplicationInfo.FLAG_SYSTEM) != 0,
                isEnabled = appInfo.enabled,
                isHidden = dpm.isApplicationHidden(componentName, appInfo.packageName)
            )
        }
    }

    data class AppInfo(
        val packageName: String,
        val label: String,
        val isSystemApp: Boolean,
        val isEnabled: Boolean,
        val isHidden: Boolean
    )
}
```

---

## Epic E12: Web Content Filtering (F19)

### Feasibility Summary
| Feature | Possible? | Method |
|---------|-----------|--------|
| Block adult websites | ✅ YES | DNS filtering via DPC |
| Category blocking | ✅ YES | CleanBrowsing/NextDNS |
| See browsing history | ❌ NO | Chrome is sandboxed |
| Block specific URLs | ✅ YES | DNS blocklist |

### Story E12-S01: DNS-Based Content Filtering
**Points**: 5
**Owner**: Android Developer
**Priority**: P2

#### Description
Use Device Policy Controller to set Private DNS to a content filtering service.

#### Tasks

##### Task E12-S01-T01: Create Content Filter Manager (Child)
**Time**: 2 hours

Create `app/src/main/java/com/kidtunes/dpc/filter/ContentFilterManager.kt`:

```kotlin
package com.kidtunes.dpc.filter

import android.app.admin.DevicePolicyManager
import android.content.ComponentName
import android.content.Context
import android.os.Build
import com.kidtunes.dpc.receivers.KidDeviceAdminReceiver
import com.kidtunes.launcher.data.local.LauncherPreferences
import dagger.hilt.android.qualifiers.ApplicationContext
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Manages DNS-based content filtering via Device Owner APIs.
 *
 * How it works:
 * 1. We set Private DNS to a filtering service (CleanBrowsing, NextDNS)
 * 2. All DNS queries go through the filter
 * 3. Blocked sites return NXDOMAIN (site not found)
 * 4. Zero battery impact, works system-wide
 */
@Singleton
class ContentFilterManager @Inject constructor(
    @ApplicationContext private val context: Context,
    private val preferences: LauncherPreferences
) {
    private val dpm = context.getSystemService(Context.DEVICE_POLICY_SERVICE)
        as DevicePolicyManager
    private val adminComponent = KidDeviceAdminReceiver.getComponentName(context)

    companion object {
        // Free DNS filtering services - no account required
        object DnsProviders {
            // CleanBrowsing - blocks adult, phishing, malware
            const val CLEANBROWSING_FAMILY = "family-filter-dns.cleanbrowsing.org"
            const val CLEANBROWSING_ADULT = "adult-filter-dns.cleanbrowsing.org"

            // Cloudflare for Families
            const val CLOUDFLARE_FAMILY = "family.cloudflare-dns.com"       // Adult + malware
            const val CLOUDFLARE_MALWARE = "security.cloudflare-dns.com"    // Malware only

            // AdGuard Family
            const val ADGUARD_FAMILY = "family.adguard-dns.com"
        }
    }

    /**
     * Check if we have Device Owner permission to set DNS
     */
    fun canSetDns(): Boolean {
        return dpm.isDeviceOwnerApp(context.packageName) &&
               Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q  // Android 10+
    }

    /**
     * Set content filtering level
     *
     * @param level The filtering level to apply
     * @return Result indicating success or failure with error message
     */
    fun setFilterLevel(level: FilterLevel): Result<Unit> {
        if (!canSetDns()) {
            return Result.failure(
                IllegalStateException("Not device owner or Android < 10")
            )
        }

        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.Q) {
            return Result.failure(
                UnsupportedOperationException("Private DNS requires Android 10+")
            )
        }

        return try {
            val dnsHost = when (level) {
                FilterLevel.STRICT -> DnsProviders.CLEANBROWSING_FAMILY
                FilterLevel.MODERATE -> DnsProviders.CLOUDFLARE_FAMILY
                FilterLevel.ADULT_ONLY -> DnsProviders.CLEANBROWSING_ADULT
                FilterLevel.OFF -> null
            }

            if (dnsHost != null) {
                // Set specific DNS host for filtering
                dpm.setGlobalPrivateDnsModeSpecifiedHost(adminComponent, dnsHost)
            } else {
                // Disable forced DNS (opportunistic = use system default)
                dpm.setGlobalPrivateDnsModeOpportunistic(adminComponent)
            }

            // Save preference
            preferences.setFilterLevel(level.name)

            Result.success(Unit)
        } catch (e: SecurityException) {
            Result.failure(SecurityException("Not authorized to change DNS: ${e.message}"))
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    /**
     * Get current filter level
     */
    fun getCurrentFilterLevel(): FilterLevel {
        val saved = preferences.getFilterLevel()
        return FilterLevel.values().find { it.name == saved } ?: FilterLevel.MODERATE
    }

    /**
     * Filtering levels with what each blocks
     */
    enum class FilterLevel(val description: String, val blockedCategories: List<String>) {
        STRICT(
            description = "Maximum safety - blocks adult, social media, games",
            blockedCategories = listOf("Adult", "Gambling", "Social Media", "Games", "Dating", "Violence")
        ),
        MODERATE(
            description = "Balanced - blocks adult content and malware",
            blockedCategories = listOf("Adult", "Gambling", "Malware", "Phishing")
        ),
        ADULT_ONLY(
            description = "Block adult content only",
            blockedCategories = listOf("Adult", "Pornography")
        ),
        OFF(
            description = "No filtering",
            blockedCategories = emptyList()
        )
    }
}
```

##### Task E12-S01-T02: Create Filter Settings Screen (Parent)
**Time**: 2 hours

Create `lib/presentation/screens/settings/web_filter_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

import '../../../core/theme/app_theme.dart';
import '../../../data/services/command_service.dart';

enum FilterLevel {
  strict,
  moderate,
  adultOnly,
  off,
}

class WebFilterScreen extends ConsumerStatefulWidget {
  final String deviceId;

  const WebFilterScreen({super.key, required this.deviceId});

  @override
  ConsumerState<WebFilterScreen> createState() => _WebFilterScreenState();
}

class _WebFilterScreenState extends ConsumerState<WebFilterScreen> {
  final CommandService _commandService = CommandService();
  FilterLevel _selectedLevel = FilterLevel.moderate;
  bool _isLoading = true;

  @override
  void initState() {
    super.initState();
    _loadCurrentLevel();
  }

  Future<void> _loadCurrentLevel() async {
    final doc = await FirebaseFirestore.instance
        .collection('devices')
        .doc(widget.deviceId)
        .get();

    final data = doc.data();
    final levelStr = data?['settings']?['filterLevel'] as String?;

    setState(() {
      _selectedLevel = FilterLevel.values.firstWhere(
        (l) => l.name == levelStr,
        orElse: () => FilterLevel.moderate,
      );
      _isLoading = false;
    });
  }

  Future<void> _updateFilterLevel(FilterLevel level) async {
    setState(() => _selectedLevel = level);

    // Send command to device
    await _commandService.setFilterLevel(widget.deviceId, level.name);

    // Update Firestore
    await FirebaseFirestore.instance
        .collection('devices')
        .doc(widget.deviceId)
        .update({
      'settings.filterLevel': level.name,
      'settings.filterUpdatedAt': FieldValue.serverTimestamp(),
    });

    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Filter level updated to ${_getLevelName(level)}')),
      );
    }
  }

  String _getLevelName(FilterLevel level) {
    switch (level) {
      case FilterLevel.strict:
        return 'Maximum Safety';
      case FilterLevel.moderate:
        return 'Balanced';
      case FilterLevel.adultOnly:
        return 'Adult Content Only';
      case FilterLevel.off:
        return 'No Filtering';
    }
  }

  String _getLevelDescription(FilterLevel level) {
    switch (level) {
      case FilterLevel.strict:
        return 'Blocks adult content, social media, games, gambling, and violent content. Best for ages 5-8.';
      case FilterLevel.moderate:
        return 'Blocks adult content, gambling, malware, and phishing sites. Best for ages 9-12.';
      case FilterLevel.adultOnly:
        return 'Only blocks adult/pornographic content. Best for ages 13+.';
      case FilterLevel.off:
        return 'No websites are blocked. Not recommended for children.';
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Web Safety'),
      ),
      body: _isLoading
          ? const Center(child: CircularProgressIndicator())
          : ListView(
              padding: const EdgeInsets.all(16),
              children: [
                // Explanation Card
                Card(
                  color: AppTheme.primaryColor.withOpacity(0.1),
                  child: Padding(
                    padding: const EdgeInsets.all(16),
                    child: Row(
                      children: [
                        Icon(Icons.info_outline, color: AppTheme.primaryColor),
                        const SizedBox(width: 12),
                        Expanded(
                          child: Text(
                            'Web filtering blocks harmful websites before they load. This works with all browsers and apps.',
                            style: TextStyle(color: AppTheme.textSecondary),
                          ),
                        ),
                      ],
                    ),
                  ),
                ),

                const SizedBox(height: 24),

                Text(
                  'Select Filtering Level',
                  style: Theme.of(context).textTheme.titleMedium?.copyWith(
                        fontWeight: FontWeight.bold,
                      ),
                ),

                const SizedBox(height: 16),

                // Filter Options
                ...FilterLevel.values.map((level) => _buildFilterOption(level)),

                const SizedBox(height: 24),

                // What's blocked section
                _buildBlockedCategoriesCard(),

                const SizedBox(height: 24),

                // Limitations notice
                Card(
                  color: Colors.orange.shade50,
                  child: Padding(
                    padding: const EdgeInsets.all(16),
                    child: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        Row(
                          children: [
                            Icon(Icons.warning_amber, color: Colors.orange.shade700),
                            const SizedBox(width: 8),
                            Text(
                              'Limitations',
                              style: TextStyle(
                                fontWeight: FontWeight.bold,
                                color: Colors.orange.shade700,
                              ),
                            ),
                          ],
                        ),
                        const SizedBox(height: 8),
                        Text(
                          '• Browsing history is not visible (Chrome keeps it private)\n'
                          '• Some sites may be incorrectly blocked or not blocked\n'
                          '• VPN apps could bypass filtering if installed',
                          style: TextStyle(color: Colors.orange.shade900, fontSize: 13),
                        ),
                      ],
                    ),
                  ),
                ),
              ],
            ),
    );
  }

  Widget _buildFilterOption(FilterLevel level) {
    final isSelected = _selectedLevel == level;

    return Card(
      margin: const EdgeInsets.only(bottom: 12),
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
        side: BorderSide(
          color: isSelected ? AppTheme.primaryColor : Colors.transparent,
          width: 2,
        ),
      ),
      child: InkWell(
        onTap: () => _updateFilterLevel(level),
        borderRadius: BorderRadius.circular(12),
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Row(
            children: [
              Radio<FilterLevel>(
                value: level,
                groupValue: _selectedLevel,
                onChanged: (v) => v != null ? _updateFilterLevel(v) : null,
              ),
              const SizedBox(width: 12),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Row(
                      children: [
                        Text(
                          _getLevelName(level),
                          style: const TextStyle(
                            fontWeight: FontWeight.bold,
                            fontSize: 16,
                          ),
                        ),
                        if (level == FilterLevel.moderate) ...[
                          const SizedBox(width: 8),
                          Container(
                            padding: const EdgeInsets.symmetric(
                              horizontal: 8,
                              vertical: 2,
                            ),
                            decoration: BoxDecoration(
                              color: AppTheme.primaryColor,
                              borderRadius: BorderRadius.circular(4),
                            ),
                            child: const Text(
                              'Recommended',
                              style: TextStyle(
                                color: Colors.white,
                                fontSize: 10,
                                fontWeight: FontWeight.bold,
                              ),
                            ),
                          ),
                        ],
                      ],
                    ),
                    const SizedBox(height: 4),
                    Text(
                      _getLevelDescription(level),
                      style: TextStyle(
                        color: AppTheme.textSecondary,
                        fontSize: 13,
                      ),
                    ),
                  ],
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildBlockedCategoriesCard() {
    final blocked = _getBlockedCategories(_selectedLevel);

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'What\'s blocked with ${_getLevelName(_selectedLevel)}',
              style: const TextStyle(fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 12),
            if (blocked.isEmpty)
              Text(
                'Nothing is blocked',
                style: TextStyle(color: AppTheme.textSecondary),
              )
            else
              Wrap(
                spacing: 8,
                runSpacing: 8,
                children: blocked
                    .map((cat) => Chip(
                          label: Text(cat, style: const TextStyle(fontSize: 12)),
                          backgroundColor: Colors.red.shade50,
                          side: BorderSide(color: Colors.red.shade200),
                        ))
                    .toList(),
              ),
          ],
        ),
      ),
    );
  }

  List<String> _getBlockedCategories(FilterLevel level) {
    switch (level) {
      case FilterLevel.strict:
        return ['Adult', 'Gambling', 'Social Media', 'Games', 'Dating', 'Violence', 'Malware'];
      case FilterLevel.moderate:
        return ['Adult', 'Gambling', 'Malware', 'Phishing'];
      case FilterLevel.adultOnly:
        return ['Adult Content'];
      case FilterLevel.off:
        return [];
    }
  }
}
```

##### Task E12-S01-T03: Handle Filter Command on Device
**Time**: 1 hour

Add to `CommandHandler.kt`:

```kotlin
// In CommandHandler.kt - add to handleCommand()
"SET_FILTER_LEVEL" -> {
    val level = data["level"] as? String ?: return
    val filterLevel = ContentFilterManager.FilterLevel.valueOf(level.uppercase())

    contentFilterManager.setFilterLevel(filterLevel).fold(
        onSuccess = {
            sendCommandAck(commandId, success = true)
        },
        onFailure = { error ->
            sendCommandAck(commandId, success = false, error = error.message)
        }
    )
}
```

---

## Epic E13: Call & Contact Control (F20)

### Feasibility Summary
| Feature | Possible? | Method |
|---------|-----------|--------|
| Block incoming calls | ✅ YES | CallScreeningService |
| Control outgoing calls | ✅ YES | Hide dialer + safe dialer |
| Read call history | ✅ YES | READ_CALL_LOG permission |
| Listen to calls | ❌ NO | Not possible on Android |

### Story E13-S01: Incoming Call Screening
**Points**: 8
**Owner**: Android Developer
**Priority**: P2

#### Description
Use Android's CallScreeningService to block calls from non-approved numbers.

#### Tasks

##### Task E13-S01-T01: Create Call Screening Service
**Time**: 3 hours

Create `app/src/main/java/com/kidtunes/dpc/call/KidSafeCallScreener.kt`:

```kotlin
package com.kidtunes.dpc.call

import android.content.ContentResolver
import android.net.Uri
import android.os.Build
import android.provider.ContactsContract
import android.telecom.Call
import android.telecom.CallScreeningService
import androidx.annotation.RequiresApi
import com.google.firebase.firestore.FirebaseFirestore
import com.kidtunes.launcher.data.local.LauncherPreferences
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.*
import javax.inject.Inject

/**
 * Screens incoming calls and blocks unauthorized numbers.
 *
 * Requirements:
 * - User must set this app as "Caller ID & spam" app in Settings
 * - Works on Android 10+ (API 29+)
 *
 * In AndroidManifest.xml:
 * <service
 *     android:name=".call.KidSafeCallScreener"
 *     android:permission="android.permission.BIND_SCREENING_SERVICE"
 *     android:exported="true">
 *     <intent-filter>
 *         <action android:name="android.telecom.CallScreeningService"/>
 *     </intent-filter>
 * </service>
 */
@AndroidEntryPoint
@RequiresApi(Build.VERSION_CODES.Q)
class KidSafeCallScreener : CallScreeningService() {

    @Inject
    lateinit var preferences: LauncherPreferences

    private val serviceScope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private val firestore = FirebaseFirestore.getInstance()

    // India emergency numbers - ALWAYS allowed
    private val emergencyNumbers = setOf(
        "100",    // Police
        "101",    // Fire
        "102",    // Ambulance
        "108",    // Emergency Ambulance
        "112",    // Universal Emergency
        "1098",   // Child Helpline
        "181",    // Women Helpline
        "1091",   // Women Helpline
        "14461",  // Senior Citizen Helpline
    )

    override fun onScreenCall(callDetails: Call.Details) {
        val incomingNumber = callDetails.handle?.schemeSpecificPart ?: ""
        val normalizedNumber = normalizePhoneNumber(incomingNumber)

        serviceScope.launch {
            val decision = screenCall(normalizedNumber, callDetails)

            // Log the call attempt (for parent visibility)
            logCallAttempt(
                number = normalizedNumber,
                allowed = decision.isAllowed,
                reason = decision.reason,
                direction = "incoming"
            )

            // Respond to the call
            respondToCall(callDetails, CallResponse.Builder()
                .setDisallowCall(!decision.isAllowed)
                .setRejectCall(!decision.isAllowed)
                .setSilenceCall(!decision.isAllowed)
                .setSkipCallLog(false)  // Always log in system
                .setSkipNotification(!decision.isAllowed)
                .build()
            )
        }
    }

    private suspend fun screenCall(number: String, details: Call.Details): CallDecision {
        // 1. Emergency numbers - ALWAYS allow
        if (isEmergencyNumber(number)) {
            return CallDecision(isAllowed = true, reason = "Emergency number")
        }

        // 2. Parent numbers - ALWAYS allow
        if (isParentNumber(number)) {
            return CallDecision(isAllowed = true, reason = "Parent")
        }

        // 3. Check call control mode
        val mode = getCallControlMode()

        return when (mode) {
            CallControlMode.APPROVED_ONLY -> {
                if (isApprovedContact(number)) {
                    CallDecision(isAllowed = true, reason = "Approved contact")
                } else {
                    CallDecision(isAllowed = false, reason = "Not in approved list")
                }
            }
            CallControlMode.CONTACTS_ONLY -> {
                if (isInDeviceContacts(number)) {
                    CallDecision(isAllowed = true, reason = "In device contacts")
                } else {
                    CallDecision(isAllowed = false, reason = "Unknown number")
                }
            }
            CallControlMode.ALL_ALLOWED -> {
                CallDecision(isAllowed = true, reason = "All calls allowed")
            }
        }
    }

    private fun isEmergencyNumber(number: String): Boolean {
        return emergencyNumbers.contains(number) ||
               emergencyNumbers.contains(number.takeLast(3)) ||
               emergencyNumbers.contains(number.takeLast(4))
    }

    private suspend fun isParentNumber(number: String): Boolean {
        val deviceId = preferences.getDeviceId() ?: return false

        return try {
            val doc = firestore.collection("devices")
                .document(deviceId)
                .get()
                .await()

            val parentNumbers = doc.get("parentPhoneNumbers") as? List<*>
            parentNumbers?.any { normalizePhoneNumber(it.toString()) == number } ?: false
        } catch (e: Exception) {
            false
        }
    }

    private suspend fun isApprovedContact(number: String): Boolean {
        val deviceId = preferences.getDeviceId() ?: return false

        return try {
            val contacts = firestore.collection("devices")
                .document(deviceId)
                .collection("approvedContacts")
                .get()
                .await()

            contacts.documents.any { doc ->
                val contactNumber = doc.getString("phoneNumber") ?: ""
                normalizePhoneNumber(contactNumber) == number
            }
        } catch (e: Exception) {
            false
        }
    }

    private fun isInDeviceContacts(number: String): Boolean {
        val uri = Uri.withAppendedPath(
            ContactsContract.PhoneLookup.CONTENT_FILTER_URI,
            Uri.encode(number)
        )

        contentResolver.query(
            uri,
            arrayOf(ContactsContract.PhoneLookup._ID),
            null, null, null
        )?.use { cursor ->
            return cursor.count > 0
        }
        return false
    }

    private suspend fun getCallControlMode(): CallControlMode {
        val deviceId = preferences.getDeviceId() ?: return CallControlMode.CONTACTS_ONLY

        return try {
            val doc = firestore.collection("devices")
                .document(deviceId)
                .get()
                .await()

            val modeStr = doc.getString("settings.callControlMode") ?: "CONTACTS_ONLY"
            CallControlMode.valueOf(modeStr)
        } catch (e: Exception) {
            CallControlMode.CONTACTS_ONLY
        }
    }

    private fun logCallAttempt(
        number: String,
        allowed: Boolean,
        reason: String,
        direction: String
    ) {
        val deviceId = preferences.getDeviceId() ?: return

        firestore.collection("devices")
            .document(deviceId)
            .collection("callLog")
            .add(mapOf(
                "number" to maskPhoneNumber(number),  // Privacy: mask for storage
                "fullNumberHash" to number.hashCode(),  // For deduplication
                "allowed" to allowed,
                "reason" to reason,
                "direction" to direction,
                "timestamp" to com.google.firebase.Timestamp.now()
            ))
    }

    private fun normalizePhoneNumber(number: String): String {
        // Remove all non-digits
        var normalized = number.replace(Regex("[^0-9]"), "")

        // Remove India country code
        if (normalized.startsWith("91") && normalized.length > 10) {
            normalized = normalized.substring(2)
        }
        if (normalized.startsWith("0") && normalized.length > 10) {
            normalized = normalized.substring(1)
        }

        return normalized
    }

    private fun maskPhoneNumber(number: String): String {
        // Show only last 4 digits for privacy
        return if (number.length > 4) {
            "****" + number.takeLast(4)
        } else {
            number
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        serviceScope.cancel()
    }

    data class CallDecision(val isAllowed: Boolean, val reason: String)

    enum class CallControlMode {
        APPROVED_ONLY,   // Only parent-approved numbers can call
        CONTACTS_ONLY,   // Any number in device contacts
        ALL_ALLOWED      // All calls allowed, just logged
    }
}

// Extension to await Firestore tasks
private suspend fun <T> com.google.android.gms.tasks.Task<T>.await(): T =
    suspendCancellableCoroutine { continuation ->
        addOnSuccessListener { continuation.resume(it) {} }
        addOnFailureListener { continuation.resumeWithException(it) }
    }
```

##### Task E13-S01-T02: Create Safe Dialer Activity (Outgoing Calls)
**Time**: 2 hours

Create `app/src/main/java/com/kidtunes/launcher/ui/SafeDialerActivity.kt`:

```kotlin
package com.kidtunes.launcher.ui

import android.Manifest
import android.content.Intent
import android.content.pm.PackageManager
import android.net.Uri
import android.os.Bundle
import android.widget.Toast
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Call
import androidx.compose.material.icons.filled.Warning
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.core.app.ActivityCompat
import com.google.firebase.firestore.FirebaseFirestore
import com.kidtunes.launcher.data.local.LauncherPreferences
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.tasks.await
import javax.inject.Inject

/**
 * Safe Dialer - Only shows approved contacts
 *
 * This replaces the system dialer for controlled calling.
 * Child can only call numbers that parent has approved.
 */
@AndroidEntryPoint
class SafeDialerActivity : ComponentActivity() {

    @Inject
    lateinit var preferences: LauncherPreferences

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            MaterialTheme {
                SafeDialerScreen(
                    preferences = preferences,
                    onCall = { number -> makeCall(number) },
                    onEmergency = { showEmergencyOptions() }
                )
            }
        }
    }

    private fun makeCall(phoneNumber: String) {
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.CALL_PHONE)
            != PackageManager.PERMISSION_GRANTED) {
            Toast.makeText(this, "Phone permission required", Toast.LENGTH_SHORT).show()
            return
        }

        val intent = Intent(Intent.ACTION_CALL).apply {
            data = Uri.parse("tel:$phoneNumber")
        }
        startActivity(intent)
    }

    private fun showEmergencyOptions() {
        // Show emergency numbers dialog
        val intent = Intent(Intent.ACTION_DIAL).apply {
            data = Uri.parse("tel:112")
        }
        startActivity(intent)
    }
}

@Composable
fun SafeDialerScreen(
    preferences: LauncherPreferences,
    onCall: (String) -> Unit,
    onEmergency: () -> Unit
) {
    var contacts by remember { mutableStateOf<List<ApprovedContact>>(emptyList()) }
    var isLoading by remember { mutableStateOf(true) }

    val deviceId = preferences.getDeviceId()

    LaunchedEffect(deviceId) {
        if (deviceId != null) {
            contacts = loadApprovedContacts(deviceId)
        }
        isLoading = false
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Call") },
                actions = {
                    TextButton(onClick = onEmergency) {
                        Icon(Icons.Default.Warning, contentDescription = null, tint = Color.Red)
                        Spacer(Modifier.width(4.dp))
                        Text("Emergency", color = Color.Red)
                    }
                }
            )
        }
    ) { padding ->
        if (isLoading) {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        } else if (contacts.isEmpty()) {
            Box(Modifier.fillMaxSize().padding(padding), contentAlignment = Alignment.Center) {
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    Text("No approved contacts", fontSize = 18.sp)
                    Spacer(Modifier.height(8.dp))
                    Text(
                        "Ask your parent to add contacts",
                        color = Color.Gray
                    )
                }
            }
        } else {
            LazyColumn(
                contentPadding = padding,
                modifier = Modifier.fillMaxSize()
            ) {
                items(contacts) { contact ->
                    ContactCard(
                        contact = contact,
                        onCall = { onCall(contact.phoneNumber) }
                    )
                }
            }
        }
    }
}

@Composable
fun ContactCard(contact: ApprovedContact, onCall: () -> Unit) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 8.dp)
            .clickable(onClick = onCall)
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            // Avatar
            Surface(
                modifier = Modifier.size(56.dp).clip(CircleShape),
                color = MaterialTheme.colorScheme.primaryContainer
            ) {
                Box(contentAlignment = Alignment.Center) {
                    Text(
                        text = contact.name.take(1).uppercase(),
                        fontSize = 24.sp,
                        fontWeight = FontWeight.Bold
                    )
                }
            }

            Spacer(Modifier.width(16.dp))

            // Name and relationship
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text = contact.name,
                    fontSize = 18.sp,
                    fontWeight = FontWeight.Medium
                )
                Text(
                    text = contact.relationship,
                    fontSize = 14.sp,
                    color = Color.Gray
                )
            }

            // Call button
            FilledIconButton(onClick = onCall) {
                Icon(Icons.Default.Call, contentDescription = "Call")
            }
        }
    }
}

data class ApprovedContact(
    val name: String,
    val phoneNumber: String,
    val relationship: String
)

private suspend fun loadApprovedContacts(deviceId: String): List<ApprovedContact> {
    return try {
        val snapshot = FirebaseFirestore.getInstance()
            .collection("devices")
            .document(deviceId)
            .collection("approvedContacts")
            .get()
            .await()

        snapshot.documents.mapNotNull { doc ->
            ApprovedContact(
                name = doc.getString("name") ?: return@mapNotNull null,
                phoneNumber = doc.getString("phoneNumber") ?: return@mapNotNull null,
                relationship = doc.getString("relationship") ?: "Contact"
            )
        }
    } catch (e: Exception) {
        emptyList()
    }
}
```

##### Task E13-S01-T03: Create Contacts Management Screen (Parent)
**Time**: 2 hours

Create `lib/presentation/screens/settings/call_control_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

import '../../../core/theme/app_theme.dart';

enum CallControlMode {
  approvedOnly,
  contactsOnly,
  allAllowed,
}

class CallControlScreen extends ConsumerStatefulWidget {
  final String deviceId;

  const CallControlScreen({super.key, required this.deviceId});

  @override
  ConsumerState<CallControlScreen> createState() => _CallControlScreenState();
}

class _CallControlScreenState extends ConsumerState<CallControlScreen> {
  CallControlMode _mode = CallControlMode.contactsOnly;
  List<Map<String, dynamic>> _approvedContacts = [];
  bool _isLoading = true;

  @override
  void initState() {
    super.initState();
    _loadSettings();
  }

  Future<void> _loadSettings() async {
    final doc = await FirebaseFirestore.instance
        .collection('devices')
        .doc(widget.deviceId)
        .get();

    final modeStr = doc.data()?['settings']?['callControlMode'] as String?;

    final contactsSnap = await FirebaseFirestore.instance
        .collection('devices')
        .doc(widget.deviceId)
        .collection('approvedContacts')
        .get();

    setState(() {
      _mode = CallControlMode.values.firstWhere(
        (m) => m.name.toUpperCase() == modeStr?.toUpperCase(),
        orElse: () => CallControlMode.contactsOnly,
      );
      _approvedContacts = contactsSnap.docs
          .map((d) => {'id': d.id, ...d.data()})
          .toList();
      _isLoading = false;
    });
  }

  Future<void> _updateMode(CallControlMode mode) async {
    setState(() => _mode = mode);

    await FirebaseFirestore.instance
        .collection('devices')
        .doc(widget.deviceId)
        .update({
      'settings.callControlMode': mode.name.toUpperCase(),
    });
  }

  Future<void> _addContact() async {
    final nameController = TextEditingController();
    final phoneController = TextEditingController();
    String relationship = 'Family';

    final result = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Add Approved Contact'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(
              controller: nameController,
              decoration: const InputDecoration(
                labelText: 'Name',
                hintText: 'e.g., Grandma',
              ),
            ),
            const SizedBox(height: 16),
            TextField(
              controller: phoneController,
              decoration: const InputDecoration(
                labelText: 'Phone Number',
                hintText: '+91 98765 43210',
              ),
              keyboardType: TextInputType.phone,
            ),
            const SizedBox(height: 16),
            DropdownButtonFormField<String>(
              value: relationship,
              decoration: const InputDecoration(labelText: 'Relationship'),
              items: ['Family', 'Friend', 'School', 'Other']
                  .map((r) => DropdownMenuItem(value: r, child: Text(r)))
                  .toList(),
              onChanged: (v) => relationship = v ?? 'Family',
            ),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('Cancel'),
          ),
          ElevatedButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('Add'),
          ),
        ],
      ),
    );

    if (result == true && nameController.text.isNotEmpty && phoneController.text.isNotEmpty) {
      final docRef = await FirebaseFirestore.instance
          .collection('devices')
          .doc(widget.deviceId)
          .collection('approvedContacts')
          .add({
        'name': nameController.text,
        'phoneNumber': phoneController.text,
        'relationship': relationship,
        'addedAt': FieldValue.serverTimestamp(),
      });

      setState(() {
        _approvedContacts.add({
          'id': docRef.id,
          'name': nameController.text,
          'phoneNumber': phoneController.text,
          'relationship': relationship,
        });
      });
    }
  }

  Future<void> _removeContact(String id) async {
    await FirebaseFirestore.instance
        .collection('devices')
        .doc(widget.deviceId)
        .collection('approvedContacts')
        .doc(id)
        .delete();

    setState(() {
      _approvedContacts.removeWhere((c) => c['id'] == id);
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Calls & Contacts'),
      ),
      body: _isLoading
          ? const Center(child: CircularProgressIndicator())
          : ListView(
              padding: const EdgeInsets.all(16),
              children: [
                // Mode selection
                Text(
                  'Who can call this device?',
                  style: Theme.of(context).textTheme.titleMedium?.copyWith(
                        fontWeight: FontWeight.bold,
                      ),
                ),
                const SizedBox(height: 12),

                _buildModeOption(
                  CallControlMode.approvedOnly,
                  'Approved contacts only',
                  'Only the contacts you add below can call',
                  Icons.verified_user,
                ),
                _buildModeOption(
                  CallControlMode.contactsOnly,
                  'Device contacts only',
                  'Anyone saved in the phone contacts can call',
                  Icons.contacts,
                ),
                _buildModeOption(
                  CallControlMode.allAllowed,
                  'Anyone can call',
                  'All calls allowed (logged for you to see)',
                  Icons.phone_enabled,
                ),

                const SizedBox(height: 24),

                // Approved contacts list
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    Text(
                      'Approved Contacts (${_approvedContacts.length})',
                      style: Theme.of(context).textTheme.titleMedium?.copyWith(
                            fontWeight: FontWeight.bold,
                          ),
                    ),
                    IconButton(
                      onPressed: _addContact,
                      icon: const Icon(Icons.add_circle),
                      color: AppTheme.primaryColor,
                    ),
                  ],
                ),

                if (_approvedContacts.isEmpty)
                  Card(
                    child: Padding(
                      padding: const EdgeInsets.all(24),
                      child: Column(
                        children: [
                          Icon(Icons.person_add, size: 48, color: Colors.grey.shade400),
                          const SizedBox(height: 12),
                          const Text('No approved contacts yet'),
                          const SizedBox(height: 8),
                          ElevatedButton.icon(
                            onPressed: _addContact,
                            icon: const Icon(Icons.add),
                            label: const Text('Add Contact'),
                          ),
                        ],
                      ),
                    ),
                  )
                else
                  ..._approvedContacts.map((c) => _buildContactCard(c)),

                const SizedBox(height: 24),

                // Call history link
                Card(
                  child: ListTile(
                    leading: const Icon(Icons.history),
                    title: const Text('View Call History'),
                    subtitle: const Text('See all incoming and outgoing calls'),
                    trailing: const Icon(Icons.chevron_right),
                    onTap: () {
                      // Navigate to call history
                    },
                  ),
                ),

                const SizedBox(height: 16),

                // Note about emergency
                Card(
                  color: Colors.green.shade50,
                  child: Padding(
                    padding: const EdgeInsets.all(16),
                    child: Row(
                      children: [
                        Icon(Icons.emergency, color: Colors.green.shade700),
                        const SizedBox(width: 12),
                        Expanded(
                          child: Text(
                            'Emergency numbers (100, 108, 112) are always allowed',
                            style: TextStyle(color: Colors.green.shade900),
                          ),
                        ),
                      ],
                    ),
                  ),
                ),
              ],
            ),
    );
  }

  Widget _buildModeOption(
    CallControlMode mode,
    String title,
    String subtitle,
    IconData icon,
  ) {
    final isSelected = _mode == mode;

    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
        side: BorderSide(
          color: isSelected ? AppTheme.primaryColor : Colors.transparent,
          width: 2,
        ),
      ),
      child: ListTile(
        leading: Icon(icon, color: isSelected ? AppTheme.primaryColor : null),
        title: Text(title),
        subtitle: Text(subtitle, style: const TextStyle(fontSize: 12)),
        trailing: Radio<CallControlMode>(
          value: mode,
          groupValue: _mode,
          onChanged: (v) => v != null ? _updateMode(v) : null,
        ),
        onTap: () => _updateMode(mode),
      ),
    );
  }

  Widget _buildContactCard(Map<String, dynamic> contact) {
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: ListTile(
        leading: CircleAvatar(
          child: Text(contact['name'].toString()[0].toUpperCase()),
        ),
        title: Text(contact['name']),
        subtitle: Text('${contact['phoneNumber']} • ${contact['relationship']}'),
        trailing: IconButton(
          icon: const Icon(Icons.delete_outline, color: Colors.red),
          onPressed: () => _removeContact(contact['id']),
        ),
      ),
    );
  }
}
```

##### Task E13-S01-T04: Hide System Dialer via DPC
**Time**: 1 hour

Add to `DevicePolicyManager.kt`:

```kotlin
/**
 * Control which dialer app child can use
 */
class DialerControlManager(
    private val context: Context,
    private val dpm: DevicePolicyManager,
    private val adminComponent: ComponentName
) {
    // System dialer packages to hide
    private val systemDialers = listOf(
        "com.google.android.dialer",
        "com.android.dialer",
        "com.samsung.android.dialer",
        "com.samsung.android.contacts"
    )

    /**
     * Hide system dialers - child can only use our safe dialer
     */
    fun enforceApprovedDialerOnly() {
        if (!dpm.isDeviceOwnerApp(context.packageName)) return

        systemDialers.forEach { pkg ->
            try {
                dpm.setApplicationHidden(adminComponent, pkg, true)
            } catch (e: Exception) {
                // Package might not exist on this device
            }
        }
    }

    /**
     * Restore system dialer (when call control is set to "all allowed")
     */
    fun restoreSystemDialer() {
        if (!dpm.isDeviceOwnerApp(context.packageName)) return

        systemDialers.forEach { pkg ->
            try {
                dpm.setApplicationHidden(adminComponent, pkg, false)
            } catch (e: Exception) {
                // Package might not exist
            }
        }
    }
}
```

---

## Epic E14: App-Specific Controls (F22)

### Feasibility Summary
| Feature | Possible? | Method |
|---------|-----------|--------|
| YouTube Restricted Mode | ✅ YES | Managed Configuration |
| Chrome Incognito block | ✅ YES | Managed Configuration |
| Block in-app purchases | ✅ YES | DPC User Restriction |
| Read YouTube history | ❌ NO | Sandboxed |

### Story E14-S01: Managed App Configurations
**Points**: 5
**Owner**: Android Developer
**Priority**: P2

#### Tasks

##### Task E14-S01-T01: Create Managed Config Manager
**Time**: 2 hours

Create `app/src/main/java/com/kidtunes/dpc/config/ManagedConfigManager.kt`:

```kotlin
package com.kidtunes.dpc.config

import android.app.admin.DevicePolicyManager
import android.content.ComponentName
import android.content.Context
import android.os.Bundle
import android.os.UserManager
import com.kidtunes.dpc.receivers.KidDeviceAdminReceiver
import dagger.hilt.android.qualifiers.ApplicationContext
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Manages app-specific configurations via Android Enterprise Managed Configurations.
 *
 * Supported apps:
 * - Chrome: Block incognito, force SafeSearch, block URLs
 * - YouTube: Restricted Mode
 * - Play Store: Require password for purchases
 */
@Singleton
class ManagedConfigManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val dpm = context.getSystemService(Context.DEVICE_POLICY_SERVICE)
        as DevicePolicyManager
    private val adminComponent = KidDeviceAdminReceiver.getComponentName(context)

    fun isDeviceOwner(): Boolean = dpm.isDeviceOwnerApp(context.packageName)

    // ============ CHROME CONFIGURATION ============

    /**
     * Configure Chrome browser restrictions
     */
    fun configureChromeRestrictions(config: ChromeConfig): Result<Unit> {
        if (!isDeviceOwner()) {
            return Result.failure(IllegalStateException("Not device owner"))
        }

        return try {
            val restrictions = Bundle().apply {
                // Block incognito mode
                // 0 = enabled, 1 = disabled, 2 = forced
                putInt("IncognitoModeAvailability", if (config.blockIncognito) 1 else 0)

                // Force SafeSearch on Google
                putBoolean("ForceGoogleSafeSearch", config.forceSafeSearch)

                // Force YouTube Restricted Mode via Chrome
                putBoolean("ForceYouTubeRestrict", config.forceYouTubeRestrict)

                // Block developer tools
                putBoolean("DeveloperToolsDisabled", true)

                // Download restrictions
                // 0 = allow, 1 = block dangerous, 2 = block dangerous types, 3 = block all
                putInt("DownloadRestrictions", if (config.blockDownloads) 3 else 1)

                // Block specific URLs (comma-separated patterns)
                if (config.blockedUrls.isNotEmpty()) {
                    putStringArray("URLBlocklist", config.blockedUrls.toTypedArray())
                }

                // Allow specific URLs (overrides blocklist)
                if (config.allowedUrls.isNotEmpty()) {
                    putStringArray("URLAllowlist", config.allowedUrls.toTypedArray())
                }
            }

            dpm.setApplicationRestrictions(adminComponent, CHROME_PACKAGE, restrictions)
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    // ============ YOUTUBE CONFIGURATION ============

    /**
     * Configure YouTube restrictions
     */
    fun configureYouTubeRestrictions(restrictedMode: Boolean): Result<Unit> {
        if (!isDeviceOwner()) {
            return Result.failure(IllegalStateException("Not device owner"))
        }

        return try {
            val restrictions = Bundle().apply {
                // Enable restricted mode (filters mature content)
                putString("restrict", if (restrictedMode) "Strict" else "Off")
            }

            dpm.setApplicationRestrictions(adminComponent, YOUTUBE_PACKAGE, restrictions)
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    // ============ IN-APP PURCHASES ============

    /**
     * Block in-app purchases system-wide
     */
    fun blockInAppPurchases(block: Boolean): Result<Unit> {
        if (!isDeviceOwner()) {
            return Result.failure(IllegalStateException("Not device owner"))
        }

        return try {
            // Play Store - require authentication for purchases
            val playRestrictions = Bundle().apply {
                // 0 = never, 1 = every 30 min, 2 = always require
                putInt("require_password_for_purchase", if (block) 2 else 0)
            }
            dpm.setApplicationRestrictions(adminComponent, PLAY_STORE_PACKAGE, playRestrictions)

            // Add user restriction to prevent modifying accounts (prevents payment methods)
            if (block) {
                dpm.addUserRestriction(adminComponent, UserManager.DISALLOW_MODIFY_ACCOUNTS)
            } else {
                dpm.clearUserRestriction(adminComponent, UserManager.DISALLOW_MODIFY_ACCOUNTS)
            }

            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    // ============ APPLY ALL SETTINGS ============

    /**
     * Apply all app-specific settings at once
     */
    fun applyAllSettings(settings: AppControlSettings): Result<Unit> {
        val results = mutableListOf<Result<Unit>>()

        // Chrome
        results.add(configureChromeRestrictions(ChromeConfig(
            blockIncognito = settings.blockChromeIncognito,
            forceSafeSearch = settings.forceSafeSearch,
            forceYouTubeRestrict = settings.youtubeRestrictedMode,
            blockDownloads = settings.blockDownloads,
            blockedUrls = settings.blockedUrls,
            allowedUrls = settings.allowedUrls
        )))

        // YouTube
        results.add(configureYouTubeRestrictions(settings.youtubeRestrictedMode))

        // In-app purchases
        results.add(blockInAppPurchases(settings.blockInAppPurchases))

        // Return first error if any
        return results.find { it.isFailure } ?: Result.success(Unit)
    }

    companion object {
        const val CHROME_PACKAGE = "com.android.chrome"
        const val YOUTUBE_PACKAGE = "com.google.android.youtube"
        const val PLAY_STORE_PACKAGE = "com.android.vending"
    }

    data class ChromeConfig(
        val blockIncognito: Boolean = true,
        val forceSafeSearch: Boolean = true,
        val forceYouTubeRestrict: Boolean = true,
        val blockDownloads: Boolean = false,
        val blockedUrls: List<String> = emptyList(),
        val allowedUrls: List<String> = emptyList()
    )

    data class AppControlSettings(
        val blockChromeIncognito: Boolean = true,
        val forceSafeSearch: Boolean = true,
        val youtubeRestrictedMode: Boolean = true,
        val blockDownloads: Boolean = false,
        val blockInAppPurchases: Boolean = true,
        val blockedUrls: List<String> = emptyList(),
        val allowedUrls: List<String> = emptyList()
    )
}
```

##### Task E14-S01-T02: Create App Controls Screen (Parent)
**Time**: 2 hours

Create `lib/presentation/screens/settings/app_controls_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

import '../../../core/theme/app_theme.dart';
import '../../../data/services/command_service.dart';

class AppControlsScreen extends ConsumerStatefulWidget {
  final String deviceId;

  const AppControlsScreen({super.key, required this.deviceId});

  @override
  ConsumerState<AppControlsScreen> createState() => _AppControlsScreenState();
}

class _AppControlsScreenState extends ConsumerState<AppControlsScreen> {
  final CommandService _commandService = CommandService();

  bool _blockChromeIncognito = true;
  bool _forceSafeSearch = true;
  bool _youtubeRestrictedMode = true;
  bool _blockInAppPurchases = true;
  bool _blockDownloads = false;
  bool _isLoading = true;

  @override
  void initState() {
    super.initState();
    _loadSettings();
  }

  Future<void> _loadSettings() async {
    final doc = await FirebaseFirestore.instance
        .collection('devices')
        .doc(widget.deviceId)
        .get();

    final settings = doc.data()?['appControls'] as Map<String, dynamic>?;

    setState(() {
      _blockChromeIncognito = settings?['blockChromeIncognito'] ?? true;
      _forceSafeSearch = settings?['forceSafeSearch'] ?? true;
      _youtubeRestrictedMode = settings?['youtubeRestrictedMode'] ?? true;
      _blockInAppPurchases = settings?['blockInAppPurchases'] ?? true;
      _blockDownloads = settings?['blockDownloads'] ?? false;
      _isLoading = false;
    });
  }

  Future<void> _updateSetting(String key, bool value) async {
    await FirebaseFirestore.instance
        .collection('devices')
        .doc(widget.deviceId)
        .update({'appControls.$key': value});

    // Send command to apply settings
    await _commandService.updateAppControls(widget.deviceId, {
      'blockChromeIncognito': _blockChromeIncognito,
      'forceSafeSearch': _forceSafeSearch,
      'youtubeRestrictedMode': _youtubeRestrictedMode,
      'blockInAppPurchases': _blockInAppPurchases,
      'blockDownloads': _blockDownloads,
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('App Controls'),
      ),
      body: _isLoading
          ? const Center(child: CircularProgressIndicator())
          : ListView(
              padding: const EdgeInsets.all(16),
              children: [
                // YouTube Controls
                _buildSectionHeader('YouTube', Icons.play_circle),
                _buildSwitch(
                  'Restricted Mode',
                  'Filters out mature content',
                  _youtubeRestrictedMode,
                  (v) {
                    setState(() => _youtubeRestrictedMode = v);
                    _updateSetting('youtubeRestrictedMode', v);
                  },
                ),

                const SizedBox(height: 24),

                // Chrome Controls
                _buildSectionHeader('Chrome Browser', Icons.language),
                _buildSwitch(
                  'Block Incognito Mode',
                  'Prevents private browsing',
                  _blockChromeIncognito,
                  (v) {
                    setState(() => _blockChromeIncognito = v);
                    _updateSetting('blockChromeIncognito', v);
                  },
                ),
                _buildSwitch(
                  'Force SafeSearch',
                  'Filters explicit search results',
                  _forceSafeSearch,
                  (v) {
                    setState(() => _forceSafeSearch = v);
                    _updateSetting('forceSafeSearch', v);
                  },
                ),
                _buildSwitch(
                  'Block Downloads',
                  'Prevents file downloads',
                  _blockDownloads,
                  (v) {
                    setState(() => _blockDownloads = v);
                    _updateSetting('blockDownloads', v);
                  },
                ),

                const SizedBox(height: 24),

                // Purchase Controls
                _buildSectionHeader('Purchases', Icons.shopping_cart),
                _buildSwitch(
                  'Block In-App Purchases',
                  'Prevents buying within apps and games',
                  _blockInAppPurchases,
                  (v) {
                    setState(() => _blockInAppPurchases = v);
                    _updateSetting('blockInAppPurchases', v);
                  },
                ),

                const SizedBox(height: 24),

                // Limitations notice
                Card(
                  color: Colors.orange.shade50,
                  child: Padding(
                    padding: const EdgeInsets.all(16),
                    child: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        Row(
                          children: [
                            Icon(Icons.info_outline, color: Colors.orange.shade700),
                            const SizedBox(width: 8),
                            Text(
                              'What these controls can\'t do',
                              style: TextStyle(
                                fontWeight: FontWeight.bold,
                                color: Colors.orange.shade700,
                              ),
                            ),
                          ],
                        ),
                        const SizedBox(height: 8),
                        Text(
                          '• Cannot see YouTube watch history (private to app)\n'
                          '• Cannot see Chrome browsing history\n'
                          '• Cannot block specific YouTube channels\n'
                          '• Some apps may not support all restrictions',
                          style: TextStyle(color: Colors.orange.shade900, fontSize: 13),
                        ),
                      ],
                    ),
                  ),
                ),
              ],
            ),
    );
  }

  Widget _buildSectionHeader(String title, IconData icon) {
    return Padding(
      padding: const EdgeInsets.only(bottom: 12),
      child: Row(
        children: [
          Icon(icon, color: AppTheme.primaryColor),
          const SizedBox(width: 8),
          Text(
            title,
            style: Theme.of(context).textTheme.titleMedium?.copyWith(
                  fontWeight: FontWeight.bold,
                ),
          ),
        ],
      ),
    );
  }

  Widget _buildSwitch(
    String title,
    String subtitle,
    bool value,
    ValueChanged<bool> onChanged,
  ) {
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: SwitchListTile(
        title: Text(title),
        subtitle: Text(subtitle, style: const TextStyle(fontSize: 12)),
        value: value,
        onChanged: onChanged,
      ),
    );
  }
}
```

---

## Sprint 4 Checklist

### Days 50-53: Location & Safety (E09)
- [ ] LocationService created
- [ ] Location updates to Firebase
- [ ] Safe zones working
- [ ] Location screen (Parent)
- [ ] Play sound command

### Days 54-57: Dashboard & Analytics (E10)
- [ ] Dashboard screen polished
- [ ] Real-time status display
- [ ] Screen time visualization
- [ ] Now Playing widget
- [ ] Quick actions working

### Days 58-60: Web Content Filtering (E12)
- [ ] ContentFilterManager created
- [ ] DNS filtering via DPC working
- [ ] Filter levels (Strict/Moderate/Off)
- [ ] Web Filter Screen (Parent)
- [ ] Command handling on device

### Days 61-65: Call & Contact Control (E13)
- [ ] KidSafeCallScreener service
- [ ] Incoming call blocking working
- [ ] SafeDialerActivity for outgoing
- [ ] Hide system dialer via DPC
- [ ] Call Control Screen (Parent)
- [ ] Approved contacts management
- [ ] Call history logging

### Days 66-68: App-Specific Controls (E14)
- [ ] ManagedConfigManager created
- [ ] YouTube Restricted Mode working
- [ ] Chrome Incognito blocking
- [ ] Chrome SafeSearch forced
- [ ] In-app purchase blocking
- [ ] App Controls Screen (Parent)

### Days 69-70: Remote Install & Polish (E11)
- [ ] ManagedPlayManager created
- [ ] App install commands
- [ ] UI polish pass
- [ ] Error handling
- [ ] Loading states
- [ ] Integration testing

---

## Sprint 4 Deliverables

### Must Have (P0/P1)
1. ✅ Real-time location tracking
2. ✅ Safe zone alerts
3. ✅ Find device (play sound)
4. ✅ Polished dashboard
5. ✅ Usage analytics display

### Advanced Controls (P2)
6. ✅ Web content filtering (DNS-based)
7. ✅ Call screening & blocking
8. ✅ Safe dialer with approved contacts
9. ✅ YouTube Restricted Mode
10. ✅ Chrome Incognito blocking
11. ✅ In-app purchase blocking
12. ✅ Remote app management

### What's NOT Included (Technically Not Possible)
- ❌ Reading WhatsApp/SMS messages
- ❌ Viewing Chrome browsing history
- ❌ Viewing YouTube watch history
- ❌ Listening to phone calls
- ❌ Blocking specific YouTube channels

---

## Summary: What Parents CAN Control

| Category | Feature | Status |
|----------|---------|--------|
| **Web** | Block adult/gambling/social sites | ✅ Via DNS |
| **Web** | Force SafeSearch on Google | ✅ Via Chrome config |
| **Web** | Block Chrome Incognito | ✅ Via Chrome config |
| **Calls** | Block unknown callers | ✅ Via CallScreeningService |
| **Calls** | Only approved contacts can call | ✅ Via safe dialer |
| **Calls** | See call history | ✅ Via call log sync |
| **Apps** | YouTube Restricted Mode | ✅ Via managed config |
| **Apps** | Block in-app purchases | ✅ Via DPC restriction |
| **Location** | Real-time tracking | ✅ Already implemented |
| **Time** | Screen time limits | ✅ Already implemented |

---

## API Routes for Advanced Controls

### Firebase Firestore Structure

```
devices/{deviceId}/
├── settings/
│   ├── filterLevel: "MODERATE"
│   ├── callControlMode: "CONTACTS_ONLY"
│   └── appControls/
│       ├── blockChromeIncognito: true
│       ├── forceSafeSearch: true
│       ├── youtubeRestrictedMode: true
│       └── blockInAppPurchases: true
├── approvedContacts/
│   └── {contactId}/
│       ├── name: "Mom"
│       ├── phoneNumber: "+919876543210"
│       └── relationship: "Family"
└── callLog/
    └── {logId}/
        ├── number: "****3210"
        ├── allowed: true
        ├── direction: "incoming"
        └── timestamp: ...
```

### FCM Commands

| Command | Payload | Description |
|---------|---------|-------------|
| `SET_FILTER_LEVEL` | `{level: "STRICT"}` | Update DNS filtering |
| `UPDATE_APP_CONTROLS` | `{...settings}` | Apply Chrome/YouTube/Purchase settings |
| `SYNC_APPROVED_CONTACTS` | - | Pull latest approved contacts |
| `SET_CALL_CONTROL_MODE` | `{mode: "APPROVED_ONLY"}` | Update call screening mode |
