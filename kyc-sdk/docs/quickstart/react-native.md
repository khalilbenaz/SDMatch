# React Native Quick Start

Get the KYC SDK running in your React Native app in under 5 minutes.

## Requirements

- React Native 0.73+
- TypeScript 5+
- React Native New Architecture (TurboModules) recommended
- iOS 14.0+ / Android API 24+

## Installation

```bash
npm install kyc-sdk-react-native
```

Or with Yarn:

```bash
yarn add kyc-sdk-react-native
```

### iOS

Run pod install after adding the package:

```bash
cd ios && pod install && cd ..
```

### Android

No additional setup required. Gradle will pick up the native module automatically on the next build.

## Camera Permission

### iOS

In `ios/<YourApp>/Info.plist`:

```xml
<key>NSCameraUsageDescription</key>
<string>Camera access is required for identity verification.</string>
```

### Android

In `android/app/src/main/AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-feature android:name="android.hardware.camera" android:required="true" />
```

Use `react-native-permissions` or the built-in `PermissionsAndroid` API to request camera permission at runtime:

```typescript
import { PermissionsAndroid, Platform } from 'react-native';

async function requestCameraPermission(): Promise<boolean> {
  if (Platform.OS === 'android') {
    const granted = await PermissionsAndroid.request(
      PermissionsAndroid.PERMISSIONS.CAMERA,
    );
    return granted === PermissionsAndroid.RESULTS.GRANTED;
  }
  return true; // iOS prompts automatically
}
```

## Quick Start

```typescript
import { NativeBridge } from 'kyc-sdk-react-native';
import type {
  KycConfig,
  KycSession,
  ImageInput,
  KycStatus,
} from 'kyc-sdk-react-native';

async function runKyc(): Promise<void> {
  // 1. Initialize the SDK
  const config: KycConfig = {
    modelsDirectory: `${documentsDir}/models`,
    licencePath: `${documentsDir}/licence.json`,
    templatesDirectory: `${documentsDir}/templates`,
  };
  await NativeBridge.initialize(config);

  // 2. Start a session
  const session = await NativeBridge.startSession();

  // 3. Scan an identity document (base64 RGB)
  const docImage: ImageInput = { base64: docBase64, width: 1920, height: 1080 };
  const docResult = await NativeBridge.scanDocument(session.sessionId, docImage);
  console.log(`Document type: ${docResult.type}, fields:`, docResult.fields);

  // 4. Capture a selfie (base64 RGB)
  const selfieImage: ImageInput = { base64: selfieBase64, width: 1080, height: 1920 };
  const selfieResult = await NativeBridge.captureSelfie(session.sessionId, selfieImage);
  console.log(`Glasses: ${selfieResult.hasGlasses}, sharpness: ${selfieResult.sharpnessScore}`);

  // 5. Verify (face match)
  const result = await NativeBridge.verify(session.sessionId);
  switch (result.status) {
    case 0: console.log(`Verified! Score: ${result.faceMatch.score}`);   break;
    case 1: console.log('Please retake your selfie.');                   break;
    case 2: console.log('Please rescan your document.');                 break;
    case 3: console.log('Verification failed.');                         break;
  }

  // 6. Clean up
  await NativeBridge.endSession(session.sessionId);
}
```

## Error Handling

All methods return Promises. Errors are rejected with objects containing a `code` and `message`:

```typescript
try {
  const docResult = await NativeBridge.scanDocument(session.sessionId, docImage);
  console.log(`Scanned document: ${docResult.type}`);
} catch (error: any) {
  switch (error.code) {
    case 100: console.warn('No document found in image');    break;
    case 101: console.warn('Image is too blurry');           break;
    case 103: console.warn('Too much glare on document');    break;
    default:  console.error(`Error: ${error.message}`);      break;
  }
}
```

## Next Steps

- [API Reference](../api-reference/README.md) for the full list of methods, types, and error codes.
- [Full Flow Example](../examples/full-flow.md) for a production-ready implementation with retry logic.
