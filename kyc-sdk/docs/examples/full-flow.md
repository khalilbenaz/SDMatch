# Full KYC Flow Example

A complete, production-ready KYC verification flow in all five supported platforms. Each example demonstrates initialization with custom configuration, document scanning, selfie capture, verification with full status handling, and proper error handling with retry logic.

---

## iOS (Swift)

```swift
import UIKit
import KycSdk

class KycFlowController {
    private let sdk = KycSdk()
    private var session: KycSession?

    func runFullFlow(documentImage: UIImage, selfieImage: UIImage) {
        // ── Initialize ──────────────────────────────────────────────
        let config = KycConfig(
            modelsDirectory: Bundle.main.resourcePath! + "/models",
            licencePath: Bundle.main.path(forResource: "licence", ofType: "json")!,
            templatesDirectory: Bundle.main.resourcePath! + "/templates",
            thresholds: FaceMatchThresholds(
                standardThreshold: 0.75,
                glassesThreshold: 0.70,
                hijabGlassesThreshold: 0.65,
                beardThreshold: 0.68,
                retryThreshold: 0.55
            ),
            logLevel: .info,
            maxImageDimension: 4096,
            jpegQuality: 90
        )

        switch sdk.initialize(config: config) {
        case .success:
            break
        case .failure(let error):
            handleFatalError(error)
            return
        }

        // ── Start session ───────────────────────────────────────────
        switch sdk.startSession() {
        case .success(let s):
            session = s
        case .failure(let error):
            handleFatalError(error)
            return
        }

        guard let session = session else { return }

        // ── Scan document (with retry) ──────────────────────────────
        var docResult: DocumentResult?
        for attempt in 1...3 {
            switch sdk.scanDocument(session: session, image: documentImage) {
            case .success(let result):
                docResult = result
                print("[\(attempt)] Document scanned: \(result.type)")
                print("  Fields: \(result.fields)")
                break
            case .failure(let error):
                print("[\(attempt)] Document scan failed: \(error.code) - \(error.message)")
                if attempt == 3 {
                    handleRecoverableError(error, session: session)
                    return
                }
            }
            if docResult != nil { break }
        }

        // ── Capture selfie (with retry) ─────────────────────────────
        var selfieResult: SelfieResult?
        for attempt in 1...3 {
            switch sdk.captureSelfie(session: session, image: selfieImage) {
            case .success(let result):
                selfieResult = result
                print("[\(attempt)] Selfie captured")
                print("  Glasses: \(result.hasGlasses)")
                print("  Head covering: \(result.hasHeadCovering)")
                print("  Sharpness: \(result.sharpnessScore)")
                break
            case .failure(let error):
                print("[\(attempt)] Selfie capture failed: \(error.code) - \(error.message)")
                if attempt == 3 {
                    handleRecoverableError(error, session: session)
                    return
                }
            }
            if selfieResult != nil { break }
        }

        // ── Verify ──────────────────────────────────────────────────
        switch sdk.verify(session: session) {
        case .success(let result):
            handleVerificationResult(result, session: session)
        case .failure(let error):
            print("Verification error: \(error.code) - \(error.message)")
            _ = sdk.endSession(session)
        }
    }

    private func handleVerificationResult(_ result: KycResult, session: KycSession) {
        print("Face match score: \(result.faceMatch.score)")
        print("Threshold used: \(result.faceMatch.threshold)")
        print("Match status: \(result.faceMatch.status)")

        switch result.status {
        case .verified:
            print("Identity VERIFIED")
            print("Document type: \(result.document.type)")
            print("Fields: \(result.document.fields)")
            _ = sdk.endSession(session)

        case .retrySelfie:
            print("Selfie quality insufficient. Please retake.")
            // In a real app: prompt user for a new selfie, then call
            // captureSelfie + verify again on the same session.
            _ = sdk.endSession(session)

        case .rescanDocument:
            print("Document quality insufficient. Please rescan.")
            // In a real app: prompt user for a new document photo, then call
            // scanDocument + captureSelfie + verify again.
            _ = sdk.endSession(session)

        case .failed:
            print("Verification FAILED. Faces do not match.")
            _ = sdk.endSession(session)
        }
    }

    private func handleFatalError(_ error: KycError) {
        print("Fatal error: \(error.code) - \(error.message)")
    }

    private func handleRecoverableError(_ error: KycError, session: KycSession) {
        print("Max retries exceeded: \(error.code) - \(error.message)")
        _ = sdk.endSession(session)
    }
}
```

