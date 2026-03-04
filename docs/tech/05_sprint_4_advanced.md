# Sprint 4: Advanced Features (Days 38-49)

## Sprint Goal
Implement location tracking, usage analytics, remote app installation via managed Play, and UI polish.

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

## Sprint 4 Checklist

### Days 38-41: Location & Safety
- [ ] LocationService created
- [ ] Location updates to Firebase
- [ ] Safe zones working
- [ ] Location screen (Parent)
- [ ] Play sound command

### Days 42-45: Dashboard & Analytics
- [ ] Dashboard screen polished
- [ ] Real-time status display
- [ ] Screen time visualization
- [ ] Now Playing widget
- [ ] Quick actions working

### Days 46-49: Remote Install & Polish
- [ ] ManagedPlayManager created
- [ ] App install commands
- [ ] UI polish pass
- [ ] Error handling
- [ ] Loading states

---

## Sprint 4 Deliverables

1. ✅ Real-time location tracking
2. ✅ Safe zone alerts
3. ✅ Find device (play sound)
4. ✅ Polished dashboard
5. ✅ Usage analytics display
6. ✅ Remote app management
7. ✅ Ready for testing in Sprint 5
