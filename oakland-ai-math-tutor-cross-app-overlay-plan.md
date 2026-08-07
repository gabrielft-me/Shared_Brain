# Pen-First AI Math Tutor — Cross-App Floating Overlay Plan

## 1. Architectural correction

The Android tutor app should **not require its own Activity to remain visible while the student works**.

The app's Activity becomes mainly a setup/control surface used to:

- request overlay permission
- request screen-capture consent
- start/stop the tutor session
- configure the tutor
- show session summaries

During tutoring, the student can open **another math app, browser, PDF, worksheet, or any normal Android app**. The tutor stays available through system-level floating overlay windows.

The core product loop becomes:

```text
Any foreground Android app
        │
        │ visible screen
        ▼
MediaProjection capture session
        │
        ▼
Periodic screenshot
        │
        ├── Visual Understanding
        │
        └── Pointing Service
                │
                ▼
        normalized (x, y) / bbox
                │
                ▼
TutorOverlayService
        │
        ▼
Floating explanation positioned
next to the selected screen region
```

The Canvas remains important, but it becomes a **transparent annotation layer above other apps**, rather than the primary worksheet surface.

---

## 2. Android API that actually enables the floating window

Use the native Android window system:

- `WindowManager`
- `WindowManager.LayoutParams`
- `WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY`
- `android.permission.SYSTEM_ALERT_WINDOW`
- `Settings.canDrawOverlays()`
- `Settings.ACTION_MANAGE_OVERLAY_PERMISSION`

`TYPE_APPLICATION_OVERLAY` is the correct API for a window that must stay above ordinary app Activities.

It was added in Android 8 / API 26 and is the replacement for old system-alert window types such as `TYPE_PHONE`, `TYPE_SYSTEM_ALERT`, and `TYPE_SYSTEM_OVERLAY` for third-party apps.

Conceptually:

```text
Chrome / IXL / PDF / other app
              │
              ▼
      Android application layer
              │
              ▼
 TYPE_APPLICATION_OVERLAY
              │
              ▼
Tutor bubble / Canvas / hint card
```

The overlay sits above normal application windows but below critical system UI such as parts of the status bar and IME.

---

## 3. Required overlay permission

Manifest:

```xml
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
```

Before starting the overlay:

```kotlin
if (!Settings.canDrawOverlays(context)) {
    val intent = Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION)
    context.startActivity(intent)
}
```

This is a special Android permission. The user grants it through Android Settings rather than a normal runtime permission dialog.

---

## 4. The app should use multiple overlay windows

Do not implement the whole tutor as one giant permanent overlay.

Use three overlay components with separate responsibilities.

### A. TutorBubbleOverlay

Small persistent control.

```text
      ● AI
```

Responsibilities:

- stays visible while another app is open
- draggable
- start annotation mode
- request hint
- collapse/expand tutor
- stop tutoring session

Recommended window configuration:

```text
TYPE_APPLICATION_OVERLAY
WRAP_CONTENT × WRAP_CONTENT
FLAG_NOT_FOCUSABLE
```

Because the bubble occupies only a small rectangle, the rest of the underlying app remains fully interactive.

---

### B. AnnotationCanvasOverlay

A transparent full-screen Canvas used only when the student wants to point, circle, underline, or annotate something.

```text
┌──────────────────────────────────────────────┐
│ underlying homework app                     │
│                                              │
│        3(x + 4) = 21                        │
│             ⭕                               │
│                                              │
│                              transparent     │
│                              Ink Canvas      │
└──────────────────────────────────────────────┘
```

This window uses:

```text
TYPE_APPLICATION_OVERLAY
MATCH_PARENT × MATCH_PARENT
PixelFormat.TRANSLUCENT
```

The Canvas should use the same low-latency pen/stylus implementation already planned.

Important: while this full-screen Canvas is touchable, it intercepts input. That means the app underneath cannot simultaneously receive the same touches.

Therefore the product needs explicit interaction modes.

---

### C. TutorCardOverlay

A small floating explanation positioned near the point returned by the backend.

Example:

```text
3(x + 4) = 21
     ⭕
       └──────────────┐
                      │
              ┌─────────────────────────┐
              │ What else does the 3    │
              │ multiply inside here?   │
              └─────────────────────────┘
```