---

## Android (Kotlin)

```kotlin
import android.graphics.Bitmap
import android.util.Log
import com.kycsdk.*

class KycFlowController {
    private val sdk = KycSdk()
    private val TAG = "KycFlow"

    fun runFullFlow(documentBitmap: Bitmap, selfieBitmap: Bitmap) {
        // ── Initialize ──────────────────────────────────────────────
        val config = KycConfig(
            modelsDirectory = "$filesDir/models",
            licencePath = "$filesDir/licence.json",
            templatesDirectory = "$filesDir/templates",
            thresholds = FaceMatchThresholds(
                standardThreshold = 0.75,
                glassesThreshold = 0.70,
                hijabGlassesThreshold = 0.65,
                beardThreshold = 0.68,
                retryThreshold = 0.55
            ),
            logLevel = LogLevel.INFO,
            maxImageDimension = 4096,
            jpegQuality = 90
        )

        when (val initResult = sdk.initialize(config)) {
            is KycResult.Failure -> {
                Log.e(TAG, "Init failed: ${initResult.error.message}")
                return
            }
            is KycResult.Success -> Log.d(TAG, "SDK initialized")
        }

        // ── Start session ───────────────────────────────────────────
        val session = when (val r = sdk.startSession()) {
            is KycResult.Success -> r.value
            is KycResult.Failure -> {
                Log.e(TAG, "Session start failed: ${r.error.message}")
                return
            }
        }

        // ── Scan document (with retry) ──────────────────────────────
        var docResult: DocumentResult? = null
        for (attempt in 1..3) {
            when (val r = sdk.scanDocument(session, documentBitmap)) {
                is KycResult.Success -> {
                    docResult = r.value
                    Log.d(TAG, "[$attempt] Document: ${docResult.type}, fields: ${docResult.fields}")
                    break
                }
                is KycResult.Failure -> {
                    Log.w(TAG, "[$attempt] Scan failed: ${r.error.code} - ${r.error.message}")
                    if (attempt == 3) {
                        sdk.endSession(session)
                        return
                    }
                }
            }
        }

        // ── Capture selfie (with retry) ─────────────────────────────
        var selfieResult: SelfieResult? = null
        for (attempt in 1..3) {
            when (val r = sdk.captureSelfie(session, selfieBitmap)) {
                is KycResult.Success -> {
                    selfieResult = r.value
                    Log.d(TAG, "[$attempt] Selfie: glasses=${selfieResult.hasGlasses}, " +
                        "headCovering=${selfieResult.hasHeadCovering}, " +
                        "sharpness=${selfieResult.sharpnessScore}")
                    break
                }
                is KycResult.Failure -> {
                    Log.w(TAG, "[$attempt] Selfie failed: ${r.error.code} - ${r.error.message}")
                    if (attempt == 3) {
                        sdk.endSession(session)
                        return
                    }
                }
            }
        }

        // ── Verify ──────────────────────────────────────────────────
        when (val r = sdk.verify(session)) {
            is KycResult.Success -> {
                val result = r.value
                Log.d(TAG, "Score: ${result.faceMatch.score}, threshold: ${result.faceMatch.threshold}")

                when (result.status) {
                    KycStatus.VERIFIED -> {
                        Log.d(TAG, "VERIFIED - ${result.document.type}: ${result.document.fields}")
                    }
                    KycStatus.RETRY_SELFIE -> {
                        Log.w(TAG, "Retry selfie recommended")
                        // Prompt user for new selfie, then captureSelfie + verify again
                    }
                    KycStatus.RESCAN_DOCUMENT -> {
                        Log.w(TAG, "Rescan document recommended")
                        // Prompt user for new document photo, then full flow again
                    }
                    KycStatus.FAILED -> {
                        Log.e(TAG, "Verification FAILED - faces do not match")
                    }
                }
            }
            is KycResult.Failure -> {
                Log.e(TAG, "Verify error: ${r.error.message}")
            }
        }

        sdk.endSession(session)
        sdk.close()
    }
}
```

---

## .NET MAUI (C#)

