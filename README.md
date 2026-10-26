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
    implementation('io.seon.streamsdk:streamsdk:2.0.0') {
        transitive = true
    }
}
```

```kotlin
// Kotlin DSL
dependencies {
    implementation("io.seon.streamsdk:streamsdk:2.0.0") {
        isTransitive = true
    }
}
```

---

### Initialization

The SDK initializes itself automatically via `SeonStreamInitProvider`, a `ContentProvider` that runs before `Application.onCreate()`. No manual `initialize()` call is required.

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

#### Feature Configuration

The `SeonStream.initialize()` accepts an optional `SeonFeatureConfig` that controls SDK-wide behavior.

The automatic initialization via `SeonStreamInitProvider` always uses the default configuration. To pass a custom `SeonFeatureConfig`, remove the provider from your manifest as described above and initialize the SDK manually. Only the first `initialize()` call takes effect, so if the provider is not removed, your custom configuration is ignored.

Features to control:

| Feature | Description | Default state |
|---|---|--|
| automaticScreenTracking | Automatically tracks screens (Activities). When disabled, report screens manually with `SeonStream.getInstance().changeScreen()`. | `true` |
| automaticViewTracking | Automatically tracks UI elements (touch, focus, typing, clipboard events). When disabled, only elements tagged with `SeonStream.tagViewElement()` / `Modifier.seonTagView()` are tracked. | `true` |

See [Manual tracking](#manual-tracking) for details.

```java
// Java

SeonFeatureConfig featureConfig = new SeonFeatureConfig.Builder()
        .withAutomaticScreenTrackingEnabled(false)
        .withAutomaticViewTrackingEnabled(false)
        .build();

SeonStream.initialize(this, featureConfig);
```

```kotlin
// Kotlin

val featureConfig = SeonFeatureConfig.Builder()
    .withAutomaticScreenTrackingEnabled(false)
    .withAutomaticViewTrackingEnabled(false)
    .build()

SeonStream.initialize(this, featureConfig)
```

## Starting and stopping a session

#### Session Configuration

Create a `SeonStreamSessionConfig` to control per-session behavior. It is constructed directly with the following parameters:

| Parameter | Description | Default |
|---|---|---|
| `authData` (`String`) | This is **required** to start a stream, can be gathered through the Seon's authentication endpoint. If this data becomes invalid for some reason, it has to be renewed manually (see [Authentication](#authentication)). |  |
| `maxBackgroundDuration` (`long`) | Maximum time (in ms) the session stays alive while the app is in the background. Must be between `0` and `48 hours`. | `0` |
| `label` (`String`) | An optional label for the session (max 32 characters). | `null` |

#### Starting a Session
```java
// Java
String authData = fetchTokenFromBackend() // This is just an example

SeonStreamSessionConfig sessionConfig = new SeonStreamSessionConfig(
        authData,        // required
        60000,           // maxBackgroundDuration
        "checkout-flow"  // label, nullable
);

SeonStream.getInstance().startStream(sessionConfig);
```

```kotlin
// Kotlin
val authData = fetchTokenFromBackend() // This is just an example
val sessionConfig = SeonStreamSessionConfig(
    authData,        // required
    60000,           // maxBackgroundDuration
    "checkout-flow"  // label, nullable
)

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

All SDK exceptions extend `SeonStreamException` (a `RuntimeException`). Errors are delivered through three channels depending on when they occur:

- **Synchronous** — Initialization and configuration errors (e.g. `initialize()`, tagging, listener management) are thrown directly and can be caught with a `try-catch` block.
- **Asynchronous in `onStreamFinished`** — Errors that occur at the end of an active session, or that cause the active session to stop are delivered through the `SeonStreamListener.onStreamFinished` callback. See [Listening to Session Events](#listening-to-session-events) for details.
- **Asynchronous in `onStreamError`** — Errors that occur during an active session but do not stop the stream, or errors that occur during stream start are delivered through the `SeonStreamListener.onStreamError()` callback. See [Listening to Session Events](#listening-to-session-events) for details.

| Exception                                | Cause                                                                                                                                                                  | Reported via         |
|------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------|
| `SeonStreamInitializeException`          | `initialize()` is called with a `null` context or more than once, `getInstance()` is called before initialization, or the SDK fails to set up its trackers.            | try-catch            |
| `SeonStreamInternalException`            | `tagActivity()`, `tagViewElement()`, `addListener()` or `removeListener()` fails internally.                                                          | try-catch            |
| `SeonStreamAlreadyRunningException`      | `startSessionMonitoring()` is called while a session is already running.                                                                                               | `onStreamError()`    |
| `SeonStreamSessionConfigException`       | `startSessionMonitoring()` is called with an invalid config: `authData` is null or invalid, the label is longer than 32 characters, or `maxBackgroundDuration` is not between 0 and 2 days. | `onStreamError()`    |
| `SeonStreamStartException`               | Starting the session fails internally, e.g. the stream ID cannot be generated.                                                                                         | `onStreamError()`    |
| `SeonStreamNotRunningException`          | `finishSessionMonitoring()` or `createCustomEvent()` is called while no session is running.                                                                            | `onStreamError()`    |
| `SeonStreamCustomEventInvalidException`  | `createCustomEvent()` is called with a missing or too long name (max 32 characters) or too long additional data (max 1024 characters), or fails internally.            | `onStreamError()`    |
| `SeonStreamRateLimitException`           | The backend rate limit is reached during an active session and data transmission is throttled. The session keeps running.                                              | `onStreamError()`    |
| `SeonStreamAuthException`                | The auth token is rejected by the backend (e.g. expired) during an active session, which stops the session.                                                            | `onStreamFinished()` |
| `SeonStreamSessionTimeoutException`      | The backend reports that the session reached its maximum duration, which stops the session.                                                                            | `onStreamFinished()` |
| `SeonStreamIpBanException`               | The backend rejects the client because its IP address is banned, which stops the session.                                                                              | `onStreamFinished()` |
| `SeonStreamStorageException`             | A serious local storage error occurs (e.g. the database insert fails or the storage is full), which stops the session.                                                 | `onStreamFinished()` |
| `SeonStreamFinishException`              | An internal error occurs while stopping the session.                                                                                                                   | `onStreamFinished()` |

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
    public void onStreamFinished(SeonStreamCriticalException exception)) {
        // Session was stopped, if exception is null the finish was successful.
    }

    @Override
    public void onStreamError(SeonStreamNonCriticalException exception) {
        // An error occured
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

    override fun onStreamFinished(exception: SeonStreamCriticalException) {
        // Session was stopped by finishStream()
    }

    override fun onStreamError(exception: SeonStreamNonCriticalException) {
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
2. Pass `authData` to the `SeonStreamSessionConfig` constructor when calling `startStream(sessionConfig)`.
3. The SDK extracts the session JWT and additional configuration from `authData` automatically.
4. If the JWT inside `authData` expires and the backend rejects further uploads, the SDK stops the stream and calls `onStreamAuthError` callback (see [Error handling](#error-handling)). The stored session is **not** cleared on auth failure, so you can fetch a fresh `authData` and call `startStream(sessionConfig)` again with the **same `label`** to continue the session, as long as you are still within `maxBackgroundDuration`.

---

## Navigation tracking

Automatic screen tracking is enabled by default, and it is switched off with `withAutomaticScreenTrackingEnabled(false)` (see [Feature Configuration](#feature-configuration)).

### Automatic navigation tracking

| Tracked | Legacy XML UI | Jetpack Compose |
|---|---|---|
| Activity appearances and disappearances | Yes | Yes |
| Route changes | - | Yes, if set up with `setComposeNavController` |
| Fragments | No | - |
| Modals and dialogs | No | No |

#### Compose navigation

If your app uses Jetpack Compose navigation and the package `androidx.navigation.navigation-compose` implemented, wrap the `NavHostController` with `NavHostControllerWrapper` and pass it to the SDK, so it can track route changes. Call `setComposeNavController(navController)` as soon as the controller is available and before starting the session.
```java
// Java
// obtain navController from your Compose host
SeonStream.getInstance().setComposeNavController(new NavHostControllerWrapper(navController));
```

```kotlin
// Kotlin
val navController = rememberNavController()
LaunchedEffect(navController){
    SeonStream.getInstance().setComposeNavController(NavHostControllerWrapper(navController))
}
```
> **Important**: Dont use the wrapper if you do not have the package `androidx.navigation.navigation-compose` implemented.
>
> Compose route tracking is not enabled automatically.
> If you register the controller after calling `startStream()`, subsequent route changes will still be tracked, but the initial visible route may be missed.

#### Screen names

The SDK derives the screen name automatically:

| Scenario | Name source | Example |
|---|---|---|
| Activity | Tagged name → Activity label → fully qualified class name | `checkout-screen` → `Checkout` → `com.example.CheckoutActivity` |
| Compose navigation route | Route from the navigation graph | `"checkout"`, `"profile/{id}"` |

For an Activity the first available name in this order is used:

1. **Tagged name** — set with `tagActivity()`.
2. **Activity label** — the `android:label` of the Activity in your manifest, or the title set with `Activity.setTitle()`.
3. **Class name** — the fully qualified class name, used when neither of the above exists.

To override it, an Activity can be renamed with `tagActivity()`. Call it before the Activity appears on screen (e.g. in `onCreate()`), otherwise the first events are recorded with the label or class name.

```java
// Java
SeonStream.tagActivity(activity, "checkout-screen");
```

```kotlin
// Kotlin
SeonStream.tagActivity(activity, "checkout-screen")
```

Compose route names cannot be renamed, they always come from the navigation graph. Use a short, stable route name, or switch to [manual navigation tracking](#manual-navigation-tracking) if you need full control over the names.

### Manual navigation tracking

Disable automatic screen tracking, then call `changeScreen()` every time a new screen becomes visible. The argument is the screen name that is shown in the SEON dashboard.
> Warning: You have to remove `SeonStreamInitProvider` from your manifest as described in [Initialization](#initialization), then call `initialize()` with your config.

```java
// Java
SeonStream.getInstance().changeScreen("checkout-screen");
```

```kotlin
// Kotlin
SeonStream.getInstance().changeScreen("checkout-screen")
```

### Limitations

| Scenario | Behaviour | Notes |
|---|---|---|
| `changeScreen()` with automatic screen tracking enabled | Ignored | Prevents duplicated screen events. Disable it with `withAutomaticScreenTrackingEnabled(false)`. |
| `tagActivity()` with automatic screen tracking disabled | No effect | It only renames automatically tracked screens. Pass the name to `changeScreen()` instead. |
| `changeScreen()` while no session is running | Not recorded | Call it for the initially visible screen right after `startStream()`. |

---

## View Tracking

Automatic view tracking is enabled by default, and it is switched off with `withAutomaticViewTrackingEnabled(false)` (see [Feature Configuration](#feature-configuration)). While it is enabled, the SDK tracks touches and text input on the UI elements of your app without additional integration code, and names them automatically.

#### Touch tracking

The SDK automatically captures touch interactions during an active session, including tap events, gestures, and screen coordinates.

Internally, the SDK wraps the following system callbacks:

| Callback | UI Framework | Purpose |
|---|---|---|
| `View.OnTouchListener` | Legacy XML UI | Wraps the existing touch listener on each view to intercept touch events. |
| `Window.Callback` | Jetpack Compose | Wraps the Activity's window callback to intercept `dispatchTouchEvent`. |

> **Important:** On Legacy XML UI only clickable elements are tracked automatically by the SDK.

#### Input tracking

The SDK automatically monitors text input interactions during an active session, including text field focus, input events, and typing characteristics.

> **Privacy:** The SDK does not capture the actual values of input fields. Only interaction metadata (e.g. focus changes, typing characteristics, paste actions) is monitored, ensuring that sensitive information such as personal data or passwords is never collected.

Internally, the SDK wraps or registers the following system callbacks:

| Callback | UI Framework | Purpose |
|---|---|---|
| `TextWatcher` | Legacy XML UI | Monitors text changes on `EditText` fields. |
| `View.OnFocusChangeListener` | Legacy XML UI | Wraps the existing focus listener to track focus gain/loss. |
| `ActionMode.Callback` | Legacy XML UI | Wraps the selection action mode callback to detect copy, paste, and cut actions. |
| `OnReceiveContentListener` | Legacy XML UI | Detects paste and drag-and-drop content insertion. |
| `SemanticsListener` | Jetpack Compose | Observes the Compose semantics tree for text and focus changes. |

#### UI element auto-tagging

The SDK assigns a name and a type to every tracked UI element in the following order. Use [Tagging UI elements](#tagging-ui-elements) to override it.

**Legacy XML UI**

```
Name:  seon override  →  android:id resource entry name  →  UNKNOWN
Type:  seon override  →  view class  →  Unknown
```

**Jetpack Compose**

```
Name:  seon override  →  Modifier.testTag()  →  contentDescription  →  Text content (truncated)  →  empty
Type:  seon override  →  semantics role  →  editable text / IME action  →  progress range  →  text  →  Unknown
```

The type is inferred as follows:

| | Legacy XML UI (view class) | Jetpack Compose (semantics) |
|---|---|---|
| `TextInputField` | `EditText` | Editable text or IME action |
| `Button` | `Button`, `ImageButton` | Role `Button` |
| `Checkbox` | `CheckBox` | Role `Checkbox` or `TriStateCheckbox` |
| `RadioButton` | `RadioButton` | Role `RadioButton` |
| `Switch` | `Switch`, `SwitchCompat` | Role `Switch` |
| `Slider` | `SeekBar` | Progress range |
| `Dropdown` | `Spinner` | Role `DropdownList` |
| `Image` | `ImageView` | Role `Image` |
| `Label` | `TextView` | Has text |
| `Unknown` | Anything else | Anything else |

The more specific class is matched first, e.g. a `CheckBox` is a `Checkbox`, not a `Button`. If the type is wrong, for example a clickable `LinearLayout` that acts as a button, set it with [Tagging UI elements](#tagging-ui-elements).

> **Note:** Legacy views without an `android:id` appear as `UNKNOWN`, and Compose elements with none of the sources appear with an empty name. Add a name to them for the element to be easy to identify.

## Tagging UI elements

By default the SDK names every UI element it tracks (see [UI element auto-tagging](#ui-element-auto-tagging)). A tag is applied the same way in both UI frameworks, but through a different API. Both take a `name` and an optional `type` (`SeonElementType`): `TEXT_INPUT_FIELD`, `BUTTON`, `CHECKBOX`, `RADIO_BUTTON`, `SWITCH`, `SLIDER`, `DROPDOWN`, `IMAGE`, `LABEL` or `UNKNOWN`. When the type is omitted it is inferred as described in [UI element auto-tagging](#ui-element-auto-tagging).

#### Legacy XML UI

Call `SeonStream.tagViewElement(view, name[, type])` on the `View` once it has been created, e.g. in `onCreate()` or `onViewCreated()`.

```java
// Java
SeonStream.tagViewElement(usernameEditText, "username");
SeonStream.tagViewElement(payButton, "pay_button", SeonElementType.BUTTON);
```

```kotlin
// Kotlin
SeonStream.tagViewElement(usernameEditText, "username")
SeonStream.tagViewElement(payButton, "pay_button", SeonElementType.BUTTON)
```

#### Jetpack Compose

Add the `Modifier.seonTagView(name, type)` extension to the composable.

```kotlin
// Kotlin
TextField(
    value = username,
    onValueChange = { username = it },
    modifier = Modifier.seonTagView("username")
)

Button(
    onClick = onPay,
    modifier = Modifier.seonTagView("pay_button", SeonElementType.BUTTON)
) {
    Text("Pay")
}
```

### Form tagging

Use `SeonFormTag` to group related input fields into a named form so they appear together on the SEON admin panel. Create one `SeonFormTag` instance per form and apply it to each field that belongs to that form.

`SeonFormTag` takes three parameters:

| Parameter | Required | Description |
|---|---|---|
| `id` | Yes | Unique identifier for the form. Case-insensitive; whitespace is replaced by `_`. Forms are grouped by this id on the admin panel. |
| `numberOfElements` | Yes | Total number of fields that make up this form. |
| `displayName` | No | Human-readable name shown on the admin panel. Defaults to `null`. |
> You can create more instances as well, because in the background we use the provided `id` value.

#### Legacy XML UI

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

#### Jetpack Compose

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

### Manual view tracking

Manual view tracking is enabled when automatic view tracking is switched off with `withAutomaticViewTrackingEnabled(false)` in `SeonFeatureConfig`. 
> Warning: You have to remove `SeonStreamInitProvider` from your manifest as described in [Initialization](#initialization), then call `initialize()` with your config.

```java
// Java
SeonFeatureConfig featureConfig = new SeonFeatureConfig.Builder()
        .withAutomaticViewTrackingEnabled(false)
        .build();

SeonStream.initialize(this, featureConfig);
```

```kotlin
// Kotlin
val featureConfig = SeonFeatureConfig.Builder()
    .withAutomaticViewTrackingEnabled(false)
    .build()

SeonStream.initialize(this, featureConfig)
```

In this mode the SDK no longer tracks UI elements on its own. The elements tagged with [`tagViewElement()`](#legacy-xml-ui) or [`Modifier.seonTagView()`](#jetpack-compose) stay trackable: their touches are reported, and for text fields their focus, typing and clipboard events as well. Untagged elements are not reported, so tag every element you want to see in the session. The name and type of a tracked element come from its tag.

### Limitations

| Scenario | Behaviour | Notes |
|---|---|---|
| Untagged element with automatic view tracking disabled | Not reported | Tag every element you want to see with `tagViewElement()` / `seonTagView()`. |
| Tagged element with an empty name | Not reported | Always pass a non-empty name. |
| Element interacted with before it is tagged | Recorded with the default name | Tag the view before it is interacted with, e.g. in `onCreate()` or `onViewCreated()`. |
| Compose `testTag()`, `contentDescription` or text with automatic view tracking disabled | Not used for naming | Only the name passed to `seonTagView()` is used. |


---

## Common integration difficulties

- **Late initialization or session start** — The SDK registers Activity lifecycle callbacks during `initialize()`, but events are only recorded once `startSessionMonitoring()` is called. If you initialize or start the session after the first Activity has already resumed, that screen will not appear in the session timeline. Call `initialize()` in `Application.onCreate()` and start the session as early as possible.

- **Default `maxBackgroundDuration` is `0`** — With the default value the session will not survive any time in the background. If you expect the session to persist across brief app switches (e.g. opening the camera or switching apps), set a positive `maxBackgroundDuration`.


- **Session config validation happens on start** — The `SeonStreamSessionConfig` constructor does not validate its values. Invalid values (negative duration, duration exceeding 48 hours, label longer than 32 characters, or malformed authData) are reported asynchronously after `startStream()` is called, so register a `SeonStreamListener` to detect them.

- **Tag screens and views early** — Tags are resolved when lifecycle and interaction events fire. If you tag an Activity or View after it has already appeared on screen, the initial events will be recorded with the default class name instead of the tag you assigned.

---
## Migrating from 1.x to 2.0

<details>
<summary>Show the 1.x to 2.0 migration guide</summary>

1. **Update the dependency** to `io.seon.streamsdk:streamsdk:2.0.0`.
2. **Replace `SeonStreamSessionConfig.Builder`** with the constructor. `withAuthData()`, `withMaxBackgroundDuration()`, `withLabel()` and `build()` are removed.
   ```kotlin
   // 1.x
   val config = SeonStreamSessionConfig.Builder()
       .withAuthData(authData)
       .withMaxBackgroundDuration(60000)
       .withLabel("checkout-flow")
       .build()

   // 2.0
   val config = SeonStreamSessionConfig(authData, 60000, "checkout-flow")
   ```
3. **Update your `SeonStreamListener`.** `onStreamFinished()`, `onStreamFinishedWithError()` and `onStreamAuthError()` are replaced:
   ```
   onStreamFinished()            ┐
   onStreamFinishedWithError(e)  ├→  onStreamFinished(SeonStreamCriticalException)   null = successful finish,
   onStreamAuthError()           ┘                                                   otherwise a critical error (e.g. SeonStreamAuthException)

   (new)                         →   onStreamError(SeonStreamNonCriticalException)   errors that do not stop the stream, or occur on start
   ```
4. **Update your exception handling.** Several exceptions are removed or renamed, see [Error Handling](#error-handling) for the full list:
   ```
   SeonStreamConfigurationException  →  SeonStreamInternalException         tagging, listener calls (try-catch)
                                     →  SeonStreamSessionConfigException     invalid session config (onStreamError)
   SeonStreamStopException           →  SeonStreamNotRunningException        no running session (onStreamError)
                                     →  SeonStreamFinishException            internal stop error (onStreamFinished)
   SeonStreamCustomEventException    →  SeonStreamCustomEventInvalidException
   SeonStreamTimeoutException        →  SeonStreamSessionTimeoutException
   ```
   `finishStream()` no longer throws.
5. **Rename the `SeonFeatureConfig` options**, if you use a custom config:
   ```
   withActivityTrackerEnabled()  →  withAutomaticScreenTrackingEnabled()
   withInputTrackerEnabled()     ┐
                                 ├→  withAutomaticViewTrackingEnabled()
   withTouchTrackerEnabled()     ┘
   ```
   Input and touch tracking can no longer be switched off separately.
6. **Do not rely on automatic Fragment, modal and dialog tracking.** Only Activity changes are tracked automatically. To report other screens, disable automatic screen tracking and call `changeScreen()` for **all** screens, see [Manual tracking](#manual-tracking).

</details>

## Changelog
### 2.0.0

> ⚠️ **Breaking changes.** See [Migrating from 1.x to 2.0](#migrating-from-1x-to-20)

- Added `SeonFeatureConfig` to turn automatic screen and UI tracking on or off, SDK can be initialized with this object
- Added manual screen tracking with `changeScreen()`
- Added manual UI element tracking with `tagViewElement()` and `Modifier.seonTagView()`
- Added optional `SeonElementType` to tagged elements
- `SeonStream.initialize()` now returns the `SeonStream` instance as well
- `SeonStreamListener` now has `onStreamStarted`, `onStreamFinished` and `onStreamError`
- Added `SeonStreamCriticalException` and `SeonStreamNonCriticalException` error groups
- Added `SeonStreamRateLimitException` when rate limit was hit
- Removed fragment, modal and dialog from automatic screen tracking
- Fixed touch and typing related bugs

### 1.3.2
- Fixed minor internal bugs

### 1.3.1
- Fixed minor internal bugs

### 1.3.0
- Changed `androidx.navigation.navigation-compose` as an optional dependency
- Changed `setComposeNavController` parameter type to `NavHostControllerWrapper`
- Increased target API level to **37**

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
