# SDK MATCH — Guide d'integration

## Installation

### Android (Kotlin / Java)

#### 1. Ajouter le Fat AAR

Telecharger `sdkmatch-sdk-v1.0.0.aar` depuis les [Releases](https://github.com/khalilbenaz/SDMatch/releases) et le placer dans `app/libs/`.

```kotlin
// app/build.gradle.kts
dependencies {
    implementation(files("libs/sdkmatch-sdk-v1.0.0.aar"))
}
```

> **Note** : Le fat AAR embarque toutes les dependances (ML Kit, modeles, libs natives). Aucune autre dependance n'est requise.

#### 2. Configuration minimale

```kotlin
// build.gradle.kts
android {
    defaultConfig {
        minSdk = 26  // Android 8.0 minimum
    }
}
```

#### 3. Permissions

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-feature android:name="android.hardware.camera" android:required="true" />
```

---

## Classes du SDK

Le SDK expose 6 classes publiques :

| Classe | Role |
|--------|------|
| `LicenceManager` | Activation et validation de la licence |
| `DocumentDetector` | Detection du type de document et extraction des champs OCR |
| `MrzParser` | Parsing MRZ conforme ICAO 9303 (TD1/TD2/TD3) |
| `FaceComparison` | Comparaison faciale document vs selfie + liveness |
| `ImageQuality` | Evaluation de la qualite image (flou, luminosite) |
| `BitmapUtils` | Chargement d'image avec correction de rotation EXIF |

---

## Flux d'integration typique

### Etape 1 — Activer la licence

```kotlin
val licence = LicenceManager(context)

// Premiere activation
val result = licence.activate("VOTRE_API_KEY", "VOTRE_API_SECRET")
if (result.success) {
    Log.d("SDK", "Licence activee: ${result.plan}, features: ${result.features}")
}

// Verifications suivantes
when (licence.getStatus()) {
    LicenceStatus.VALID         -> // OK
    LicenceStatus.EXPIRED_GRACE -> // Grace period 72h, renouveler
    LicenceStatus.EXPIRED       -> // Bloque
    LicenceStatus.NO_LICENCE    -> // Pas de licence
}
```

### Etape 2 — Capturer et analyser le document

```kotlin
// Capturer une image avec CameraX (cote app)
// Puis analyser le texte OCR avec ML Kit (inclus dans le SDK) :

val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)
val image = InputImage.fromBitmap(bitmap, 0)

recognizer.process(image)
    .addOnSuccessListener { visionText ->
        val ocrText = visionText.text

        // Detecter le type de document
        val detection = DocumentDetector.detect(ocrText)
        // detection.type = "CIN_V2"
        // detection.fields = {nom: "BENAZZOUZ", prenom: "KHALIL", cin: "XX123456", ...}

        // Parser la MRZ si presente
        val mrz = MrzParser.parse(ocrText)
        if (mrz != null && mrz.isValid) {
            // mrz.format = "TD1"
            // mrz.lastName, mrz.firstName
            // mrz.documentNumber, mrz.cinNumber
            // mrz.birthDate, mrz.expiryDate
            // mrz.nationality, mrz.sex
        }
    }
```

### Etape 3 — Verifier la qualite de l'image

```kotlin
val quality = ImageQuality.assess(documentBitmap)

if (!quality.isAcceptable) {
    when {
        quality.isBlurry  -> // "Image floue, veuillez reprendre la photo"
        quality.isTooDark -> // "Image trop sombre"
        quality.isTooLight -> // "Image trop claire"
    }
}
```

### Etape 4 — Face Matching + Liveness

```kotlin
val result = FaceComparison.compare(context, documentBitmap, selfieBitmap)

if (result.isMatch) {
    // Identite verifiee
    val score = result.score           // 0.0 - 1.0
    val isLive = result.liveness.isLive  // Passive liveness
    val timing = result.processingTime   // Temps en ms
} else {
    // Echec : score trop bas ou pas de visage detecte
}
```

---

## Utilitaire : BitmapUtils

```kotlin
// Charger une image capturee par CameraX avec la bonne rotation
val bitmap = BitmapUtils.loadWithRotation("/path/to/captured/image.jpg")
```

---

## Gestion des erreurs

```kotlin
try {
    val mrz = MrzParser.parse(ocrText)
} catch (e: Exception) {
    // MRZ non trouvee ou invalide
}

val match = FaceComparison.compare(context, docBitmap, selfieBitmap)
if (match.score == 0.0f) {
    // Aucun visage detecte dans l'une des images
}
```

---

## ProGuard

Les regles ProGuard sont automatiquement incluses via `consumer-rules.pro` dans le AAR. Aucune configuration supplementaire n'est necessaire.

---

## Configuration requise

| Parametre | Valeur |
|-----------|--------|
| Android minimum | API 26 (Android 8.0) |
| Compile SDK | 35 |
| Java | 1.8+ |
| Camera | Requise |
| Internet | Activation licence uniquement |
| Taille du SDK | ~43 MB (fat AAR) |