```csharp
using KycSdk.Maui;

public class KycFlowController
{
    public void RunFullFlow(byte[] documentPixels, int docW, int docH,
                            byte[] selfiePixels, int selfieW, int selfieH)
    {
        // ── Initialize ──────────────────────────────────────────────
        using var sdk = new KycSdk();
        var config = new KycConfig
        {
            ModelsDirectory = Path.Combine(FileSystem.AppDataDirectory, "models"),
            LicencePath = Path.Combine(FileSystem.AppDataDirectory, "licence.json"),
            TemplatesDirectory = Path.Combine(FileSystem.AppDataDirectory, "templates"),
            Thresholds = new FaceMatchThresholds
            {
                StandardThreshold = 0.75,
                GlassesThreshold = 0.70,
                HijabGlassesThreshold = 0.65,
                BeardThreshold = 0.68,
                RetryThreshold = 0.55
            },
            LogLevel = LogLevel.Info,
            MaxImageDimension = 4096,
            JpegQuality = 90
        };

        var initResult = sdk.Initialize(config);
        if (initResult.IsFailure)
        {
            Console.WriteLine($"Init failed: {initResult.GetError()!.Message}");
            return;
        }

        // ── Start session ───────────────────────────────────────────
        var sessionResult = sdk.StartSession();
        if (sessionResult.IsFailure)
        {
            Console.WriteLine($"Session failed: {sessionResult.GetError()!.Message}");
            return;
        }
        var session = sessionResult.GetOrThrow();

        // ── Scan document (with retry) ──────────────────────────────
        DocumentResult? docResult = null;
        for (int attempt = 1; attempt <= 3; attempt++)
        {
            var r = sdk.ScanDocument(session, documentPixels, docW, docH);
            if (r.IsSuccess)
            {
                docResult = r.GetOrThrow();
                Console.WriteLine($"[{attempt}] Document: {docResult.Type}");
                foreach (var kv in docResult.Fields)
                    Console.WriteLine($"  {kv.Key}: {kv.Value}");
                break;
            }
            else
            {
                var err = r.GetError()!;
                Console.WriteLine($"[{attempt}] Scan failed: {err.Code} - {err.Message}");
                if (attempt == 3)
                {
                    sdk.EndSession(session);
                    return;
                }
            }
        }

        // ── Capture selfie (with retry) ─────────────────────────────
        SelfieResult? selfieResult = null;
        for (int attempt = 1; attempt <= 3; attempt++)
        {
            var r = sdk.CaptureSelfie(session, selfiePixels, selfieW, selfieH);
            if (r.IsSuccess)
            {
                selfieResult = r.GetOrThrow();
                Console.WriteLine($"[{attempt}] Selfie: glasses={selfieResult.HasGlasses}, " +
                    $"headCovering={selfieResult.HasHeadCovering}, " +
                    $"sharpness={selfieResult.SharpnessScore}");
                break;
            }
            else
            {
                var err = r.GetError()!;
                Console.WriteLine($"[{attempt}] Selfie failed: {err.Code} - {err.Message}");
                if (attempt == 3)
                {
                    sdk.EndSession(session);
                    return;
                }
            }
        }

        // ── Verify ──────────────────────────────────────────────────
        var verifyResult = sdk.Verify(session);
        if (verifyResult.IsSuccess)
        {
            var result = verifyResult.GetOrThrow();
            Console.WriteLine($"Score: {result.FaceMatch.Score}, " +
                $"threshold: {result.FaceMatch.Threshold}");

            switch (result.Status)
            {
                case KycStatus.Verified:
                    Console.WriteLine($"VERIFIED - {result.Document.Type}");
                    break;
                case KycStatus.RetrySelfie:
                    Console.WriteLine("Retry selfie recommended");
                    // Prompt user for new selfie
                    break;
                case KycStatus.RescanDocument:
                    Console.WriteLine("Rescan document recommended");
                    // Prompt user for new document photo
                    break;
                case KycStatus.Failed:
                    Console.WriteLine("Verification FAILED");
                    break;
            }
        }
        else
        {
            Console.WriteLine($"Verify error: {verifyResult.GetError()!.Message}");
        }

        sdk.EndSession(session);
        // sdk.Dispose() called automatically by using statement
    }
}
```

