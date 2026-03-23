# Android Quick Start

Get the KYC SDK running in your Android app in under 5 minutes.

## Requirements

- Android API 24+ (Android 7.0)
- Kotlin 1.9+
- Gradle 8+

## Installation

Add the dependency to your module-level `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.kycsdk:kycsdk:1.0.0")
}
```

Or if using Groovy `build.gradle`:

```groovy
dependencies {
    implementation 'com.kycsdk:kycsdk:1.0.0'
}
```

Sync your project with Gradle files.

## Camera Permission

Add to your `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-feature android:name="android.hardware.camera" android:required="true" />
```

Request the permission at runtime before capturing images:

```kotlin
if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA)
    != PackageManager.PERMISSION_GRANTED) {
    ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.CAMERA), REQUEST_CAMERA)
}
```

## Quick Start

```kotlin
import android.graphics.Bitmap
import com.kycsdk.*

// 1. Create and initialize the SDK
val sdk = KycSdk()
val config = KycConfig(
    modelsDirectory = "$filesDir/models",
    licencePath = "$filesDir/licence.json",
    templatesDirectory = "$filesDir/templates"
)
sdk.initialize(config).getOrThrow()

// 2. Start a session
val session = sdk.startSession().getOrThrow()

// 3. Scan an identity document
val docBitmap: Bitmap = /* captured document photo */
val docResult = sdk.scanDocument(session, docBitmap).getOrThrow()
Log.d("KYC", "Document type: ${docResult.type}, fields: ${docResult.fields}")

// 4. Capture a selfie
val selfieBitmap: Bitmap = /* captured selfie photo */
val selfieResult = sdk.captureSelfie(session, selfieBitmap).getOrThrow()
Log.d("KYC", "Glasses: ${selfieResult.hasGlasses}, sharpness: ${selfieResult.sharpnessScore}")

// 5. Verify (face match)
val result = sdk.verify(session).getOrThrow()
when (result.status) {
    KycStatus.VERIFIED        -> Log.d("KYC", "Verified! Score: ${result.faceMatch.score}")
    KycStatus.RETRY_SELFIE    -> Log.d("KYC", "Please retake your selfie.")
    KycStatus.RESCAN_DOCUMENT -> Log.d("KYC", "Please rescan your document.")
    KycStatus.FAILED          -> Log.d("KYC", "Verification failed.")
}

// 6. Clean up
sdk.endSession(session)
sdk.close()
```

## Error Handling

All SDK methods return `KycResult<T>`, a sealed class with `Success` and `Failure` variants:

```kotlin
when (val result = sdk.scanDocument(session, docBitmap)) {
    is KycResult.Success -> {
        val doc = result.value
        Log.d("KYC", "Scanned ${doc.type}")
    }
    is KycResult.Failure -> {
        when (result.error.code) {
            KycErrorCode.DOCUMENT_NOT_DETECTED -> Log.w("KYC", "No document found")
            KycErrorCode.DOCUMENT_BLURRY       -> Log.w("KYC", "Image too blurry")
            KycErrorCode.DOCUMENT_GLARE        -> Log.w("KYC", "Too much glare")
            else -> Log.e("KYC", "Error: ${result.error.message}")
        }
    }
}
```

## Next Steps

- [API Reference](../api-reference/README.md) for the full list of methods, types, and error codes.
- [Full Flow Example](../examples/full-flow.md) for a production-ready implementation with retry logic.