This is another `TYPE_APPLICATION_OVERLAY` window whose `x` and `y` are controlled through `WindowManager.LayoutParams`.

---

## 5. Interaction modes

The overlay architecture should have three modes.

### PASSIVE mode

```text
Underlying app: interactive
Canvas: hidden
Tutor bubble: visible
Tutor card: optionally visible
```

The student uses the normal homework app without interference.

### ANNOTATE mode

```text
Underlying app: visible but temporarily not interactive
Full-screen transparent Canvas: touchable
S Pen / finger: captured by tutor
```

The student circles the region they want explained.

When the circle is complete, exit annotation mode.

### RESPONSE mode

```text
Canvas: removed or non-touchable
Underlying app: interactive again
Tutor card: positioned next to selected region
```

This prevents the transparent Canvas from permanently blocking the app underneath.

---

## 6. Moving overlay windows anywhere on screen

Use `WindowManager.LayoutParams.x` and `.y` with top-left gravity.

```kotlin
val params = WindowManager.LayoutParams(
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.WRAP_CONTENT,
    WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
    WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
        WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN,
    PixelFormat.TRANSLUCENT
).apply {
    gravity = Gravity.TOP or Gravity.START
    x = targetX
    y = targetY
}

windowManager.addView(view, params)
```

To move it later:

```kotlin
params.x = newX
params.y = newY
windowManager.updateViewLayout(view, params)
```

This is the mechanism used for both:

- user dragging the tutor bubble
- automatically positioning the hint near the backend's pointing result

---

## 7. Coordinate system

For this architecture, the **device screen** should be the canonical coordinate system.

Every AI frame should include:

```json
{
  "screen_width_px": 2560,
  "screen_height_px": 1600,
  "image_width_px": 2560,
  "image_height_px": 1600,
  "orientation": "landscape",
  "density_dpi": 280
}
```

The pointing service returns normalized values:

```json
{
  "point": {
    "x": 0.62,
    "y": 0.44
  },
  "bbox": {
    "x": 0.57,
    "y": 0.40,
    "width": 0.12,
    "height": 0.09
  }
}
```

Android converts them back:

```text
screenX = normalizedX × screenWidth
screenY = normalizedY × screenHeight
```

Then the overlay renderer offsets the card so it does not cover the selected expression.

---

## 8. Screen capture must now use MediaProjection

Because the tutor must understand **another app's screen**, rendering your own Canvas to a Bitmap is no longer sufficient.

Use:

- `MediaProjectionManager`
- `MediaProjection`
- `VirtualDisplay`
- `ImageReader`

Flow:

```text
User starts tutor session
        ↓
createScreenCaptureIntent()
        ↓
Android system consent UI
        ↓
MediaProjection token
        ↓
Foreground mediaProjection service
        ↓
VirtualDisplay
        ↓
ImageReader
        ↓
periodic screen frames
```

On Android 14+, user consent is required for each new MediaProjection capture session. Keep one projection session alive during the tutoring session instead of repeatedly starting and stopping projection for each screenshot.

---

## 9. MediaProjection foreground service

For Android 14+ declare:

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION" />
```

Service:

```xml
<service
    android:name=".capture.ScreenCaptureService"
    android:exported="false"
    android:foregroundServiceType="mediaProjection" />
```

The active projection must run as a foreground service.

This service owns:

- `MediaProjection`
- `VirtualDisplay`
- `ImageReader`
- capture cadence
- screenshot frame lifecycle

---

## 10. Overlay service

Keep overlay UI ownership separate from capture ownership.

Suggested service:

```text
TutorOverlayService
```

Responsibilities:

- own `WindowManager`
- create/remove TutorBubbleOverlay
- create/remove AnnotationCanvasOverlay
- create/remove TutorCardOverlay
- update x/y positions
- transition between PASSIVE / ANNOTATE / RESPONSE modes

For a long-running foreground overlay service on Android 14+, if you need it to be independently persistent and it does not fit another foreground-service category, Android provides the `specialUse` foreground-service type. Its use must be explained in the manifest and is reviewed for Play distribution.

For the hackathon, you can also keep overlay lifecycle coordinated with the active MediaProjection foreground service rather than creating unnecessary service complexity.

---

## 11. Critical screenshot + Canvas design

The student circles something on the **overlay Canvas**, but the semantic content lives in the app underneath.

Do not permanently flatten the whole product into a Canvas.

Use this pipeline:

```text
Underlying app pixels
       +