---

## Flutter (Dart)

```dart
import 'dart:typed_data';
import 'package:kyc_sdk/kyc_sdk.dart';

class KycFlowController {
  KycSdk? _sdk;

  Future<void> runFullFlow(
    Uint8List documentPixels, int docW, int docH,
    Uint8List selfiePixels, int selfieW, int selfieH,
  ) async {
    // ── Initialize ──────────────────────────────────────────────
    _sdk = KycSdk();
    try {
      _sdk!.initialize(KycConfig(
        modelsDirectory: '$appDocDir/models',
        licencePath: '$appDocDir/licence.json',
        templatesDirectory: '$appDocDir/templates',
        thresholds: const FaceMatchThresholds(
          standardThreshold: 0.75,
          glassesThreshold: 0.70,
          hijabGlassesThreshold: 0.65,
          beardThreshold: 0.68,
          retryThreshold: 0.55,
        ),
        logLevel: LogLevel.info,
        maxImageDimension: 4096,
        jpegQuality: 90,
      ));
    } on KycError catch (e) {
      print('Init failed: ${e.code} - ${e.message}');
      return;
    }

    // ── Start session ───────────────────────────────────────────
    KycSession session;
    try {
      session = _sdk!.startSession();
    } on KycError catch (e) {
      print('Session failed: ${e.code} - ${e.message}');
      _sdk!.dispose();
      return;
    }

    // ── Scan document (with retry) ──────────────────────────────
    DocumentResult? docResult;
    for (var attempt = 1; attempt <= 3; attempt++) {
      try {
        docResult = _sdk!.scanDocument(
          session: session, pixels: documentPixels, width: docW, height: docH,
        );
        print('[$attempt] Document: ${docResult.type}');
        docResult.fields.forEach((k, v) => print('  $k: $v'));
        break;
      } on KycError catch (e) {
        print('[$attempt] Scan failed: ${e.code} - ${e.message}');
        if (attempt == 3) {
          _sdk!.endSession(session);
          _sdk!.dispose();
          return;
        }
      }
    }

    // ── Capture selfie (with retry) ─────────────────────────────
    SelfieResult? selfieResult;
    for (var attempt = 1; attempt <= 3; attempt++) {
      try {
        selfieResult = _sdk!.captureSelfie(
          session: session, pixels: selfiePixels, width: selfieW, height: selfieH,
        );
        print('[$attempt] Selfie: glasses=${selfieResult.hasGlasses}, '
            'headCovering=${selfieResult.hasHeadCovering}, '
            'sharpness=${selfieResult.sharpnessScore}');
        break;
      } on KycError catch (e) {
        print('[$attempt] Selfie failed: ${e.code} - ${e.message}');
        if (attempt == 3) {
          _sdk!.endSession(session);
          _sdk!.dispose();
          return;
        }
      }
    }

    // ── Verify ──────────────────────────────────────────────────
    try {
      final result = _sdk!.verify(session: session);
      print('Score: ${result.faceMatch.score}, threshold: ${result.faceMatch.threshold}');

      switch (result.status) {
        case KycStatus.verified:
          print('VERIFIED - ${result.document.type}');
          result.document.fields.forEach((k, v) => print('  $k: $v'));
        case KycStatus.retrySelfie:
          print('Retry selfie recommended');
          // Prompt user for new selfie, then captureSelfie + verify again
        case KycStatus.rescanDocument:
          print('Rescan document recommended');
          // Prompt user for new document photo, then full flow again
        case KycStatus.failed:
          print('Verification FAILED - faces do not match');
      }
    } on KycError catch (e) {
      print('Verify error: ${e.code} - ${e.message}');
    }

    _sdk!.endSession(session);
    _sdk!.dispose();
  }
}
```

---

## React Native (TypeScript)

