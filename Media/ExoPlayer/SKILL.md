---
name: exoplayer
description: >
  ExoPlayer (Media3) setup and usage for advanced media playback on Android.
  Load this skill when implementing audio/video streaming, HLS/DASH playback,
  background audio with MediaSession, playlist management, or custom renderers.
---

# ExoPlayer (Media3)

## Overview
ExoPlayer, now part of AndroidX Media3, is the recommended media player for Android. It supports HTTP streaming (HLS, DASH, SmoothStreaming), local files, playlists, gapless audio, and deep MediaSession integration. It replaces `MediaPlayer` for all but the simplest use cases.

---

## Core Principles

- Use **Media3 ExoPlayer** — not the standalone ExoPlayer library (deprecated)
- Create one `ExoPlayer` instance per playback session — not per screen
- Use **MediaSession** for background audio and system/notification controls
- Release the player when done — `player.release()` frees all resources
- For background audio, use a `MediaSessionService` (foreground service)

---

## Setup

```toml
[versions]
media3 = "1.4.0"

[libraries]
media3-exoplayer     = { module = "androidx.media3:media3-exoplayer", version.ref = "media3" }
media3-exoplayer-hls = { module = "androidx.media3:media3-exoplayer-hls", version.ref = "media3" }
media3-ui            = { module = "androidx.media3:media3-ui", version.ref = "media3" }
media3-session       = { module = "androidx.media3:media3-session", version.ref = "media3" }
media3-datasource-okhttp = { module = "androidx.media3:media3-datasource-okhttp", version.ref = "media3" }
```

```kotlin
dependencies {
    implementation(libs.media3.exoplayer)
    implementation(libs.media3.exoplayer.hls)
    implementation(libs.media3.ui)
    implementation(libs.media3.session)
}
```

---

## Basic Playback

```kotlin
// ✅ Create and configure player
val player = ExoPlayer.Builder(context)
    .setHandleAudioBecomingNoisy(true)  // pause on headphone unplug
    .setAudioAttributes(
        AudioAttributes.Builder()
            .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
            .setUsage(C.USAGE_MEDIA)
            .build(),
        true  // handle audio focus automatically
    )
    .build()

// ✅ Set media item and play
val mediaItem = MediaItem.fromUri("https://example.com/audio.mp3")
player.setMediaItem(mediaItem)
player.prepare()
player.play()

// ✅ Local file
val mediaItem = MediaItem.fromUri(Uri.fromFile(File(context.filesDir, "audio.mp3")))

// ✅ HLS stream
val mediaItem = MediaItem.Builder()
    .setUri("https://example.com/stream.m3u8")
    .setMimeType(MimeTypes.APPLICATION_M3U8)
    .build()
```

---

## Playlist Management

```kotlin
// ✅ Multiple media items
val playlist = listOf(
    MediaItem.fromUri("https://example.com/track1.mp3"),
    MediaItem.fromUri("https://example.com/track2.mp3"),
    MediaItem.fromUri("https://example.com/track3.mp3")
)

player.setMediaItems(playlist)
player.prepare()
player.play()

// ✅ Navigate playlist
player.seekToNextMediaItem()
player.seekToPreviousMediaItem()
player.seekToMediaItem(index = 2)

// ✅ Repeat and shuffle
player.repeatMode = Player.REPEAT_MODE_ALL   // repeat playlist
player.shuffleModeEnabled = true
```

---

## Player Controls in Compose

```kotlin
// ✅ PlayerView in Compose
@Composable
fun VideoPlayer(videoUri: Uri) {
    val context = LocalContext.current

    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(videoUri))
            prepare()
            playWhenReady = true
        }
    }

    DisposableEffect(Unit) {
        onDispose { player.release() }
    }

    AndroidView(
        factory = {
            PlayerView(it).apply {
                this.player = player
                useController = true
            }
        },
        modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
    )
}
```

---

## Player Listeners

```kotlin
// ✅ Listen to playback state changes
player.addListener(object : Player.Listener {

    override fun onPlaybackStateChanged(playbackState: Int) {
        when (playbackState) {
            Player.STATE_IDLE      -> { /* not started */ }
            Player.STATE_BUFFERING -> showBufferingIndicator()
            Player.STATE_READY     -> hideBufferingIndicator()
            Player.STATE_ENDED     -> onPlaybackEnded()
        }
    }

    override fun onIsPlayingChanged(isPlaying: Boolean) {
        updatePlayPauseButton(isPlaying)
    }

    override fun onPlayerError(error: PlaybackException) {
        Log.e("ExoPlayer", "Playback error: ${error.message}")
        showError(error.message)
    }

    override fun onMediaItemTransition(mediaItem: MediaItem?, reason: Int) {
        updateNowPlayingUI(mediaItem)
    }
})
```

---

## Background Audio with MediaSession

```kotlin
// ✅ MediaSessionService for background audio
@AndroidEntryPoint
class PlaybackService : MediaSessionService() {

    private lateinit var player: ExoPlayer
    private lateinit var mediaSession: MediaSession

    override fun onCreate() {
        super.onCreate()
        player = ExoPlayer.Builder(this)
            .setHandleAudioBecomingNoisy(true)
            .build()

        mediaSession = MediaSession.Builder(this, player)
            .setCallback(MediaSessionCallback())
            .build()
    }

    override fun onGetSession(controllerInfo: MediaSession.ControllerInfo): MediaSession =
        mediaSession

    override fun onDestroy() {
        mediaSession.release()
        player.release()
        super.onDestroy()
    }

    private inner class MediaSessionCallback : MediaSession.Callback {
        override fun onAddMediaItems(
            mediaSession: MediaSession,
            controller: MediaSession.ControllerInfo,
            mediaItems: List<MediaItem>
        ): ListenableFuture<List<MediaItem>> = Futures.immediateFuture(mediaItems)
    }
}

// AndroidManifest.xml
// <service android:name=".PlaybackService"
//          android:exported="true"
//          android:foregroundServiceType="mediaPlayback">
//     <intent-filter>
//         <action android:name="androidx.media3.session.MediaSessionService"/>
//     </intent-filter>
// </service>
```

---

## Custom OkHttp Data Source

```kotlin
// ✅ Use OkHttp for network requests (auth headers, custom interceptors)
val dataSourceFactory = OkHttpDataSource.Factory(okHttpClient)
    .setDefaultRequestProperties(mapOf("Authorization" to "Bearer $token"))

val player = ExoPlayer.Builder(context)
    .setMediaSourceFactory(
        DefaultMediaSourceFactory(context).setDataSourceFactory(dataSourceFactory)
    )
    .build()
```

---

## Anti-Patterns

- Creating a new `ExoPlayer` instance per screen — expensive, loses state on navigation
- Not calling `player.release()` — holds audio/video hardware
- Not setting `setHandleAudioBecomingNoisy(true)` — audio leaks to speaker on unplug
- Background audio without `MediaSessionService` — system kills playback
- Using legacy ExoPlayer library instead of Media3 — deprecated

---

## Related Skills
- `audio` — basic audio playback with MediaPlayer
- `video` — basic video playback
- `foreground-service` — required for background audio
- `notification` — media notification controls
- `lifecycle` — releasing player with lifecycle
