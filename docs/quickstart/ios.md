# iOS — Quick Start

Integrer SDK MATCH dans votre application iOS en 5 minutes.

## Pre-requis

- Xcode 15+
- iOS 15.0+
- Swift 5.9+

## Installation

### Swift Package Manager (recommande)

Dans Xcode : File > Add Package Dependencies, puis ajouter :

```
https://github.com/khalilbenaz/SDMatch.git
```

Ou dans `Package.swift` :

```swift
dependencies: [
    .package(url: "https://github.com/khalilbenaz/SDMatch.git", from: "1.0.0")
]
```

### CocoaPods

```ruby
# Podfile
pod 'SDKMatch', '~> 1.0'
```

## Permissions

```xml
<!-- Info.plist -->
<key>NSCameraUsageDescription</key>
<string>Camera requise pour scanner les documents et prendre un selfie</string>
```

## Utilisation

```swift
import SDKMatch

// 1. Activer la licence
let result = try await LicenceManager.shared.activate(
    apiKey: "VOTRE_API_KEY",
    apiSecret: "VOTRE_API_SECRET"
)

guard result.success else {
    print("Erreur licence: \(result.message)")
    return
}

// 2. Scanner et analyser un document
let ocrText = // ... texte OCR obtenu via ML Kit iOS
let detection = DocumentDetector.detect(ocrText: ocrText)
print("Type: \(detection.type)")     // CIN_V1, CIN_V2, PASSPORT, PERMIS
print("Champs: \(detection.fields)") // [nom: "...", prenom: "...", cin: "..."]

// 3. Parser la MRZ
if let mrz = MrzParser.parse(text: ocrText), mrz.isValid {
    print("MRZ: \(mrz.lastName) \(mrz.firstName)")
    print("CIN: \(mrz.cinNumber ?? "N/A")")
}

// 4. Qualite image
let quality = ImageQuality.assess(image: documentImage)
guard quality.isAcceptable else {
    print("Image de mauvaise qualite")
    return
}

// 5. Face Matching
let match = try await FaceComparison.compare(
    documentImage: documentImage,
    selfieImage: selfieImage
)

if match.isMatch && match.liveness.isLive {
    print("Identite verifiee ! Score: \(match.score)")
}
```

## Voir aussi

- [Reference API](../api-reference/README.md)
- [Exemple complet](../examples/full-flow.md)
- [Source iOS SDK](../../sdk/ios/)
