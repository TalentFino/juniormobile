# Feature Specifications

## Feature Priority Matrix

| Priority | Definition | MVP Scope |
|----------|------------|-----------|
| **P0** | Must have for launch | Yes |
| **P1** | Important, needed soon | Yes |
| **P2** | Nice to have | Partial |
| **P3** | Future consideration | No |

---

## P0 Features (Must Have)

### F01: Device Setup & Pairing

**Description**: Parent pairs child device to their account via QR code.

| Aspect | Specification |
|--------|---------------|
| **Trigger** | Parent taps "Add Device" in parent app |
| **Input** | Child name, age, protection level |
| **Output** | QR code valid for 15 minutes |
| **Child Action** | Scan QR during device setup |
| **Result** | Device linked to parent account |

**Acceptance Criteria**:
- [ ] QR code generates in < 2 seconds
- [ ] QR code is scannable from 30cm distance
- [ ] Pairing completes in < 30 seconds
- [ ] Device appears in parent dashboard immediately
- [ ] Works on first boot (factory reset device)

---

### F02: Custom Launcher

**Description**: Replacement home screen that shows only approved apps.

| Aspect | Specification |
|--------|---------------|
| **Default Home** | Launcher is set as default, cannot be changed |
| **App Display** | Only whitelisted apps visible |
| **Modes** | Kids Mode (2x3 grid), Standard Mode (4x4 grid) |
| **Mode Switch** | Requires 4-digit PIN |
| **Persist** | Survives restart, cannot be uninstalled |

**Acceptance Criteria**:
- [ ] Launcher loads in < 500ms
- [ ] Only whitelisted apps appear
- [ ] Home button always returns to launcher
- [ ] Cannot access other launchers
- [ ] Recent apps only shows whitelisted apps

---

### F03: Device Policy Controller (DPC)

**Description**: System-level control via Android Device Owner APIs.

| Restriction | Kids Mode | Standard Mode |
|-------------|-----------|---------------|
| Install apps | Blocked | Blocked (unless remote install) |
| Uninstall apps | Blocked | Blocked |
| Access Play Store | Blocked | Blocked |
| Access Settings | Blocked | Limited |
| Factory Reset | PIN required | PIN required |
| USB Debugging | Blocked | Blocked |
| Safe Boot | Blocked | Blocked |

**Acceptance Criteria**:
- [ ] Device Owner mode active after setup
- [ ] Cannot uninstall DPC or Launcher
- [ ] Restrictions survive device restart
- [ ] Cannot bypass via ADB

---

### F04: App Whitelist Management

**Description**: Parent controls which apps child can access.

| Aspect | Specification |
|--------|---------------|
| **View Apps** | Parent sees all installed apps on child device |
| **Toggle** | ON = visible to child, OFF = hidden |
| **Sync Time** | Changes reflect on device within 5 seconds |
| **Default** | New devices have preset app list based on age |
| **Categories** | Music, Education, Utility, Communication, Payment, Games |

**Default Whitelist by Protection Level**:

| Protection Level | Default Apps |
|-----------------|--------------|
| Maximum (5-8) | Clock, Calculator, 2 music apps |
| Balanced (9-12) | Above + messaging, educational |
| Trust (13+) | Above + more apps, fewer restrictions |

**Acceptance Criteria**:
- [ ] All installed apps listed in parent app
- [ ] Toggle state syncs to device in < 5 seconds
- [ ] Hidden apps not visible in launcher
- [ ] Hidden apps cannot be launched via shortcuts

---

### F05: Remote Mode Toggle

**Description**: Parent can switch child device between Kids and Standard mode.

| Aspect | Specification |
|--------|---------------|
| **Trigger** | Parent taps mode toggle in dashboard |
| **Confirmation** | "Switch to Kids Mode?" dialog |
| **Execution** | Command sent via Firebase RTDB |
| **Result** | Device switches mode immediately |
| **Notification** | Child sees brief notification |

**Acceptance Criteria**:
- [ ] Mode switches within 3 seconds
- [ ] Works when device is online
- [ ] Queued when device is offline
- [ ] Parent sees confirmation

---

### F06: Panic Button

**Description**: Child can send emergency alert to parent.

| Aspect | Specification |
|--------|---------------|
| **Location** | Prominent button in Kids Mode |
| **Activation** | Long press for 3 seconds |
| **Feedback** | Vibration + visual confirmation |
| **Data Sent** | Device ID, timestamp, GPS location |
| **Parent Alert** | High-priority push notification |

**Acceptance Criteria**:
- [ ] Panic button always visible in Kids Mode
- [ ] Cannot be accidentally triggered (3 sec hold)
- [ ] Alert received by parent within 5 seconds
- [ ] Works even with low battery
- [ ] Includes location if GPS enabled

---

### F07: Real-time Device Status

**Description**: Parent sees current state of child device.

| Status | Displayed |
|--------|-----------|
| Online/Offline | Green/gray dot |
| Battery Level | Percentage + icon |
| Current Mode | Kids/Standard |
| Last Seen | "2 minutes ago" |
| Screen State | On/Off |

**Acceptance Criteria**:
- [ ] Status updates within 10 seconds
- [ ] Offline shown if no update for 2 minutes
- [ ] Battery level accurate within 5%

---

## P1 Features (Important)

### F08: Screen Time Limits

**Description**: Parent sets daily usage limits.

| Aspect | Specification |
|--------|---------------|
| **Daily Limit** | 30 min to 4 hours (or unlimited) |
| **Per-App Limit** | Optional limit for specific apps |
| **Warning** | Alert at 10 minutes remaining |
| **Enforcement** | Device locks when limit reached |
| **Reset** | Daily at midnight (device timezone) |

**Lock Screen When Limit Reached**:
- Shows "Time's up for today!"
- Shows time until reset
- Emergency call still possible
- Parent can extend via app

**Acceptance Criteria**:
- [ ] Time tracked accurately (± 1 minute)
- [ ] Warning notification at 10 min remaining
- [ ] Device locks when limit hit
- [ ] Parent can extend remotely
- [ ] Resets at midnight correctly

---

### F09: Bedtime Mode

**Description**: Device locks during sleep hours.

