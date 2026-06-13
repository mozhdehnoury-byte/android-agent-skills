---
name: video
description: >
  Android video playback and capture patterns.
  Load this skill when playing local or streaming video, capturing video,
  or handling video output on Android. For advanced playback use ExoPlayer.
---

# Video

## Overview
Android provides `VideoView` and `MediaPlayer` for simple local video playback, and `MediaRecorder` for video capture. For streaming, adaptive bitrate, or advanced playback, use ExoPlayer. This skill covers the foundational video APIs.

---

## Core Principles

- Use **ExoPlayer** for streaming video, adaptive bitrate (HLS/DASH), or any non-trivial playback
- Use `VideoView` only for simple local file playback with minimal customization
- Video capture requires **both** `CAMERA` and `RECORD_AUDIO` permissions
- Always **release** `MediaPlayer`/`MediaRecorder` — they hold exclusive hardware
- Handle **lifecycle** — pause video when app goes to background

---

## VideoView — Simple Playback

```kotlin
// ✅ Play local video in XML layout
// layout/fragment_video.xml
// <VideoView android:id="@+id/videoView" ... />

// In Fragment
val videoView = binding.videoView
val mediaController = MediaController(requireContext())
mediaController.setAnchorView(videoView)

videoView.apply {
    setMediaController(mediaController)
    setVideoURI(Uri.parse("android.resource://${context.packageName}/${R.raw.sample_video}"))
    setOnPreparedListener { start() }
    setOnCompletionListener { /* handle completion */ }
    setOnErrorListener { _, what, extra ->
        Log.e("VideoView", "Error: what=$what extra=$extra")
        true
    }
    requestFocus()
}
```

---

## MediaPlayer — Programmatic Video Playback

```kotlin
// ✅ VideoView alternative with more control
class VideoPlayer(private val context: Context) {

    private var mediaPlayer: MediaPlayer? = null

    fun playOnSurface(surface: Surface, videoUri: Uri) {
        release()
        mediaPlayer = MediaPlayer().apply {
            setAudioAttributes(
                AudioAttributes.Builder()
                    .setContentType(AudioAttributes.CONTENT_TYPE_MOVIE)
                    .setUsage(AudioAttributes.USAGE_MEDIA)
                    .build()
            )
            setDataSource(context, videoUri)
            setSurface(surface)
            setOnPreparedListener { start() }
            prepareAsync()
        }
    }

    fun pause()   { mediaPlayer?.pause() }
    fun resume()  { mediaPlayer?.start() }

    fun release() {
        mediaPlayer?.release()
        mediaPlayer = null
    }

    val currentPosition: Int get() = mediaPlayer?.currentPosition ?: 0
    val duration: Int get() = mediaPlayer?.duration ?: 0
}
```

---

## Video Capture with CameraX

```kotlin
// ✅ Preferred — use CameraX VideoCapture use case
// See camerax skill for full setup

val recorder = Recorder.Builder()
    .setQualitySelector(
        QualitySelector.fromOrderedList(
            listOf(Quality.FHD, Quality.HD, Quality.SD),
            FallbackStrategy.lowerQualityOrHigherThan(Quality.SD)
        )
    )
    .build()

val videoCapture = VideoCapture.withOutput(recorder)

// Start recording
var recording: Recording? = null

fun startRecording(context: Context, outputFile: File) {
    recording = videoCapture.output
        .prepareRecording(context, FileOutputOptions.Builder(outputFile).build())
        .withAudioEnabled()
        .start(ContextCompat.getMainExecutor(context)) { event ->
            when (event) {
                is VideoRecordEvent.Start    -> onRecordingStarted()
                is VideoRecordEvent.Finalize -> {
                    if (!event.hasError()) onRecordingSaved(event.outputResults.outputUri)
                    else onRecordingError(event.error)
                }
            }
        }
}

fun stopRecording() {
    recording?.stop()
    recording = null
}
```

---

## Video Capture with MediaRecorder (Legacy)

```kotlin
// ✅ Use when CameraX is not available or Camera2 integration is needed
class LegacyVideoRecorder(private val context: Context) {

    private var mediaRecorder: MediaRecorder? = null

    fun startRecording(surface: Surface, outputFile: File) {
        mediaRecorder = MediaRecorder(context).apply {
            setAudioSource(MediaRecorder.AudioSource.MIC)
            setVideoSource(MediaRecorder.VideoSource.SURFACE)
            setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
            setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
            setVideoEncoder(MediaRecorder.VideoEncoder.H264)
            setVideoSize(1920, 1080)
            setVideoFrameRate(30)
            setVideoEncodingBitRate(10_000_000)
            setOutputFile(outputFile.absolutePath)
            prepare()
        }
        mediaRecorder?.start()
    }

    fun stopRecording() {
        mediaRecorder?.apply {
            stop()
            release()
        }
        mediaRecorder = null
    }
}
```

---

## Thumbnail Generation

```kotlin
// ✅ Generate video thumbnail
suspend fun getVideoThumbnail(context: Context, videoUri: Uri): Bitmap? {
    return withContext(Dispatchers.IO) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            context.contentResolver.loadThumbnail(videoUri, Size(640, 360), null)
        } else {
            MediaMetadataRetriever().use { retriever ->
                retriever.setDataSource(context, videoUri)
                retriever.getFrameAtTime(0)
            }
        }
    }
}
```

---

## Lifecycle Handling

```kotlin
// ✅ In Fragment — pause/resume with lifecycle
override fun onPause() {
    super.onPause()
    videoPlayer.pause()
}

override fun onResume() {
    super.onResume()
    videoPlayer.resume()
}

override fun onDestroyView() {
    videoPlayer.release()
    super.onDestroyView()
}
```

---

## Anti-Patterns

- Using `VideoView` for streaming — no buffering control, poor UX; use ExoPlayer
- Not releasing `MediaPlayer`/`MediaRecorder` — hardware lock
- Video playback in background without a `ForegroundService` — system kills it
- Capturing video without checking both `CAMERA` and `RECORD_AUDIO` permissions
- Blocking main thread with `prepare()` — always use `prepareAsync()`

---

## Related Skills
- `exoplayer` — advanced video/audio playback
- `camerax` — video capture via CameraX
- `audio` — audio-only recording and playback
- `foreground-service` — background video playback
- `lifecycle` — releasing video resources on lifecycle events
