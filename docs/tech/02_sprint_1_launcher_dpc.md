# Sprint 1: Launcher & DPC Foundation (Days 8-17)

## Sprint Goal
Build a functional custom launcher with Kids Mode and Standard Mode, and implement the Device Policy Controller for device lockdown.

---

## Epic E02: Custom Launcher

### Story E02-S01: Launcher Activity Foundation
**Points**: 5
**Owner**: Android Developer
**Priority**: P0

#### Description
Create the main launcher activity that responds to HOME intent and can replace the default launcher.

#### Acceptance Criteria
- [ ] App can be set as default launcher
- [ ] Pressing HOME button opens our app
- [ ] Activity handles configuration changes
- [ ] Proper lifecycle management

#### Tasks

##### Task E02-S01-T01: Create LauncherApplication Class
**Time**: 30 minutes

Create `app/src/main/java/com/kidtunes/launcher/LauncherApplication.kt`:

```kotlin
package com.kidtunes.launcher

import android.app.Application
import com.google.firebase.FirebaseApp
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class LauncherApplication : Application() {

    override fun onCreate() {
        super.onCreate()
        FirebaseApp.initializeApp(this)
    }
}
```

Update `AndroidManifest.xml` to use this Application class:
```xml
<application
    android:name=".LauncherApplication"
    ... >
```

##### Task E02-S01-T02: Configure AndroidManifest for Launcher
**Time**: 45 minutes

Update `app/src/main/AndroidManifest.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- Permissions -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
    <uses-permission android:name="android.permission.QUERY_ALL_PACKAGES"
        tools:ignore="QueryAllPackagesPermission" />
    <uses-permission android:name="android.permission.PACKAGE_USAGE_STATS"
        tools:ignore="ProtectedPermissions" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.VIBRATE" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />

    <application
        android:name=".LauncherApplication"
        android:allowBackup="false"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.KidTunesLauncher"
        tools:targetApi="34">

        <!-- Main Launcher Activity -->
        <activity
            android:name=".ui.LauncherActivity"
            android:exported="true"
            android:launchMode="singleTask"
            android:clearTaskOnLaunch="true"
            android:stateNotNeeded="true"
            android:screenOrientation="portrait"
            android:windowSoftInputMode="adjustPan"
            android:excludeFromRecents="true"
            android:configChanges="orientation|screenSize|keyboard|keyboardHidden|navigation">

            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.HOME" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>

        <!-- Screen Time Lock Activity -->
        <activity
            android:name=".ui.lockscreen.ScreenTimeLockActivity"
            android:exported="false"
            android:launchMode="singleTask"
            android:screenOrientation="portrait"
            android:excludeFromRecents="true" />

        <!-- Bedtime Lock Activity -->
        <activity
            android:name=".ui.lockscreen.BedtimeLockActivity"
            android:exported="false"
            android:launchMode="singleTask"
            android:screenOrientation="portrait"
            android:excludeFromRecents="true" />

        <!-- Panic Activity -->
        <activity
            android:name=".ui.panic.PanicActivity"
            android:exported="false"
            android:launchMode="singleTask"
            android:screenOrientation="portrait" />

    </application>

</manifest>
```

##### Task E02-S01-T03: Create LauncherActivity
**Time**: 1 hour

Create `app/src/main/java/com/kidtunes/launcher/ui/LauncherActivity.kt`:

```kotlin
package com.kidtunes.launcher.ui

import android.os.Bundle
import android.view.KeyEvent
import androidx.activity.OnBackPressedCallback
import androidx.appcompat.app.AppCompatActivity
import androidx.fragment.app.Fragment
import androidx.fragment.app.commit
import com.kidtunes.launcher.R
import com.kidtunes.launcher.databinding.ActivityLauncherBinding
import com.kidtunes.launcher.data.local.LauncherPreferences
import com.kidtunes.launcher.ui.kids.KidsModeFragment
import com.kidtunes.launcher.ui.standard.StandardModeFragment
import com.kidtunes.launcher.ui.pin.PinDialogFragment
import dagger.hilt.android.AndroidEntryPoint
import javax.inject.Inject

@AndroidEntryPoint
class LauncherActivity : AppCompatActivity() {

    private lateinit var binding: ActivityLauncherBinding

    @Inject
    lateinit var preferences: LauncherPreferences

    private var isKidsMode: Boolean = true

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityLauncherBinding.inflate(layoutInflater)
        setContentView(binding.root)

        setupBackPressHandler()
        loadCurrentMode()
    }

    private fun setupBackPressHandler() {
        onBackPressedDispatcher.addCallback(this, object : OnBackPressedCallback(true) {
            override fun handleOnBackPressed() {
                // Do nothing - don't allow back press to exit launcher
            }
        })
    }

    private fun loadCurrentMode() {
        isKidsMode = preferences.isKidsMode()
        showModeFragment()
    }

    private fun showModeFragment() {
        val fragment: Fragment = if (isKidsMode) {
            KidsModeFragment.newInstance()
        } else {
            StandardModeFragment.newInstance()
        }

        supportFragmentManager.commit {
            replace(R.id.fragmentContainer, fragment)
        }
    }

    fun requestModeSwitch() {
        if (isKidsMode) {
            // Switching from Kids to Standard requires PIN
            showPinDialog()
        } else {
            // Switching from Standard to Kids - no PIN needed
            switchToKidsMode()
        }
    }

    private fun showPinDialog() {
        val dialog = PinDialogFragment.newInstance(
            onSuccess = { switchToStandardMode() },
            onCancel = { /* Stay in Kids Mode */ }
        )
        dialog.show(supportFragmentManager, "pin_dialog")
    }

    fun switchToKidsMode() {
        isKidsMode = true
        preferences.setKidsMode(true)
        showModeFragment()
        notifyModeChange()
    }

    fun switchToStandardMode() {
        isKidsMode = false
        preferences.setKidsMode(false)
        showModeFragment()
        notifyModeChange()
    }

    private fun notifyModeChange() {
        // Notify Firebase of mode change
        // Will implement in Sprint 3
    }

    override fun onKeyLongPress(keyCode: Int, event: KeyEvent?): Boolean {
        if (keyCode == KeyEvent.KEYCODE_POWER && isKidsMode) {
            // Long press power in Kids Mode - show PIN to switch
            requestModeSwitch()
            return true
        }
        return super.onKeyLongPress(keyCode, event)
    }

    override fun onNewIntent(intent: android.content.Intent) {
        super.onNewIntent(intent)
        // Handle when HOME is pressed while we're already running
        // Just refresh the current view
    }
}
```

##### Task E02-S01-T04: Create Activity Layout
**Time**: 20 minutes

Create `app/src/main/res/layout/activity_launcher.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/fragmentContainer"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/launcher_background" />
```

##### Task E02-S01-T05: Create LauncherPreferences
**Time**: 45 minutes

Create `app/src/main/java/com/kidtunes/launcher/data/local/LauncherPreferences.kt`:

```kotlin
package com.kidtunes.launcher.data.local

import android.content.Context
import android.content.SharedPreferences
import androidx.core.content.edit
import androidx.security.crypto.EncryptedSharedPreferences
import androidx.security.crypto.MasterKey
import dagger.hilt.android.qualifiers.ApplicationContext
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class LauncherPreferences @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    private val securePrefs: SharedPreferences = EncryptedSharedPreferences.create(
        context,
        "kidtunes_secure_prefs",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    private val prefs: SharedPreferences = context.getSharedPreferences(
        "kidtunes_prefs",
        Context.MODE_PRIVATE
    )

    companion object {
        private const val KEY_KIDS_MODE = "kids_mode"
        private const val KEY_PIN = "pin"
        private const val KEY_DEVICE_ID = "device_id"
        private const val KEY_PAIRED = "is_paired"
        private const val KEY_CHILD_NAME = "child_name"
        private const val KEY_VOLUME_LIMIT = "volume_limit"
        private const val KEY_DAILY_LIMIT_MINUTES = "daily_limit_minutes"
    }

    // Mode
    fun isKidsMode(): Boolean = prefs.getBoolean(KEY_KIDS_MODE, true)
    fun setKidsMode(enabled: Boolean) = prefs.edit { putBoolean(KEY_KIDS_MODE, enabled) }

    // PIN (stored encrypted)
    fun getPin(): String? = securePrefs.getString(KEY_PIN, null)
    fun setPin(pin: String) = securePrefs.edit { putString(KEY_PIN, pin) }
    fun hasPin(): Boolean = getPin() != null

    // Device ID
    fun getDeviceId(): String? = prefs.getString(KEY_DEVICE_ID, null)
    fun setDeviceId(id: String) = prefs.edit { putString(KEY_DEVICE_ID, id) }

    // Pairing
    fun isPaired(): Boolean = prefs.getBoolean(KEY_PAIRED, false)
    fun setPaired(paired: Boolean) = prefs.edit { putBoolean(KEY_PAIRED, paired) }

    // Child Name
    fun getChildName(): String = prefs.getString(KEY_CHILD_NAME, "Child") ?: "Child"
    fun setChildName(name: String) = prefs.edit { putString(KEY_CHILD_NAME, name) }

    // Settings
    fun getVolumeLimit(): Int = prefs.getInt(KEY_VOLUME_LIMIT, 70)
    fun setVolumeLimit(percent: Int) = prefs.edit { putInt(KEY_VOLUME_LIMIT, percent) }

    fun getDailyLimitMinutes(): Int = prefs.getInt(KEY_DAILY_LIMIT_MINUTES, 120)
    fun setDailyLimitMinutes(minutes: Int) = prefs.edit { putInt(KEY_DAILY_LIMIT_MINUTES, minutes) }
}
```

Add dependency for encrypted shared preferences in `build.gradle.kts`:
```kotlin
implementation("androidx.security:security-crypto:1.1.0-alpha06")
```

##### Task E02-S01-T06: Create Hilt Module
**Time**: 30 minutes

Create `app/src/main/java/com/kidtunes/launcher/di/AppModule.kt`:

```kotlin
package com.kidtunes.launcher.di

import android.content.Context
import com.kidtunes.launcher.data.local.LauncherPreferences
import com.kidtunes.launcher.data.local.AppWhitelistManager
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    @Singleton
    fun provideLauncherPreferences(
        @ApplicationContext context: Context
    ): LauncherPreferences {
        return LauncherPreferences(context)
    }

    @Provides
    @Singleton
    fun provideAppWhitelistManager(
        @ApplicationContext context: Context,
        preferences: LauncherPreferences
    ): AppWhitelistManager {
        return AppWhitelistManager(context, preferences)
    }
}
```

---

### Story E02-S02: Kids Mode UI
**Points**: 5
**Owner**: Android Developer
**Priority**: P0

#### Description
Create the Kids Mode fragment with 2x3 grid of large app icons.

#### Acceptance Criteria
- [ ] Shows 2x3 grid of apps
- [ ] Large, touch-friendly icons (120dp)
- [ ] Shows greeting with child's name
- [ ] Panic button visible at bottom
- [ ] Status bar shows battery/signal

#### Tasks

##### Task E02-S02-T01: Create KidsModeFragment
**Time**: 1 hour

Create `app/src/main/java/com/kidtunes/launcher/ui/kids/KidsModeFragment.kt`:

```kotlin
package com.kidtunes.launcher.ui.kids

import android.content.Intent
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.fragment.app.Fragment
import androidx.fragment.app.viewModels
import androidx.lifecycle.lifecycleScope
import androidx.recyclerview.widget.GridLayoutManager
import com.kidtunes.launcher.databinding.FragmentKidsModeBinding
import com.kidtunes.launcher.data.local.LauncherPreferences
import com.kidtunes.launcher.data.local.AppWhitelistManager
import com.kidtunes.launcher.data.models.AppInfo
import com.kidtunes.launcher.ui.LauncherActivity
import com.kidtunes.launcher.ui.panic.PanicActivity
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch
import java.util.Calendar
import javax.inject.Inject

@AndroidEntryPoint
class KidsModeFragment : Fragment() {

    private var _binding: FragmentKidsModeBinding? = null
    private val binding get() = _binding!!

    @Inject
    lateinit var preferences: LauncherPreferences

    @Inject
    lateinit var appWhitelistManager: AppWhitelistManager

    private lateinit var appsAdapter: KidsModeAdapter

    companion object {
        fun newInstance() = KidsModeFragment()
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentKidsModeBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        setupGreeting()
        setupAppsGrid()
        setupPanicButton()
        observeApps()
    }

    private fun setupGreeting() {
        val childName = preferences.getChildName()
        val greeting = getGreetingText()
        binding.greetingText.text = "$greeting, $childName!"
    }

    private fun getGreetingText(): String {
        val hour = Calendar.getInstance().get(Calendar.HOUR_OF_DAY)
        return when {
            hour < 12 -> "Good morning"
            hour < 17 -> "Good afternoon"
            else -> "Good evening"
        }
    }

    private fun setupAppsGrid() {
        appsAdapter = KidsModeAdapter { appInfo ->
            launchApp(appInfo)
        }

        binding.appsRecyclerView.apply {
            layoutManager = GridLayoutManager(requireContext(), 2)
            adapter = appsAdapter
            setHasFixedSize(true)
        }
    }

    private fun setupPanicButton() {
        binding.panicButton.setOnClickListener {
            startActivity(Intent(requireContext(), PanicActivity::class.java))
        }
    }

    private fun observeApps() {
        viewLifecycleOwner.lifecycleScope.launch {
            appWhitelistManager.getWhitelistedApps().collectLatest { apps ->
                appsAdapter.submitList(apps)
            }
        }
    }

    private fun launchApp(appInfo: AppInfo) {
        val intent = requireContext().packageManager
            .getLaunchIntentForPackage(appInfo.packageName)

        if (intent != null) {
            startActivity(intent)
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

##### Task E02-S02-T02: Create Kids Mode Layout
**Time**: 45 minutes

Create `app/src/main/res/layout/fragment_kids_mode.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@drawable/bg_kids_mode_gradient"
    android:padding="24dp">

    <!-- Greeting -->
    <TextView
        android:id="@+id/greetingText"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="48dp"
        android:textSize="28sp"
        android:textColor="@color/white"
        android:textStyle="bold"
        android:fontFamily="@font/roboto_medium"
        tools:text="Good morning, Rahul!"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <!-- Apps Grid -->
    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/appsRecyclerView"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:layout_marginTop="32dp"
        android:layout_marginBottom="16dp"
        android:clipToPadding="false"
        android:overScrollMode="never"
        app:layout_constraintTop_toBottomOf="@id/greetingText"
        app:layout_constraintBottom_toTopOf="@id/panicButton"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <!-- Panic Button -->
    <com.google.android.material.button.MaterialButton
        android:id="@+id/panicButton"
        android:layout_width="0dp"
        android:layout_height="56dp"
        android:layout_marginBottom="24dp"
        android:text="🆘  I need help"
        android:textSize="18sp"
        android:textColor="@color/white"
        android:textAllCaps="false"
        android:backgroundTint="@color/panic_red"
        app:cornerRadius="28dp"
        app:layout_constraintBottom_toTopOf="@id/statusBar"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <!-- Status Bar -->
    <LinearLayout
        android:id="@+id/statusBar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginBottom="16dp"
        android:orientation="horizontal"
        android:gravity="center"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent">

        <TextView
            android:id="@+id/batteryText"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textColor="@color/white_70"
            android:textSize="14sp"
            tools:text="🔋 78%" />

        <View
            android:layout_width="16dp"
            android:layout_height="0dp" />

        <TextView
            android:id="@+id/signalText"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textColor="@color/white_70"
            android:textSize="14sp"
            tools:text="📶" />

        <View
            android:layout_width="16dp"
            android:layout_height="0dp" />

        <TextView
            android:id="@+id/volumeText"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textColor="@color/white_70"
            android:textSize="14sp"
            tools:text="🔊 70%" />

    </LinearLayout>

