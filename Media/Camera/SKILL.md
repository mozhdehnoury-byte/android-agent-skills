---
name: camera
description: >
  Android Camera2 API usage and patterns.
  Load this skill when accessing the camera directly via Camera2 API,
  building custom camera experiences, or when CameraX doesn't meet requirements.
---

# Camera (Camera2)

## Overview
Camera2 is Android's low-level camera API providing full control over capture pipeline, exposure, focus, and output formats. It's complex but powerful. For most use cases, prefer CameraX — use Camera2 only when fine-grained control is required.

---

## Core Principles

- Camera2 is **complex and verbose** — use CameraX unless you need its specific capabilities
- Camera operations must run on a **background thread** — never on main thread
- Always **close** `CameraDevice` and `CameraCaptureSession` when done — resource leaks crash other apps
- Request permissions before accessing camera — `CAMERA` is a runtime permission
- Handle camera **disconnection and errors** — hardware can become unavailable

---

## When to Use Camera2 vs CameraX

| Need | Use |
|------|-----|
| Simple photo/video capture | CameraX |
| Preview + capture | CameraX |
| RAW capture | Camera2 |
| Manual exposure/ISO/shutter | Camera2 |
| Custom image processing pipeline | Camera2 |
| YUV frame access | Camera2 |
| Multi-camera (logical/physical) | Camera2 |

---

## Permission

```kotlin
// AndroidManifest.xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-feature android:name="android.hardware.camera" android:required="true" />

// ✅ Request at runtime
val requestPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { isGranted ->
    if (isGranted) openCamera()
    else showPermissionDeniedMessage()
}

requestPermissionLauncher.launch(Manifest.permission.CAMERA)
```

---

## Camera Manager — Enumerate Cameras

```kotlin
// ✅ List available cameras
val cameraManager = context.getSystemService(CameraManager::class.java)

val cameraIds = cameraManager.cameraIdList
cameraIds.forEach { id ->
    val characteristics = cameraManager.getCameraCharacteristics(id)
    val facing = characteristics.get(CameraCharacteristics.LENS_FACING)
    when (facing) {
        CameraCharacteristics.LENS_FACING_BACK  -> Log.d("Camera", "Back camera: $id")
        CameraCharacteristics.LENS_FACING_FRONT -> Log.d("Camera", "Front camera: $id")
    }
}

// ✅ Find back camera
fun findBackCamera(manager: CameraManager): String? {
    return manager.cameraIdList.firstOrNull { id ->
        manager.getCameraCharacteristics(id)
            .get(CameraCharacteristics.LENS_FACING) == CameraCharacteristics.LENS_FACING_BACK
    }
}
```

---

## Opening Camera

```kotlin
// ✅ Open camera on background thread
private lateinit var cameraDevice: CameraDevice
private val cameraThread = HandlerThread("CameraThread").also { it.start() }
private val cameraHandler = Handler(cameraThread.looper)

@SuppressLint("MissingPermission")
fun openCamera(cameraId: String) {
    cameraManager.openCamera(cameraId, object : CameraDevice.StateCallback() {
        override fun onOpened(camera: CameraDevice) {
            cameraDevice = camera
            createCaptureSession()
        }

        override fun onDisconnected(camera: CameraDevice) {
            camera.close()
        }

        override fun onError(camera: CameraDevice, error: Int) {
            camera.close()
            // Handle error — show user message
        }
    }, cameraHandler)
}
```

---

## Capture Session and Preview

```kotlin
// ✅ Create capture session for preview
private lateinit var captureSession: CameraCaptureSession

fun createCaptureSession(surface: Surface) {
    val targets = listOf(surface)

    cameraDevice.createCaptureSession(
        targets,
        object : CameraCaptureSession.StateCallback() {
            override fun onConfigured(session: CameraCaptureSession) {
                captureSession = session
                startPreview(surface)
            }

            override fun onConfigureFailed(session: CameraCaptureSession) {
                // Handle configuration failure
            }
        },
        cameraHandler
    )
}

// ✅ Start repeating preview request
fun startPreview(surface: Surface) {
    val previewRequest = cameraDevice
        .createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW)
        .apply {
            addTarget(surface)
            set(CaptureRequest.CONTROL_AF_MODE, CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_PICTURE)
            set(CaptureRequest.CONTROL_AE_MODE, CaptureRequest.CONTROL_AE_MODE_ON)
        }
        .build()

    captureSession.setRepeatingRequest(previewRequest, null, cameraHandler)
}
```

---

## Capture Photo

```kotlin
// ✅ Capture still image
fun capturePhoto(imageReader: ImageReader) {
    val captureRequest = cameraDevice
        .createCaptureRequest(CameraDevice.TEMPLATE_STILL_CAPTURE)
        .apply {
            addTarget(imageReader.surface)
            set(CaptureRequest.CONTROL_AF_MODE, CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_PICTURE)
            set(CaptureRequest.JPEG_ORIENTATION, getJpegOrientation())
        }
        .build()

    captureSession.capture(captureRequest, object : CameraCaptureSession.CaptureCallback() {
        override fun onCaptureCompleted(
            session: CameraCaptureSession,
            request: CaptureRequest,
            result: TotalCaptureResult
        ) {
            // Photo captured
        }
    }, cameraHandler)
}

// ✅ Read image from ImageReader
imageReader.setOnImageAvailableListener({ reader ->
    val image = reader.acquireLatestImage()
    val buffer = image.planes[0].buffer
    val bytes = ByteArray(buffer.remaining())
    buffer.get(bytes)
    image.close()  // ✅ Always close image
    savePhoto(bytes)
}, cameraHandler)
```

---

## Cleanup

```kotlin
// ✅ Always release camera resources
fun closeCamera() {
    captureSession.close()
    cameraDevice.close()
    imageReader.close()
}

// ✅ Release in lifecycle
override fun onPause() {
    super.onPause()
    closeCamera()
}
```

---

## Anti-Patterns

- Camera operations on the main thread — causes ANR
- Not closing CameraDevice on error/disconnect — blocks other apps from using camera
- Not closing Image from ImageReader — buffer pool exhaustion, new images drop
- Ignoring orientation — photos appear rotated
- Not handling permission denial gracefully — crashes without permission

---

## Related Skills
- `camerax` — high-level camera API (preferred)
- `manifest` — camera permission declaration
- `coroutine` — background thread management