```typescript
import { NativeBridge } from 'kyc-sdk-react-native';
import {
  KycConfig,
  KycSession,
  KycStatus,
  KycErrorCode,
  ImageInput,
} from 'kyc-sdk-react-native';

const MAX_RETRIES = 3;

export async function runFullKycFlow(
  documentBase64: string,
  docWidth: number,
  docHeight: number,
  selfieBase64: string,
  selfieWidth: number,
  selfieHeight: number,
): Promise<void> {
  // ── Initialize ──────────────────────────────────────────────────
  const config: KycConfig = {
    modelsDirectory: `${documentsDir}/models`,
    licencePath: `${documentsDir}/licence.json`,
    templatesDirectory: `${documentsDir}/templates`,
    thresholds: {
      standardThreshold: 0.75,
      glassesThreshold: 0.70,
      hijabGlassesThreshold: 0.65,
      beardThreshold: 0.68,
      retryThreshold: 0.55,
    },
    logLevel: 1, // Info
    maxImageDimension: 4096,
    jpegQuality: 90,
  };

  try {
    await NativeBridge.initialize(config);
    console.log('SDK initialized');
  } catch (error: any) {
    console.error(`Init failed: ${error.message}`);
    return;
  }

  // ── Start session ───────────────────────────────────────────────
  let session: KycSession;
  try {
    session = await NativeBridge.startSession();
    console.log(`Session started: ${session.sessionId}`);
  } catch (error: any) {
    console.error(`Session failed: ${error.message}`);
    return;
  }

  // ── Scan document (with retry) ──────────────────────────────────
  const docImage: ImageInput = {
    base64: documentBase64,
    width: docWidth,
    height: docHeight,
  };

  let docResult;
  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    try {
      docResult = await NativeBridge.scanDocument(session.sessionId, docImage);
      console.log(`[${attempt}] Document: type=${docResult.type}`);
      console.log(`  Fields:`, docResult.fields);
      break;
    } catch (error: any) {
      console.warn(`[${attempt}] Scan failed: code=${error.code}, ${error.message}`);
      if (attempt === MAX_RETRIES) {
        await NativeBridge.endSession(session.sessionId);
        return;
      }
    }
  }

  // ── Capture selfie (with retry) ─────────────────────────────────
  const selfieImage: ImageInput = {
    base64: selfieBase64,
    width: selfieWidth,
    height: selfieHeight,
  };

  let selfieResult;
  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    try {
      selfieResult = await NativeBridge.captureSelfie(
        session.sessionId,
        selfieImage,
      );
      console.log(
        `[${attempt}] Selfie: glasses=${selfieResult.hasGlasses}, ` +
          `headCovering=${selfieResult.hasHeadCovering}, ` +
          `sharpness=${selfieResult.sharpnessScore}`,
      );
      break;
    } catch (error: any) {
      console.warn(
        `[${attempt}] Selfie failed: code=${error.code}, ${error.message}`,
      );
      if (attempt === MAX_RETRIES) {
        await NativeBridge.endSession(session.sessionId);
        return;
      }
    }
  }

  // ── Verify ──────────────────────────────────────────────────────
  try {
    const result = await NativeBridge.verify(session.sessionId);
    console.log(
      `Score: ${result.faceMatch.score}, threshold: ${result.faceMatch.threshold}`,
    );

    switch (result.status) {
      case KycStatus.Verified:
        console.log('VERIFIED');
        console.log(`Document type: ${result.document.type}`);
        console.log('Fields:', result.document.fields);
        break;

      case KycStatus.RetrySelfie:
        console.log('Retry selfie recommended');
        // Prompt user for new selfie, then captureSelfie + verify again
        break;

      case KycStatus.RescanDocument:
        console.log('Rescan document recommended');
        // Prompt user for new document photo, then full flow again
        break;

      case KycStatus.Failed:
        console.log('Verification FAILED - faces do not match');
        break;
    }
  } catch (error: any) {
    console.error(`Verify error: ${error.message}`);
  }

  // ── Clean up ────────────────────────────────────────────────────
  await NativeBridge.endSession(session.sessionId);
}
```

---

## Status Handling Summary

| Status | Meaning | Recommended Action |
|--------|---------|-------------------|
| `Verified` | Face match succeeded. Identity confirmed. | Proceed with the verified identity data. |
| `RetrySelfie` | Score is marginal (between retry and match thresholds). | Ask the user to retake their selfie with better conditions. Call `captureSelfie` + `verify` again on the same session. |
| `RescanDocument` | Document quality too low for reliable comparison. | Ask the user to retake the document photo. Call `scanDocument` + `captureSelfie` + `verify` again. |
| `Failed` | Score is below the retry threshold. Faces do not match. | End the session. The user may need to start over or contact support. |
