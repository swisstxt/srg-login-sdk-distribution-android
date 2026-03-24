# SRG Login SDK — Android Distribution

![SDK Version](https://img.shields.io/badge/dynamic/xml?url=https%3A%2F%2Fswisstxt.github.io%2Fsrg-login-sdk-distribution-android%2Fch%2Fsrg%2Flogin%2Fsrglogin-core-android%2Fmaven-metadata.xml&query=%2F%2Flatest&label=SDK%20Version&color=blue)

## Integration

### Gradle (settings.gradle.kts)

Add the Maven repository — no authentication needed:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://swisstxt.github.io/srg-login-sdk-distribution-android/") }
    }
}
```

### Dependency (build.gradle.kts)

```kotlin
dependencies {
    implementation("ch.srg.login:srglogin-core-android:<VERSION>")
}
```

Replace `<VERSION>` with the desired version (e.g., `1.0.0-beta.9`, `1.0.0`).

---

## Quickstart

### Requirements

- Android SDK 21+ (minSdk)
- JDK 17+
- Gradle 8.11+

### Initialize the SDK

Call `SrgLoginSdk.initialize()` once in your `Application.onCreate()`:

```kotlin
import ch.srg.login.sdk.SrgLoginSdk

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // Minimal — uses default token storage config
        SrgLoginSdk.initialize(
            platformContext = this,
            isDebugBuild = BuildConfig.DEBUG
        )

        // Or customise token storage keys to avoid collisions with other SDK instances
        // SrgLoginSdk.initialize(
        //     platformContext = this,
        //     isDebugBuild = BuildConfig.DEBUG,
        //     tokenStorageConfig = TokenStorageConfig(
        //         keystoreAlias = "your_app_token_key",
        //         fileName = "your_app_tokens"
        //     )
        // )
    }
}
```

### Create an SrgLogin instance

Use `environment` to select the OIDC endpoints: `Environment.DEV`, `Environment.INT`, or `Environment.PROD`.

#### AppIdentity

> **`appId`, `appName`, and `appVersion` must be resolved dynamically at runtime — never hardcode these values.**

`businessUnit` and `businessUnitName` must use exact values from the table below. These values are used as Sentry filters to categorize and route error reports — incorrect or custom values will break Sentry dashboards and alerting.

| `businessUnit` | `businessUnitName` |
|---|---|
| `"SRF"` | `"Schweizer Radio und Fernsehen"` |
| `"RTS"` | `"Radio Télévision Suisse"` |
| `"RSI"` | `"Radiotelevisione svizzera di lingua italiana"` |
| `"RTR"` | `"Radiotelevisiun Svizra Rumantscha"` |
| `"SWI"` | `"SWI swissinfo.ch"` |
| `"SWISSTXT"` | `"SWISS TXT"` |
| `"SRG"` | `"SRG SSR"` |

```kotlin
import ch.srg.login.sdk.SrgLoginSdk
import ch.srg.login.sdk.config.SrgLoginConfig
import ch.srg.login.sdk.config.AppIdentity
import ch.srg.login.sdk.config.Environment

val appVersion = packageManager.getPackageInfo(packageName, 0).versionName ?: "unknown"

val config = SrgLoginConfig(
    clientId = "your-oauth-client-id",
    redirectUri = "your-app-scheme://callback",
    postLogoutRedirectUri = "your-app-scheme://logout",
    appIdentity = AppIdentity(
        appId = packageName,
        appName = applicationInfo.loadLabel(packageManager).toString(),
        appVersion = appVersion,
        businessUnit = "SRF",
        businessUnitName = "Schweizer Radio und Fernsehen"
    ),
    environment = Environment.INT
)

val srgLogin = SrgLoginSdk.create(config)
```

> **Tip**: Store the `srgLogin` instance in a ViewModel, singleton, or DI container. Avoid recreating it on every screen.

### Login

```kotlin
import ch.srg.login.sdk.auth.AndroidAuthContext
import ch.srg.login.sdk.auth.Credentials
import ch.srg.login.sdk.auth.LoginState