| Aspect | Specification |
|--------|---------------|
| **Schedule** | Start time + End time |
| **Days** | Can set different for weekdays/weekends |
| **Lock** | Device shows bedtime screen |
| **Exceptions** | Alarm app still works |
| **Emergency** | Panic button still works |

**Acceptance Criteria**:
- [ ] Locks at exact scheduled time
- [ ] Unlocks at exact scheduled time
- [ ] Alarm sounds even when locked
- [ ] Panic button works when locked

---

### F10: Remote Device Lock

**Description**: Parent can instantly lock child device.

| Aspect | Specification |
|--------|---------------|
| **Trigger** | Parent taps "Lock Now" |
| **Execution** | Immediate (< 3 seconds) |
| **Lock Screen** | Shows parent-defined message |
| **Unlock** | Parent taps "Unlock" or schedules |

**Acceptance Criteria**:
- [ ] Device locks within 3 seconds
- [ ] Custom message displayed
- [ ] Cannot be bypassed by child
- [ ] Parent can unlock remotely

---

### F11: Volume Limiting

**Description**: Maximum volume enforced for hearing protection.

| Aspect | Specification |
|--------|---------------|
| **Default Limit** | 70% of max volume |
| **Range** | 50% to 100% |
| **Enforcement** | Cannot exceed via buttons or apps |
| **Setting** | Parent configures in app |

**Acceptance Criteria**:
- [ ] Volume cannot exceed set limit
- [ ] Works with all audio apps
- [ ] Works with headphones and speaker
- [ ] Persists after restart

---

### F12: Usage Reports

**Description**: Parent sees how child used the device.

| Report | Content |
|--------|---------|
| **Today** | Total time, per-app breakdown |
| **This Week** | Daily totals, trends |
| **This Month** | Weekly totals, most used apps |

**Acceptance Criteria**:
- [ ] Accurate within 1 minute
- [ ] Updates in near real-time
- [ ] Shows app icons and names
- [ ] Visual charts (bar/pie)

---

## P2 Features (Nice to Have)

### F13: Location Tracking

**Description**: Parent sees child device location.

| Aspect | Specification |
|--------|---------------|
| **Update Frequency** | Every 5 minutes (configurable) |
| **Accuracy** | Within 50 meters |
| **History** | Last 7 days of locations |
| **Privacy** | Opt-in, can be disabled |

**Acceptance Criteria**:
- [ ] Location accurate within 50m
- [ ] Map displays correctly
- [ ] History scrollable by date
- [ ] Battery impact < 3%/hour

---

### F14: Safe Zones

**Description**: Parent defines allowed locations, alerts when child leaves.

| Aspect | Specification |
|--------|---------------|
| **Create** | Drop pin on map, set radius |
| **Radius** | 100m to 2km |
| **Alert** | Push notification when child leaves |
| **Zones** | Up to 5 zones (Home, School, etc.) |

**Acceptance Criteria**:
- [ ] Geofence triggers within 100m accuracy
- [ ] Alert received within 1 minute of exit
- [ ] Can enable/disable per zone

---

### F15: Remote App Installation

**Description**: Parent can install apps on child device remotely.

| Aspect | Specification |
|--------|---------------|
| **Method** | Via Managed Google Play |
| **Trigger** | Parent selects app in parent app |
| **Execution** | App installs silently on child device |
| **Catalog** | Pre-approved safe apps |

**Acceptance Criteria**:
- [ ] App installs without child interaction
- [ ] Parent sees installation status
- [ ] Only works with approved apps

---

### F16: Now Playing Display

**Description**: Parent sees what child is currently listening to.

| Aspect | Specification |
|--------|---------------|
| **Display** | Song title, artist, app |
| **Update** | Real-time when playing |
| **Apps** | Spotify, JioSaavn, Amazon Music, YouTube Music |

**Acceptance Criteria**:
- [ ] Shows current song within 5 seconds
- [ ] Works with major music apps
- [ ] Shows "Not playing" when idle

---

### F19: Web Content Filtering & Browser Control

**Description**: Parent controls which websites child can access.

#### Feasibility Assessment

| Method | Possible? | Difficulty | Notes |
|--------|-----------|------------|-------|
| **Block browser apps entirely** | ✅ YES | Easy | Already have this via app whitelist |
| **DNS-based filtering** | ✅ YES | Medium | Via DPC Private DNS setting |
| **VPN-based filtering** | ✅ YES | Hard | Battery drain, complex |
| **Read Chrome history** | ❌ NO | N/A | Chrome doesn't expose to other apps |
| **Force SafeSearch** | ✅ YES | Easy | Via DNS or managed config |

#### Recommended Approach: DNS Filtering via DPC

**How it works**: Device Owner can set Private DNS to a filtering service (CleanBrowsing, NextDNS). All DNS queries go through filter → blocked sites don't resolve.

```kotlin
// In DevicePolicyController - Set Private DNS for content filtering
class ContentFilterManager(
    private val context: Context,
    private val dpm: DevicePolicyManager,
    private val adminComponent: ComponentName
) {
    companion object {
        // Free DNS filtering services
        const val CLEANBROWSING_FAMILY = "family-filter-dns.cleanbrowsing.org"
        const val CLEANBROWSING_ADULT = "adult-filter-dns.cleanbrowsing.org"
        const val CLOUDFLARE_FAMILY = "family.cloudflare-dns.com"

        // For custom filtering (requires backend)
        // const val CUSTOM_DNS = "filter.kidsafe.example.com"
    }

    /**
     * Set DNS-based content filtering
     * Requires: Device Owner mode
     */
    fun setContentFilter(level: FilterLevel): Result<Unit> {
        if (!dpm.isDeviceOwnerApp(context.packageName)) {
            return Result.Error(Exception("Not device owner"))
        }

        return try {
            val dnsHost = when (level) {
                FilterLevel.STRICT -> CLEANBROWSING_FAMILY  // Blocks adult + social + gaming
                FilterLevel.MODERATE -> CLEANBROWSING_ADULT // Blocks adult content only
                FilterLevel.OFF -> null
            }

            // Android 10+ Private DNS setting
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                if (dnsHost != null) {
                    dpm.setGlobalPrivateDnsModeSpecifiedHost(
                        adminComponent,
                        dnsHost
                    )
                } else {
                    dpm.setGlobalPrivateDnsModeOpportunistic(adminComponent)
                }
                Result.Success(Unit)
            } else {
                Result.Error(Exception("Requires Android 10+"))
            }
        } catch (e: Exception) {
            Result.Error(e)
        }
    }

    enum class FilterLevel {
        STRICT,    // Family filter - blocks adult, social media, games
        MODERATE,  // Adult filter - blocks adult content only
        OFF        // No filtering
    }
}
```

