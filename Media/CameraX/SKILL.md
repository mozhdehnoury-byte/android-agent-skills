---
name: camerax
description: >
  CameraX setup and usage for Android camera features.
  Load this skill when implementing photo capture, video recording,
  camera preview, QR scanning, or image analysis with CameraX.
---

# CameraX

## Overview
CameraX is Jetpack's high-level camera library built on Camera2. It handles lifecycle management, device compatibility, and camera session complexity automatically. It's the recommended approach for most camera use cases in Android.

---

## Core Principles

- CameraX is **lifecycle-aware** — bind to a LifecycleOwner, it handles open/close
- Use **use cases** to compose camera behavior — Preview, ImageCapture, VideoCapture, ImageAnalysis
- Maximum **3 use cases** can be active simultaneously — choose wisely
- CameraX handles **device compatibility** automatically — no per-device workarounds
- Always run image analysis on a **background executor**

---

## Setup

```toml
[versions]
camerax = "1.3.4"

[libraries]
camerax-core     = { module = "androidx.camera:camera-core", version.ref = "camerax" }
camerax-camera2  = { module = "androidx.camera:camera-camera2", version.ref = "camerax" }
camerax-lifecycle = { module = "androidx.camera:camera-lifecycle", version.ref = "camerax" }
camerax-video    = { module = "androidx.camera:camera-video", version.ref = "camerax" }
camerax-view     = { module = "androidx.camera:camera-view", version.ref = "camerax" }
camerax-extensions = { module = "androidx.camera:camera-extensions", version.ref = "camerax" }
```

```kotlin
dependencies {
    implementation(libs.camerax.core)
    implementation(libs.camerax.camera2)
    implementation(libs.camerax.lifecycle)
    implementation(libs.camerax.video)
    implementation(libs.camerax.view)
}
```

---

## Preview in Compose

```kotlin
// ✅ CameraX Preview with Compose
@Composable
fun CameraPreview(
    modifier: Modifier = Modifier,
    onImageCaptured: (Uri) -> Unit
) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current

    val previewView = remember { PreviewView(context) }
    val imageCapture = remember { ImageCapture.Builder().build() }

    LaunchedEffect(Unit) {
        val cameraProvider = ProcessCameraProvider.getInstance(context).await()

        val preview = Preview.Builder().build().also {
            it.setSurfaceProvider(previewView.surfaceProvider)
        }

        try {
            cameraProvider.unbindAll()
            cameraProvider.bindToLifecycle(
                lifecycleOwner,
                CameraSelector.DEFAULT_BACK_CAMERA,
                preview,
                imageCapture
            )
        } catch (e: Exception) {
            Log.e("CameraX", "Use case binding failed", e)
        }
    }

    Box(modifier = modifier) {
        AndroidView(factory = { previewView }, modifier = Modifier.fillMaxSize())

        Button(
            onClick = { capturePhoto(context, imageCapture, onImageCaptured) },
            modifier = Modifier.align(Alignment.BottomCenter).padding(16.dp)
        ) {
            Text("Capture")
        }
    }
}
```

---

## Photo Capture

```kotlin
// ✅ Capture photo to file
fun capturePhoto(
    context: Context,
    imageCapture: ImageCapture,
    onImageCaptured: (Uri) -> Unit
) {
    val photoFile = File(
        context.getExternalFilesDir(Environment.DIRECTORY_PICTURES),
        "photo_${System.currentTimeMillis()}.jpg"
    )

    val outputOptions = ImageCapture.OutputFileOptions.Builder(photoFile).build()

    imageCapture.takePicture(
        outputOptions,
        ContextCompat.getMainExecutor(context),
        object : ImageCapture.OnImageSavedCallback {
            override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                val savedUri = output.savedUri ?: Uri.fromFile(photoFile)
                onImageCaptured(savedUri)
            }

            override fun onError(exception: ImageCaptureException) {
                Log.e("CameraX", "Photo capture failed: ${exception.message}", exception)
            }
        }
    )
}

// ✅ Capture to in-memory buffer (no file)
imageCapture.takePicture(
    ContextCompat.getMainExecutor(context),
    object : ImageCapture.OnImageCapturedCallback() {
        override fun onCaptureSuccess(image: ImageProxy) {
            val buffer = image.planes[0].buffer
            val bytes = ByteArray(buffer.remaining())
            buffer.get(bytes)
            image.close()  // ✅ Always close
            processImage(bytes)
        }

        override fun onError(exception: ImageCaptureException) { }
    }
)
```

