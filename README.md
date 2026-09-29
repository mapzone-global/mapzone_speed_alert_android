# MapZone Speed Alert SDK — Android

[![](https://jitpack.io/v/mapzone-global/mapzone_speed_alert_android.svg)](https://jitpack.io/#mapzone-global/mapzone_speed_alert_android)

This SDK is provided only for Vietmap MAPs API enterprise customers. Contact your Vietmap [Vietmap Solutions](https://zalo.me/3189066936017422854) Zalo OA if you are interested in becoming a customer.

**Zone-based speed alert — no route required.** You feed raw GPS frames; nothing
else. The native engine fetches the road network around the vehicle from the
MapZone service, map-matches every fix onto it, and hands you back speed-limit
signs, upcoming camera / toll previews, restriction signs, and voice cues. There
is no route to hand in and no reroute to handle: the driver can go anywhere and
the engine follows.

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

### 3. Requirements

| Setting | Value |
|---|---|
| `minSdk` | **21** or higher |
| `compileSdk` | 34 or higher |
| Java / Kotlin JVM target | **8** or higher (`sourceCompatibility` / `targetCompatibility` = `VERSION_1_8`) |
| Native ABIs shipped | `arm64-v8a`, `armeabi-v7a` (16 KB page size supported) — **no x86/x86_64** |
| R8 / ProGuard | Nothing to add — the AAR ships its own consumer rules |

> **Emulators:** the AAR contains no x86_64 slice, so the standard Intel system
> images cannot load the engine. Test on a physical device or on an **arm64
> emulator image**. On an unsupported ABI the SDK stays inert and reports error
> code `-5` (see Error Codes).

## Permissions

### `AndroidManifest.xml`

The AAR declares exactly one permission — the engine only talks to the network:

```xml
<!-- Declared by the AAR and merged into your manifest -->
<uses-permission android:name="android.permission.INTERNET" />
```

Everything else belongs in **your** manifest, because your app is the one that
collects GPS: the SDK never touches the location APIs, it only consumes the
coordinates you hand to `updateLocation` / `processGps`.

```xml
<!-- Add to your own manifest — required -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

<!-- Only if you keep alerting while the app is in the background -->
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />

<!-- Only if you run your GPS collection inside a foreground service -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />

<!-- Only if that service posts a notification on Android 13+ (API 33) -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

> **⚠️ Upgrading from 2.0.1 — breaking change.** 2.0.1 declared the location,
> foreground-service and wake-lock permissions inside the AAR, so manifest
> merging handed them to your app whether you asked or not. 2.0.2 and later declare only
> `INTERNET`. If you were relying on that merge and never declared the location
> permissions yourself, your app loses them on upgrade and hits
> `SecurityException` on the first GPS read — check your own manifest before
> shipping.
>
> In exchange, `ACCESS_BACKGROUND_LOCATION` is no longer forced on you, so the
> `tools:node="remove"` workaround the 2.0.1 guide recommended is obsolete.
> Delete it if you have it, otherwise it will strip a permission you now declare
> on purpose.

> **Foreground service on Android 14+ (API 34+):** if you collect GPS from a
> foreground service, declare it with `android:foregroundServiceType="location"`
> and start it with the matching type. This is about your GPS collection, not
> the SDK — the engine has no service of its own.

### Runtime permission request (Android 6.0+)

```kotlin
private val permissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { results ->
    val granted = results.values.all { it }
    if (granted) startTracking() else showRationale()
}

permissionLauncher.launch(
    arrayOf(
        Manifest.permission.ACCESS_FINE_LOCATION,
        Manifest.permission.ACCESS_COARSE_LOCATION
    )
)
```

---

## Quick Integration — `ZoneNetworkManager`

`ZoneNetworkManager` is the only public entry point. It owns the native engine
and serialises all zone / GPS work on an internal single-thread executor — the
public API is safe to call from the main thread, and all callbacks arrive back
on the main thread.

### 1. Create and configure

Call `configure(...)` **before the first GPS frame**.

```java
import map.zone.speedalertsdk.ZoneNetworkManager;

ZoneNetworkManager manager = new ZoneNetworkManager();

manager.configure(
        /* baseUrl     */ "https://driving.map.zone",   // MapZone service base URL
        /* apiKeyId    */ "<your-api-key-id>",          // issued by Vietmap
        /* apiKey      */ "<your-api-key>",             // issued by Vietmap
        /* vehicleId   */ "<your-vehicle-id>",          // issued by Vietmap
        /* vehicleType */ 1,                            // 1..9 (see Vehicle Types)
        /* seats       */ 1,                            // seats — filters applicable alerts
        /* weights     */ 1.0);                         // gross weight in kg — filters applicable alerts
```

Everything that fails *after* configuration is reported through
`ResultCallback` as a numeric code, never as an exception.

> **Note:** Treat `apiKey` as a credential — do not log it, commit it to a public
> repo, or expose it in user-facing UI.
>
> **Bundle id / allow-listing:** you do **not** pass a bundle id. The SDK resolves
> your app's *installed* `applicationId` automatically (via an auto-installed
> `ContentProvider`) and sends it as the auth bundle id — **including any
> `applicationIdSuffix` or flavor suffix** (e.g. `com.foo.debug`). The auth server
> accepts a single exact application id and does **not** whitelist suffixed
> variants. Register the exact `applicationId` of every build you ship with the
> Vietmap team, or build those variants without a suffix.

**Optional configuration** (safe to call at any time, including mid-session):

| Method | Purpose |
|---|---|
| `setMutedAlertTypes(VoiceAlertType...)` | Mute specific alert voices (camera, toll, restriction signs, …). Speed-limit & speeding voices can never be muted. Call with no arguments to re-enable all. Persists across zone reloads. |
| `setMutedAlertTypes(Set<VoiceAlertType>)` | Same, taking a set. `null` or empty re-enables all. |
| `setLogFilePath(String)` | Append native engine logs to a file for field reports. **Debug builds only** — returns `false` and writes nothing on the release AAR. |
| `getLogFilePath()` / `flushLog()` | Read the current log path / flush before sharing the file. |

### 2. Register callbacks

All callbacks fire on the main thread. Wire only the ones you need.

```java
// (a) Per-tick speed signs — the core driver of your UI.
manager.setBitmapCallback((currentBmp, speedStatus,
                           nextBmp,    nextDistMeters,
                           cameraBmp,  cameraDistMeters,
                           tollBmp,    tollDistMeters,
                           voiceWav) -> {

    // Current speed-limit sign (null = no speed limit on the matched road).
    ivSpeedSign.setImageBitmap(currentBmp);

    // Overspeed indicator: 0 = within limit, 1 = within 5 km/h of it, 2 = over limit.
    statusBar.setBackgroundColor(
        speedStatus == 2 ? Color.parseColor("#F44336") :   // red
        speedStatus == 1 ? Color.parseColor("#FF9800") :   // orange
                           Color.parseColor("#4CAF50"));    // green

    // Upcoming sign / camera / toll previews. Hide the slot when the bitmap is null.
    bindMiniSign(ivNext,   nextBmp,   nextDistMeters);
    bindMiniSign(ivCamera, cameraBmp, cameraDistMeters);
    bindMiniSign(ivToll,   tollBmp,   tollDistMeters);

    // voiceWav: legacy parameter, always null in this version — voice is
    // delivered through the built-in player / VoiceCallback instead.
});

// (b) Zone loaded — fires after each successful fetch of the surrounding network.
manager.setReadyCallback((isReady, linkCount, alertCount) -> {
    statusLabel.setText("ready: " + linkCount + " links, " + alertCount + " alerts");
});

// (c) Fetch outcome — surfaces server and local errors. errorCode == 0 means success.
manager.setResultCallback((success, errorCode, errorMessage) -> {
    if (!success && errorCode != 0) {
        Log.w(TAG, "zone load failed code=" + errorCode + ": " + errorMessage);
    }
});
```

> **Important:** `onBitmap` fires only on GPS frames the matcher could place on a
> road. When the vehicle is off-network (parking garage, brand-new road, GPS
> spike) no callback arrives at all — your last sign stays on screen. If that
> matters for your UI, clear the slots yourself after N seconds without a
> callback.

**Restriction signs** are delivered through three independent callbacks, because
they answer different questions and genuinely co-occur — a street inside a
built-up area can also ban stopping:

```
  access │ đường cấm, cấm loại xe  │ may I drive here at all?
  stop   │ cấm dừng, cấm đỗ        │ may I stop here?
  bua    │ khu dân cư              │ what speed regime applies?
```

Give each its own display slot; one firing says nothing about the other two.

```java
manager.setAccessRestrictionCallback((bmp, kind, distMeters) -> {
    // kind: 0 = none (hide the slot), 3 = đường cấm, 4 = cấm loại xe
    bindSignSlot(accessSlot, bmp, kind, distMeters);
});

manager.setStopRestrictionCallback((bmp, kind, distMeters) -> {
    // kind: 0 = none, 1 = cấm đỗ, 2 = cấm dừng
    // Only raised while the driver is actually slowing down or stopped.
    bindSignSlot(stopSlot, bmp, kind, distMeters);
});

manager.setBuildupAreaCallback((bmp, kind, distMeters) -> {
    // kind: 0 = none, 5 = khu dân cư, 6 = hết khu dân cư
    // Artwork is a landscape rectangle, not a circle — size the slot for it.
    bindSignSlot(buaSlot, bmp, kind, distMeters);
});
```

Each group fires **only when it changes** — not on every GPS frame. `distMeters == 0`
means "you are on it now" (hide the distance label); `> 0` means it starts that
far ahead. A `null` bitmap with a non-zero `kind` means the artwork for that sign
is not embedded yet — hide the slot, never treat it as an error.

> **Note (voice):** The Android SDK has a **built-in, priority-aware
> `MediaPlayer` queue** that plays voice cues automatically. You do not have to
> wire anything.
>
> Register a `VoiceCallback` only if you want to mix audio yourself, route it to
> a specific stream, or apply your own prioritisation:
>
> ```java
> manager.setVoiceCallback(new VoiceCallback() {
>     // Richer variant — the SDK always calls this one.
>     @Override public void onVoice(byte[] wavBytes, int trigger, int priority) {
>         // priority: 0 = "hiện tại" (lowest, skipped first when congested)
>         //           1 = normal (approaching / camera / toll / sign)
>         //           2 = speeding (most urgent)
>         // trigger:  native VoiceTrigger value (see Voice Alert Types)
>         myPlayer.enqueue(wavBytes, priority);
>     }
>     // Legacy single-arg variant — still part of the interface; the two-arg
>     // default forwards here, so existing code keeps working.
>     @Override public void onVoice(byte[] wavBytes) { myPlayer.enqueue(wavBytes, 1); }
> });
> // Once a callback is set, the built-in player is disabled and your app becomes
> // responsible for ALL voice playback. Pass null to restore the default player.
> ```
>
> WAV format: PCM 16-bit little-endian, mono, 22 050 Hz — ready for
> `MediaPlayer`, `AudioTrack`, or `ExoPlayer`.

### 3. Feed GPS

Every GPS frame goes through **two** calls, in this order:

```
  Location fix
       │
       ├─► updateLocation(...)   keeps the loaded road network around the vehicle
       │                          (fetches from the server only when needed)
       │
       └─► processGps(...)        matches the fix, renders signs, fires voice
```

```kotlin
private val locationListener = LocationListener { loc ->
    val speedKmh = if (loc.hasSpeed()) loc.speed * 3.6 else 0.0   // Location.speed is m/s
    val hasBearing = loc.hasBearing()
    val bearing = if (hasBearing) loc.bearing.toDouble() else 0.0

    // 1) Keep the zone around the vehicle fresh.
    //    Double.NaN here means "heading unknown" — the engine then skips the
    //    forward projection instead of aiming the request at a made-up heading.
    manager.updateLocation(loc.latitude, loc.longitude, speedKmh,
        if (hasBearing) bearing else Double.NaN)

    // 2) Drive the per-tick UI.
    manager.processGps(
        loc.latitude,
        loc.longitude,
        bearing,                  // degrees, 0 = N, 90 = E — never NaN
        speedKmh,                 // km/h — the SDK converts to m/s internally
        loc.accuracy.toDouble(),  // horizontal accuracy in metres
        loc.time)                 // fix timestamp, ms since epoch (UTC)
}
```
### 4. Reset

```kotlin
manager.reset()    // clears loaded zone data + engine state
```

Call this when the vehicle profile changes (driver swaps `vehicleType` / `seats`
/ `weights` — re-`configure` afterwards) or when the session ends and you want
the memory back. The next `updateLocation` triggers a fresh fetch.

---

## Voice Alert Types (mute control)

Pass the categories you want to **mute** to `setMutedAlertTypes(...)`; everything
not listed keeps announcing. Speed-limit and speeding warnings are core safety
cues and can never be muted — they have no entry in the enum.

Muting a **camera or toll** category also hides its on-screen icon. The other
categories currently have no icon, so muting them only silences the voice.

```java
import map.zone.speedalertsdk.VoiceAlertType;

// Mute camera + toll voices; keep every other voice and all visual signs.
manager.setMutedAlertTypes(VoiceAlertType.SPEED_CAMERA, VoiceAlertType.TOLL);

// Re-enable all voices:
manager.setMutedAlertTypes();
```

| `VoiceAlertType` | Trigger | Meaning |
|---|---|---|
| `SPEED_CAMERA` | 3 | Speed camera |
| `TOLL` | 4 | Toll booth |
| `TRAFFIC_ENFORCEMENT_CAMERA` | 6 | Traffic-enforcement camera (phạt nguội) |
| `RED_LIGHT_CAMERA` | 7 | Red-light / surveillance camera |
| `AI_CAMERA` | 8 | AI camera |
| `NO_LEFT_TURN` / `NO_RIGHT_TURN` / `NO_UTURN` / `NO_STRAIGHT` | 9 / 10 / 11 / 15 | Turn restrictions |
| `NO_OVERTAKING` / `NO_OVERTAKING_END` | 12 / 13 | No-overtaking zone start / end |
| `NO_PARKING` | 14 | Cấm đỗ |
| `BUILDUP_AREA_START` / `BUILDUP_AREA_END` | 16 / 17 | Built-up area (khu dân cư) start / end |
| `REST_STATION` | 18 | Rest station |
| `NO_STOPPING` | 19 | Cấm dừng |
| `ROAD_CLOSED` | 20 | Đường cấm |
| `VEHICLE_RESTRICTED` | 21 | Cấm loại xe đang cấu hình |

---

## Vehicle Types

`configure(...)` takes a raw integer `vehicleType` in **1..9**. The
`VehicleType` enum in the SDK carries the same values plus display names.

| `vehicleType` | Description |
|---|---|
| `1` | Car (xe ô tô) |
| `2` | Motorcycle (xe mô tô) |
| `3` | Truck (xe tải) |
| `4` | Coach (xe khách) |
| `5` | Bus (xe bus) |
| `6` | Taxi (xe taxi) |
| `7` | Bicycle (xe đạp) |
| `8` | Pedestrian (người đi bộ) |
| `9` | Emergency (xe ưu tiên) |

`vehicleType` also selects which "cấm loại xe" artwork the access-restriction
callback renders.

---

## Error Codes (`ResultCallback.onResult`)

`errorMessage` is a short, display-ready English sentence derived from
`errorCode` **alone** — the server's own wording never reaches your app.
Localise off the numeric code if you need another language, and quote the number
in bug reports.

| Code | Meaning |
|---|---|
| `0` | Success — zone data loaded |
| `1001` | This location is outside the supported area |
| `1000`–`1999` | The request was invalid |
| `2000`–`2999` | Session expired / unauthorized — `apiKeyId`, `apiKey`, or application id mismatch |
| `3003` | Vehicle type not supported (must be `1..9`) |
| `3004` | No map data found for this area |
| `3000`–`3999` | Requested data not found |
| `5000`–`5999` | Server temporarily unavailable |
| `-1` | Response could not be verified |
| `-2` | Map data empty, malformed, or unusable |
| `-3` | Secure session could not be established |
| `-4` | Server unreachable — check connectivity |
| `-5` | Native bridge failed — library missing for this device's ABI, a bridge call threw, or JNI could not marshal the result back to Java. Reported **once per outage**, not on every GPS tick. |
| `-99` | Legacy spelling of `-5`, emitted by older engine builds. Same meaning, same message. |

---

## Demo

Check the demo app on [GitHub](https://github.com/mapzone-global/mapzone-speed-alert-app-android).
