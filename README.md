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

The SDK saves this setting across launches and uses it for hosted login, native authentication, token exchange, and token refresh. API hosts and the OAuth issuer/audience remain derived from `baseURL`, so changing the authentication origin preserves existing tokens.

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
page's **Last used** indicator can carry across login attempts. When the
authorization server or provider returns an OAuth error (for example
`access_denied`), `login` throws an `APIError` whose message is the
`error_description` (or the `error` code when no description is sent).

Apps can also present a native email-code UI. The SDK completes OAuth with
authorization code + PKCE and securely stores the resulting session:

```kotlin
val fleetFlow = FleetFlow.shared(context)
fleetFlow.sendLoginCode(email)
fleetFlow.login(email, oneTimeCode = code)
```

The native flow uses the authentication methods and organization boundary of
the configured OAuth client. It does not expose or store a password in the app.

Check which in-app methods an email can use before showing UI:

```kotlin
val methods = fleetFlow.loginMethods(email)
// methods.email, methods.passkey, methods.passkeyRegistration
```

### Provider-token sign-in (Google, Apple)

Obtain the provider's ID token in your app (for example Google via Credential
Manager's `GetGoogleIdOption` / Sign in with Google) and hand it to the SDK. FleetFlow
verifies the token server side and completes OAuth (authorization code + PKCE) in app.
If you requested the token with a nonce, pass the same nonce. The SDK itself does not
depend on Google Play Services for this.

```kotlin
fleetFlow.login(
    SocialLoginProvider.GOOGLE,
    idToken = googleIdTokenCredential.idToken,
    nonce = nonce,               // optional
    firstName = googleIdTokenCredential.givenName,
    lastName = googleIdTokenCredential.familyName,
    email = googleIdTokenCredential.id
)
```

### Passkeys

Passkeys use Android Credential Manager (`androidx.credentials`, with the Play
services provider for Android 13 and lower). Sign in with an existing passkey:

```kotlin
fleetFlow.loginWithPasskey(activity, email)
```

Offer passkey creation right after an email-code sign-in, while the short-lived
session from `login(email, oneTimeCode)` is still held, and release it when the user
finishes or skips:

```kotlin
fleetFlow.login(email, oneTimeCode = code)
if (fleetFlow.loginMethods(email).passkeyRegistration) {
    runCatching { fleetFlow.createPasskey(activity, name = "My phone") } // completes setup on success
}
fleetFlow.finishNativePasskeySetup() // safe to call when already finished
```

If the user dismisses the passkey sheet, these calls throw an `APIError` with the
message `passkey_cancelled` (`PASSKEY_CANCELLED_ERROR`).

Passkey requirements:

- The authentication host (the relying party, e.g. `auth.fleetflow.io` or your
  `authenticationURL` host) must serve `/.well-known/assetlinks.json` listing your app's
  package name and signing certificate fingerprint with the
  `delegate_permission/common.handle_all_urls` and
  `delegate_permission/common.get_login_creds` relations.
- The FleetFlow authentication server must accept your app's
  `android:apk-key-hash:<base64url SHA-256 of the signing certificate>` WebAuthn origin.

## API keys and organization context

The `apiKey` passed to `configure` is the default. After `selectOrganization(...)`
or `selectPartner(...)`, requests use that context's API key, also after a relaunch
and a later `configure` call. `clearAPIContext()` (and logout) switch back to the
configured key.

GET requests that pass `email` and `password` send them only as an HTTP Basic
`Authorization` header, never in the URL.

## Realtime and thread WebSockets

The realtime, customer threads and organization thread sockets reconnect
automatically with exponential backoff (realtime from 1s, threads from 2s, doubling
up to 60s with jitter). The delay resets after a successful connection or a manual
disconnect. `onRealtimeWebSocketDisconnect` fires for both dropped connections and
`disconnectRealtimeWebSocket()`. `connectThreadsWebSocket` and
`connectToOrganizationThreadWebSocket` call exactly one of `onConnected` / `onFailure`
per connect request, for its initial outcome: `onConnected` when the socket first
opens, `onFailure` when it can't be opened (`api_key_not_configured`,
`not_authenticated` or a connection error). Reconnects don't call them again. Logout
closes all sockets.

## Session storage and backups