</androidx.constraintlayout.widget.ConstraintLayout>
```

##### Task E02-S02-T03: Create Kids Mode Adapter
**Time**: 45 minutes

Create `app/src/main/java/com/kidtunes/launcher/ui/kids/KidsModeAdapter.kt`:

```kotlin
package com.kidtunes.launcher.ui.kids

import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.DiffUtil
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView
import coil.load
import com.kidtunes.launcher.databinding.ItemKidsModeAppBinding
import com.kidtunes.launcher.data.models.AppInfo

class KidsModeAdapter(
    private val onAppClick: (AppInfo) -> Unit
) : ListAdapter<AppInfo, KidsModeAdapter.AppViewHolder>(AppDiffCallback()) {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): AppViewHolder {
        val binding = ItemKidsModeAppBinding.inflate(
            LayoutInflater.from(parent.context),
            parent,
            false
        )
        return AppViewHolder(binding)
    }

    override fun onBindViewHolder(holder: AppViewHolder, position: Int) {
        holder.bind(getItem(position))
    }

    inner class AppViewHolder(
        private val binding: ItemKidsModeAppBinding
    ) : RecyclerView.ViewHolder(binding.root) {

        init {
            binding.root.setOnClickListener {
                val position = bindingAdapterPosition
                if (position != RecyclerView.NO_POSITION) {
                    onAppClick(getItem(position))
                }
            }
        }

        fun bind(appInfo: AppInfo) {
            binding.apply {
                appIcon.setImageDrawable(appInfo.icon)
                appName.text = appInfo.label
            }
        }
    }

    class AppDiffCallback : DiffUtil.ItemCallback<AppInfo>() {
        override fun areItemsTheSame(oldItem: AppInfo, newItem: AppInfo): Boolean {
            return oldItem.packageName == newItem.packageName
        }

        override fun areContentsTheSame(oldItem: AppInfo, newItem: AppInfo): Boolean {
            return oldItem == newItem
        }
    }
}
```

##### Task E02-S02-T04: Create Kids Mode App Item Layout
**Time**: 30 minutes

Create `app/src/main/res/layout/item_kids_mode_app.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.cardview.widget.CardView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="12dp"
    app:cardCornerRadius="24dp"
    app:cardElevation="8dp"
    app:cardBackgroundColor="@color/white">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:gravity="center"
        android:padding="20dp">

        <ImageView
            android:id="@+id/appIcon"
            android:layout_width="80dp"
            android:layout_height="80dp"
            android:scaleType="fitCenter"
            tools:src="@mipmap/ic_launcher" />

        <TextView
            android:id="@+id/appName"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="12dp"
            android:textSize="16sp"
            android:textColor="@color/text_primary"
            android:textStyle="bold"
            android:maxLines="1"
            android:ellipsize="end"
            tools:text="Spotify" />

    </LinearLayout>

</androidx.cardview.widget.CardView>
```

##### Task E02-S02-T05: Create AppInfo Model
**Time**: 20 minutes

Create `app/src/main/java/com/kidtunes/launcher/data/models/AppInfo.kt`:

```kotlin
package com.kidtunes.launcher.data.models

import android.graphics.drawable.Drawable

data class AppInfo(
    val packageName: String,
    val label: String,
    val icon: Drawable?,
    val isSystemApp: Boolean = false,
    val isEnabled: Boolean = true,
    val category: AppCategory = AppCategory.OTHER
)

enum class AppCategory {
    MUSIC,
    EDUCATION,
    GAMES,
    COMMUNICATION,
    FINANCE,
    UTILITY,
    OTHER
}
```

##### Task E02-S02-T06: Create Gradient Background
**Time**: 15 minutes

Create `app/src/main/res/drawable/bg_kids_mode_gradient.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <gradient
        android:type="linear"
        android:angle="135"
        android:startColor="#4A90E2"
        android:centerColor="#5BA0F0"
        android:endColor="#7BB8F5" />
</shape>
```

##### Task E02-S02-T07: Add Colors
**Time**: 15 minutes

Update `app/src/main/res/values/colors.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!-- Primary Colors -->
    <color name="primary">#4A90E2</color>
    <color name="primary_dark">#357ABD</color>
    <color name="primary_light">#7BB8F5</color>

    <!-- Background -->
    <color name="launcher_background">#F5F5F5</color>
    <color name="white">#FFFFFF</color>
    <color name="white_70">#B3FFFFFF</color>

    <!-- Text -->
    <color name="text_primary">#333333</color>
    <color name="text_secondary">#666666</color>

    <!-- Status -->
    <color name="panic_red">#FF4444</color>
    <color name="success_green">#4CAF50</color>
    <color name="warning_orange">#FF9800</color>

    <!-- Kids Mode -->
    <color name="kids_bg_start">#4A90E2</color>
    <color name="kids_bg_end">#7BB8F5</color>
