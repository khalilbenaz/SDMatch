# ⚙️ SDK MATCH — Core SDK

## 📦 Intégration

### 🤖 Android (Kotlin / JNI)

```kotlin
implementation("com.sdkmatch:kycsdk:1.0.0")
```

### 🍎 iOS (CocoaPods)

```ruby
pod 'KycSdk'
```

### 💜 .NET MAUI (NuGet)

```bash
dotnet add package KycSdk.Maui
```

### 💙 Flutter (pub.dev)

```yaml
dependencies:
  kyc_sdk: ^1.0.0
```

### ⚛️ React Native (npm)

```bash
npm install kyc-sdk-react-native
```

---

## 🚀 Utilisation (4 méthodes)

```kotlin
val sdk = KycSdk()

// 1️⃣ Démarrer une session
val session = sdk.startSession()

// 2️⃣ Scanner le document
val docResult = sdk.scanDocument(session, image)

// 3️⃣ Capturer un selfie
val selfieResult = sdk.captureSelfie(session, selfieImage)

// 4️⃣ Vérifier l'identité
val result = sdk.verify(session)

when (result.status) {
    KycStatus.VERIFIED -> // ✅ Identité vérifiée
    KycStatus.RETRY_SELFIE -> // 🔄 Reprendre le selfie
    KycStatus.RESCAN_DOCUMENT -> // 📄 Rescanner le document
    KycStatus.FAILED -> // ❌ Échec
}
```

---

## 📄 Documents supportés

| | Document | Extraction |
|---|----------|-----------|
| 🇲🇦 | CIN v1 (ancienne) | OCR bilingue (FR+AR) |
| 🇲🇦 | CIN v2 (biométrique) | MRZ TD1 + OCR |
| 🌍 | Passeport | MRZ TD3 |
| 🚗 | Permis de conduire | OCR intelligent |

---

## 👤 Face Matching

| | Feature | Détail |
|---|---------|--------|
| 👁️ | Détection visage | ML Kit Face Detection |
| 🔄 | Comparaison 1:1 | Document ↔ Selfie |
| 🧕 | Seuils adaptatifs | 0.75 standard, 0.70 lunettes, 0.65 hijab+lunettes |
| ✅ | Liveness passif | Yeux ouverts + angle tête naturel |

---

## 🔒 Privacy

- ✅ 100% on-device — zéro appel réseau pendant le KYC
- ✅ Données éphémères — effacées après vérification
- ✅ Aucun stockage — ni local ni cloud

---

## 📚 Documentation complète

- [📖 API Reference](docs/api-reference/README.md)
- [💡 Exemples complets](docs/examples/full-flow.md)
- [🤖 Quick Start Android](docs/quickstart/android.md)
- [🍎 Quick Start iOS](docs/quickstart/ios.md)
- [💜 Quick Start MAUI](docs/quickstart/maui.md)
- [💙 Quick Start Flutter](docs/quickstart/flutter.md)
- [⚛️ Quick Start React Native](docs/quickstart/react-native.md)

---

## 🔑 Licence

Obtenez une licence d'essai sur [sdmatch.netlify.app](https://sdmatch.netlify.app/)

Propriétaire — SDK MATCH © 2026
