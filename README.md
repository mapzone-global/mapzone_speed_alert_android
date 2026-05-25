# MapZone Speed Alert Android SDK

[![](https://jitpack.io/v/mapzone-global/mapzone_speed_alert_android.svg)](https://jitpack.io/#mapzone-global/mapzone_speed_alert_android)

This SDK is provided only for Vietmap MAPs API enterprise customers. Contact your Vietmap account manager for access or [Vietmap Solutions](https://zalo.me/3189066936017422854) Zalo OA if you are interested in becoming a customer.

---

## Requirements

| | Minimum |
|---|---|
| `minSdk` | 21 (Android 5.0 Lollipop) |
| `compileSdk` | 34 |
| Java / Kotlin | Java 8 source / Kotlin 1.8+ |
| ABIs | `armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64` |
| Page-size support | 16 KB-page devices (Android 15+) |
| Network | HTTPS access to `*.map.zone` |
| Location | GPS hardware required |

> **Note:** The AAR ships prebuilt `.so` libraries — your project does not need the Android NDK to consume the SDK. NDK is only required if you rebuild from source.

---

## Installation

### 1. Add the JitPack repository

**Gradle 7.0+ (`settings.gradle` / `settings.gradle.kts`):**

```gradle
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

**Legacy (`build.gradle` at project root):**

```gradle
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

### 2. Add the dependency

```gradle
dependencies {
    implementation 'com.github.mapzone-global:mapzone_speed_alert_android:<latest_version>'
}
```

Pick the version number from the JitPack badge at the top of this README.

> **Note:** The AAR bundles its own native libraries. If your app also ships other native code, make sure your `packagingOptions` do not strip `lib/*/libmapzone_native.so`.

---

## Permissions

### `AndroidManifest.xml`

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- Only if you need speed alerts when the screen is off / app is backgrounded -->
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
```

### Runtime permission request (Android 6.0+)

```kotlin
private val permissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { results ->
    val granted = results.values.all { it }
    if (granted) startEngine() else showRationale()
}

permissionLauncher.launch(
    arrayOf(
        Manifest.permission.ACCESS_FINE_LOCATION,
        Manifest.permission.ACCESS_COARSE_LOCATION,
        Manifest.permission.ACCESS_BACKGROUND_LOCATION   // request *after* FINE/COARSE
    )
)
```

> **Note:** On Android 10+ (API 29+), `ACCESS_BACKGROUND_LOCATION` must be requested in a **separate** prompt *after* the user has already granted `ACCESS_FINE_LOCATION`. The system silently denies otherwise. On Android 14+ (API 34+), declare `FOREGROUND_SERVICE_LOCATION` if the engine runs inside a foreground service.

---

## Quick Integration — `ZoneNetworkManager`

`ZoneNetworkManager` is the only public entry point. It owns the engine and serialises all GPS / network work on an internal background executor — the public API is safe to call from the main thread.

### 1. Create and configure

```kotlin
import map.zone.speedalertsdk.ZoneNetworkManager
import map.zone.speedalertsdk.VehicleType

val zoneManager = ZoneNetworkManager(
    /* baseUrl     */ "https://driving.map.zone",     // endpoint
    /* apiKeyId    */ "<your-api-key-id>",            // issued by Vietmap
    /* apiKey      */ "<your-api-key>",               // issued by Vietmap
    /* bundleId    */ packageName,                    // typically context.getPackageName()
    /* vehicleId   */ "<your-vehicle-id>",            // issued by Vietmap
    /* vehicleType */ VehicleType.TRUCK.value,        // see Vehicle Types table below
    /* seats       */ 4,
    /* weights     */ 3500.0                          // gross weight in kg
)
```

> **Note:** Treat `apiKey` as a credential. Do not log it, embed it in a public repo, or expose it in user-facing UI. The constructor throws `IllegalArgumentException` if `baseUrl` does not start with `https://`.

### 2. Register callbacks

All four callbacks fire on the main thread. Wire only the ones you need.

```kotlin
// (a) Engine ready — fires after each successful zone load.
zoneManager.setReadyCallback { isReady, linkCount, alertCount ->
    Log.d(TAG, "ready=$isReady, links=$linkCount, alerts=$alertCount")
}

// (b) Network result — surfaces server errors. errorCode == 0 means success.
zoneManager.setResultCallback { success, errorCode, errorMessage ->
    if (!success && errorCode != 0) {
        Log.w(TAG, "error code=$errorCode: $errorMessage")
    }
}

// (c) Per-tick bitmaps + first voice clip — the core driver of your UI.
zoneManager.setBitmapCallback { currentBmp, speedStatus,
                                nextBmp,    nextDistMeters,
                                cameraBmp,  cameraDistMeters,
                                tollBmp,    tollDistMeters,
                                voiceWav ->

    // Current speed-limit sign (null = matcher has no link under GPS).
    binding.ivSpeedSign.setImageBitmap(currentBmp)

    // Overspeed indicator: 0 = safe, 1 = approaching limit, 2 = over limit.
    binding.statusBar.setBackgroundColor(when (speedStatus) {
        1    -> Color.parseColor("#FF9800")    // orange
        2    -> Color.parseColor("#F44336")    // red
        else -> Color.parseColor("#4CAF50")    // green
    })

    // Upcoming sign / camera / toll. Hide the container when bitmap == null.
    bindMiniSign(binding.flNextSign,   binding.ivNext,   binding.tvNext,   nextBmp,   nextDistMeters)
    bindMiniSign(binding.flCameraSign, binding.ivCamera, binding.tvCamera, cameraBmp, cameraDistMeters)
    bindMiniSign(binding.flTollSign,   binding.ivToll,   binding.tvToll,   tollBmp,   tollDistMeters)

    // voiceWav is null when nothing is queued. See the voice note below.
}
```

> **Note (voice):** The Android SDK has a **built-in `MediaPlayer` queue** that plays voice cues automatically. You only need to register a `VoiceCallback` if you want to mix audio yourself, route it to a specific stream, or pause cues based on app state:
>
> ```kotlin
> zoneManager.setVoiceCallback { wavBytes ->
>     // Once a callback is set, the built-in player is disabled and the
>     // app becomes responsible for ALL voice playback (including the
>     // voiceWav delivered through onBitmap).
>     myCustomPlayer.enqueue(wavBytes)
> }
> // Pass null to restore default playback.
> ```
>
> WAV format: PCM 16-bit little-endian, mono, 22 050 Hz — ready for `MediaPlayer`, `AudioTrack`, or `ExoPlayer`.

### 3. Feed GPS updates

Call both APIs on every GPS frame. The engine throttles HTTP fetches internally — calling at 1 Hz when the vehicle hasn't moved costs nothing.

```kotlin
private val locationListener = LocationListener { loc ->
    val lat      = loc.latitude
    val lng      = loc.longitude
    val bearing  = if (loc.hasBearing()) loc.bearing.toDouble() else 0.0
    val speedKmh = if (loc.hasSpeed()) loc.speed * 3.6 else 0.0    // Location.speed is m/s
    val accuracy = loc.accuracy.toDouble()

    // (a) Refresh zone cache. Returns immediately; HTTP happens on a background queue.
    zoneManager.updateLocation(lat, lng, speedKmh, bearing)

    // (b) Drive the per-tick UI: bitmaps, speed status, voice cues.
    zoneManager.processGps(lat, lng, bearing, speedKmh, accuracy)
}

locationManager.requestLocationUpdates(
    LocationManager.GPS_PROVIDER, /* minTimeMs */ 1000L, /* minDistanceM */ 2f, locationListener
)
```

> **Note:** Pass *snapped-to-route* coordinates here if your app already runs map-matched navigation (e.g. VietmapNavigation `SnapToRoute`) — raw `Location` samples can be 10-20 m off the road centre even with good `accuracy`, which causes the matcher to pick the wrong link.

### 4. Reset

```kotlin
zoneManager.reset()    // clears loaded zone data; next updateLocation() refetches
```

Call this when the driver swaps `vehicleType` / `seats` / `weights`, or at the end of a navigation session if you want to free engine memory immediately.

---

## Vehicle Types

Use the `VehicleType` enum or pass the raw integer value.

| Enum | `value` | Description |
|---|---|---|
| `VehicleType.CAR` | 1 | Xe ô tô |
| `VehicleType.MOTORCYCLE` | 2 | Xe mô tô |
| `VehicleType.TRUCK` | 3 | Xe tải — supported for speed alerts |
| `VehicleType.COACH` | 4 | Xe khách |
| `VehicleType.BUS` | 5 | Xe bus |
| `VehicleType.TAXI` | 6 | Xe taxi |
| `VehicleType.BICYCLE` | 7 | Xe đạp |
| `VehicleType.PEDESTRIAN` | 8 | Người đi bộ |
| `VehicleType.EMERGENCY` | 9 | Xe ưu tiên |

```kotlin
val vt: Int = VehicleType.TRUCK.value          // 3
val parsed: VehicleType = VehicleType.fromValue(3)   // VehicleType.TRUCK
```

---

## Error Codes (`ResultCallback.onResult`)

| Code | Meaning |
|---|---|
| `0` | Success (zone loaded) |
| `1001` | Invalid parameter (check `vehicleId`, `seats`, `weights`) |
| `2003` | Unauthorized — `apiKeyId` / `apiKey` / `bundleId` mismatch |
| `3003` | Vehicle type not supported for this account |
| negative values | Local SDK failure (network unreachable, parse error) |

---

## Threading Model

```
 Main thread ──► updateLocation / processGps
                          │
                          ▼
              Internal single-thread ExecutorService
              (HTTP + matching + bitmap render)
                          │
                          ▼
              Main thread ◄── ReadyCallback / ResultCallback /
                              BitmapCallback / VoiceCallback
```

All public methods are safe to call from the Android main thread. Callbacks always arrive back on the main thread via the application's `Handler(Looper.getMainLooper())`, so you can update views directly without `runOnUiThread {}`.

---

## Troubleshooting

- **No bitmaps and `ReadyCallback` never fires:** check `ResultCallback` — a non-zero `errorCode` tells you whether it is an auth issue (2003), a parameter issue (1001), or a network failure (negative code).
- **`UnsatisfiedLinkError` for `libmapzone_native.so`:** your app's `packagingOptions` is stripping the SDK's native libs, or your `abiFilters` excludes the device ABI. Add the full set `armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64`.
- **Wrong sign shown on the wrong road:** pass map-matched coordinates instead of raw GPS (see step 3 note).
- **Voice doesn't stop when the app goes to background:** the default `MediaPlayer` queue keeps playing on the music stream. Either set a `VoiceCallback` and gate playback yourself, or call `zoneManager.setVoiceCallback(null)` after the lifecycle event you want to honour.
- **Crash on Android 14 background service start:** add `FOREGROUND_SERVICE_LOCATION` to your manifest and start the service with the matching `foregroundServiceType`.

---

## License

Proprietary — MapZone Global.