</resources>
```

---

### Story E02-S03: Standard Mode UI
**Points**: 5
**Owner**: Android Developer
**Priority**: P0

#### Description
Create the Standard Mode fragment with 4x4 grid and Now Playing widget.

#### Acceptance Criteria
- [ ] Shows 4x4 grid of apps
- [ ] Standard icon sizes (56dp)
- [ ] Now Playing widget at bottom
- [ ] Bottom navigation bar
- [ ] Can switch to Kids Mode easily

#### Tasks

##### Task E02-S03-T01: Create StandardModeFragment
**Time**: 1 hour

Create `app/src/main/java/com/kidtunes/launcher/ui/standard/StandardModeFragment.kt`:

```kotlin
package com.kidtunes.launcher.ui.standard

import android.content.Intent
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.fragment.app.Fragment
import androidx.lifecycle.lifecycleScope
import androidx.recyclerview.widget.GridLayoutManager
import com.kidtunes.launcher.databinding.FragmentStandardModeBinding
import com.kidtunes.launcher.data.local.AppWhitelistManager
import com.kidtunes.launcher.data.models.AppInfo
import com.kidtunes.launcher.ui.LauncherActivity
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch
import javax.inject.Inject

@AndroidEntryPoint
class StandardModeFragment : Fragment() {

    private var _binding: FragmentStandardModeBinding? = null
    private val binding get() = _binding!!

    @Inject
    lateinit var appWhitelistManager: AppWhitelistManager

    private lateinit var appsAdapter: StandardModeAdapter

    companion object {
        fun newInstance() = StandardModeFragment()
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentStandardModeBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        setupAppsGrid()
        setupBottomNav()
        setupKidsModeButton()
        observeApps()
    }

    private fun setupAppsGrid() {
        appsAdapter = StandardModeAdapter { appInfo ->
            launchApp(appInfo)
        }

        binding.appsRecyclerView.apply {
            layoutManager = GridLayoutManager(requireContext(), 4)
            adapter = appsAdapter
            setHasFixedSize(true)
        }
    }

    private fun setupBottomNav() {
        binding.bottomNav.setOnItemSelectedListener { item ->
            when (item.itemId) {
                // Handle navigation items
            }
            true
        }
    }

    private fun setupKidsModeButton() {
        binding.kidsModeButton.setOnClickListener {
            (activity as? LauncherActivity)?.switchToKidsMode()
        }
    }

    private fun observeApps() {
        viewLifecycleOwner.lifecycleScope.launch {
            appWhitelistManager.getWhitelistedApps().collectLatest { apps ->
                appsAdapter.submitList(apps)
            }
        }
    }

