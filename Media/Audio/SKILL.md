---
name: audio
description: >
  Android audio playback and recording patterns.
  Load this skill when playing audio files, recording from microphone,
  managing audio focus, or handling audio streams in Android.
---

# Audio

## Overview
Android provides multiple APIs for audio — `MediaPlayer` for simple playback, `AudioTrack` for low-latency PCM, `MediaRecorder` for recording, and `AudioRecord` for raw PCM capture. Choosing the right API depends on the use case: streaming, local file, or real-time processing.

---

## Core Principles

- Always **request audio focus** before playback — respect other apps
- **Release resources** (`MediaPlayer`, `MediaRecorder`) when done — they hold hardware
- Audio operations that block belong on **Dispatchers.IO**
- Handle **audio output changes** (headphone unplug, Bluetooth) — pause when needed
- Declare `RECORD_AUDIO` permission for recording — runtime permission required

---

## Permissions

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.RECORD_AUDIO" />
```

---

## MediaPlayer — Simple Playback

```kotlin
// ✅ Play from resource
class AudioPlayer(private val context: Context) {

    private var mediaPlayer: MediaPlayer? = null

    fun playFromResource(@RawRes resId: Int) {
        release()
        mediaPlayer = MediaPlayer.create(context, resId).apply {
            setOnCompletionListener { release() }
            start()
        }
    }

    // ✅ Play from URL (streaming)
    fun playFromUrl(url: String) {
        release()
        mediaPlayer = MediaPlayer().apply {
            setAudioAttributes(
                AudioAttributes.Builder()
                    .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
                    .setUsage(AudioAttributes.USAGE_MEDIA)
                    .build()
            )
            setDataSource(url)
            setOnPreparedListener { start() }
            setOnErrorListener { _, what, extra ->
                Log.e("AudioPlayer", "Error: what=$what extra=$extra")
                true
            }
            prepareAsync()  // ✅ Non-blocking prepare for streams
        }
    }

    fun pause() { mediaPlayer?.pause() }
    fun resume() { mediaPlayer?.start() }
    fun stop() { mediaPlayer?.stop() }

    // ✅ Always release — frees hardware resource
    fun release() {
        mediaPlayer?.release()
        mediaPlayer = null
    }
}
```

---

## Audio Focus

```kotlin
// ✅ Request audio focus before playback
class AudioFocusManager(private val context: Context) {

    private val audioManager = context.getSystemService(AudioManager::class.java)
    private var focusRequest: AudioFocusRequest? = null

    fun requestFocus(onGranted: () -> Unit, onLost: () -> Unit): Boolean {
        val request = AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN)
            .setAudioAttributes(
                AudioAttributes.Builder()
                    .setUsage(AudioAttributes.USAGE_MEDIA)
                    .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
                    .build()
            )
            .setOnAudioFocusChangeListener { focusChange ->
                when (focusChange) {
                    AudioManager.AUDIOFOCUS_LOSS           -> onLost()
                    AudioManager.AUDIOFOCUS_LOSS_TRANSIENT -> onLost()  // pause temporarily
                    AudioManager.AUDIOFOCUS_GAIN           -> onGranted() // resume
                }
            }
            .build()

        focusRequest = request
        val result = audioManager.requestAudioFocus(request)
        return result == AudioManager.AUDIOFOCUS_REQUEST_GRANTED
    }

    fun abandonFocus() {
        focusRequest?.let { audioManager.abandonAudioFocusRequest(it) }
        focusRequest = null
    }
}
```

---

## Handle Headphone Unplug

```kotlin
// ✅ Pause when headphones are disconnected
class NoisyReceiver(private val onBecomingNoisy: () -> Unit) : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == AudioManager.ACTION_AUDIO_BECOMING_NOISY) {
            onBecomingNoisy()  // pause playback
        }
    }
}

// Register/unregister with lifecycle
override fun onStart() {
    super.onStart()
    val filter = IntentFilter(AudioManager.ACTION_AUDIO_BECOMING_NOISY)
    registerReceiver(noisyReceiver, filter)
}

override fun onStop() {
    unregisterReceiver(noisyReceiver)
    super.onStop()
}
```

---

## MediaRecorder — Simple Recording

```kotlin
// ✅ Record audio to file
class AudioRecorder(private val context: Context) {

    private var recorder: MediaRecorder? = null

    fun startRecording(outputFile: File) {
        recorder = MediaRecorder(context).apply {
            setAudioSource(MediaRecorder.AudioSource.MIC)
            setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
            setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
            setAudioSamplingRate(44100)
            setAudioEncodingBitRate(128_000)
            setOutputFile(outputFile.absolutePath)
            prepare()
            start()
        }
    }

    fun stopRecording(): File? {
        return try {
            recorder?.apply {
                stop()
                release()
            }
            recorder = null
            outputFile
        } catch (e: Exception) {
            null
        }
    }

    fun release() {
        recorder?.release()
        recorder = null
    }
}
```

---

## AudioRecord — Raw PCM Capture

```kotlin
// ✅ For real-time audio processing (speech recognition, audio analysis)
class RawAudioCapture {

    private val sampleRate = 44100
    private val bufferSize = AudioRecord.getMinBufferSize(
        sampleRate,
        AudioFormat.CHANNEL_IN_MONO,
        AudioFormat.ENCODING_PCM_16BIT
    )

    private var audioRecord: AudioRecord? = null
    private var isRecording = false

    fun startCapture(onAudioData: (ShortArray) -> Unit) {
        audioRecord = AudioRecord(
            MediaRecorder.AudioSource.MIC,
            sampleRate,
            AudioFormat.CHANNEL_IN_MONO,
            AudioFormat.ENCODING_PCM_16BIT,
            bufferSize
        )

        isRecording = true
        audioRecord?.startRecording()

        Thread {
            val buffer = ShortArray(bufferSize)
            while (isRecording) {
                val read = audioRecord?.read(buffer, 0, buffer.size) ?: 0
                if (read > 0) onAudioData(buffer.copyOf(read))
            }
        }.start()
    }

    fun stopCapture() {
        isRecording = false
        audioRecord?.stop()
        audioRecord?.release()
        audioRecord = null
    }
}
```

---

## Anti-Patterns

- Not releasing `MediaPlayer` or `MediaRecorder` — holds hardware, other apps can't use it
- Playing audio without requesting audio focus — overlaps with other apps' audio
- `prepare()` instead of `prepareAsync()` for network streams — blocks main thread
- Not handling `AudioManager.ACTION_AUDIO_BECOMING_NOISY` — audio leaks from speaker
- Recording without checking/requesting `RECORD_AUDIO` permission

---

## Related Skills
- `exoplayer` — advanced media playback
- `foreground-service` — background audio playback
- `lifecycle` — releasing audio resources with lifecycle
- `manifest` — audio permission declaration