val authContext = AndroidAuthContext(context = this, activity = this)

srgLogin.login(
    credentials = Credentials.Web,
    authContext = authContext,
    additionalScopes = listOf("offline_access")
).collect { state ->
    when (state) {
        is LoginState.Success -> {
            val token = state.tokenSet.accessToken.value
            // User is authenticated
        }
        is LoginState.Failure -> {
            handleError(state.error)
        }
        else -> { /* intermediate states */ }
    }
}
```

### Handle redirect

In your Activity, add `launchMode="singleTask"` and handle the OAuth callback:

```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".MainActivity"
    android:launchMode="singleTask"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="your-app-scheme" android:host="callback" />
    </intent-filter>
</activity>
```

```kotlin
import ch.srg.login.sdk.handleRedirect

override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    handleRedirect(intent)
}
```

> **Note**: `handleRedirect` is a top-level function — not a method on `SrgLoginSdk`.

### Observe token state

```kotlin
srgLogin.observeTokenState().collect { state ->
    when (state) {
        TokenState.Valid -> { /* normal operation */ }
        TokenState.ExpiringSoon -> { /* optional: warn user */ }
        TokenState.Refreshing -> { /* show spinner */ }
        is TokenState.Refreshed -> { /* new token available */ }
        TokenState.Expired -> { /* wait for refresh or re-auth */ }
        is TokenState.RefreshFailed -> { /* handle error */ }
        TokenState.NoTokens -> { /* show login screen */ }
    }
}
```

### Get access token

```kotlin
// Check if the user is currently authenticated
val isAuthenticated = srgLogin.isAuthenticated()

// Get the current access token (always fetch at call time — the SDK refreshes it automatically)
when (val result = srgLogin.getAccessToken()) {
    is SdkResult.Success -> {
        val authHeader = "Bearer ${result.data.value}"
        // Use authHeader for your API calls
    }
    is SdkResult.Failure -> {
        // e.g. SrgLoginError.NotAuthenticated if the user is not logged in
        handleError(result.error)
    }
}
```

> Always call `getAccessToken()` at the point of use rather than caching the value. The SDK transparently refreshes the token when it is near expiry.

### Logout

**Front-channel logout** — clears the server session and local tokens:

```kotlin
val authContext = AndroidAuthContext(context = this, activity = this)
srgLogin.logout(
    logoutType = LogoutType.FrontChannel(postLogoutRedirectUri = "your-app-scheme://logout"),
    authContext = authContext
)
```

**Local-only logout** — clears local tokens only:

```kotlin
srgLogin.logout(logoutType = LogoutType.LocalOnly)
```

### SSO — open a URL with the user's active session

`openSsoClient` opens any URL in a Chrome Custom Tab with the user's IDP session cookie already injected. The page loads without prompting for credentials — the user is silently authenticated via SSO.

```kotlin
val authContext = AndroidAuthContext(context = this, activity = this)

srgLogin.openSsoClient(
    ssoClientUrl = "https://settings.srgssr.ch/profile",
    authContext = authContext
)
```

### Reconfiguration

To switch environments or update the OAuth config at runtime:

```kotlin
SrgLoginSdk.shutdown()
// Then call initialize(...) and create(config) again with the new config
```

> Always cancel any active `observeTokenState()` collection before calling `shutdown()`.

---

## Sample app

The [srg-login-sdk-sample-android](https://github.com/swisstxt/srg-login-sdk-sample-android) repository contains a fully working reference app demonstrating all SDK features.

---

## Releases

All releases are served via GitHub Pages from the `gh-pages` branch of this repository. Each version is immutable once published.

---

## Questions

Teams: [SRG-Login SDK](https://teams.microsoft.com/l/channel/19%3A77f7201d0aa44ccfb8f2e856c47c7cae%40thread.skype/SRG-Login%20SDK?groupId=4c35f1b2-811c-40ef-aa8e-01eede3277f2&tenantId=2598639a-d083-492d-bdbe-f1dd8066b03a)

For SDK developers and contributors, see the [main repository](https://github.com/swisstxt/srg-login-mobile-sdk).