transparent annotation Canvas
       ↓
visual frame used for inference
       ↓
backend
```

There are two implementation options.

### Option A — capture overlay together with the screen

If the device's MediaProjection output includes the overlay exactly as expected, send the resulting frame directly.

This is simplest but needs device testing.

### Option B — recommended deterministic approach

Maintain the annotation layer independently.

1. MediaProjection captures the underlying display.
2. AnnotationCanvasOverlay stores the temporary visual annotation.
3. Render the annotation layer onto a copy of the captured frame using the exact same screen dimensions.
4. Send the composited image to the backend.
5. Clear/remove the annotation overlay.

```text
MediaProjection frame        Annotation layer
      2560×1600                  2560×1600
           │                        │
           └──────────┬─────────────┘
                      ▼
                 composite
                      ↓
             inference image
                 2560×1600
```

This preserves exact geometry while still allowing the vision model to see the student's circle.

The stroke coordinates exist only to render the annotation; the AI still performs visual analysis from the image.

---

## 12. Pointing pipeline

Use one captured/composited frame for both understanding and localization.

```text
                    screen frame
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
     Visual Understanding      Pointing Service
              │                     │
              │                  x / y / bbox
              └──────────┬──────────┘
                         ▼
                   Tutor Reasoning
                         │
                         ▼
                 structured response
```

Pointing output:

```json
{
  "selection_detected": true,
  "target_description": "the +4 term",
  "point": {
    "x": 0.621,
    "y": 0.437
  },
  "bbox": {
    "x": 0.580,
    "y": 0.405,
    "width": 0.091,
    "height": 0.063
  },
  "confidence": 0.95
}
```

---

## 13. Floating tutor placement algorithm

Backend returns a normalized point/bounding box.

Android should perform final placement locally.

```text
backend normalized coordinates
          ↓
convert to display pixels
          ↓
calculate target bbox
          ↓
try right side
          ↓
if collision → left
          ↓
if collision → above/below
          ↓
clamp to safe display bounds
          ↓
WindowManager.updateViewLayout()
```

Do not ask the model to decide the final card rectangle.

The model identifies the target. Android owns layout.

---

## 14. Jetpack Compose inside an overlay

You can continue using Jetpack Compose.

The native approach is:

```text
Service
  ↓
WindowManager
  ↓
ComposeView
  ↓
