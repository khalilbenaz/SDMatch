# .NET MAUI Quick Start

Get the KYC SDK running in your .NET MAUI app in under 5 minutes.

## Requirements

- .NET 8.0+
- .NET MAUI workload installed
- Visual Studio 2022 17.8+ or JetBrains Rider

## Installation

Install via the .NET CLI:

```bash
dotnet add package KycSdk.Maui
```

Or via the NuGet Package Manager in Visual Studio, search for `KycSdk.Maui`.

## Camera Permission

### Android

In `Platforms/Android/AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-feature android:name="android.hardware.camera" android:required="true" />
```

### iOS

In `Platforms/iOS/Info.plist`:

```xml
<key>NSCameraUsageDescription</key>
<string>Camera access is required for identity verification.</string>
```

## Quick Start

```csharp
using KycSdk.Maui;

// 1. Create and initialize the SDK
using var sdk = new KycSdk();
var config = new KycConfig
{
    ModelsDirectory = Path.Combine(FileSystem.AppDataDirectory, "models"),
    LicencePath = Path.Combine(FileSystem.AppDataDirectory, "licence.json"),
    TemplatesDirectory = Path.Combine(FileSystem.AppDataDirectory, "templates")
};
sdk.Initialize(config).GetOrThrow();

// 2. Start a session
var session = sdk.StartSession().GetOrThrow();

// 3. Scan an identity document (raw RGB byte array)
byte[] docPixels = /* captured document as RGB bytes */;
var docResult = sdk.ScanDocument(session, docPixels, width: 1920, height: 1080).GetOrThrow();
Console.WriteLine($"Document type: {docResult.Type}, fields: {docResult.Fields.Count}");

// 4. Capture a selfie (raw RGB byte array)
byte[] selfiePixels = /* captured selfie as RGB bytes */;
var selfieResult = sdk.CaptureSelfie(session, selfiePixels, width: 1080, height: 1920).GetOrThrow();
Console.WriteLine($"Glasses: {selfieResult.HasGlasses}, sharpness: {selfieResult.SharpnessScore}");

// 5. Verify (face match)
var result = sdk.Verify(session).GetOrThrow();
switch (result.Status)
{
    case KycStatus.Verified:
        Console.WriteLine($"Verified! Score: {result.FaceMatch.Score}");
        break;
    case KycStatus.RetrySelfie:
        Console.WriteLine("Please retake your selfie.");
        break;
    case KycStatus.RescanDocument:
        Console.WriteLine("Please rescan your document.");
        break;
    case KycStatus.Failed:
        Console.WriteLine("Verification failed.");
        break;
}

// 6. Clean up
sdk.EndSession(session);
// sdk.Dispose() is called automatically by the using statement
```

## Error Handling

All SDK methods return `KycResult<T>`. Check `IsSuccess` or call `GetOrThrow()`:

```csharp
var result = sdk.ScanDocument(session, imageData, width, height);
if (result.IsSuccess)
{
    var doc = result.GetOrThrow();
    Console.WriteLine($"Scanned {doc.Type}");
}
else
{
    var error = result.GetError()!;
    switch (error.Code)
    {
        case KycErrorCode.DocumentNotDetected:
            Console.WriteLine("No document found in image");
            break;
        case KycErrorCode.DocumentBlurry:
            Console.WriteLine("Image is too blurry");
            break;
        case KycErrorCode.DocumentGlare:
            Console.WriteLine("Too much glare on document");
            break;
        default:
            Console.WriteLine($"Error: {error.Message}");
            break;
    }
}
```

## Next Steps

- [API Reference](../api-reference/README.md) for the full list of methods, types, and error codes.
- [Full Flow Example](../examples/full-flow.md) for a production-ready implementation with retry logic.
