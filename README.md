# SEON Stream SDK - Android
The SEON Stream SDK continuously collects behavioural signals from your Android application — touch patterns, typing dynamics, sensor data, network state and screen flows — and streams them to the SEON platform for fraud detection and risk assessment. The SDK supports both Jetpack Compose and legacy Android Views (Activities, Fragments, XML layouts).

---

## Requirements

- Android 7.0 or higher (API level 24)
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

An API key issued by SEON is required. Contact your SEON account representative to obtain one.

---

## Installation

Add the dependency to your module-level `build.gradle` file.

```groovy
// Groovy DSL
dependencies {
    implementation('io.seon.streamsdk:streamsdk:1.0.0') {
        transitive = true
    }
}
```

```kotlin
// Kotlin DSL
dependencies {
    implementation("io.seon.streamsdk:streamsdk:1.0.0") {
        isTransitive = true
    }
}
```

---

## Getting started

### Configuration

The SDK is configured in two stages: a **global configuration** (set once at initialization) and a **session configuration** (set each time you start a session).

#### Global Configuration

Use `SeonStreamGlobalConfig.Builder` to set the API token and the data-center region.

| Method | Description | Default |
|---|---|---|
| `withToken(String)` | Sets the API token. | — |
| `withRegion(Region)` | Target region: `Region.EU`, `Region.US`, or `Region.APAC`. | `Region.EU` |

#### Session Configuration

Use `SeonStreamSessionConfig.Builder` to control per-session behavior.

| Method | Description | Default |
|---|---|---|
| `withMaxBackgroundDuration(long)` | Maximum time (in ms) the session stays alive while the app is in the background. Must be between `0` and `48 hours`. | `0` |
| `withLabel(String)` | An optional label for the session (max 32 characters). | `null` |

### Initialization

Call `SeonStream.initialize()` once — typically in your `Application.onCreate()` or main `Activity.onCreate()`.

```java
// Java
SeonStreamGlobalConfig config = new SeonStreamGlobalConfig.Builder()
        .withToken("YOUR_API_TOKEN")
        .withRegion(Region.EU)
        .build();

SeonStream.initialize(this, config);
```

```kotlin
// Kotlin
val config = SeonStreamGlobalConfig.Builder()
    .withToken("YOUR_API_TOKEN")
    .withRegion(Region.EU)
    .build()

SeonStream.initialize(this, config)
```

#### Updating the Token

If the token is not available at initialization time, you can set or update it later:

```java
// Java
SeonStream.getInstance().setToken("YOUR_API_TOKEN");
```

```kotlin
// Kotlin
SeonStream.getInstance().setToken("YOUR_API_TOKEN")
```

---

## Starting and stopping a session
#### Starting a Session
```java
// Java
SeonStreamSessionConfig sessionConfig = new SeonStreamSessionConfig.Builder()
        .withMaxBackgroundDuration(60000)
        .withLabel("checkout-flow")
        .build();

SeonStream.getInstance().startStream(sessionConfig);
```

```kotlin
// Kotlin
val sessionConfig = SeonStreamSessionConfig.Builder()
    .withMaxBackgroundDuration(60000)
    .withLabel("checkout-flow")
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
| `SeonStreamConfigurationException` | `SeonStreamGlobalConfig.Builder.build()` is called with an invalid region, `SeonStreamSessionConfig.Builder.build()` is called with invalid values (negative duration, duration exceeding 48 hours, or label longer than 32 characters), or `setToken()`, `tagActivity()`, `tagFragment()`, `tagViewElement()`, `addListener()`, or `removeListener()` fails. | Synchronous (try-catch) |
| `SeonStreamCustomEventException`   | `createCustomEvent()` fails — e.g. name exceeds 32 characters, additional data exceeds 1024 characters, or an internal error.                                                                                                                                                                                                                                 | Async (`onStreamFinishedWithError`) |
| `SeonStreamAuthException`          | The backend throws back a 401 error, so the provided api token is invalid                                                                                                                                                                                                                                                                                     | Async (`onStreamFinishedWithError`) |
| `SeonStreamTimeoutException`       | The backend informs the client that it reached its session duration's hard limit                                                                                                                                                                                                                                                                              | Async (`onStreamFinishedWithError`) |
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

- **Token must be set before starting a session** — If no token has been provided (either through `SeonStreamGlobalConfig` or `setToken()`) before calling `startSessionMonitoring()`, the start will fail asynchronously via the `onFailure` callback.

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

### 1.0.0
- Initial release
