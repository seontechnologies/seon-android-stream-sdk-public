# SEON Stream SDK - Android
The SEON Stream SDK continuously collects behavioural signals from your Android application — touch patterns, typing dynamics, sensor data, network state and screen flows — and streams them to the SEON platform for fraud detection and risk assessment. The SDK supports both Jetpack Compose and legacy Android Views (Activities, Fragments, XML layouts).

---

## Requirements

- Android 7.0 or higher (API level 24)
- SEON Stream SDK 1.1.0 or higher (minimum supported version)
- `INTERNET` permission
- _(optional)_ `ACCESS_NETWORK_STATE` permission for IP address change detection, proxy, and VPN detection
- _(optional)_ `READ_PHONE_STATE` permission for active call tracking

> **Note:** `INTERNET` and `ACCESS_NETWORK_STATE` are declared in the SDK's `AndroidManifest.xml` and will be merged into your app automatically. If you do not want `ACCESS_NETWORK_STATE`, remove it in your app's manifest with the `tools:node="remove"` attribute:
>
> ```xml
> <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" tools:node="remove" />
> ```

>
> `READ_PHONE_STATE` is **not** declared by the SDK. If you want active call tracking by permission, add it to your app's manifest yourself:
>
> ```xml
> <uses-permission android:name="android.permission.READ_PHONE_STATE" />
> ```

> If an **optional** permission is not granted, not declared, or removed, the corresponding data points will simply be omitted. We recommend enabling as many of these permissions as your use-case allows to maximize detection accuracy.

Integrations authenticate with an **authData** token obtained from the SEON authentication API (contact your SEON account representative for access).
---
## Getting started

### Installation

Add the dependency to your module-level `build.gradle` file.

```groovy
// Groovy DSL
dependencies {
    implementation('io.seon.streamsdk:streamsdk:1.2.0') {
        transitive = true
    }
}
```

```kotlin
// Kotlin DSL
dependencies {
    implementation("io.seon.streamsdk:streamsdk:1.2.0") {
        isTransitive = true
    }
}
```

---

### Initialization

Starting from SDK version 1.2.0, the SDK initializes itself automatically via `SeonStreamInitProvider`, a `ContentProvider` that runs before `Application.onCreate()`. No manual `initialize()` call is required.

If you need to control when initialization happens — for example, to conditionally initialize based on user consent — disable the auto-init provider in your app's manifest and call `SeonStream.initialize()` manually:

```xml
<provider
    android:name="io.seon.streamsdk.SeonStreamInitProvider"
    tools:node="remove" />
```

Then call `initialize()` yourself, typically in `Application.onCreate()` or your main `Activity.onCreate()`:

```java
// Java

SeonStream.initialize(this);
```

```kotlin
// Kotlin

SeonStream.initialize(this)
```

## Starting and stopping a session

#### Session Configuration

Use `SeonStreamSessionConfig.Builder` to control per-session behavior.

| Method | Description | Default |
|---|---|---|
| `authData(String)` | This is **required** to start a stream, can be gathered through the Seon's authentication endpoint. If this data becomes invalid for some reason, it has to be renewed manually (see [Authentication](#authentication)). |  |
| `withMaxBackgroundDuration(long)` | Maximum time (in ms) the session stays alive while the app is in the background. Must be between `0` and `48 hours`. | `0` |
| `withLabel(String)` | An optional label for the session (max 32 characters). | `null` |

#### Starting a Session
```java
// Java
String authData = fetchTokenFromBackend() // This is just an example

SeonStreamSessionConfig sessionConfig = new SeonStreamSessionConfig.Builder()
        .withAuthData(authData) //required
        .withMaxBackgroundDuration(60000) //optional
        .withLabel("checkout-flow") //optional
        .build();

SeonStream.getInstance().startStream(sessionConfig);
```

```kotlin
// Kotlin
val authData = fetchTokenFromBackend() // This is just an example
val sessionConfig = SeonStreamSessionConfig.Builder()
    .withAuthData(authData) //required
    .withMaxBackgroundDuration(60000) //optional
    .withLabel("checkout-flow") //optional
    .build()

SeonStream.getInstance().startStream(sessionConfig)
```

