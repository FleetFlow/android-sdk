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

## Custom authentication URL

Pass the optional `authenticationURL` origin to use a verified FleetFlow custom authentication domain. Use an HTTP(S) origin without a path, query, fragment, or credentials; a trailing slash is accepted. Omit it (or pass `nil`/`null`) to keep the default authentication host.

```kotlin
FleetFlow.shared(context).configure(
    baseURL = "fleetflow.io",
    clientID = "your-client-id",
    redirectURI = "your-app://auth-callback",
    authenticationURL = "https://auth.example.com"
)
```

The SDK saves this setting across launches and uses it for hosted login, native authentication, token exchange, and token refresh. API hosts and the OAuth issuer/audience remain derived from `baseURL`, so changing the authentication origin preserves existing tokens. For native iOS passkeys, include the authentication host in the app's `webcredentials` associated domains; the server must also support that host's passkey origin.

## Authentication

Use the activity-based overload for FleetFlow's hosted OAuth experience:

```kotlin
FleetFlow.shared(context).login(activity)
```

To open the hosted flow with Google or Apple already selected, use the
provider-specific overload. The provider exchange and verified-email account
linking remain on FleetFlow's authentication service:

```kotlin
val fleetFlow = FleetFlow.shared(context)
fleetFlow.login(activity, SocialLoginProvider.GOOGLE)
fleetFlow.login(activity, SocialLoginProvider.APPLE)
```

The hosted flow runs in a regular Custom Tab, so provider SSO and the hosted
page's **Last used** indicator can carry across login attempts.

Apps can also present a native email-code UI. The SDK completes OAuth with
authorization code + PKCE and securely stores the resulting session:

```kotlin
val fleetFlow = FleetFlow.shared(context)
fleetFlow.sendLoginCode(email)
fleetFlow.login(email, oneTimeCode = code)
```

The native flow uses the authentication methods and organization boundary of
the configured OAuth client. It does not expose or store a password in the app.

## Full documentation

Start with the official docs at the **Android SDK** tab:

- https://account.fleetflow.io/developer/docs?sdk=android
