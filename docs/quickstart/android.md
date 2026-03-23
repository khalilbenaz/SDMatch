# Android — Quick Start

Integrer SDK MATCH dans votre application Android en 5 minutes.

## Pre-requis

- Android Studio Hedgehog (2023.1.1) ou plus recent
- Android API 26+ (Android 8.0)
- Kotlin 1.9+

## 1. Ajouter le SDK

Telecharger `sdkmatch-sdk-v1.0.0.aar` depuis les [Releases](https://github.com/khalilbenaz/SDMatch/releases) et le copier dans `app/libs/`.

```kotlin
// app/build.gradle.kts
dependencies {
    implementation(files("libs/sdkmatch-sdk-v1.0.0.aar"))

    // CameraX (pour la capture — cote app uniquement)
    implementation("androidx.camera:camera-core:1.4.1")
    implementation("androidx.camera:camera-camera2:1.4.1")
    implementation("androidx.camera:camera-lifecycle:1.4.1")
    implementation("androidx.camera:camera-view:1.4.1")
}
```

> Le fat AAR inclut deja ML Kit, les modeles et toutes les dependances du SDK. Seul CameraX est a ajouter cote app pour gerer la camera.

## 2. Permissions

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-feature android:name="android.hardware.camera" android:required="true" />
```

## 3. Activer la licence

```kotlin
import com.sdkmatch.sdk.LicenceManager

val licence = LicenceManager(this)
val result = licence.activate("VOTRE_API_KEY", "VOTRE_API_SECRET")

if (!licence.isLicenced()) {
    // Afficher erreur — licence requise
    return
}
```

## 4. Scanner un document

```kotlin
import com.sdkmatch.sdk.DocumentDetector
import com.sdkmatch.sdk.MrzParser
import com.sdkmatch.sdk.ImageQuality
import com.sdkmatch.sdk.BitmapUtils
import com.google.mlkit.vision.text.TextRecognition
import com.google.mlkit.vision.text.latin.TextRecognizerOptions
import com.google.mlkit.vision.common.InputImage

// Charger l'image capturee
val bitmap = BitmapUtils.loadWithRotation(imagePath)

// Verifier la qualite
val quality = ImageQuality.assess(bitmap)
if (!quality.isAcceptable) {
    // Demander a l'utilisateur de reprendre la photo
    return
}

// OCR
val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)
recognizer.process(InputImage.fromBitmap(bitmap, 0))
    .addOnSuccessListener { visionText ->
        // Detecter le document
        val doc = DocumentDetector.detect(visionText.text)
        Log.d("SDK", "Type: ${doc.type}")
        Log.d("SDK", "Champs: ${doc.fields}")

        // Parser la MRZ
        val mrz = MrzParser.parse(visionText.text)
        mrz?.let {
            Log.d("SDK", "MRZ: ${it.lastName} ${it.firstName}, CIN: ${it.cinNumber}")
        }
    }
```

## 5. Face Matching

```kotlin
import com.sdkmatch.sdk.FaceComparison

val result = FaceComparison.compare(this, documentBitmap, selfieBitmap)

if (result.isMatch && result.liveness.isLive) {
    // Identite verifiee avec succes
    Log.d("SDK", "Score: ${result.score}")
} else {
    Log.d("SDK", "Echec: ${result.details}")
}
```

## 6. Resultat complet

```kotlin
// Resume du KYC
val kycResult = mapOf(
    "document_type" to doc.type,
    "nom" to doc.fields["nom"],
    "prenom" to doc.fields["prenom"],
    "cin" to doc.fields["cin"],
    "face_match_score" to result.score,
    "is_match" to result.isMatch,
    "is_live" to result.liveness.isLive,
    "processing_time_ms" to result.processingTime
)
```

---

## Voir aussi

- [Reference API](../api-reference/README.md) — Documentation complete de chaque classe
- [Exemple complet](../examples/full-flow.md) — Flux KYC production-ready avec gestion d'erreurs