#### Stopping a Session

```java
// Java
SeonStream.getInstance().finishStream();
```

```kotlin
// Kotlin
SeonStream.getInstance().finishStream()
```

#### Custom Events

Send named custom events with optional additional data (name max 32 chars, data max 1024 chars):

```java
// Java
SeonStream.getInstance().createCustomEvent("purchase_attempt", "{\"amount\":99.99}");
```

```kotlin
// Kotlin
SeonStream.getInstance().createCustomEvent("purchase_attempt", "{\"amount\":99.99}")
```

#### Checking Session Status

```java
// Java
boolean running = SeonStream.getInstance().getIsRunning();
```

```kotlin
// Kotlin
val running = SeonStream.getInstance().isRunning
```

### Error Handling

All SDK exceptions extend `SeonStreamException` (a `RuntimeException`). Errors are delivered through two channels depending on when they occur:

- **Synchronous** — Initialization and configuration errors (e.g. `initialize()`, tagging, listener management) are thrown directly and can be caught with a `try-catch` block.
- **Asynchronous** — Errors that occur during an active session (e.g. starting, stopping, runtime failures) are delivered through the `SeonStreamListener.onFailure()` callback. See [Listening to Session Events](#listening-to-session-events) for details.

| Exception                          | Thrown when                                                                                                                                                                                                                                                                                                                                                   | Delivery |
|------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---|
| `SeonStreamInitializeException`    | `initialize()` is called with invalid arguments or is called more than once, or `getInstance()` is called before initialization, or SDK initialization otherwise fails.                                                                                                                                                                                       | Synchronous (try-catch) |
| `SeonStreamStartException`         | `startSessionMonitoring()` fails — e.g. no token set, session already running, invalid config, stream ID could not be generated, or an internal error.                                                                                                                                                                                                        | Async (`onStreamFinishedWithError`) |
| `SeonStreamStopException`          | `finishSessionMonitoring()` fails — e.g. no session is currently running, or an internal error.                                                                                                                                                                                                                                                               | Async (`onStreamFinishedWithError`) |
| `SeonStreamConfigurationException` | `SeonStreamSessionConfig.Builder.build()` is called with invalid values (invalid authData, negative duration, duration exceeding 48 hours, or label longer than 32 characters), or `tagActivity()`, `tagFragment()`, `tagViewElement()`, `addListener()`, or `removeListener()` fails. | Synchronous (try-catch) |
| `SeonStreamCustomEventException`   | `createCustomEvent()` fails — e.g. name exceeds 32 characters, additional data exceeds 1024 characters, or an internal error.                                                                                                                                                                                                                                 | Async (`onStreamFinishedWithError`) |
| `SeonStreamTimeoutException`       | The backend informs the client that it reached its session duration's hard limit                                                                                                                                                                                                                                                                              | Async (`onStreamFinishedWithError`) |
| `SeonStreamStorageException`       | When there is a serious storage related issue, e.g. the storage is full.                                                                                                                                                                                                                                                                              | Async (`onStreamFinishedWithError`) |

#### Listening to Session Events

Implement `SeonStreamListener` to receive callbacks on the main thread:

```java
// Java
SeonStreamListener listener = new SeonStreamListener() {
    @Override
    public void onStreamStarted(String streamId) {
        // Session is active — streamId can be sent to your backend
    }

    @Override
    public void onStreamFinished() {
        // Session was stopped by finishStream()
    }

    @Override
    public void onStreamFinishedWithError(SeonStreamException exception) {
        // An error occurred
    }

    @Override
    public void onStreamAuthError() {
        // The authData expired and has to be renewed manually
    }
};

SeonStream.getInstance().addListener(listener);
```

```kotlin
// Kotlin
val listener = object : SeonStreamListener {
    override fun onStreamStarted(streamId: String) {
        // Session is active — streamId can be sent to your backend
    }

    override fun onStreamFinished() {
        // Session was stopped by finishStream()
    }

    override fun onStreamFinishedWithError(exception: SeonStreamException) {
        // An error occurred
    }

    override fun onStreamAuthError() {
        // An error occurred
    }
}

SeonStream.getInstance().addListener(listener)
```

Remove a listener when it is no longer needed:

```java
// Java
SeonStream.getInstance().removeListener(listener);
```

```kotlin
// Kotlin
SeonStream.getInstance().removeListener(listener)
```

---

## Authentication

> **Version note:** The `authData`-based authentication introduced in 1.1.0 is required for production integrations. Version 1.0.0 was an open preview release without this mechanism; **1.1.0 is the minimum supported version**.

Each stream session is authorized with an **authData** value from the SEON authentication API. It is short-lived; your app should fetch a new one before starting a session and again after an auth failure.

1. Your backend calls `POST {environment}/session-monitoring-api/v1/auth` with the header `X-API-KEY: <your api key>` and passes the response body to the app as the `authData` string. The API key must never be embedded in the mobile application.
2. Pass `authData` in `SeonStreamSessionConfig.Builder` when calling `startStream(sessionConfig)`.
3. The SDK extracts the session JWT and additional configuration from `authData` automatically.
4. If the JWT inside `authData` expires and the backend rejects further uploads, the SDK stops the stream and calls `onStreamAuthError` callback (see [Error handling](#error-handling)). The stored session is **not** cleared on auth failure, so you can fetch a fresh `authData` and call `startStream(sessionConfig)` again with the **same `label`** to continue the session, as long as you are still within `maxBackgroundDuration`.

---

## Tagging

Tag UI elements with human-readable names so tracked interactions are easier to interpret in the SEON dashboard.

### Tagging Activities, Fragments, and Views

```java
// Java
SeonStream.tagActivity(activity, "checkout-screen");
SeonStream.tagFragment(fragment, "payment-form");
SeonStream.tagViewElement(submitButton, "submit-btn");
```

```kotlin
// Kotlin
SeonStream.tagActivity(activity, "checkout-screen")
SeonStream.tagFragment(fragment, "payment-form")
SeonStream.tagViewElement(submitButton, "submit-btn")
```

### Compose Navigation Support

If your app uses Jetpack Compose navigation, pass the `NavHostController` so the SDK can track route changes:

```java
// Java
// obtain navController from your Compose host
SeonStream.getInstance().setComposeNavController(navController);
```

```kotlin
// Kotlin
val navController = rememberNavController()
LaunchedEffect(navController){
    SeonStream.getInstance().setComposeNavController(navController)
}
```
Call setComposeNavController(navController) as soon as the controller is available and before starting session monitoring.
> **Important**: Compose route tracking is not enabled automatically. 
> If you register the controller after calling startSessionMonitoring(), subsequent route changes will still be tracked, but the initial visible route may be missed.

### Form tagging

Use `SeonFormTag` to group related input fields into a named form so they appear together on the SEON admin panel. Create one `SeonFormTag` instance per form and apply it to each field that belongs to that form.

`SeonFormTag` takes three parameters:

| Parameter | Required | Description |
|---|---|---|
| `id` | Yes | Unique identifier for the form. Case-insensitive; whitespace is replaced by `_`. Forms are grouped by this id on the admin panel. |
| `numberOfElements` | Yes | Total number of fields that make up this form. |
| `displayName` | No | Human-readable name shown on the admin panel. Defaults to `null`. |
> You can create more instances as well, because in the background we use the provided `id` value.

#### Legacy Views

Call `tagFormElement()` on the `SeonFormTag` instance for each `View` that belongs to the form. Pass `true` as the second argument to mark a field as optional.

```java
// Java
SeonFormTag loginForm = new SeonFormTag("login_form", 2, "Login Form");
loginForm.tagFormElement(usernameEditText);
loginForm.tagFormElement(passwordEditText, true); // optional field
```

```kotlin
// Kotlin
val loginForm = SeonFormTag("login_form", 2, "Login Form")
loginForm.tagFormElement(usernameEditText)
loginForm.tagFormElement(passwordEditText, true) // optional field
```

#### Compose

Use the `seonTagForm` `Modifier` extension to associate a composable with a form. Pass `optional = true` to mark a field as optional.

```kotlin
// Kotlin
val loginForm = remember { SeonFormTag("login_form", 2, "Login Form") }

TextField(
    value = username,
    onValueChange = { username = it },
    modifier = Modifier.seonTagForm(loginForm)
)

TextField(
    value = password,
    onValueChange = { password = it },
    modifier = Modifier.seonTagForm(loginForm, optional = true)
)
```

> **Note:** `tagFormElement()` and `seonTagForm` can be combined with `tagViewElement()` / `testTag()` on the same view — form tagging and element naming are independent.

---

## Screen (Activity and Fragment) Tracking

During an active session, the SDK automatically tracks Activity and Fragment lifecycle transitions without additional integration code. Screen appearances, disappearances, and navigation events (including Compose routes, if it is set up by `setComposeNavController`) are captured and sent to the SEON platform.

Internally, the SDK registers the following system callbacks:

| Callback | Purpose |
|---|---|
| `Application.ActivityLifecycleCallbacks` | Monitors Activity resume/pause transitions. |
| `FragmentManager.FragmentLifecycleCallbacks` | Monitors Fragment resume/pause transitions. |
| `NavController.OnDestinationChangedListener` | Tracks Compose navigation route changes. |
---

## Touch Tracking

The SDK automatically captures touch interactions during an active session, including tap events, gestures, and screen coordinates. This data is used to build behavioural profiles for fraud detection. No additional integration is required.

Internally, the SDK wraps the following system callbacks:

| Callback | UI Framework | Purpose |
|---|---|---|
| `View.OnTouchListener` | Legacy Views | Wraps the existing touch listener on each view to intercept touch events. |
| `Window.Callback` | Compose | Wraps the Activity's window callback to intercept `dispatchTouchEvent`. |
---

## Input Tracking

The SDK automatically monitors text input interactions during an active session, including text field focus, input events, and typing characteristics. These signals help detect anomalies such as automated or copy-pasted input. No additional integration is required.

> **Privacy:** The SDK does not capture the actual values of input fields. Only interaction metadata (e.g. focus changes, typing characteristics, paste actions) is monitored, ensuring that sensitive information such as personal data or passwords is never collected.

Internally, the SDK wraps or registers the following system callbacks:

| Callback | UI Framework | Purpose |
|---|---|---|
| `TextWatcher` | Legacy Views | Monitors text changes on `EditText` fields. |
| `View.OnFocusChangeListener` | Legacy Views | Wraps the existing focus listener to track focus gain/loss. |
| `ActionMode.Callback` | Legacy Views | Wraps the selection action mode callback to detect copy, paste, and cut actions. |
| `OnReceiveContentListener` | Legacy Views | Detects paste and drag-and-drop content insertion. |
| `SemanticsListener` | Compose | Observes the Compose semantics tree for text and focus changes. |

---

## Common integration difficulties

- **Late initialization or session start** — The SDK registers Activity and Fragment lifecycle callbacks during `initialize()`, but events are only recorded once `startSessionMonitoring()` is called. If you initialize or start the session after the first Activity has already resumed, that screen will not appear in the session timeline. Call `initialize()` in `Application.onCreate()` and start the session as early as possible.

- **Default `maxBackgroundDuration` is `0`** — With the default value the session will not survive any time in the background. If you expect the session to persist across brief app switches (e.g. opening the camera or switching apps), set a positive value with `withMaxBackgroundDuration()`.


- **Session config validation is synchronous** — `SeonStreamSessionConfig.Builder.build()` throws a `SeonStreamConfigurationException` immediately if the values are invalid (negative duration, duration exceeding 48 hours, or label longer than 32 characters). This is not delivered through `onFailure` — wrap the `build()` call in a try-catch if you use dynamic values.

- **Tag screens and views early** — Tags are resolved when lifecycle and interaction events fire. If you tag an Activity or View after it has already appeared on screen, the initial events will be recorded with the default class name instead of the tag you assigned.

---

## Limitations & Auto-tagging Behaviour

The SDK automatically assigns names to screens and UI elements where possible. The table below summarises what is auto-tagged, what requires manual intervention, and what is not supported.

### Screen auto-tagging

| Scenario | Auto-tagged? | Name source | Example | Notes |
|---|--------------|---|---|---|
| Activity (Legacy Views) | Yes          | Fully qualified class name | `com.example.CheckoutActivity` | Override with `tagActivity()`. |
| Fragment (Legacy Views) | Yes          | Fully qualified class name | `com.example.PaymentFragment` | Override with `tagFragment()`. Only leaf fragments (no child fragments) generate events. |
| Compose navigation route | No           | Route name from the navigation graph | `"checkout"`, `"profile/{id}"` | You must call `setComposeNavController()` to enable route tracking. Without it, Compose screen transitions are not captured. |

### UI element auto-tagging

| Scenario | Auto-tagged? | Name source               | Example | Notes |
|---|---|---------------------------|--|---|
| Legacy View with `android:id` | Yes | XML resource entry name   | `submit_button` from `android:id="@+id/submit_button"` | Override with `tagViewElement()`. |
| Legacy View **without** `android:id` | No | Appears as `UNKNOWN`      | `UNKNOWN` | Assign an `android:id` in XML or call `tagViewElement()`. |
| Compose element with `Modifier.testTag()` | Yes | `testTag` value           | `"pay-btn"` from `Modifier.testTag("pay-btn")` | Best option for stable, meaningful names. |
| Compose element with `contentDescription` | Yes | `contentDescription` value | `"Submit order"` from `Modifier.semantics { contentDescription = "Submit order" }` | Used when no `testTag` is set. |
| Compose `Text()` element | Yes | Text content (truncated)  | `"Welcome back, John"` from `Text("Welcome back, John")` | Used when neither `testTag` nor `contentDescription` is set. |
| Compose element with none of the above | No | - | "" | Add a `testTag` or `contentDescription` for the element to appear with a name. |

### Input tracking

| Scenario | Tracked? | Notes |
|---|---|---|
| `EditText` (Legacy Views) | Yes | Focus, typing characteristics, copy/paste/cut, and drag-and-drop are captured. |
| Custom input views (not extending `EditText`) | No | Only `EditText` instances are detected by the legacy input strategy. |
| Compose text fields (`TextField`, `OutlinedTextField`) | Yes | Tracked via the Compose semantics tree. |
| Compose custom `Canvas`-based input | No | Elements that do not expose semantics properties are invisible to the SDK. |

### Touch tracking

| Scenario | Tracked? | Notes |
|---|---|---|
| Legacy Views — clickable views | Yes | Touch events are captured via `View.OnTouchListener` wrapping. |
| Legacy Views — non-clickable views | No | Only views where `isClickable() == true` are tracked. |
| Compose elements | Yes | Touch events are captured via `Window.Callback` wrapping and resolved against the semantics tree. |

---

## Changelog

### 1.2.0
- Added custom `android.content.ContentProvider` to initialize SDK automatically as soon as possible
- Added `SeonFormTag` class to define custom forms
- Improved network transmission logic
- Improved error handling

### 1.1.0
**First production-ready release.**
- `SeonStreamSessionConfig`'s `authData` replaces token: pass the opaque authData string from `GET /session-monitoring-api/v1/auth` to start a session.
- Authentication failure ends the stream and calls `onStreamAuthError` callback, while preserving the session for resumption within maxBackgroundDuration
- Exposed storage related errors through `onStreamFinishedWithError` callback with `SeonStreamStorageException`
- Fixed paste tracking related issues

### 1.0.0
- Initial open preview release — not supported for production use.