#### Option 2: Block All Browsers Except Custom Safe Browser

```kotlin
// Simple approach: Only allow a kid-safe browser in whitelist
class BrowserControlManager(
    private val appWhitelistManager: AppWhitelistManager
) {
    // Block all standard browsers
    private val blockedBrowsers = listOf(
        "com.android.chrome",
        "com.chrome.beta",
        "org.mozilla.firefox",
        "com.opera.browser",
        "com.brave.browser",
        "com.microsoft.emmx",
        "com.UCMobile.intl"
    )

    // Allow only kid-safe browsers
    private val safeBrowsers = listOf(
        "com.kidsafe.browser",           // Our own browser (if we build one)
        "com.kidzapps.safebrowser",       // Third-party kid browser
        "com.nickelodeon.nickjr.browser"  // Example kid browser
    )

    fun enforceKidSafeBrowsing() {
        blockedBrowsers.forEach { pkg ->
            appWhitelistManager.setAppHidden(pkg, hidden = true)
        }
        // Only pre-approved browsers visible
    }
}
```

#### What We CANNOT Do
- ❌ Read browsing history from Chrome/Firefox (sandboxed)
- ❌ See exactly which URLs were visited (without VPN)
- ❌ Block specific pages within allowed sites

#### What We CAN Do
- ✅ Block entire categories (adult, social media, gambling) via DNS
- ✅ Block/allow specific domains via DNS service
- ✅ Remove all browsers from child device
- ✅ Force SafeSearch on Google via DNS

**Acceptance Criteria**:
- [ ] DNS filtering configurable from parent app
- [ ] Filter level syncs to device in < 10 seconds
- [ ] Blocked sites show "site not found" (DNS level)
- [ ] Works with all apps making network requests

---

### F20: Call & Contact Control

**Description**: Parent controls who child can call and receive calls from.

#### Feasibility Assessment

| Feature | Possible? | Difficulty | Notes |
|---------|-----------|------------|-------|
| **Block incoming calls** | ✅ YES | Medium | CallScreeningService (Android 10+) |
| **Block outgoing calls** | ✅ YES | Medium | Hide dialer + custom dialer |
| **Read call log** | ✅ YES | Easy | READ_CALL_LOG permission |
| **Log call duration** | ✅ YES | Easy | From call log |
| **Listen to calls** | ❌ NO | N/A | Not possible on Android |
| **Record calls** | ⚠️ LIMITED | Hard | Very restricted on Android 10+ |

#### Implementation: Call Screening Service (Incoming Calls)

```kotlin
// AndroidManifest.xml - Declare the service
// <service
//     android:name=".call.KidSafeCallScreener"
//     android:permission="android.permission.BIND_SCREENING_SERVICE">
//     <intent-filter>
//         <action android:name="android.telecom.CallScreeningService"/>
//     </intent-filter>
// </service>

/**
 * Screens incoming calls and blocks unauthorized numbers
 * Requires: User must set this app as "Caller ID & spam" app in settings
 * Works on: Android 10+ (API 29+)
 */
class KidSafeCallScreener : CallScreeningService() {

    @Inject lateinit var approvedContactsRepo: ApprovedContactsRepository
    @Inject lateinit var callLogRepo: CallLogRepository

    // India emergency numbers
    private val emergencyNumbers = setOf(
        "100",   // Police
        "101",   // Fire
        "102",   // Ambulance
        "108",   // Emergency
        "112",   // Universal emergency
        "1098",  // Child helpline
        "181"    // Women helpline
    )

    override fun onScreenCall(callDetails: Call.Details) {
        val incomingNumber = callDetails.handle?.schemeSpecificPart ?: ""
        val normalizedNumber = normalizePhoneNumber(incomingNumber)

        // Check if call should be allowed
        val decision = screenCall(normalizedNumber)

        // Log the call attempt
        CoroutineScope(Dispatchers.IO).launch {
            callLogRepo.logIncomingCall(
                number = normalizedNumber,
                allowed = decision.isAllowed,
                timestamp = System.currentTimeMillis()
            )
        }

        // Respond with decision
        respondToCall(callDetails, CallResponse.Builder()
            .setDisallowCall(!decision.isAllowed)
            .setRejectCall(!decision.isAllowed)
            .setSkipCallLog(false)  // Always log in system call log
            .setSkipNotification(!decision.isAllowed)
            .build()
        )
    }

    private fun screenCall(number: String): CallDecision {
        // Always allow emergency numbers
        if (emergencyNumbers.contains(number)) {
            return CallDecision(isAllowed = true, reason = "Emergency number")
        }

        // Always allow parent numbers
        if (approvedContactsRepo.isParentNumber(number)) {
            return CallDecision(isAllowed = true, reason = "Parent")
        }

        // Check call control mode
        return when (approvedContactsRepo.getCallControlMode()) {
            CallControlMode.APPROVED_ONLY -> {
                if (approvedContactsRepo.isApproved(number)) {
                    CallDecision(isAllowed = true, reason = "Approved contact")
                } else {
                    CallDecision(isAllowed = false, reason = "Not in approved list")
                }
            }
            CallControlMode.CONTACTS_ONLY -> {
                if (isInDeviceContacts(number)) {
                    CallDecision(isAllowed = true, reason = "In contacts")
                } else {
                    CallDecision(isAllowed = false, reason = "Not in contacts")
                }
            }
            CallControlMode.ALL_ALLOWED -> {
                CallDecision(isAllowed = true, reason = "All calls allowed")
            }
        }
    }

    private fun normalizePhoneNumber(number: String): String {
        // Remove spaces, dashes, country code variations
        return number.replace(Regex("[\\s\\-()]"), "")
            .replace(Regex("^\\+91"), "")
            .replace(Regex("^91"), "")
            .replace(Regex("^0"), "")
    }

    private fun isInDeviceContacts(number: String): Boolean {
        val uri = Uri.withAppendedPath(
            ContactsContract.PhoneLookup.CONTENT_FILTER_URI,
            Uri.encode(number)
        )
        contentResolver.query(uri, arrayOf(ContactsContract.PhoneLookup._ID), null, null, null)?.use {
            return it.count > 0
        }
        return false
    }

    data class CallDecision(val isAllowed: Boolean, val reason: String)
}

enum class CallControlMode {
    APPROVED_ONLY,   // Only parent-approved numbers
    CONTACTS_ONLY,   // Any number in device contacts
    ALL_ALLOWED      // All calls allowed, just log them
}
```

