# API Reference

Complete API reference for the KYC SDK. The SDK exposes the same logical API across all five platforms (iOS Swift, Android Kotlin, .NET MAUI C#, Flutter Dart, React Native TypeScript), with platform-idiomatic naming conventions.

---

## KycSdk Class

The primary entry point for all KYC operations.

### Constructor

| Platform | Signature |
|----------|-----------|
| iOS | `KycSdk()` |
| Android | `KycSdk()` |
| MAUI | `new KycSdk()` (implements `IDisposable`) |
| Flutter | `KycSdk()` |
| React Native | N/A (uses `NativeBridge` singleton) |

### Methods

#### initialize

Initialize the engine with the provided configuration. Must be called exactly once before `startSession`.

| Platform | Signature | Returns |
|----------|-----------|---------|
| iOS | `initialize(config: KycConfig)` | `Result<Bool, KycError>` |
| Android | `initialize(config: KycConfig)` | `KycResult<Boolean>` |
| MAUI | `Initialize(KycConfig config)` | `KycResult<bool>` |
| Flutter | `initialize(KycConfig config)` | `void` (throws `KycError`) |
| React Native | `initialize(config: KycConfig)` | `Promise<boolean>` |

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| `config` | `KycConfig` | Engine configuration including paths and thresholds. |

---

#### startSession

Start a new KYC verification session. Optionally override the engine configuration for this session.

| Platform | Signature | Returns |
|----------|-----------|---------|
| iOS | `startSession(config: KycConfig? = nil)` | `Result<KycSession, KycError>` |
| Android | `startSession(config: KycConfig? = null)` | `KycResult<KycSession>` |
| MAUI | `StartSession(KycConfig? config = null)` | `KycResult<KycSession>` |
| Flutter | `startSession({KycConfig? config})` | `KycSession` (throws `KycError`) |
| React Native | `startSession(config?: KycConfig)` | `Promise<KycSession>` |

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `config` | `KycConfig` | No | Optional per-session configuration override. |

---

#### scanDocument

Scan a document image within an active session. Performs document detection, classification, OCR field extraction, and face photo extraction.

| Platform | Signature | Returns |
|----------|-----------|---------|
| iOS | `scanDocument(session: KycSession, image: UIImage)` | `Result<DocumentResult, KycError>` |
| Android | `scanDocument(session: KycSession, image: Bitmap)` | `KycResult<DocumentResult>` |
| MAUI | `ScanDocument(KycSession session, byte[] imageData, int width, int height)` | `KycResult<DocumentResult>` |
| Flutter | `scanDocument({required KycSession session, required Uint8List pixels, required int width, required int height})` | `DocumentResult` (throws `KycError`) |
| React Native | `scanDocument(sessionId: string, image: ImageInput)` | `Promise<DocumentResult>` |

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| `session` / `sessionId` | `KycSession` / `string` | The active session handle. |
| `image` / `imageData` / `pixels` | Platform image type | The document photo. iOS uses `UIImage`, Android uses `Bitmap`, MAUI and Flutter use raw RGB byte arrays, React Native uses `ImageInput` (base64). |
| `width` | `int` / `number` | Image width in pixels. Required for MAUI, Flutter, and React Native. |
| `height` | `int` / `number` | Image height in pixels. Required for MAUI, Flutter, and React Native. |

---

#### captureSelfie

Capture and process a selfie within an active session. Performs face detection, accessory analysis (glasses, head covering), and image quality checks.

| Platform | Signature | Returns |
|----------|-----------|---------|
| iOS | `captureSelfie(session: KycSession, image: UIImage)` | `Result<SelfieResult, KycError>` |
| Android | `captureSelfie(session: KycSession, image: Bitmap)` | `KycResult<SelfieResult>` |
| MAUI | `CaptureSelfie(KycSession session, byte[] imageData, int width, int height)` | `KycResult<SelfieResult>` |
| Flutter | `captureSelfie({required KycSession session, required Uint8List pixels, required int width, required int height})` | `SelfieResult` (throws `KycError`) |
| React Native | `captureSelfie(sessionId: string, image: ImageInput)` | `Promise<SelfieResult>` |

**Parameters:**

Same pattern as `scanDocument`. The image should be a front-facing photo of the user.

---

#### verify

Run face-match verification comparing the document photo to the selfie. Both `scanDocument` and `captureSelfie` must have been called on this session before calling `verify`.

| Platform | Signature | Returns |
|----------|-----------|---------|
| iOS | `verify(session: KycSession)` | `Result<KycResult, KycError>` |
| Android | `verify(session: KycSession)` | `KycResult<KycVerificationResult>` |
| MAUI | `Verify(KycSession session)` | `KycResult<KycVerificationResult>` |
| Flutter | `verify({required KycSession session})` | `KycVerificationResult` (throws `KycError`) |
| React Native | `verify(sessionId: string)` | `Promise<KycVerificationResult>` |

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| `session` / `sessionId` | `KycSession` / `string` | The active session with both document and selfie data. |

---

#### endSession

End a session and release its resources in the native engine.

| Platform | Signature | Returns |
|----------|-----------|---------|
| iOS | `endSession(_ session: KycSession)` | `Result<Bool, KycError>` |
| Android | `endSession(session: KycSession)` | `KycResult<Boolean>` |
| MAUI | `EndSession(KycSession session)` | `KycResult<bool>` |
| Flutter | `endSession(KycSession session)` | `void` (throws `KycError`) |
| React Native | `endSession(sessionId: string)` | `Promise<boolean>` |

---

#### dispose / close

Release all native resources. After calling this method, no further SDK calls are valid.

| Platform | Method | Notes |
|----------|--------|-------|
| iOS | Automatic via `deinit` | No explicit call needed. |
| Android | `close()` | Also called by `finalize()`. |
| MAUI | `Dispose()` | Use `using` statement for automatic cleanup. |
| Flutter | `dispose()` | Call when SDK is no longer needed. |
| React Native | N/A | Managed by the native module lifecycle. |

---

## KycConfig

Configuration for the KYC engine.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `modelsDirectory` | `string` | `""` | Path to the directory containing `.tflite` model files. |
| `licencePath` | `string` | `""` | Path to the licence file (RSA-signed JSON). |
| `templatesDirectory` | `string` | `""` | Path to the document templates directory. |
| `thresholds` | `FaceMatchThresholds` | See below | Face matching threshold configuration. |
| `logLevel` | `LogLevel` | `Error` | Logging verbosity (`Debug`, `Info`, `Error`). |
| `maxImageDimension` | `int` | `4096` | Maximum image dimension in pixels before automatic downscaling. |
| `jpegQuality` | `int` | `90` | JPEG compression quality (0-100) for internal image encoding. |

---

## FaceMatchThresholds

Cosine-similarity thresholds in the range [0.0, 1.0] used during face matching. The SDK automatically selects the appropriate threshold based on detected accessories.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `standardThreshold` | `double` | `0.75` | Threshold when no accessories are detected on either face. |
| `glassesThreshold` | `double` | `0.70` | Threshold when glasses are detected on either face. |
| `hijabGlassesThreshold` | `double` | `0.65` | Threshold when both glasses and a head covering are detected. |
| `beardThreshold` | `double` | `0.68` | Threshold when a head covering (but not glasses) is detected. |
| `retryThreshold` | `double` | `0.55` | Below the selected threshold but above this value, the SDK recommends a retry instead of outright failure. |

**Threshold selection logic:**

1. If both glasses and head covering are detected: use `hijabGlassesThreshold` (0.65)
2. If only head covering is detected: use `beardThreshold` (0.68)
3. If only glasses are detected: use `glassesThreshold` (0.70)
4. Otherwise: use `standardThreshold` (0.75)

**Result determination:**

- Score >= selected threshold: `MATCH` (status = `Verified`)
- Score >= `retryThreshold` but < selected threshold: `RETRY` (status = `RetrySelfie`)
- Score < `retryThreshold`: `NO_MATCH` (status = `Failed`)

---

## Types

### KycSession

An opaque handle representing an active KYC session.

| Property | Type | Description |
|----------|------|-------------|
| `sessionId` | `string` | The unique session identifier issued by the C++ core. |

---

### DocumentType

The type of identity document detected.

| Value | Int | Description |
|-------|-----|-------------|
| `CIN_V1` / `cinV1` | `0` | Moroccan National Identity Card, version 1. |
| `CIN_V2` / `cinV2` | `1` | Moroccan National Identity Card, version 2. |
| `PASSPORT` / `passport` | `2` | Passport (MRZ-based extraction). |
| `UNKNOWN` / `unknown` | `3` | Document type could not be determined. |

---

### DocumentResult

Result of scanning a document image.

| Property | Type | Description |
|----------|------|-------------|
| `type` | `DocumentType` | The classified document type. |
| `fields` | `Map<String, String>` | Extracted OCR fields (e.g., `"firstName"`, `"lastName"`, `"dateOfBirth"`, `"documentNumber"`, `"expiryDate"`). |
| `documentFaceImageData` / `documentFaceBitmap` / `documentFaceImage` | Platform image type | The face region cropped from the document photo, or null. |
| `originalImageData` / `originalBitmap` / `originalImage` | Platform image type | The original captured image, or null. |

---

### SelfieResult

Result of capturing and processing a selfie.

| Property | Type | Description |
|----------|------|-------------|
| `faceImageData` / `faceBitmap` / `faceImage` | Platform image type | The detected face image, or null. |
| `hasGlasses` | `bool` | Whether glasses were detected on the face. |
| `hasHeadCovering` | `bool` | Whether a head covering was detected. |
| `sharpnessScore` | `double` | Image sharpness score from 0.0 (blurry) to 1.0 (sharp). |

---

### FaceMatchResult

The outcome of comparing the document face with the selfie.

| Property | Type | Description |
|----------|------|-------------|
| `score` | `double` | Cosine similarity score between the two face embeddings (0.0 to 1.0). |
| `threshold` | `double` | The threshold that was applied for this comparison. |
| `glassesDetected` | `bool` | Whether glasses were detected during matching. |
| `headCoveringDetected` | `bool` | Whether a head covering was detected during matching. |
| `status` | `string` | Human-readable status: `"MATCH"`, `"NO_MATCH"`, or `"RETRY"`. |

---

### KycStatus

Overall verification status returned by `verify`.

| Value | Int | Description |
|-------|-----|-------------|
| `VERIFIED` / `verified` | `0` | Face match succeeded. Identity verified. |
| `RETRY_SELFIE` / `retrySelfie` | `1` | Score is between the retry and match thresholds. Recommend retaking the selfie. |
| `RESCAN_DOCUMENT` / `rescanDocument` | `2` | Document quality was too low for reliable comparison. Recommend rescanning. |
| `FAILED` / `failed` | `3` | Face match score is below the retry threshold. Verification failed. |

---

### KycResult / KycVerificationResult

The final verification result combining document, selfie, and face match outcomes.

| Property | Type | Description |
|----------|------|-------------|
| `status` | `KycStatus` | The overall verification status. |
| `document` | `DocumentResult` | Document scan data (type and extracted fields). |
| `selfie` | `SelfieResult` | Selfie processing data (accessories and quality). |
| `faceMatch` | `FaceMatchResult` | Face comparison details (score, threshold, status). |

> **Note:** On iOS, the return type of `verify` is named `KycResult`. On Android, MAUI, Flutter, and React Native it is named `KycVerificationResult` to avoid collision with the `KycResult<T>` wrapper type.

---

### ImageInput (React Native only)

Image data passed to the SDK over the React Native bridge.

| Property | Type | Description |
|----------|------|-------------|
| `base64` | `string` | Base64-encoded RGB pixel data. Buffer must be exactly `width * height * 3` bytes before encoding. |
| `width` | `number` | Image width in pixels. |
| `height` | `number` | Image height in pixels. |

---

### LogLevel

Logging verbosity levels.

| Value | Int | Description |
|-------|-----|-------------|
| `Debug` / `debug` | `0` | Verbose diagnostic output. |
| `Info` / `info` | `1` | Informational messages. |
| `Error` / `error` | `2` | Errors only (default). |

---

## Error Codes

All errors use the `KycErrorCode` enum. Each code has a numeric value, a description, and a recommended action.

### Document Errors (100-199)

| Code | Name | Value | Description | Recommended Action |
|------|------|-------|-------------|-------------------|
| `DOCUMENT_NOT_DETECTED` | DocumentNotDetected | 100 | No identity document was found in the image. | Ask the user to place the document fully within the camera frame. |
| `DOCUMENT_BLURRY` | DocumentBlurry | 101 | The document image is too blurry for reliable extraction. | Ask the user to hold the camera steady and ensure good focus. |
| `DOCUMENT_LOW_LIGHT` | DocumentLowLight | 102 | The image is too dark for reliable processing. | Ask the user to move to a well-lit area. |
| `DOCUMENT_GLARE` | DocumentGlare | 103 | Excessive glare or reflection detected on the document. | Ask the user to tilt the document to reduce reflections. |
| `DOCUMENT_UNKNOWN_TYPE` | DocumentUnknownType | 104 | The document was detected but its type is not supported. | Inform the user that only CIN (v1/v2) and passports are supported. |

### MRZ / OCR Errors (200-299)

| Code | Name | Value | Description | Recommended Action |
|------|------|-------|-------------|-------------------|
| `MRZ_PARSE_FAILED` | MrzParseFailed | 200 | The MRZ (Machine Readable Zone) could not be parsed. | Ask the user to ensure the full MRZ is visible and not obscured. |
| `OCR_FAILED` | OcrFailed | 201 | Optical character recognition failed to extract text. | Ask the user to retake the photo with better lighting and focus. |

### Face / Selfie Errors (300-399)

| Code | Name | Value | Description | Recommended Action |
|------|------|-------|-------------|-------------------|
| `FACE_NOT_DETECTED` | FaceNotDetected | 300 | No face was detected in the image. | Ask the user to look directly at the camera and ensure their face is visible. |
| `FACE_MULTIPLE_FACES` | FaceMultipleFaces | 301 | Multiple faces were detected in the image. | Ask the user to ensure only one person is in the frame. |
| `FACE_TOO_SMALL` | FaceTooSmall | 302 | The detected face is too small for reliable processing. | Ask the user to move closer to the camera. |
| `SELFIE_BLURRY` | SelfieBlurry | 303 | The selfie image is too blurry. | Ask the user to hold the device steady. |

### Matching Errors (400-499)

| Code | Name | Value | Description | Recommended Action |
|------|------|-------|-------------|-------------------|
| `MATCH_SCORE_TOO_LOW` | MatchScoreTooLow | 400 | The face match score is below all thresholds. | The document and selfie faces do not match. May indicate a different person. |

### System Errors (500-599)

| Code | Name | Value | Description | Recommended Action |
|------|------|-------|-------------|-------------------|
| `MODEL_LOAD_FAILED` | ModelLoadFailed | 500 | One or more TFLite model files could not be loaded. | Verify that `modelsDirectory` is correct and contains all required `.tflite` files. |
| `LICENCE_INVALID` | LicenceInvalid | 501 | The licence file is missing, corrupt, or has an invalid signature. | Verify that `licencePath` points to a valid licence file. |
| `LICENCE_EXPIRED` | LicenceExpired | 502 | The licence has expired. | Contact support to renew your licence. |
| `INSUFFICIENT_MEMORY` | InsufficientMemory | 503 | The device does not have enough available memory. | Free memory by closing other apps, or reduce `maxImageDimension`. |

---

## Result Types

The SDK uses platform-idiomatic result types instead of exceptions (except Flutter and React Native which use throw/reject patterns).

### iOS: `Result<T, KycError>`

Swift's built-in `Result` type. Use `.get()` to unwrap or `switch` to pattern match.

### Android: `KycResult<T>`

A sealed class with `Success<T>` and `Failure` variants.

| Method | Description |
|--------|-------------|
| `getOrThrow()` | Returns the value or throws `KycException`. |
| `getOrNull()` | Returns the value or `null`. |
| `getOrElse(default)` | Returns the value or the result of the default lambda. |
| `map(transform)` | Transforms the success value. |
| `flatMap(transform)` | Monadic bind. |

### MAUI: `KycResult<T>`

A class with `IsSuccess` / `IsFailure` properties.

| Method | Description |
|--------|-------------|
| `GetOrThrow()` | Returns the value or throws `KycException`. |
| `GetOrDefault(defaultValue)` | Returns the value or the default. |
| `GetError()` | Returns the error, or null on success. |
| `Map<TResult>(transform)` | Transforms the success value. |

### Flutter

Methods throw `KycError` on failure. Use try/catch.

### React Native

Methods return `Promise`. Failures reject the promise with an error object containing `code` (int) and `message` (string).