The SDK stores session tokens only in encrypted storage (`FleetFlow.secure.xml`,
backed by an Android Keystore key). If that storage is temporarily unreadable the
SDK keeps the session and retries before the next request instead of signing out.

Keystore keys are never included in backups, so a restored `FleetFlow.secure.xml`
can't be decrypted; the SDK then discards it and the user signs in again. Exclude
the file from backups in your app to avoid that. With `android:fullBackupContent`
(Android 11 and lower):

```xml
<full-backup-content>
    <exclude domain="sharedpref" path="FleetFlow.secure.xml" />
</full-backup-content>
```

With `android:dataExtractionRules` (Android 12+), add the same `<exclude>` under
both `<cloud-backup>` and `<device-transfer>`.

## Parity

The iOS and Android SDKs implement the same behavior as of 2.0.0. Only the API
shapes differ where each platform has its own idiom.

Behavior that is identical on both platforms includes:

- **Errors:** `APIError.message` is a stable snake_case code for SDK-generated
  errors (for example `invalid_response`, `invalid_token`, `token_exchange_failed`,
  `api_key_not_configured`, `not_authenticated`, `threads_websocket_not_connected`,
  `organization_thread_websocket_not_connected`), or the server's message for API
  errors. `request` and multipart uploads map responses the same way: a non-JSON
  error page is `invalid_response` with its HTTP status, a final 498 is
  `invalid_token` with `httpCode` 498. Native sign-in validation errors
  (for example "Enter a valid email address.") are user-facing text on both platforms.
- **Profile and vehicles:** customer users load `account` and their vehicles from
  customer-api in the API version their token was issued for (v5 today); organization
  users load `account` and the organization's vehicles from organization-api v1.
  Without a stored token API and version the profile is cleared and nothing is loaded.
  `loadFromCache()` publishes the cached profile, vehicles and selection without a
  network request. The previous selection is kept when it still
  exists, otherwise the first vehicle (sorted by UUID) is selected. The selected
  vehicle loads its details and `products`; a failing products request keeps the
  vehicle with an empty `products` list, a failing vehicle request clears it.
- **Thread sockets:** each connect request reports its initial outcome exactly
  once (opened, or failed to open). Automatic reconnects (exponential backoff from
  2s up to 60s) don't report again; messages keep arriving through `onMessage`, and
  sending while disconnected throws `*_websocket_not_connected`. A new connect
  request replaces the previous one and drops its pending callback.
- **Diagnostics:** the SDK's own app-session starts (enabling diagnostics,
  returning to the foreground, sign-in) are best effort: a failure is logged, never
  fails that flow, and no app session is stored.

Platform-idiomatic differences:

| | iOS | Android |
|---|---|---|
| Async model | `async throws`, `@Published` properties | `suspend` functions, `StateFlow`s |
| Hosted login | Native Apple authentication sheet (`ASWebAuthenticationSession`); Sign in with Apple uses the system sheet | Custom Tabs; forward the redirect with `FleetFlow.handleOAuthRedirect(uri)` |
| Token storage | Keychain (`kSecAttrAccessibleAfterFirstUnlock`) | `EncryptedSharedPreferences` backed by the Android Keystore |
| Push token | `setAPNSToken` | `setFCMToken` |
| Thread socket callbacks | `completion: (Result<Void, Error>)` | `onConnected` / `onFailure` |
| Customer threads socket | `connectToThreadsWebSocket` / `disconnectFromThreadsWebSocket` | `connectThreadsWebSocket` / `disconnectThreadsWebSocket` |
| `startAppSession` | `async throws` (direct callers see the error) | `suspend`, swallows the error (no crash in a background scope) |
| API builders | Dynamic members for any API (`customer("v5").vehicles(uuid).get()`) | `customer`, `organization`, `partner`, `admin`, `orchestrator` with `path(...)` |
| Error text | `APIError.message` (also `errorDescription`) | `APIError.message` |

Android also has `connectVehicleWebSocket` / `disconnectVehicleWebSocket` /
`writeVehicleStream` aliases for the realtime socket. iOS also has
`loginWithApple()` as a shortcut for `login(with: .apple)`.

## Full documentation

Start with the official docs at the **Android SDK** tab:

- https://account.fleetflow.io/developer/docs?sdk=android