#### Implementation: Outgoing Call Control

```kotlin
/**
 * Control outgoing calls by:
 * 1. Hiding default dialer app
 * 2. Providing our own dialer that checks approved list
 *
 * Note: We CANNOT intercept outgoing calls directly.
 * We can only control by hiding dialer and providing custom one.
 */
class OutgoingCallControl(
    private val context: Context,
    private val dpm: DevicePolicyManager,
    private val adminComponent: ComponentName
) {
    // System dialer packages to hide
    private val dialerPackages = listOf(
        "com.google.android.dialer",
        "com.android.dialer",
        "com.samsung.android.dialer",
        "com.android.contacts"
    )

    /**
     * Hide system dialer - child can only use our safe dialer
     */
    fun enforceApprovedDialerOnly() {
        if (!dpm.isDeviceOwnerApp(context.packageName)) return

        dialerPackages.forEach { pkg ->
            try {
                dpm.setApplicationHidden(adminComponent, pkg, true)
            } catch (e: Exception) {
                // Package might not exist on this device
            }
        }
    }

    /**
     * Restore system dialer (when switching to "all allowed" mode)
     */
    fun restoreSystemDialer() {
        dialerPackages.forEach { pkg ->
            try {
                dpm.setApplicationHidden(adminComponent, pkg, false)
            } catch (e: Exception) {
                // Package might not exist
            }
        }
    }
}

/**
 * Simple safe dialer activity that only allows approved numbers
 */
class SafeDialerActivity : AppCompatActivity() {

    @Inject lateinit var approvedContacts: ApprovedContactsRepository

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_safe_dialer)

        // Show only approved contacts in a simple list
        val contacts = approvedContacts.getAllApproved()
        // Display contacts with big tap-to-call buttons
        // No manual number entry for maximum safety
    }

    fun makeCall(phoneNumber: String) {
        // Verify one more time before calling
        if (!approvedContacts.isApproved(phoneNumber) &&
            !approvedContacts.isEmergency(phoneNumber)) {
            Toast.makeText(this, "This number is not approved", Toast.LENGTH_SHORT).show()
            return
        }

        val intent = Intent(Intent.ACTION_CALL).apply {
            data = Uri.parse("tel:$phoneNumber")
        }
        startActivity(intent)
    }
}
```

#### Reading Call History

```kotlin
/**
 * Read call log to show parents who child talked to
 * Requires: READ_CALL_LOG permission
 */
class CallHistoryReader(private val context: Context) {

    data class CallRecord(
        val number: String,
        val name: String?,
        val type: CallType,
        val duration: Int,  // seconds
        val timestamp: Long
    )

    enum class CallType { INCOMING, OUTGOING, MISSED, BLOCKED }

    fun getRecentCalls(limit: Int = 50): List<CallRecord> {
        val calls = mutableListOf<CallRecord>()

        if (ContextCompat.checkSelfPermission(context, Manifest.permission.READ_CALL_LOG)
            != PackageManager.PERMISSION_GRANTED) {
            return emptyList()
        }

        val cursor = context.contentResolver.query(
            CallLog.Calls.CONTENT_URI,
            arrayOf(
                CallLog.Calls.NUMBER,
                CallLog.Calls.CACHED_NAME,
                CallLog.Calls.TYPE,
                CallLog.Calls.DURATION,
                CallLog.Calls.DATE
            ),
            null, null,
            "${CallLog.Calls.DATE} DESC"
        )

        cursor?.use {
            val numberIdx = it.getColumnIndex(CallLog.Calls.NUMBER)
            val nameIdx = it.getColumnIndex(CallLog.Calls.CACHED_NAME)
            val typeIdx = it.getColumnIndex(CallLog.Calls.TYPE)
            val durationIdx = it.getColumnIndex(CallLog.Calls.DURATION)
            val dateIdx = it.getColumnIndex(CallLog.Calls.DATE)

            var count = 0
            while (it.moveToNext() && count < limit) {
                calls.add(CallRecord(
                    number = it.getString(numberIdx) ?: "",
                    name = it.getString(nameIdx),
                    type = when (it.getInt(typeIdx)) {
                        CallLog.Calls.INCOMING_TYPE -> CallType.INCOMING
                        CallLog.Calls.OUTGOING_TYPE -> CallType.OUTGOING
                        CallLog.Calls.MISSED_TYPE -> CallType.MISSED
                        CallLog.Calls.BLOCKED_TYPE -> CallType.BLOCKED
                        else -> CallType.INCOMING
                    },
                    duration = it.getInt(durationIdx),
                    timestamp = it.getLong(dateIdx)
                ))
                count++
            }
        }

        return calls
    }

    /**
     * Sync call history to Firebase for parent to view
     */
    suspend fun syncToParent(deviceId: String) {
        val calls = getRecentCalls(limit = 100)
        val callData = calls.map { call ->
            mapOf(
                "number" to call.number,
                "name" to (call.name ?: "Unknown"),
                "type" to call.type.name,
                "duration" to call.duration,
                "timestamp" to call.timestamp
            )
        }

        Firebase.firestore
            .collection("devices")
            .document(deviceId)
            .collection("call_history")
            .document("recent")
            .set(mapOf("calls" to callData, "synced_at" to FieldValue.serverTimestamp()))
    }
}
```

#### What We CANNOT Do
- ❌ Listen to call audio
- ❌ Record calls (heavily restricted on Android 10+)
- ❌ Intercept outgoing calls in real-time (can only hide dialer)

