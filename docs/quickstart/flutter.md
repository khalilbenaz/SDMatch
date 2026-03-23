# Flutter — Quick Start

Integrer SDK MATCH dans votre application Flutter en 5 minutes.

## Pre-requis

- Flutter 3.10+
- Dart SDK 3.0+
- Android API 26+ / iOS 15+

## Installation

```yaml
# pubspec.yaml
dependencies:
  sdkmatch:
    path: sdk/flutter  # ou depuis pub.dev quand disponible
```

```bash
flutter pub get
```

## Permissions

### Android (`android/app/src/main/AndroidManifest.xml`)
```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.INTERNET" />
```

### iOS (`ios/Runner/Info.plist`)
```xml
<key>NSCameraUsageDescription</key>
<string>Camera requise pour le KYC</string>
```

## Utilisation

```dart
import 'package:sdkmatch/sdkmatch.dart';

// 1. Activer la licence
final activation = await SdkMatch.activate('API_KEY', 'API_SECRET');
if (!activation.success) {
  print('Erreur: ${activation.message}');
  return;
}

// 2. Verifier la licence
final isLicenced = await SdkMatch.isLicenced();
final status = await SdkMatch.getStatus(); // LicenceStatus.valid, .expired, etc.

// 3. Detecter le document
final detection = await DocumentDetector.detect(ocrText);
print('Type: ${detection.type}');
print('Champs: ${detection.fields}');

// 4. Parser la MRZ
final mrz = await MrzParser.parse(ocrText);
if (mrz != null && mrz.isValid) {
  print('${mrz.lastName} ${mrz.firstName} — CIN: ${mrz.cinNumber}');
}

// 5. Qualite image
final quality = await ImageQuality.assess(imageBytes);
if (!quality.isAcceptable) {
  print('Image floue ou mal eclairee');
  return;
}

// 6. Face Matching
final match = await FaceComparison.compare(documentBytes, selfieBytes);
if (match.isMatch && match.liveness.isLive) {
  print('Identite verifiee ! Score: ${match.score}');
}
```

## Exemple complet

Voir `sdk/flutter/example/lib/main.dart` pour un exemple de flux KYC complet avec camera.

## Voir aussi

- [Reference API](../api-reference/README.md)
- [Source Flutter Plugin](../../sdk/flutter/)