    private fun launchApp(appInfo: AppInfo) {
        val intent = requireContext().packageManager
            .getLaunchIntentForPackage(appInfo.packageName)

        if (intent != null) {
            startActivity(intent)
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

##### Task E02-S03-T02: Create Standard Mode Layout
**Time**: 45 minutes

Create `app/src/main/res/layout/fragment_standard_mode.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/launcher_background">

    <!-- Kids Mode Button -->
    <com.google.android.material.button.MaterialButton
        android:id="@+id/kidsModeButton"
        style="@style/Widget.Material3.Button.TonalButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        android:text="🔒 Kids Mode"
        android:textAllCaps="false"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <!-- Apps Grid -->
    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/appsRecyclerView"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:layout_marginTop="8dp"
        android:padding="8dp"
        android:clipToPadding="false"
        app:layout_constraintTop_toBottomOf="@id/kidsModeButton"
        app:layout_constraintBottom_toTopOf="@id/nowPlayingCard"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <!-- Now Playing Card -->
    <androidx.cardview.widget.CardView
        android:id="@+id/nowPlayingCard"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        app:cardCornerRadius="16dp"
        app:cardElevation="4dp"
        app:layout_constraintBottom_toTopOf="@id/bottomNav"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent">

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:padding="12dp"
            android:gravity="center_vertical">

            <ImageView
                android:id="@+id/albumArt"
                android:layout_width="48dp"
                android:layout_height="48dp"
                android:scaleType="centerCrop"
                tools:src="@mipmap/ic_launcher" />

            <LinearLayout
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:layout_marginStart="12dp"
                android:orientation="vertical">

                <TextView
                    android:id="@+id/trackTitle"
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:textSize="14sp"
                    android:textStyle="bold"
                    android:textColor="@color/text_primary"
                    android:maxLines="1"
                    android:ellipsize="end"
                    tools:text="Kesariya" />

                <TextView
                    android:id="@+id/artistName"
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:textSize="12sp"
                    android:textColor="@color/text_secondary"
                    android:maxLines="1"
                    android:ellipsize="end"
                    tools:text="Arijit Singh" />

            </LinearLayout>

            <ImageButton
                android:id="@+id/playPauseButton"
                android:layout_width="40dp"
                android:layout_height="40dp"
                android:background="?attr/selectableItemBackgroundBorderless"
                android:src="@drawable/ic_play"
                android:contentDescription="Play/Pause" />

        </LinearLayout>

    </androidx.cardview.widget.CardView>

    <!-- Bottom Navigation -->
    <com.google.android.material.bottomnavigation.BottomNavigationView
        android:id="@+id/bottomNav"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:background="@color/white"
        app:menu="@menu/bottom_nav_menu"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

##### Task E02-S03-T03: Create Standard Mode Adapter
**Time**: 30 minutes

Create `app/src/main/java/com/kidtunes/launcher/ui/standard/StandardModeAdapter.kt`:

```kotlin
package com.kidtunes.launcher.ui.standard

import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.DiffUtil
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView
import com.kidtunes.launcher.databinding.ItemStandardModeAppBinding
import com.kidtunes.launcher.data.models.AppInfo

class StandardModeAdapter(
    private val onAppClick: (AppInfo) -> Unit
) : ListAdapter<AppInfo, StandardModeAdapter.AppViewHolder>(AppDiffCallback()) {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): AppViewHolder {
        val binding = ItemStandardModeAppBinding.inflate(
            LayoutInflater.from(parent.context),
            parent,
            false
        )
        return AppViewHolder(binding)
    }

    override fun onBindViewHolder(holder: AppViewHolder, position: Int) {
        holder.bind(getItem(position))
    }

    inner class AppViewHolder(
        private val binding: ItemStandardModeAppBinding
    ) : RecyclerView.ViewHolder(binding.root) {

        init {
            binding.root.setOnClickListener {
                val position = bindingAdapterPosition
                if (position != RecyclerView.NO_POSITION) {
                    onAppClick(getItem(position))
                }
            }
        }

        fun bind(appInfo: AppInfo) {
            binding.apply {
                appIcon.setImageDrawable(appInfo.icon)
                appName.text = appInfo.label
            }
        }
    }

    class AppDiffCallback : DiffUtil.ItemCallback<AppInfo>() {
        override fun areItemsTheSame(oldItem: AppInfo, newItem: AppInfo): Boolean {
            return oldItem.packageName == newItem.packageName
        }

        override fun areContentsTheSame(oldItem: AppInfo, newItem: AppInfo): Boolean {
            return oldItem == newItem
        }
    }
}
```

##### Task E02-S03-T04: Create Standard Mode App Item Layout
**Time**: 20 minutes

Create `app/src/main/res/layout/item_standard_mode_app.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:gravity="center"
    android:padding="8dp"
    android:background="?attr/selectableItemBackgroundBorderless">

    <ImageView
        android:id="@+id/appIcon"
        android:layout_width="56dp"
        android:layout_height="56dp"
        android:scaleType="fitCenter"
        tools:src="@mipmap/ic_launcher" />

    <TextView
        android:id="@+id/appName"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="4dp"
        android:textSize="11sp"
        android:textColor="@color/text_primary"
        android:maxLines="1"
        android:ellipsize="end"
        tools:text="Spotify" />

</LinearLayout>
```

##### Task E02-S03-T05: Create Bottom Navigation Menu
**Time**: 15 minutes

Create `app/src/main/res/menu/bottom_nav_menu.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">

    <item
        android:id="@+id/nav_home"
        android:icon="@drawable/ic_home"
        android:title="Home" />

    <item
        android:id="@+id/nav_recent"
        android:icon="@drawable/ic_recent"
        android:title="Recent" />

    <item
        android:id="@+id/nav_search"
        android:icon="@drawable/ic_search"
        android:title="Search" />

</menu>
```

---

### Story E02-S04: PIN Dialog
**Points**: 3
**Owner**: Android Developer
**Priority**: P0

#### Description
Create PIN entry dialog for mode switching and sensitive operations.

#### Acceptance Criteria
- [ ] 4-6 digit PIN entry
- [ ] Numeric keypad
- [ ] Dot indicators for entered digits
- [ ] Wrong PIN feedback
- [ ] Lockout after 3 failed attempts

#### Tasks

##### Task E02-S04-T01: Create PinDialogFragment
**Time**: 1 hour

Create `app/src/main/java/com/kidtunes/launcher/ui/pin/PinDialogFragment.kt`:

```kotlin
package com.kidtunes.launcher.ui.pin

import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.view.WindowManager
import androidx.fragment.app.DialogFragment
import com.kidtunes.launcher.R
import com.kidtunes.launcher.databinding.DialogPinBinding
import com.kidtunes.launcher.data.local.LauncherPreferences
import dagger.hilt.android.AndroidEntryPoint
import javax.inject.Inject

@AndroidEntryPoint
class PinDialogFragment : DialogFragment() {

    private var _binding: DialogPinBinding? = null
    private val binding get() = _binding!!

    @Inject
    lateinit var preferences: LauncherPreferences

    private var enteredPin = ""
    private var failedAttempts = 0
    private var onSuccess: (() -> Unit)? = null
    private var onCancel: (() -> Unit)? = null

    companion object {
        private const val MAX_PIN_LENGTH = 6
        private const val MIN_PIN_LENGTH = 4
        private const val MAX_FAILED_ATTEMPTS = 3
        private const val LOCKOUT_DURATION_MS = 5 * 60 * 1000L // 5 minutes

        fun newInstance(
            onSuccess: () -> Unit,
            onCancel: () -> Unit
        ): PinDialogFragment {
            return PinDialogFragment().apply {
                this.onSuccess = onSuccess
                this.onCancel = onCancel
            }
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setStyle(STYLE_NO_FRAME, R.style.Theme_PinDialog)
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = DialogPinBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        setupKeypad()
        setupCancelButton()
        updatePinDots()
    }

    override fun onStart() {
        super.onStart()
        dialog?.window?.apply {
            setLayout(
                WindowManager.LayoutParams.MATCH_PARENT,
                WindowManager.LayoutParams.MATCH_PARENT
            )
            setBackgroundDrawableResource(android.R.color.transparent)
        }
    }

    private fun setupKeypad() {
        val numberButtons = listOf(
            binding.btn0, binding.btn1, binding.btn2, binding.btn3,
            binding.btn4, binding.btn5, binding.btn6, binding.btn7,
            binding.btn8, binding.btn9
        )

        numberButtons.forEachIndexed { index, button ->
            button.setOnClickListener {
                onNumberPressed(index)
            }
        }

        binding.btnDelete.setOnClickListener {
            onDeletePressed()
        }
    }

    private fun setupCancelButton() {
        binding.btnCancel.setOnClickListener {
            onCancel?.invoke()
            dismiss()
        }
    }

    private fun onNumberPressed(number: Int) {
        if (enteredPin.length < MAX_PIN_LENGTH) {
            enteredPin += number.toString()
            updatePinDots()

            if (enteredPin.length >= MIN_PIN_LENGTH) {
                validatePin()
            }
        }
    }

    private fun onDeletePressed() {
        if (enteredPin.isNotEmpty()) {
            enteredPin = enteredPin.dropLast(1)
            updatePinDots()
        }
    }

    private fun updatePinDots() {
        val dots = listOf(
            binding.dot1, binding.dot2, binding.dot3,
            binding.dot4, binding.dot5, binding.dot6
        )

        dots.forEachIndexed { index, dot ->
            dot.isActivated = index < enteredPin.length
        }
    }

    private fun validatePin() {
        val correctPin = preferences.getPin()

        if (enteredPin == correctPin) {
            onSuccess?.invoke()
            dismiss()
        } else if (enteredPin.length == MAX_PIN_LENGTH) {
            handleWrongPin()
        }
    }

    private fun handleWrongPin() {
        failedAttempts++
        enteredPin = ""
        updatePinDots()

        if (failedAttempts >= MAX_FAILED_ATTEMPTS) {
            showLockout()
        } else {
            showWrongPinFeedback()
        }
    }

    private fun showWrongPinFeedback() {
        binding.pinTitle.text = "Wrong PIN. ${MAX_FAILED_ATTEMPTS - failedAttempts} attempts left"
        binding.pinContainer.animate()
            .translationX(20f)
            .setDuration(50)
            .withEndAction {
                binding.pinContainer.animate()
                    .translationX(-20f)
                    .setDuration(50)
                    .withEndAction {
                        binding.pinContainer.animate()
                            .translationX(0f)
                            .setDuration(50)
                            .start()
                    }
                    .start()
            }
            .start()
    }

    private fun showLockout() {
        binding.pinTitle.text = "Too many attempts. Try again in 5 minutes."
        binding.keypadContainer.visibility = View.GONE
        // Could implement countdown timer here
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

##### Task E02-S04-T02: Create PIN Dialog Layout
**Time**: 45 minutes

Create `app/src/main/res/layout/dialog_pin.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#CC000000"
    android:padding="32dp">

    <LinearLayout
        android:id="@+id/pinContainer"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:gravity="center"
        android:padding="24dp"
        android:background="@drawable/bg_pin_dialog"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent">

        <TextView
            android:id="@+id/pinTitle"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Enter Parent PIN"
            android:textSize="20sp"
            android:textColor="@color/text_primary"
            android:textStyle="bold" />

        <!-- PIN Dots -->
        <LinearLayout
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="24dp"
            android:layout_marginBottom="24dp"
            android:orientation="horizontal">

            <View
                android:id="@+id/dot1"
                android:layout_width="16dp"
                android:layout_height="16dp"
                android:layout_margin="8dp"
                android:background="@drawable/pin_dot_selector" />

            <View
                android:id="@+id/dot2"
                android:layout_width="16dp"
                android:layout_height="16dp"
                android:layout_margin="8dp"
                android:background="@drawable/pin_dot_selector" />

            <View
                android:id="@+id/dot3"
                android:layout_width="16dp"
                android:layout_height="16dp"
                android:layout_margin="8dp"
                android:background="@drawable/pin_dot_selector" />

            <View
                android:id="@+id/dot4"
                android:layout_width="16dp"
                android:layout_height="16dp"
                android:layout_margin="8dp"
                android:background="@drawable/pin_dot_selector" />

            <View
                android:id="@+id/dot5"
                android:layout_width="16dp"
                android:layout_height="16dp"
                android:layout_margin="8dp"
                android:background="@drawable/pin_dot_selector" />

            <View
                android:id="@+id/dot6"
                android:layout_width="16dp"
                android:layout_height="16dp"
                android:layout_margin="8dp"
                android:background="@drawable/pin_dot_selector" />

        </LinearLayout>

        <!-- Keypad -->
        <GridLayout
            android:id="@+id/keypadContainer"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:columnCount="3"
            android:rowCount="4">

            <Button android:id="@+id/btn1" style="@style/PinButton" android:text="1" />
            <Button android:id="@+id/btn2" style="@style/PinButton" android:text="2" />
            <Button android:id="@+id/btn3" style="@style/PinButton" android:text="3" />
            <Button android:id="@+id/btn4" style="@style/PinButton" android:text="4" />
            <Button android:id="@+id/btn5" style="@style/PinButton" android:text="5" />
            <Button android:id="@+id/btn6" style="@style/PinButton" android:text="6" />
            <Button android:id="@+id/btn7" style="@style/PinButton" android:text="7" />
            <Button android:id="@+id/btn8" style="@style/PinButton" android:text="8" />
            <Button android:id="@+id/btn9" style="@style/PinButton" android:text="9" />
            <View android:layout_width="64dp" android:layout_height="64dp" />
            <Button android:id="@+id/btn0" style="@style/PinButton" android:text="0" />
            <ImageButton
                android:id="@+id/btnDelete"
                style="@style/PinButton"
                android:src="@drawable/ic_backspace"
                android:background="?attr/selectableItemBackgroundBorderless" />

        </GridLayout>

        <Button
            android:id="@+id/btnCancel"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="16dp"
            android:text="Cancel"
            android:textColor="@color/text_secondary"
            android:background="@android:color/transparent" />

    </LinearLayout>

</androidx.constraintlayout.widget.ConstraintLayout>
```

---

### Story E02-S05: App Whitelist Manager
**Points**: 5
**Owner**: Android Developer
**Priority**: P0

#### Description
Create the app whitelist manager that controls which apps are visible.

#### Acceptance Criteria
- [ ] Can query all installed apps
- [ ] Maintains whitelist in preferences
- [ ] Syncs whitelist from Firebase
- [ ] Filters apps based on whitelist
- [ ] Detects new app installations

#### Tasks

##### Task E02-S05-T01: Create AppWhitelistManager
**Time**: 1.5 hours

Create `app/src/main/java/com/kidtunes/launcher/data/local/AppWhitelistManager.kt`:

```kotlin
package com.kidtunes.launcher.data.local

import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.content.pm.ResolveInfo
import android.graphics.drawable.Drawable
import com.kidtunes.launcher.data.models.AppCategory
import com.kidtunes.launcher.data.models.AppInfo
import dagger.hilt.android.qualifiers.ApplicationContext
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.withContext
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class AppWhitelistManager @Inject constructor(
    @ApplicationContext private val context: Context,
    private val preferences: LauncherPreferences
) {
    private val packageManager = context.packageManager
    private val _whitelist = MutableStateFlow<Set<String>>(emptySet())
    private val _installedApps = MutableStateFlow<List<AppInfo>>(emptyList())

    // Default apps that are always available
    private val defaultWhitelist = setOf(
        "com.android.deskclock",  // Clock
        "com.sec.android.app.clockpackage", // Samsung Clock
    )

    // Known app categories
    private val appCategories = mapOf(
        "com.spotify.music" to AppCategory.MUSIC,
        "com.jio.saavn" to AppCategory.MUSIC,
        "com.amazon.mp3" to AppCategory.MUSIC,
        "com.google.android.apps.youtube.music" to AppCategory.MUSIC,
        "com.gaana" to AppCategory.MUSIC,
        "com.wynk.music" to AppCategory.MUSIC,
        "com.google.android.apps.nbu.paisa.user" to AppCategory.FINANCE,
        "com.phonepe.app" to AppCategory.FINANCE,
        "com.whatsapp" to AppCategory.COMMUNICATION,
        "org.telegram.messenger" to AppCategory.COMMUNICATION,
        "com.duolingo" to AppCategory.EDUCATION,
        "com.byjus" to AppCategory.EDUCATION,
    )

    init {
        loadWhitelist()
    }

    private fun loadWhitelist() {
        val stored = preferences.getWhitelist()
        _whitelist.value = stored.ifEmpty { defaultWhitelist }
    }

    suspend fun refreshInstalledApps() = withContext(Dispatchers.IO) {
        val intent = Intent(Intent.ACTION_MAIN).apply {
            addCategory(Intent.CATEGORY_LAUNCHER)
        }

        val resolveInfos: List<ResolveInfo> = packageManager.queryIntentActivities(
            intent,
            PackageManager.MATCH_ALL
        )

        val apps = resolveInfos.mapNotNull { resolveInfo ->
            try {
                val packageName = resolveInfo.activityInfo.packageName
                val appInfo = packageManager.getApplicationInfo(packageName, 0)

                AppInfo(
                    packageName = packageName,
                    label = packageManager.getApplicationLabel(appInfo).toString(),
                    icon = packageManager.getApplicationIcon(appInfo),
                    isSystemApp = (appInfo.flags and android.content.pm.ApplicationInfo.FLAG_SYSTEM) != 0,
                    isEnabled = appInfo.enabled,
                    category = appCategories[packageName] ?: AppCategory.OTHER
                )
            } catch (e: Exception) {
                null
            }
        }.sortedBy { it.label.lowercase() }

        _installedApps.value = apps
    }

    fun getWhitelistedApps(): Flow<List<AppInfo>> {
        return _installedApps.map { apps ->
            apps.filter { app ->
                _whitelist.value.contains(app.packageName)
            }
        }
    }

    fun getAllInstalledApps(): Flow<List<AppInfo>> = _installedApps

    fun isWhitelisted(packageName: String): Boolean {
        return _whitelist.value.contains(packageName)
    }

    fun addToWhitelist(packageName: String) {
        val updated = _whitelist.value + packageName
        _whitelist.value = updated
        preferences.saveWhitelist(updated)
    }

    fun removeFromWhitelist(packageName: String) {
        if (defaultWhitelist.contains(packageName)) return // Can't remove defaults

        val updated = _whitelist.value - packageName
        _whitelist.value = updated
        preferences.saveWhitelist(updated)
    }

    fun setWhitelist(packageNames: Set<String>) {
        _whitelist.value = packageNames + defaultWhitelist
        preferences.saveWhitelist(_whitelist.value)
    }

    fun getAppInfo(packageName: String): AppInfo? {
        return _installedApps.value.find { it.packageName == packageName }
    }

    fun getAppIcon(packageName: String): Drawable? {
        return try {
            packageManager.getApplicationIcon(packageName)
        } catch (e: Exception) {
            null
        }
    }
}
```

##### Task E02-S05-T02: Add Whitelist to Preferences
**Time**: 20 minutes

Add to `LauncherPreferences.kt`:

```kotlin
companion object {
    // ... existing keys
    private const val KEY_WHITELIST = "app_whitelist"
}

fun getWhitelist(): Set<String> {
    val json = prefs.getString(KEY_WHITELIST, "[]") ?: "[]"
    return try {
        org.json.JSONArray(json).let { array ->
            (0 until array.length()).map { array.getString(it) }.toSet()
        }
    } catch (e: Exception) {
        emptySet()
    }
}

fun saveWhitelist(whitelist: Set<String>) {
    val json = org.json.JSONArray(whitelist.toList()).toString()
    prefs.edit { putString(KEY_WHITELIST, json) }
}
```

##### Task E02-S05-T03: Create Package Change Receiver
**Time**: 30 minutes

Create `app/src/main/java/com/kidtunes/launcher/receivers/PackageChangeReceiver.kt`:

```kotlin
package com.kidtunes.launcher.receivers

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import com.kidtunes.launcher.data.local.AppWhitelistManager
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import javax.inject.Inject

@AndroidEntryPoint
class PackageChangeReceiver : BroadcastReceiver() {

    @Inject
    lateinit var appWhitelistManager: AppWhitelistManager

    override fun onReceive(context: Context, intent: Intent) {
        when (intent.action) {
            Intent.ACTION_PACKAGE_ADDED,
            Intent.ACTION_PACKAGE_REMOVED,
            Intent.ACTION_PACKAGE_REPLACED -> {
                val packageName = intent.data?.schemeSpecificPart
                CoroutineScope(Dispatchers.IO).launch {
                    appWhitelistManager.refreshInstalledApps()
                }

                // Notify parent app about package change
                // Will implement in Sprint 3
            }
        }
    }
}
```

Add to `AndroidManifest.xml`:
```xml
<receiver
    android:name=".receivers.PackageChangeReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.PACKAGE_ADDED" />
        <action android:name="android.intent.action.PACKAGE_REMOVED" />
        <action android:name="android.intent.action.PACKAGE_REPLACED" />
        <data android:scheme="package" />
    </intent-filter>
</receiver>
```

---

## Epic E03: Device Policy Controller (DPC)

### Story E03-S01: DPC Foundation
**Points**: 5
**Owner**: Android Developer
**Priority**: P0

#### Description
Create the Device Policy Controller app that enforces device restrictions.

#### Acceptance Criteria
- [ ] DPC can be set as Device Owner
- [ ] QR provisioning works
- [ ] Basic policies can be enforced
- [ ] Cannot be uninstalled

#### Tasks

##### Task E03-S01-T01: Create DeviceAdminReceiver
**Time**: 1 hour

Create `app/src/main/java/com/kidtunes/dpc/receivers/KidDeviceAdminReceiver.kt`:

```kotlin
package com.kidtunes.dpc.receivers

import android.app.admin.DeviceAdminReceiver
import android.app.admin.DevicePolicyManager
import android.content.ComponentName
import android.content.Context
import android.content.Intent
import android.os.UserHandle
import android.util.Log

class KidDeviceAdminReceiver : DeviceAdminReceiver() {

    companion object {
        private const val TAG = "KidDeviceAdmin"

        fun getComponentName(context: Context): ComponentName {
            return ComponentName(context, KidDeviceAdminReceiver::class.java)
        }

        fun isDeviceOwner(context: Context): Boolean {
            val dpm = context.getSystemService(Context.DEVICE_POLICY_SERVICE)
                as DevicePolicyManager
            return dpm.isDeviceOwnerApp(context.packageName)
        }
    }

    override fun onEnabled(context: Context, intent: Intent) {
        super.onEnabled(context, intent)
        Log.d(TAG, "Device admin enabled")
    }

    override fun onDisabled(context: Context, intent: Intent) {
        super.onDisabled(context, intent)
        Log.d(TAG, "Device admin disabled")
    }

    override fun onProfileProvisioningComplete(context: Context, intent: Intent) {
        super.onProfileProvisioningComplete(context, intent)
        Log.d(TAG, "Profile provisioning complete")

        // Apply initial policies
        val dpm = context.getSystemService(Context.DEVICE_POLICY_SERVICE)
            as DevicePolicyManager
        val componentName = getComponentName(context)

        // Start the main setup activity
        val setupIntent = Intent(context, com.kidtunes.dpc.provisioning.ProvisioningActivity::class.java).apply {
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        }
        context.startActivity(setupIntent)
    }

    override fun onLockTaskModeEntering(context: Context, intent: Intent, pkg: String) {
        super.onLockTaskModeEntering(context, intent, pkg)
        Log.d(TAG, "Lock task mode entering: $pkg")
    }

    override fun onLockTaskModeExiting(context: Context, intent: Intent) {
        super.onLockTaskModeExiting(context, intent)
        Log.d(TAG, "Lock task mode exiting")
    }
}
```

##### Task E03-S01-T02: Configure DPC AndroidManifest
**Time**: 30 minutes

Update DPC's `AndroidManifest.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />

    <application
        android:name=".DPCApplication"
        android:allowBackup="false"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/Theme.KidTunesDPC"
        tools:targetApi="34">

        <!-- Device Admin Receiver -->
        <receiver
            android:name=".receivers.KidDeviceAdminReceiver"
            android:permission="android.permission.BIND_DEVICE_ADMIN"
            android:exported="true">
            <meta-data
                android:name="android.app.device_admin"
                android:resource="@xml/device_admin_receiver" />
            <intent-filter>
                <action android:name="android.app.action.DEVICE_ADMIN_ENABLED" />
                <action android:name="android.app.action.PROFILE_PROVISIONING_COMPLETE" />
            </intent-filter>
        </receiver>

        <!-- Provisioning Activity -->
        <activity
            android:name=".provisioning.ProvisioningActivity"
            android:exported="true"
            android:launchMode="singleTask">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

    </application>

</manifest>
```

##### Task E03-S01-T03: Create Device Admin XML
**Time**: 15 minutes

Create `app/src/main/res/xml/device_admin_receiver.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<device-admin xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-policies>
        <limit-password />
        <watch-login />
        <reset-password />
        <force-lock />
        <wipe-data />
        <expire-password />
        <encrypted-storage />
        <disable-camera />
        <disable-keyguard-features />
    </uses-policies>
</device-admin>
```

##### Task E03-S01-T04: Create PolicyManager
**Time**: 1.5 hours

Create `app/src/main/java/com/kidtunes/dpc/policies/PolicyManager.kt`:

```kotlin
package com.kidtunes.dpc.policies

import android.app.admin.DevicePolicyManager
import android.content.ComponentName
import android.content.Context
import android.os.UserManager
import com.kidtunes.dpc.receivers.KidDeviceAdminReceiver

class PolicyManager(private val context: Context) {

    private val dpm = context.getSystemService(Context.DEVICE_POLICY_SERVICE)
        as DevicePolicyManager
    private val componentName = KidDeviceAdminReceiver.getComponentName(context)

    fun isDeviceOwner(): Boolean {
        return dpm.isDeviceOwnerApp(context.packageName)
    }

    fun applyInitialPolicies() {
        if (!isDeviceOwner()) return

        // Block app installations
        blockAppInstallation()

        // Block settings access
        blockDangerousSettings()

        // Set our launcher as default
        setDefaultLauncher()

        // Protect our apps from uninstall
        protectApps()
    }

    fun blockAppInstallation() {
        if (!isDeviceOwner()) return

        // Block unknown sources
        dpm.addUserRestriction(componentName, UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES)

        // Block installing apps
        dpm.addUserRestriction(componentName, UserManager.DISALLOW_INSTALL_APPS)

        // Hide Play Store (optional - might want to keep for managed installs)
        // setPackageHidden("com.android.vending", true)
    }

    fun allowAppInstallation() {
        if (!isDeviceOwner()) return

        dpm.clearUserRestriction(componentName, UserManager.DISALLOW_INSTALL_APPS)
    }

    fun blockDangerousSettings() {
        if (!isDeviceOwner()) return

        // Block modifying accounts
        dpm.addUserRestriction(componentName, UserManager.DISALLOW_MODIFY_ACCOUNTS)

        // Block factory reset (can still be done with recovery)
        dpm.addUserRestriction(componentName, UserManager.DISALLOW_FACTORY_RESET)

        // Block safe boot (power menu)
        dpm.addUserRestriction(componentName, UserManager.DISALLOW_SAFE_BOOT)

        // Block USB debugging
        dpm.addUserRestriction(componentName, UserManager.DISALLOW_DEBUGGING_FEATURES)

        // Block changing WiFi
        dpm.addUserRestriction(componentName, UserManager.DISALLOW_CONFIG_WIFI)

        // Block airplane mode
        dpm.addUserRestriction(componentName, UserManager.DISALLOW_AIRPLANE_MODE)
    }

    fun setDefaultLauncher() {
        if (!isDeviceOwner()) return

        val launcherComponent = ComponentName(
            "com.kidtunes.launcher",
            "com.kidtunes.launcher.ui.LauncherActivity"
        )

        val filter = android.content.IntentFilter(android.content.Intent.ACTION_MAIN).apply {
            addCategory(android.content.Intent.CATEGORY_HOME)
            addCategory(android.content.Intent.CATEGORY_DEFAULT)
        }

        dpm.addPersistentPreferredActivity(componentName, filter, launcherComponent)
    }

    fun protectApps() {
        if (!isDeviceOwner()) return

        // Make our apps uninstallable
        val protectedPackages = listOf(
            "com.kidtunes.launcher",
            "com.kidtunes.dpc"
        )

        protectedPackages.forEach { packageName ->
            dpm.setUninstallBlocked(componentName, packageName, true)
        }
    }

    fun setPackageHidden(packageName: String, hidden: Boolean) {
        if (!isDeviceOwner()) return
        dpm.setApplicationHidden(componentName, packageName, hidden)
    }

    fun setPackageSuspended(packageName: String, suspended: Boolean) {
        if (!isDeviceOwner()) return
        dpm.setPackagesSuspended(componentName, arrayOf(packageName), suspended)
    }

    fun lockDevice() {
        if (!isDeviceOwner()) return
        dpm.lockNow()
    }

    fun enableLockTaskMode(packageNames: Array<String>) {
        if (!isDeviceOwner()) return
        dpm.setLockTaskPackages(componentName, packageNames)
    }

    fun disableLockTaskMode() {
        if (!isDeviceOwner()) return
        dpm.setLockTaskPackages(componentName, arrayOf())
    }

    fun setKeyguardDisabled(disabled: Boolean) {
        if (!isDeviceOwner()) return
        dpm.setKeyguardDisabled(componentName, disabled)
    }

    fun clearRestrictions() {
        if (!isDeviceOwner()) return

        val restrictions = listOf(
            UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES,
            UserManager.DISALLOW_INSTALL_APPS,
            UserManager.DISALLOW_MODIFY_ACCOUNTS,
            UserManager.DISALLOW_FACTORY_RESET,
            UserManager.DISALLOW_SAFE_BOOT,
            UserManager.DISALLOW_DEBUGGING_FEATURES,
            UserManager.DISALLOW_CONFIG_WIFI,
            UserManager.DISALLOW_AIRPLANE_MODE
        )

        restrictions.forEach { restriction ->
            dpm.clearUserRestriction(componentName, restriction)
        }
    }
}
```

---

## Sprint 1 Checklist

### Days 8-10: Launcher Foundation
- [ ] LauncherApplication created
- [ ] AndroidManifest configured
- [ ] LauncherActivity handles HOME intent
- [ ] Can be set as default launcher
- [ ] LauncherPreferences working

### Days 11-12: Kids Mode
- [ ] KidsModeFragment created
- [ ] 2x3 grid layout working
- [ ] Large icons display
- [ ] Greeting shows
- [ ] Panic button visible

### Days 13-14: Standard Mode
- [ ] StandardModeFragment created
- [ ] 4x4 grid layout working
- [ ] Bottom navigation
- [ ] Now Playing widget (placeholder)
- [ ] Kids Mode button

### Day 15: PIN Dialog
- [ ] PIN entry UI
- [ ] PIN validation
- [ ] Wrong PIN feedback
- [ ] Lockout after failures

### Days 16-17: App Whitelist & DPC
- [ ] AppWhitelistManager created
- [ ] Whitelist filtering works
- [ ] DPC compiles
- [ ] Device Admin Receiver works
- [ ] PolicyManager created

---

## Sprint 1 Deliverables

1. ✅ Custom launcher responds to HOME button
2. ✅ Kids Mode UI with 2x3 grid
3. ✅ Standard Mode UI with 4x4 grid
4. ✅ Mode switching with PIN
5. ✅ App whitelist filtering
6. ✅ DPC foundation ready
7. ✅ Ready for Firebase integration in Sprint 2