#### What We CAN Do
- ✅ Block incoming calls from non-approved numbers
- ✅ Hide dialer so child can only call approved contacts
- ✅ Read full call history (who, when, how long)
- ✅ Sync call history to parent app
- ✅ Always allow emergency numbers

**Acceptance Criteria**:
- [ ] CallScreeningService registered and active
- [ ] Emergency numbers (100, 101, 108, 112) always allowed
- [ ] Blocked calls logged for parent to see
- [ ] Call history synced to parent app
- [ ] Safe dialer only shows approved contacts

---

### F21: SMS & Messaging Control

**Description**: Parent monitors and controls text messaging.

#### Feasibility Assessment

| Feature | Possible? | Difficulty | Notes |
|---------|-----------|------------|-------|
| **Read SMS content** | ⚠️ HARD | High | Google Play restricts this heavily |
| **Read SMS metadata (who, when)** | ⚠️ HARD | High | Same restriction |
| **Block incoming SMS** | ❌ NO | N/A | Cannot block at system level |
| **Hide SMS app entirely** | ✅ YES | Easy | Via DPC app hiding |
| **Read WhatsApp/Telegram** | ❌ NO | N/A | End-to-end encrypted, sandboxed |

#### ⚠️ Google Play Policy Warning

**This is a MAJOR limitation:**

Google Play has very strict policies for SMS/Call Log permissions. Apps can ONLY use these permissions if:
1. App is the **default SMS/Dialer app**, OR
2. App has a **specific approved use case** (backup, caller ID)

**If we request READ_SMS without being default SMS app, our app will be REJECTED from Play Store.**

#### Practical Options

**Option A: Disable SMS Entirely (Recommended for Kids)**
```kotlin
/**
 * Simplest approach: Just hide the SMS app entirely
 * Kids use WhatsApp/Telegram anyway, which we can control via whitelist
 */
class SmsControlManager(
    private val context: Context,
    private val dpm: DevicePolicyManager,
    private val adminComponent: ComponentName
) {
    private val smsApps = listOf(
        "com.google.android.apps.messaging",  // Google Messages
        "com.samsung.android.messaging",       // Samsung Messages
        "com.android.mms"                      // Stock SMS
    )

    /**
     * Hide SMS apps - child cannot send/receive SMS
     * This is the SIMPLEST and PLAY STORE SAFE approach
     */
    fun disableSms() {
        smsApps.forEach { pkg ->
            try {
                dpm.setApplicationHidden(adminComponent, pkg, true)
            } catch (e: Exception) {
                // Package might not exist
            }
        }
    }

    fun enableSms() {
        smsApps.forEach { pkg ->
            try {
                dpm.setApplicationHidden(adminComponent, pkg, false)
            } catch (e: Exception) {
                // Package might not exist
            }
        }
    }
}
```

**Option B: Become Default SMS App (Complex)**
```kotlin
/**
 * WARNING: This is complex and changes user experience significantly.
 * Only do this if SMS monitoring is absolutely required.
 *
 * If our app becomes the default SMS app:
 * - We must handle ALL SMS functionality (receive, send, display)
 * - We can read/monitor all messages
 * - We must show notifications for new SMS
 * - Complex to implement correctly
 */

// To request becoming default SMS app:
class SmsAppRequestActivity : AppCompatActivity() {

    private val smsRoleLauncher = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if (result.resultCode == RESULT_OK) {
            // We are now the default SMS app - can read SMS
        }
    }

    fun requestSmsRole() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            val roleManager = getSystemService(RoleManager::class.java)
            if (roleManager.isRoleAvailable(RoleManager.ROLE_SMS)) {
                if (!roleManager.isRoleHeld(RoleManager.ROLE_SMS)) {
                    val intent = roleManager.createRequestRoleIntent(RoleManager.ROLE_SMS)
                    smsRoleLauncher.launch(intent)
                }
            }
        }
    }
}

/**
 * If we ARE the default SMS app, we can read messages like this:
 */
class SmsReader(private val context: Context) {

    fun getAllSms(): List<SmsMessage> {
        // This ONLY works if we are default SMS app
        val messages = mutableListOf<SmsMessage>()

        context.contentResolver.query(
            Telephony.Sms.CONTENT_URI,
            arrayOf(
                Telephony.Sms.ADDRESS,
                Telephony.Sms.BODY,
                Telephony.Sms.DATE,
                Telephony.Sms.TYPE
            ),
            null, null,
            "${Telephony.Sms.DATE} DESC"
        )?.use { cursor ->
            val addressIdx = cursor.getColumnIndex(Telephony.Sms.ADDRESS)
            val bodyIdx = cursor.getColumnIndex(Telephony.Sms.BODY)
            val dateIdx = cursor.getColumnIndex(Telephony.Sms.DATE)
            val typeIdx = cursor.getColumnIndex(Telephony.Sms.TYPE)

            while (cursor.moveToNext()) {
                messages.add(SmsMessage(
                    address = cursor.getString(addressIdx),
                    body = cursor.getString(bodyIdx),
                    date = cursor.getLong(dateIdx),
                    isIncoming = cursor.getInt(typeIdx) == Telephony.Sms.MESSAGE_TYPE_INBOX
                ))
            }
        }
        return messages
    }

    data class SmsMessage(
        val address: String,
        val body: String,
        val date: Long,
        val isIncoming: Boolean
    )
}
```

#### What About WhatsApp/Telegram/Instagram DMs?

| App | Can We Read Messages? | Can We Block? |
|-----|----------------------|---------------|
| WhatsApp | ❌ NO (encrypted) | ✅ YES (hide app) |
| Telegram | ❌ NO (encrypted) | ✅ YES (hide app) |
| Instagram | ❌ NO (sandboxed) | ✅ YES (hide app) |
| Signal | ❌ NO (encrypted) | ✅ YES (hide app) |

**Reality**: We CANNOT read messages from any modern messaging app. They are all encrypted and sandboxed.

**What We CAN Do**: Block the entire app via our whitelist system.

#### Recommended Approach for MVP

