# Flutter Quick Start

Get the KYC SDK running in your Flutter app in under 5 minutes.

## Requirements

- Flutter 3.16+
- Dart 3.2+
- iOS 14.0+ / Android API 24+

## Installation

Add the SDK to your project:

```bash
flutter pub add kyc_sdk
```

Or add it manually to `pubspec.yaml`:

```yaml
dependencies:
  kyc_sdk: ^1.0.0
```

Then run:

```bash
flutter pub get
```

## Camera Permission

### Android

In `android/app/src/main/AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-feature android:name="android.hardware.camera" android:required="true" />
```

### iOS

In `ios/Runner/Info.plist`:

```xml
<key>NSCameraUsageDescription</key>
<string>Camera access is required for identity verification.</string>
```

Use the `permission_handler` package to request camera permission at runtime:

```dart
import 'package:permission_handler/permission_handler.dart';

final status = await Permission.camera.request();
if (!status.isGranted) {
  // Handle permission denied
}
```

## Quick Start

```dart
import 'dart:typed_data';
import 'package:kyc_sdk/kyc_sdk.dart';

// 1. Create and initialize the SDK
final sdk = KycSdk();
sdk.initialize(KycConfig(
  modelsDirectory: '$appDocDir/models',
  licencePath: '$appDocDir/licence.json',
  templatesDirectory: '$appDocDir/templates',
));

// 2. Start a session
final session = sdk.startSession();

// 3. Scan an identity document (raw RGB Uint8List)
final Uint8List docPixels = /* captured document as RGB bytes */;
final docResult = sdk.scanDocument(
  session: session, pixels: docPixels, width: 1920, height: 1080,
);
print('Document type: ${docResult.type}, fields: ${docResult.fields}');

// 4. Capture a selfie (raw RGB Uint8List)
final Uint8List selfiePixels = /* captured selfie as RGB bytes */;
final selfieResult = sdk.captureSelfie(
  session: session, pixels: selfiePixels, width: 1080, height: 1920,
);
print('Glasses: ${selfieResult.hasGlasses}, sharpness: ${selfieResult.sharpnessScore}');

// 5. Verify (face match)
final result = sdk.verify(session: session);
switch (result.status) {
  case KycStatus.verified:        print('Verified! Score: ${result.faceMatch.score}');
  case KycStatus.retrySelfie:     print('Please retake your selfie.');
  case KycStatus.rescanDocument:  print('Please rescan your document.');
  case KycStatus.failed:          print('Verification failed.');
}

// 6. Clean up
sdk.endSession(session);
sdk.dispose();
```

## Error Handling

SDK methods throw `KycError` on failure. Use try/catch to handle errors:

```dart
try {
  final docResult = sdk.scanDocument(
    session: session, pixels: docPixels, width: w, height: h,
  );
  print('Scanned ${docResult.type}');
} on KycError catch (e) {
  switch (e.code) {
    case KycErrorCode.documentNotDetected:
      print('No document found in image');
    case KycErrorCode.documentBlurry:
      print('Image is too blurry');
    case KycErrorCode.documentGlare:
      print('Too much glare on document');
    default:
      print('Error: ${e.message}');
  }
}
```

## Next Steps

- [API Reference](../api-reference/README.md) for the full list of methods, types, and error codes.
- [Full Flow Example](../examples/full-flow.md) for a production-ready implementation with retry logic.
