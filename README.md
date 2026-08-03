# FleetFlow Android SDK

The FleetFlow Android SDK gives you a native Kotlin client for authentication, API requests, realtime messaging, and diagnostics in one package.

Use this SDK when you want to integrate FleetFlow features directly in your Android app without manually building OAuth flows, request signing, or websocket handling.

## Before you jump in

Have these ready:

- OAuth client ID
- Redirect URI configured for your Android app
- FleetFlow API key (when needed for your use case)

## Installation

The Android SDK is published to Maven Central. If your project does not already include Maven Central, add it like so:

```kotlin
repositories {
    google()
    mavenCentral()
}
```

Add the FleetFlow SDK to your app module:

```kotlin
dependencies {
    implementation("io.fleetflow:fleetflow-android-sdk:{VERSION}")
}
```

## Full documentation

Start with the official docs at the **Android SDK** tab:

- https://developer.fleetflow.io?sdk=android