```kotlin
/**
 * RECOMMENDED: Don't try to read SMS/messages.
 * Instead, control which messaging apps are available.
 *
 * This is:
 * 1. Play Store compliant
 * 2. Simple to implement
 * 3. Actually effective
 * 4. Privacy-respecting
 */
class MessagingControlManager(
    private val appWhitelistManager: AppWhitelistManager
) {
    // Messaging apps by safety level
    enum class MessagingTier {
        BLOCKED,        // Not allowed
        FAMILY_ONLY,    // WhatsApp with family contacts only
        MONITORED       // App allowed, but parent sees usage time
    }

    private val messagingApps = mapOf(
        "com.whatsapp" to "WhatsApp",
        "org.telegram.messenger" to "Telegram",
        "com.instagram.android" to "Instagram",
        "com.snapchat.android" to "Snapchat",
        "com.facebook.orca" to "Messenger",
        "com.discord" to "Discord"
    )

    /**
     * Set which messaging apps child can use
     * We can't read messages, but we can control access
     */
    fun setMessagingPolicy(policy: Map<String, MessagingTier>) {
        policy.forEach { (packageName, tier) ->
            when (tier) {
                MessagingTier.BLOCKED -> {
                    appWhitelistManager.setAppHidden(packageName, true)
                }
                MessagingTier.FAMILY_ONLY, MessagingTier.MONITORED -> {
                    appWhitelistManager.setAppHidden(packageName, false)
                    // App usage will still be tracked via screen time
                }
            }
        }
    }
}
```

#### Summary

| What Parents Want | What's Possible | Our Solution |
|------------------|-----------------|--------------|
| Read child's WhatsApp | ❌ Not possible | Control if WhatsApp is allowed |
| See who child texts | ⚠️ Only if we're SMS app | Just disable SMS app |
| Block unknown contacts | ❌ Not at message level | Block entire messaging apps |
| Keyword monitoring | ❌ Can't read messages | Not possible |

