# React Native — Quick Start

Integrer SDK MATCH dans votre application React Native en 5 minutes.

## Pre-requis

- React Native 0.72+
- Node.js 18+
- Android API 26+ / iOS 15+

## Installation

```bash
npm install @sdkmatch/react-native
# ou
yarn add @sdkmatch/react-native
```

### iOS
```bash
cd ios && pod install
```

## Permissions

### Android (`android/app/src/main/AndroidManifest.xml`)
```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.INTERNET" />
```

### iOS (`ios/Info.plist`)
```xml
<key>NSCameraUsageDescription</key>
<string>Camera requise pour le KYC</string>
```

## Utilisation

```typescript
import {
  activate,
  isLicenced,
  detectDocument,
  parseMrz,
  assessImageQuality,
  compareFaces,
} from '@sdkmatch/react-native';

// 1. Activer la licence
const activation = await activate('API_KEY', 'API_SECRET');
if (!activation.success) {
  console.error('Erreur:', activation.message);
  return;
}

// 2. Verifier la licence
const licenced = await isLicenced();

// 3. Detecter le document
const detection = await detectDocument(ocrText);
console.log('Type:', detection.type);
console.log('Champs:', detection.fields);

// 4. Parser la MRZ
const mrz = await parseMrz(ocrText);
if (mrz?.isValid) {
  console.log(`${mrz.lastName} ${mrz.firstName} — CIN: ${mrz.cinNumber}`);
}

// 5. Qualite image
const quality = await assessImageQuality(imageBase64);
if (!quality.isAcceptable) {
  console.warn('Image de mauvaise qualite');
  return;
}

// 6. Face Matching
const match = await compareFaces(documentBase64, selfieBase64);
if (match.isMatch && match.liveness.isLive) {
  console.log(`Identite verifiee ! Score: ${match.score}`);
}
```

## Voir aussi

- [Reference API](../api-reference/README.md)
- [Source React Native Module](../../sdk/react-native/)