setContent { TutorCard(...) }
```

However, a `ComposeView` added directly from a Service does not automatically have the Activity lifecycle/view-model owners that Compose normally expects.

Plan for a small overlay UI host abstraction that provides:

- `LifecycleOwner`
- `SavedStateRegistryOwner`
- optionally `ViewModelStoreOwner`

Or keep overlay state in the service/state-holder and keep the Composable itself intentionally small.

---

## 15. Libraries evaluated

### Recommended core: native Android APIs

Use native `WindowManager` + `TYPE_APPLICATION_OVERLAY` for the actual production architecture.

Why:

- exact x/y positioning matters to this product
- you need multiple overlay window types
- you need custom stylus behavior
- you need tight MediaProjection integration
- you need control over touch pass-through
- the native API is not especially large once wrapped

Create an internal abstraction such as:

```text
OverlayManager
├── showBubble()
├── showAnnotationCanvas()
├── showTutorCard(x, y)
├── hideAnnotationCanvas()
├── moveWindow(...)
└── removeAll()
```

### FloatingX

A modern third-party option worth evaluating.

It supports:

- Kotlin
- system floating windows
- Jetpack Compose
- custom positions
- edge snapping
- animation
- saved positions
- multi-touch
- split-screen/window position handling

This is the strongest third-party candidate found for quickly implementing the draggable bubble/window layer.

For the hackathon, it could save time on bubble dragging and edge behavior, while MediaProjection and coordinate mapping remain your own code.

### Floating Bubble View

Another library that supports both XML and Jetpack Compose floating bubbles.

Useful for:

- chat-head style bubble
- expanded bubble UI
- quick prototype

Less attractive for this project than FloatingX/native because the tutor needs precise arbitrary target positioning and a full-screen annotation layer, not only a bubble interaction.

### Recruit Lifestyle FloatingView

Do not choose this for a new build.

The project is archived and explicitly points users toward Android bubbles.

### Android Notification Bubbles

Not the right primitive for this use case.

Bubbles are notification/windowing UX controlled heavily by Android and are intended mainly around bubble-style application/conversation experiences. They do not give the same arbitrary pixel-level positioning needed for your pointing-service response.

### Picture-in-Picture

Also not appropriate.

PiP is designed primarily for video, calls, and navigation; Android controls the floating window placement. It does not solve arbitrary tutoring overlays.

---

## 16. Recommended stack

### Android application

```text
Kotlin
Jetpack Compose
Jetpack Ink / stylus handling
WindowManager
TYPE_APPLICATION_OVERLAY
MediaProjectionManager
MediaProjection
VirtualDisplay
ImageReader
Retrofit or Ktor
Coroutines / Flow
```

### Optional library

```text
FloatingX
```

Use it only for overlay ergonomics if it saves time. Keep the screen-capture, coordinate system, tutor pipeline, and annotation Canvas under your control.

### Backend

```text
Python
FastAPI
Pydantic
Vision-capable multimodal model
Pointing/localization model or service
Tutor reasoning layer
Session state
```

---

## 17. Suggested Android project structure

```text
app/
├── MainActivity.kt
│
├── permissions/
│   ├── OverlayPermissionManager.kt
│   └── MediaProjectionPermissionManager.kt
│
├── overlay/
│   ├── TutorOverlayService.kt
│   ├── OverlayManager.kt
│   ├── OverlayMode.kt
│   ├── BubbleOverlay.kt
│   ├── AnnotationOverlay.kt
│   ├── TutorCardOverlay.kt
│   └── OverlayCoordinateMapper.kt
│
├── ink/
│   ├── InkCanvas.kt
│   ├── AnnotationState.kt
│   └── AnnotationRenderer.kt
│
├── capture/
│   ├── ScreenCaptureService.kt
│   ├── MediaProjectionController.kt
│   ├── VirtualDisplayController.kt
│   ├── FrameReader.kt
│   └── FrameComposer.kt
│
├── network/
│   ├── TutorClient.kt
│   └── TutorApi.kt
│
├── tutor/
│   ├── TutorSession.kt
│   └── TutorResponse.kt
│
└── ui/
    ├── setup/
    └── summary/
```

---

## 18. End-to-end runtime sequence

### Session startup

```text
Open tutor app
    ↓
Check SYSTEM_ALERT_WINDOW
    ↓
Request overlay permission if needed
    ↓
Request MediaProjection consent
    ↓
Start mediaProjection foreground service
    ↓
Start overlay controller/service
    ↓
Show small tutor bubble
    ↓
User opens any homework app
```

### Normal monitoring

```text
Other app visible
    ↓
MediaProjection receives frames
    ↓
periodic snapshot / change detection
    ↓
visual understanding
```

### Student asks about a region

```text
Tap tutor bubble
    ↓
enter ANNOTATE mode
    ↓
full-screen transparent Canvas appears
    ↓
student circles something
    ↓
pointer-up / circle complete
    ↓
capture/composite frame
    ↓
exit ANNOTATE mode
    ↓
send frame to backend
```

### AI response

```text
Visual understanding
        +
Pointing service
        ↓
normalized target
        +
hint
        ↓
Android maps point → screen px
        ↓
TutorCardOverlay appears nearby
        ↓
