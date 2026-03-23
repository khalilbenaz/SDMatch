# iOS Quick Start

Get the KYC SDK running in your iOS app in under 5 minutes.

## Requirements

- iOS 14.0+
- Xcode 15+
- Swift 5.9+

## Installation

### CocoaPods

Add to your `Podfile`:

```ruby
pod 'KycSdk'
```

Then run:

```bash
pod install
```

### Swift Package Manager

In Xcode, go to **File > Add Package Dependencies** and enter the repository URL:

```
https://github.com/kycsdk/kyc-sdk-ios.git
```

Select version `1.0.0` or later.

## Camera Permission

Add the following key to your `Info.plist`:

```xml
<key>NSCameraUsageDescription</key>
<string>The app needs camera access to scan your identity document and capture a selfie for verification.</string>
```

## Quick Start

```swift
import UIKit
import KycSdk

// 1. Create and initialize the SDK
let sdk = KycSdk()
let config = KycConfig(
    modelsDirectory: Bundle.main.resourcePath! + "/models",
    licencePath: Bundle.main.path(forResource: "licence", ofType: "json")!,
    templatesDirectory: Bundle.main.resourcePath! + "/templates"
)
try sdk.initialize(config: config).get()

// 2. Start a session
let session = try sdk.startSession().get()

// 3. Scan an identity document
let docImage: UIImage = /* captured document photo */
let docResult = try sdk.scanDocument(session: session, image: docImage).get()
print("Document type: \(docResult.type), fields: \(docResult.fields)")

// 4. Capture a selfie
let selfieImage: UIImage = /* captured selfie photo */
let selfieResult = try sdk.captureSelfie(session: session, image: selfieImage).get()
print("Glasses: \(selfieResult.hasGlasses), sharpness: \(selfieResult.sharpnessScore)")

// 5. Verify (face match)
let result = try sdk.verify(session: session).get()
switch result.status {
case .verified:        print("Identity verified! Score: \(result.faceMatch.score)")
case .retrySelfie:     print("Please retake your selfie.")
case .rescanDocument:  print("Please rescan your document.")
case .failed:          print("Verification failed.")
}

// 6. Clean up
_ = sdk.endSession(session)
```

## Error Handling

All SDK methods return `Result<T, KycError>`. Use Swift's `Result` API to handle errors:

```swift
let result = sdk.scanDocument(session: session, image: docImage)
switch result {
case .success(let doc):
    print("Scanned \(doc.type)")
case .failure(let error):
    switch error.code {
    case .documentNotDetected: print("No document found in image")
    case .documentBlurry:      print("Image is too blurry")
    case .documentGlare:       print("Too much glare on document")
    default:                   print("Error: \(error.message)")
    }
}
```

## Next Steps

- [API Reference](../api-reference/README.md) for the full list of methods, types, and error codes.
- [Full Flow Example](../examples/full-flow.md) for a production-ready implementation with retry logic.