---

## Image Analysis (QR / ML)

```kotlin
// ✅ Real-time frame analysis
val imageAnalysis = ImageAnalysis.Builder()
    .setTargetResolution(Size(1280, 720))
    .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
    .build()
    .also { analysis ->
        analysis.setAnalyzer(
            Executors.newSingleThreadExecutor()
        ) { imageProxy ->
            analyzeFrame(imageProxy)
            imageProxy.close()  // ✅ Always close
        }
    }

// ✅ QR code scanning with ML Kit
fun analyzeFrame(imageProxy: ImageProxy) {
    val mediaImage = imageProxy.image ?: return
    val image = InputImage.fromMediaImage(mediaImage, imageProxy.imageInfo.rotationDegrees)

    barcodeScanner.process(image)
        .addOnSuccessListener { barcodes ->
            barcodes.firstOrNull()?.rawValue?.let { value ->
                onQrCodeDetected(value)
            }
        }
        .addOnCompleteListener {
            imageProxy.close()
        }
}
```

---

## Video Capture

```kotlin
// ✅ Video recording
private var recording: Recording? = null

fun startRecording(
    context: Context,
    videoCapture: VideoCapture<Recorder>,
    onVideoSaved: (Uri) -> Unit
) {
    val videoFile = File(
        context.getExternalFilesDir(Environment.DIRECTORY_MOVIES),
        "video_${System.currentTimeMillis()}.mp4"
    )

    recording = videoCapture.output
        .prepareRecording(context, FileOutputOptions.Builder(videoFile).build())
        .withAudioEnabled()
        .start(ContextCompat.getMainExecutor(context)) { event ->
            when (event) {
                is VideoRecordEvent.Finalize -> {
                    if (!event.hasError()) {
                        onVideoSaved(event.outputResults.outputUri)
                    }
                }
            }
        }
}

fun stopRecording() {
    recording?.stop()
    recording = null
}

// ✅ Setup VideoCapture use case
val recorder = Recorder.Builder()
    .setQualitySelector(QualitySelector.from(Quality.HD))
    .build()
val videoCapture = VideoCapture.withOutput(recorder)
```

---

## Camera Selector

```kotlin
// ✅ Select camera
CameraSelector.DEFAULT_BACK_CAMERA   // main back camera
CameraSelector.DEFAULT_FRONT_CAMERA  // selfie camera

// ✅ Custom selector
val cameraSelector = CameraSelector.Builder()
    .requireLensFacing(CameraSelector.LENS_FACING_BACK)
    .build()

// ✅ Switch camera
fun switchCamera(currentSelector: CameraSelector): CameraSelector =
    if (currentSelector == CameraSelector.DEFAULT_BACK_CAMERA)
        CameraSelector.DEFAULT_FRONT_CAMERA
    else
        CameraSelector.DEFAULT_BACK_CAMERA
```

---

## Camera Controls

```kotlin
// ✅ Tap to focus
fun tapToFocus(x: Float, y: Float, meteringPointFactory: MeteringPointFactory, camera: Camera) {
    val point = meteringPointFactory.createPoint(x, y)
    val action = FocusMeteringAction.Builder(point, FocusMeteringAction.FLAG_AF)
        .setAutoCancelDuration(3, TimeUnit.SECONDS)
        .build()
    camera.cameraControl.startFocusAndMetering(action)
}

// ✅ Zoom
fun setZoom(camera: Camera, zoomRatio: Float) {
    camera.cameraControl.setZoomRatio(zoomRatio)
}

// ✅ Torch
fun setTorch(camera: Camera, enabled: Boolean) {
    camera.cameraControl.enableTorch(enabled)
}
```

---

## Anti-Patterns

- Not closing `ImageProxy` in analysis — blocks future frames
- Binding more than 3 use cases simultaneously — exceeds hardware limits
- Running image analysis on the main executor — blocks UI
- Not handling `cameraProvider.unbindAll()` before rebinding — stale sessions
- Ignoring `ImageCapture.Builder.setTargetRotation()` — rotated photos

---

## Related Skills
- `camera` — Camera2 for advanced/custom use cases
- `lifecycle` — binding CameraX to lifecycle
- `compose` — CameraX in Compose with AndroidView
- `manifest` — camera permission setup