underlying app remains usable
```

---

## 19. MVP implementation order

### Phase 1 — Overlay proof

1. Request `SYSTEM_ALERT_WINDOW`.
2. Create `TutorOverlayService`.
3. Add one `TYPE_APPLICATION_OVERLAY` Compose bubble.
4. Make it draggable using `updateViewLayout()`.
5. Confirm it stays visible over Chrome and another app.

### Phase 2 — Arbitrary positioning

6. Create `TutorCardOverlay`.
7. Give it test normalized coordinates such as `(0.6, 0.4)`.
8. Map them to screen pixels.
9. Verify card placement in portrait and landscape.
10. Add edge/collision clamping.

### Phase 3 — Annotation Canvas

11. Add full-screen transparent `AnnotationCanvasOverlay`.
12. Add stylus drawing.
13. Add PASSIVE/ANNOTATE mode switching.
14. Ensure underlying app works normally when annotation mode is off.
15. Detect annotation completion.

### Phase 4 — Screen capture

16. Add MediaProjection consent.
17. Start `mediaProjection` foreground service.
18. Create VirtualDisplay + ImageReader.
19. Capture a full-screen frame.
20. Verify screenshot dimensions against display dimensions.

### Phase 5 — Composite geometry

21. Render annotation layer at exactly the same pixel dimensions as the captured frame.
22. Composite screenshot + annotation.
23. Upload frame + geometry metadata.
24. Validate normalized backend coordinates against the physical screen.

### Phase 6 — AI pipeline

25. Implement visual-understanding call.
26. Implement pointing service.
27. Return normalized point/bbox.
28. Return structured tutor hint.
29. Render TutorCardOverlay at the target.

### Phase 7 — Polish

30. Add periodic frame change detection.
31. Add session state.
32. Add three strong misconception demos.
33. Add teacher summary.
34. Test Samsung tablet + S Pen specifically.

---

## 20. Hackathon architecture

```text
┌──────────────────────────────────────────────────────────────┐
│                  ANY ANDROID APP                             │
│                                                              │
│    browser / worksheet / PDF / education app                │
│                                                              │
│                    visible content                           │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           │ MediaProjection
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                SCREEN CAPTURE SERVICE                        │
│                                                              │
│ VirtualDisplay → ImageReader → periodic screen frame         │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           │             ┌─────────────────────┐
                           │             │ ANNOTATION OVERLAY  │
                           │             │ transparent Canvas  │
                           │             │ circle / underline  │
                           │             └──────────┬──────────┘
                           │                        │
                           └────────────┬───────────┘
                                        ▼
                                  composite frame
                                        │
                                        │ HTTPS
                                        ▼
┌──────────────────────────────────────────────────────────────┐
│                       BACKEND                                │
│                                                              │
│ Visual Understanding ──┐                                    │
│                        ├── Tutor Reasoning                    │
│ Pointing Service ──────┘        │                             │
│                                 ▼                             │
│                       hint + normalized x/y                  │
└─────────────────────────────────┬────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────┐
│                 TUTOR OVERLAY SERVICE                        │
│                                                              │
│ WindowManager + TYPE_APPLICATION_OVERLAY                     │
│                                                              │
│  ● floating bubble                                           │
│  transparent Ink Canvas                                      │
│  contextual tutor card                                       │
└──────────────────────────────────────────────────────────────┘
```

---

## 21. Important Android limitations

"Works over any app" should be understood as **works over ordinary application windows**, not literally every system surface.

Important cases:

- `TYPE_APPLICATION_OVERLAY` stays below critical system windows such as some status-bar/IME surfaces.
- Android can alter overlay position/visibility in some situations.
- Apps can use Android security mechanisms to suppress non-system overlays on sensitive screens.
- Full-screen touchable overlays block touches to the app underneath; use explicit annotation mode.
- MediaProjection requires visible user consent.
- On Android 14+, every new MediaProjection session requires new consent; keep the active session alive during tutoring.
- Overlay and MediaProjection permissions have Play Store/policy implications and should be justified by the app's core functionality.

For a hackathon prototype on a controlled Samsung tablet, these constraints are manageable.

---

## 22. Final recommendation

Use this combination:

```text
WindowManager + TYPE_APPLICATION_OVERLAY
                ↓
    cross-app tutor UI

MediaProjection + VirtualDisplay + ImageReader
                ↓
    cross-app visual perception

Transparent Jetpack Ink Canvas overlay
                ↓
    student pointing / circling

Normalized screen coordinates
                ↓
    backend ↔ Android spatial contract
```

Do **not** make the tutor's Activity the workspace anymore.

The Activity starts the session; the **Android display itself becomes the workspace**.