**Honest Recommendation**: For a kid's device, the simplest approach is:
1. Hide SMS apps entirely (they'll use WhatsApp anyway)
2. Control WhatsApp via app whitelist (allow/block)
3. Track time spent in messaging apps (already have this)
4. Don't promise message reading - it's not reliably possible

**Acceptance Criteria**:
- [ ] SMS apps can be hidden via DPC
- [ ] Messaging apps controllable via whitelist
- [ ] Time spent in messaging apps tracked
- [ ] Clear documentation that message content is NOT readable

---

### F22: App-Specific Controls

**Description**: Granular controls for specific apps beyond simple on/off.

#### Feasibility Assessment

| Control | Possible? | Method | Notes |
|---------|-----------|--------|-------|
| **YouTube Restricted Mode** | ✅ YES | Managed Configuration | Works via Android Enterprise |
| **Chrome SafeSearch** | ✅ YES | DNS filtering | Via F19 |
| **Chrome Incognito Block** | ✅ YES | Managed Configuration | Chrome supports this |
| **Block In-App Purchases** | ✅ YES | DPC Restriction | System-wide restriction |
| **Read YouTube History** | ❌ NO | N/A | Not accessible |
| **Read Chrome History** | ❌ NO | N/A | Sandboxed |
| **Block Comments in YouTube** | ❌ NO | N/A | Not configurable |
| **Block Instagram DMs** | ❌ NO | N/A | Not configurable |

#### Implementation: Managed App Configurations

Android Enterprise allows Device Owners to push configurations to apps that support it.

```kotlin
/**
 * Push managed configurations to enterprise-ready apps
 * Works with: Chrome, YouTube, Gmail, Google apps, some third-party apps
 */
class ManagedConfigManager(
    private val context: Context,
    private val dpm: DevicePolicyManager,
    private val adminComponent: ComponentName
) {

    /**
     * Configure Chrome browser restrictions
     */
    fun configureChromeRestrictions(config: ChromeConfig) {
        if (!dpm.isDeviceOwnerApp(context.packageName)) return

        val restrictions = Bundle().apply {
            // Block incognito mode
            putBoolean("IncognitoModeAvailability", !config.allowIncognito)

            // Force SafeSearch (also use DNS filtering as backup)
            putBoolean("ForceGoogleSafeSearch", config.forceSafeSearch)
            putBoolean("ForceYouTubeSafetyMode", config.forceYouTubeSafe)

            // Block dangerous features
            putBoolean("DeveloperToolsDisabled", true)
            putBoolean("DownloadRestrictions", if (config.blockDownloads) 3 else 0)

            // Homepage and startup
            if (config.homepage != null) {
                putString("HomepageLocation", config.homepage)
                putBoolean("HomepageIsNewTabPage", false)
            }

            // Block specific URLs (managed via policy)
            if (config.blockedUrls.isNotEmpty()) {
                putStringArray("URLBlocklist", config.blockedUrls.toTypedArray())
            }
        }

        dpm.setApplicationRestrictions(
            adminComponent,
            "com.android.chrome",
            restrictions
        )
    }

    /**
     * Configure YouTube restrictions
     */
    fun configureYouTubeRestrictions(restrictedMode: Boolean) {
        if (!dpm.isDeviceOwnerApp(context.packageName)) return

        val restrictions = Bundle().apply {
            // Enable restricted mode (filters mature content)
            putBoolean("restrict", restrictedMode)
        }

        // YouTube app package
        dpm.setApplicationRestrictions(
            adminComponent,
            "com.google.android.youtube",
            restrictions
        )
    }

    /**
     * Block in-app purchases system-wide
     */
    fun blockInAppPurchases(block: Boolean) {
        if (!dpm.isDeviceOwnerApp(context.packageName)) return

        // This blocks ALL in-app purchases on the device
        dpm.addUserRestriction(
            adminComponent,
            UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES
        )

        // Also restrict Play Store purchases
        val playRestrictions = Bundle().apply {
            putBoolean("require_password_for_purchase", true)
        }

        dpm.setApplicationRestrictions(
            adminComponent,
            "com.android.vending",  // Play Store
            playRestrictions
        )
    }

    data class ChromeConfig(
        val allowIncognito: Boolean = false,
        val forceSafeSearch: Boolean = true,
        val forceYouTubeSafe: Boolean = true,
        val blockDownloads: Boolean = false,
        val homepage: String? = null,
        val blockedUrls: List<String> = emptyList()
    )
}
```

#### Blocking In-App Purchases Completely

```kotlin
/**
 * Multiple layers to prevent unauthorized purchases
 */
class PurchaseBlocker(
    private val context: Context,
    private val dpm: DevicePolicyManager,
    private val adminComponent: ComponentName
) {

    /**
     * Apply all purchase restrictions
     */
    fun blockAllPurchases() {
        // 1. Disable unknown sources (APK installs)
        dpm.addUserRestriction(adminComponent, UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES)

        // 2. Require authentication for Play Store
        val playConfig = Bundle().apply {
            // Require password/fingerprint for every purchase
            putInt("require_password_for_purchase", 2)  // Always require
        }
        dpm.setApplicationRestrictions(adminComponent, "com.android.vending", playConfig)

        // 3. Disable adding accounts (prevents signing into payment-enabled accounts)
        dpm.addUserRestriction(adminComponent, UserManager.DISALLOW_MODIFY_ACCOUNTS)

        // 4. Block NFC (prevents tap-to-pay)
        dpm.addUserRestriction(adminComponent, UserManager.DISALLOW_OUTGOING_BEAM)
    }

    /**
     * For specific game apps, we can check if they have IAP and warn parent
     */
    suspend fun checkAppForIAP(packageName: String): Boolean {
        // This would require checking Play Store listing
        // or maintaining a database of apps with IAP
        val knownIAPApps = setOf(
            "com.supercell.clashofclans",
            "com.king.candycrushsaga",
            "com.pubg.mobile",
            "com.roblox.client"
        )
        return packageName in knownIAPApps
    }
}
```

#### What We CAN Control Per-App

| App | Possible Controls |
|-----|------------------|
| **Chrome** | Incognito off, SafeSearch on, block URLs, block downloads |
| **YouTube** | Restricted Mode on (filters adult content) |
| **Play Store** | Require password for purchases, block mature apps |
| **Gmail** | Restrict external sharing (enterprise accounts only) |
| **Any App** | Time limits, completely block via whitelist |

#### What We CANNOT Control

| What Parents Want | Reality |
|------------------|---------|
| See YouTube watch history | ❌ Not accessible to other apps |
| Block specific YouTube channels | ❌ Not configurable |
| See Chrome browsing history | ❌ Sandboxed |
| Block Instagram DMs | ❌ Instagram doesn't expose this |
| See Spotify listening history | ⚠️ Only via Spotify API (requires login) |
| Block in-app chat in games | ❌ Per-game, not controllable |

#### Example: Parent App UI for App Controls

```dart
// In Parent App (Flutter)
class AppControlsScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('App Controls')),
      body: ListView(
        children: [
          // YouTube controls
          ExpansionTile(
            leading: Icon(Icons.play_circle),
            title: Text('YouTube'),
            children: [
              SwitchListTile(
                title: Text('Restricted Mode'),
                subtitle: Text('Filters out mature content'),
                value: true,  // This we CAN control
                onChanged: (v) => updateYouTubeRestriction(v),
              ),
              ListTile(
                title: Text('Daily Time Limit'),
                subtitle: Text('1 hour'),  // This we CAN control via screen time
                trailing: Icon(Icons.chevron_right),
              ),
              // Note: No option for "view watch history" - not possible
            ],
          ),

          // Chrome controls
          ExpansionTile(
            leading: Icon(Icons.language),
            title: Text('Chrome'),
            children: [
              SwitchListTile(
                title: Text('Block Incognito Mode'),
                value: true,  // This we CAN control
                onChanged: (v) => updateChromeIncognito(v),
              ),
              SwitchListTile(
                title: Text('Safe Search'),
                subtitle: Text('Filter adult search results'),
                value: true,  // This we CAN control
                onChanged: (v) => updateChromeSafeSearch(v),
              ),
              // Note: No option for "view browsing history" - not possible
            ],
          ),

          // In-app purchases (global)
          SwitchListTile(
            title: Text('Block In-App Purchases'),
            subtitle: Text('Prevents buying within any app'),
            value: true,  // This we CAN control
            onChanged: (v) => updatePurchaseBlock(v),
          ),
        ],
      ),
    );
  }
}
```

**Acceptance Criteria**:
- [ ] Chrome Incognito can be disabled via managed config
- [ ] YouTube Restricted Mode can be forced
- [ ] In-app purchases can be blocked system-wide
- [ ] Per-app time limits work (already have via F08)
- [ ] UI clearly shows only controls that actually work

---

## P3 Features (Future)

| Feature | Possible? | Description |
|---------|-----------|-------------|
| **Multiple Parents** | ✅ Yes | Both parents can control device |
| **Chore Rewards** | ✅ Yes | Earn screen time for completing tasks |
| **App Install Requests** | ✅ Yes | Child requests apps, parent approves |
| **Homework Mode** | ✅ Yes | Allow only educational apps during study time |
| **Family Calendar** | ✅ Yes | Shared schedule, automatic mode switching |
| **Photo Monitoring** | ⚠️ Partial | Can see photo count, not content easily |
| **AI Alerts** | ⚠️ Limited | Can detect usage patterns, not content |
| **Read Messages** | ❌ No | Technically impossible on modern apps |
| **Browser History** | ❌ No | Apps are sandboxed |

---

## Feature Dependencies

```
F01 Device Setup
 └── F02 Custom Launcher
      └── F03 DPC
           └── F04 App Whitelist
                └── F05 Remote Mode Toggle
                     └── F08 Screen Time
                          └── F09 Bedtime
                               └── F10 Remote Lock

F06 Panic Button (independent, P0)
F07 Device Status (independent, P0)
F11 Volume Limit (depends on F03)
F12 Usage Reports (depends on F04)
F13 Location (depends on F01)
F14 Safe Zones (depends on F13)
F15 Remote Install (depends on F03)
F16 Now Playing (depends on F07)

Advanced Parental Controls (P2):
F19 Web Content Filtering (depends on F03 DPC)
 └── Uses: Private DNS via DPC (setGlobalPrivateDnsModeSpecifiedHost)
 └── Uses: CleanBrowsing/NextDNS for category filtering
 └── Status: ✅ FULLY POSSIBLE

F20 Call & Contact Control (depends on F03 DPC)
 └── Uses: CallScreeningService (block incoming)
 └── Uses: Hide dialer + custom safe dialer (control outgoing)
 └── Uses: READ_CALL_LOG permission (view history)
 └── Status: ✅ FULLY POSSIBLE

F21 SMS & Messaging Control - DEPRIORITIZED
 └── Problem: Google Play heavily restricts SMS permissions
 └── Solution: Just hide SMS apps entirely via DPC
 └── Status: ⚠️ NOT RECOMMENDED FOR MVP

F22 App-Specific Controls (depends on F03 DPC)
 └── Uses: setApplicationRestrictions for Chrome, YouTube
 └── Uses: DPC restrictions for in-app purchases
 └── Status: ✅ FULLY POSSIBLE (for supported apps)
```

---

## P0.5 Features (Security - Must Have)

### F17: Device Security Validation

**Description**: Validate device integrity to prevent tampering and bypass.

| Aspect | Specification |
|--------|---------------|
| **Play Integrity API** | Verify device is genuine, not rooted |
| **Root Detection** | Block operation on rooted devices |
| **Tamper Detection** | Detect attempts to modify launcher/DPC |
| **Certificate Pinning** | Prevent MITM attacks on API calls |
| **Secure Boot Check** | Verify device booted securely |

**Validation Checks:**

| Check | Action on Failure |
|-------|-------------------|
| Play Integrity fails | Alert parent, restrict features |
| Root detected | Block operation, notify parent |
| USB debugging enabled | Warn parent, log event |
| Unknown sources enabled | Auto-disable if DO active |
| Bootloader unlocked | Alert parent, document risk |

**Acceptance Criteria**:
- [ ] Play Integrity API integrated and working
- [ ] Root detection catches common root methods (Magisk, SuperSU)
- [ ] Certificate pinning on all API calls
- [ ] Security status reported to parent app
- [ ] Tamper attempts logged and alerted

---

### F18: DPDP Act Compliance

**Description**: Ensure compliance with India's Digital Personal Data Protection Act 2023.

| Requirement | Implementation |
|-------------|----------------|
| **Parental Consent** | OTP verification before child device activation |
| **Data Minimization** | Only collect essential operational data |
| **No Profiling** | No behavioral analysis or ad targeting |
| **Data Access** | Parents can export all child data |
| **Data Deletion** | Complete erasure on account deletion |
| **Consent Logging** | Record consent with timestamp |

**Consent Flow:**
1. Parent creates account
2. Parent verifies phone/email with OTP
3. Parent explicitly consents to data collection
4. Consent recorded with IP, timestamp, version
5. Child device links after consent verified

**Acceptance Criteria**:
- [ ] OTP-based parental verification working
- [ ] Consent logged with full audit trail
- [ ] Data export feature functional
- [ ] Data deletion removes all child data
- [ ] Privacy policy clearly explains data usage

---

---

## Advanced Parental Controls - Honest Assessment

### What's ACTUALLY Possible vs What Parents Expect

| What Parents Want | Technically Possible? | Our Solution |
|-------------------|----------------------|--------------|
| Block adult websites | ✅ YES | DNS filtering (F19) |
| See browsing history | ❌ NO | Just block browsers or use DNS logs |
| Block unknown callers | ✅ YES | CallScreeningService (F20) |
| See call history | ✅ YES | Read system call log |
| Read child's WhatsApp | ❌ NO | Block WhatsApp entirely if needed |
| Read child's SMS | ⚠️ VERY HARD | Google Play restricts this |
| Block in-app purchases | ✅ YES | DPC restriction |
| See YouTube history | ❌ NO | Force Restricted Mode instead |
| Block specific games | ✅ YES | App whitelist |
| Location tracking | ✅ YES | Already in F13 |

### Implementation Roadmap

| Feature | Priority | Sprint | Complexity | Play Store Risk |
|---------|----------|--------|------------|-----------------|
| F19 Web Filtering | P2 | Sprint 4 | Medium | ✅ Safe |
| F20 Call Control | P2 | Sprint 4 | Medium | ✅ Safe |
| F21 SMS Control | P3 | Future | High | ⚠️ May be rejected |
| F22 App Controls | P2 | Sprint 4 | Easy | ✅ Safe |

### Quick Reference: What We CAN Do

```
✅ POSSIBLE (Implement These):
├── Block websites by category (DNS filtering)
├── Block unknown callers (CallScreeningService)
├── View call history (who called, when, how long)
├── Block any app entirely (whitelist)
├── Set per-app time limits (screen time)
├── Force YouTube Restricted Mode
├── Block Chrome Incognito mode
├── Block in-app purchases (DPC)
├── Track location (GPS)
├── Remote lock device
└── View screen time reports

❌ NOT POSSIBLE (Don't Promise These):
├── Read WhatsApp/Telegram messages (encrypted)
├── See browsing history from Chrome (sandboxed)
├── See YouTube watch history (sandboxed)
├── Read SMS easily (Play Store blocks this)
├── Listen to phone calls (not possible)
├── Record calls (heavily restricted)
├── Block specific Instagram DMs (not exposed)
└── See what child searches in apps (sandboxed)
```

### Recommended MVP Scope for Advanced Controls

**Sprint 4 (Days 50-65) - Add These:**

1. **DNS Content Filtering (F19)** - ~3 days
   - Use CleanBrowsing free tier for MVP
   - Set via DPC Private DNS
   - Three levels: Strict/Moderate/Off

2. **Call Control (F20)** - ~4 days
   - CallScreeningService for incoming calls
   - Hide dialer, provide safe contacts-only dialer
   - Read and sync call history

3. **App-Specific Settings (F22)** - ~2 days
   - YouTube Restricted Mode
   - Chrome Incognito block
   - Block in-app purchases

**NOT for MVP (Maybe Never):**
- SMS monitoring (Play Store risk too high)
- Message reading (technically impossible)
- Browsing history (not accessible)

---

## Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| **Latency** | Commands execute within 5 seconds |
| **Availability** | 99.5% uptime for cloud services |
| **Battery** | < 5% daily drain from our services |
| **Storage** | < 100MB total app size |
| **Offline** | Core features work offline, sync when online |
| **Security** | All data encrypted in transit and at rest |
| **Privacy** | Data stored in India (Firebase asia-south1) |
| **Integrity** | Play Integrity API validation on device |
| **Compliance** | DPDP Act 2023 compliant |
