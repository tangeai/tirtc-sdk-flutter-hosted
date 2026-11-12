# tirtc_flutter

Flutter client plugin for TiRTC remote audio/video and Ti Cloud Storage cloud-recording playback.


## Install

Choose an exact version from the [Hosted releases](https://github.com/tangeai/tirtc-sdk-flutter-hosted/releases), then replace `<version>` below:

```yaml
dependencies:
  tirtc_flutter:
    hosted: https://tangeai.github.io/tirtc-sdk-flutter-hosted
    version: <version>
```

Run `flutter pub get` and commit the resulting `pubspec.lock`. The lock pins the source, version and archive SHA-256. CI can use `flutter pub get --enforce-lockfile`.

The package includes all four native SDKs. Public installs need no publishing token and do not require changing `PUB_HOSTED_URL`; other dependencies keep their normal source. A wrapper that depends on this custom Hosted source cannot be published unchanged to pub.dev. RAPID `rapid_kit` can use its own Hosted URL in the same application.

Versions published through `2.6.4-alpha.1` retain their [original Git tags](https://github.com/tangeai/tirtc-sdk-flutter/releases). To migrate, replace the Git dependency with the Hosted dependency above and regenerate and commit the lock file. The two sources have separate package identities.

## Import

```dart
import 'package:tirtc_flutter/tirtc_flutter.dart';
```

## Platforms

- Android
- iOS arm64 device（行为资格门禁）与 Apple Silicon iOS Simulator（最低 iOS 14.0，编译及下行音视频主流程）
- macOS
- OpenHarmony / HarmonyOS NEXT

No separate TiRTC Maven, CocoaPods, or OHPM dependency is required. The package contains the Android AAR/POM, TiRTC and WebRTC XCFrameworks for iOS and macOS, and the OHOS HAR. `native-manifest.json` records their version, platform scope, size, SHA-256, source identity, and compliance links. These native components keep their own license terms; the package `LICENSE` and manifest identify the included material.

An Apple host that also declares the same TiRTC CocoaPod must select one framework owner. Set `TIRTC_DARWIN_NATIVE_OWNER=direct` and `TIRTC_DARWIN_NATIVE_VERSION=2.6.2` in the Podfile before Flutter installs plugin pods, then point `pod 'TiRTC'` at the package's platform `Frameworks` directory. The plugin changes to an exact `TiRTC` dependency and does not vendor a second copy. A different direct version fails during Podfile evaluation with a `Conflicting TiRTC Darwin version` error.

The iOS closure includes arm64 device and Apple Silicon arm64 simulator slices. The macOS TiRTC and WebRTC frameworks are Universal and contain arm64 and x86_64 slices. Flutter 2.6.2 requires macOS 12.0 or later.

## Minimal Downlink Flow

The client downlink path is:

1. Initialize the shared runtime with `TiRtc.init(...)`.
2. Create a `TiRtcConn`.
3. Create `TiRtcAudioOutput` and `TiRtcVideoOutput`.
4. Connect with `remoteId` and `token`.
5. Attach audio and video outputs to the target stream ids.
6. Mount `videoOutput.view()` in your widget tree.
7. On teardown, detach outputs, disconnect, dispose objects, then call `TiRtc.shutdown()` when the runtime is no longer needed.

```dart
import 'dart:async';

import 'package:flutter/material.dart';
import 'package:tirtc_flutter/tirtc_flutter.dart';

class TiRtcDownlinkView extends StatefulWidget {
  const TiRtcDownlinkView({
    super.key,
    required this.appId,
    required this.remoteId,
    required this.token,
    this.endpoint = '',
    this.audioStreamId = 10,
    this.videoStreamId = 11,
  });

  final String appId;
  final String endpoint;
  final String remoteId;
  final String token;
  final int audioStreamId;
  final int videoStreamId;

  @override
  State<TiRtcDownlinkView> createState() => _TiRtcDownlinkViewState();
}

class _TiRtcDownlinkViewState extends State<TiRtcDownlinkView> {
  final TiRtcConn _connection = TiRtcConn();
  final TiRtcAudioOutput _audioOutput = TiRtcAudioOutput();
  final TiRtcVideoOutput _videoOutput = TiRtcVideoOutput();

  String _status = 'initializing';

  @override
  void initState() {
    super.initState();
    unawaited(_start());
  }

  Future<void> _start() async {
    final int initCode = await TiRtc.init(
      TiRtcInitOptions(
        appId: widget.appId,
        endpoint: widget.endpoint,
        consoleLogEnabled: true,
      ),
    );
    if (initCode != 0) {
      _setStatus('initialize failed: ${TiRtc.formatError(initCode)}');
      return;
    }

    final int connectCode = _connection.connect(
      remoteId: widget.remoteId,
      token: widget.token,
    );
    if (connectCode != 0) {
      _stop();
      _setStatus('connect failed: ${TiRtc.formatError(connectCode)}');
      return;
    }

    final int audioCode = _audioOutput.attach(
      connection: _connection,
      streamId: widget.audioStreamId,
    );
    if (audioCode != 0) {
      _stop();
      _setStatus('audio attach failed: ${TiRtc.formatError(audioCode)}');
      return;
    }

    final int videoCode = _videoOutput.attach(
      connection: _connection,
      streamId: widget.videoStreamId,
    );
    if (videoCode != 0) {
      _stop();
      _setStatus('video attach failed: ${TiRtc.formatError(videoCode)}');
      return;
    }

    _setStatus('playing');
  }

  void _setStatus(String status) {
    if (!mounted) {
      return;
    }
    setState(() {
      _status = status;
    });
  }

  void _stop() {
    _videoOutput.detach();
    _audioOutput.detach();
    _connection.disconnect();
  }

  @override
  void dispose() {
    _stop();
    final int videoDisposeCode = _videoOutput.dispose();
    final int audioDisposeCode = _audioOutput.dispose();
    final int connectionDisposeCode = _connection.dispose();
    final int shutdownCode = TiRtc.shutdown();
    if (videoDisposeCode != 0 ||
        audioDisposeCode != 0 ||
        connectionDisposeCode != 0 ||
        shutdownCode != 0) {
      debugPrint(
        'TiRTC teardown failed: video=$videoDisposeCode '
        'audio=$audioDisposeCode connection=$connectionDisposeCode '
        'shutdown=$shutdownCode',
      );
    }
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Stack(
      fit: StackFit.expand,
      children: <Widget>[
        Center(child: _videoOutput.view()),
        if (_status != 'playing')
          Center(
            child: Text(_status),
          ),
      ],
    );
  }
}
```

If your app keeps one TiRTC runtime for multiple pages, initialize it once at the app or session boundary and call `TiRtc.shutdown()` only after every runtime-backed object has been released.

## Local Audio Capture And Send

Use `TiRtcAudioInput` when the app needs to capture microphone audio and send it through an existing connection. The app is responsible for checking and requesting microphone permission before calling `start()`.

The example app uses stream id `14` for local audio by default. You can use another stream id in the `0..15` range as long as the remote side listens on the same id.

Supported local audio input formats:

- media: integer Nano audio number; see the table below
- sample rate: `TiRtcAudioSampleRate.rate8k` or `TiRtcAudioSampleRate.rate16k`
- channel count: `TiRtcAudioChannelCount.mono`
- AEC, AGC, and ANS are configured before start through `TiRtcAudioInputOptions`

Configure, attach, and start the input after the connection is available:

```dart
final TiRtcAudioInput audioInput = TiRtcAudioInput();

final int optionsCode = await audioInput.setOptions(
  const TiRtcAudioInputOptions(
    media: 2,
    sampleRate: TiRtcAudioSampleRate.rate16k,
    channels: TiRtcAudioChannelCount.mono,
    aecMode: TiRtcAudioAecMode.disabled,
    agcLevel: TiRtcAudioAgcLevel.disabled,
    ansLevel: TiRtcAudioAnsLevel.disabled,
  ),
);
if (optionsCode != 0) {
  debugPrint('audio input options failed: ${TiRtc.formatError(optionsCode)}');
  return;
}

final int attachCode = await audioInput.attach(
  connection: _connection,
  streamId: 14,
);
if (attachCode != 0) {
  debugPrint('audio input attach failed: ${TiRtc.formatError(attachCode)}');
  return;
}

final int startCode = await audioInput.start();
if (startCode != 0) {
  debugPrint('audio input start failed: ${TiRtc.formatError(startCode)}');
}
```

`stop()` stops capture and transmission but keeps the connection and stream binding. Call `start()` again to reuse the same binding. When the page or session ends, stop first, detach from the connection, then dispose the input:

```dart
await audioInput.stop();
await audioInput.detach(connection: _connection);
final int disposeCode = await audioInput.dispose();
if (disposeCode != 0) {
  debugPrint('audio input dispose failed: ${TiRtc.formatError(disposeCode)}');
}
```

Changing media, sample rate, channel count, or 3A settings while the input is running returns an in-use error. Stop the input before applying new options.

## Debug Snapshots

`TiRtcAudioOutput.getDebugSnapshot()` and `TiRtcVideoOutput.getDebugSnapshot()` expose best-effort runtime facts for diagnostic UI and logs. They are query APIs, not control APIs: use `TiRtcVideoOutput.setOptions(...)` before `attach(...)` when you need to request a decoder preference.

```dart
final TiRtcVideoOutputDebugSnapshotResult videoDebug = _videoOutput.getDebugSnapshot();
if (videoDebug.code == 0) {
  final TiRtcVideoOutputDebugSnapshot snapshot = videoDebug.snapshot!;
  debugPrint(
    'video ${snapshot.width}x${snapshot.height} '
    'codec=${snapshot.codec} backend=${snapshot.resolvedDecoderBackend}',
  );
}
```

When `code == 0`, `snapshot` is always non-null, even if every field is still zero because the runtime has not observed the corresponding fact yet. When `code != 0`, `snapshot` is null. Calling a debug snapshot method does not trigger output `onError` callbacks and does not change output state.

## Saving Temporary Media

RTC recordings, RTC snapshots, Ti Cloud Storage recordings, and Ti Cloud Storage snapshots return temporary cache file objects. Save one with an optional final filename:

```dart
final asset = await file.moveToGallery(
  fileName: '客厅角落摄像头-2026-08-20-14-30-05-123.mp4',
);
```

Android, iOS, and OHOS publish to the system media library. macOS saves directly to the user's Downloads directory. The plugin does not declare or request gallery permissions; the app must prepare Android or iOS authorization before calling this method. On macOS, a sandboxed app must enable Downloads read-write access.

Omit `fileName` to use an epoch-millisecond name. The plugin appends the required `.mp4` or `.jpg` extension when it is absent. An invalid name completes with code `6000` without publishing or deleting the cache file. A successful move deletes the cache source and returns an opaque asset URI; a failed move keeps the source available for retry.

## Token Service Used By The Example

The bundled example viewer asks a token signing HTTP service for a TiRTC token before connecting. During development, start the service with TiRTC DevTools CLI and paste the service URL into the viewer configuration page. The same field also accepts a URL that directly returns a one-time `v1...` token.

Viewer configuration fields:

```text
app_id: your-app-id
endpoint: https://your-service-endpoint
remote_id: remote-device-id
Token signing service address: http://127.0.0.1:8966
audio stream id: optional-audio-stream-id
video stream id: optional-video-stream-id
```

For a standard signing service base URL, the viewer appends the fixed token path and sends this request before `connect(remoteId, token)`:

```http
POST /v1/tokens
Content-Type: application/json

{"remote_id":"remote-device-id"}
```

The token service returns the token used for the SDK connection. The response may be a JSON object with a `token` string or a plain `v1...` token body. If the configured address already contains a custom path or query, the viewer sends a `GET` request to that exact URL and applies the returned token. The example stores the viewer configuration locally after editing so the same values are available next time.

## Logs

Use `TiRtcLogging.upload()` when you need a log id for diagnostics:

```dart
final ({int code, String? logId}) result = await TiRtcLogging.upload();
```

`code == 0` means the upload request succeeded. Share the returned `logId` with the TiRTC support channel when investigating connection, audio, or video issues.

## Example

The package includes a Flutter example with configuration, settings, and output pages:

- viewer configuration page: enter `app_id`, `endpoint`, `remote_id`, token signing service URL, and stream ids
- settings page: choose video decoder preference, console output behavior, and talkback audio settings
- output page: connects automatically, renders one remote video output, plays one remote audio output, provides a microphone start / stop button, and provides log upload plus metrics and debug facts

Run it from the package example directory:

```sh
cd example
flutter pub get
flutter run
```

## Documentation

Check [Ti RTC Doc](https://docs.tange.ai/products/tirtc/) for platform setup and service-side token guidance.

## Microphone media configuration

`TiRtcAudioInputOptions.media` accepts the integer audio media number returned by
Nano device capabilities:

| media | Audio format |
| --- | --- |
| 1 | PCM |
| 2 | G711A (A-law) |
| 3 | AAC |
| 4 | Opus |
| 5 | AMR (8 kHz mono) |

```dart
final code = await input.setOptions(const TiRtcAudioInputOptions(media: 2));
```

Omitting `media` or setting it to `-1` uses the deprecated `codec` field, whose
default remains G711A. Existing AAC calls keep selecting AAC. An explicit supported
`media` takes precedence over `codec`; any other value returns the existing
parameter error and leaves the accepted configuration intact. Stop capture before
applying new options, and follow the input's existing detach requirements.

The deprecated audio codec values keep their original meanings. Codec fields on
received frames and metrics retain their output numbering; they are not Nano
input media numbers.
