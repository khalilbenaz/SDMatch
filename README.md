# SDK MATCH — KYC Mobile SDK (On-Device)

**Scanner. Verifier. Valider.**
SDK de verification d'identite 100% on-device pour les documents marocains.

---

## Fonctionnalites

| Feature | Description |
|---------|-------------|
| **OCR & MRZ** | Lecture automatique (CIN v1, CIN v2, Passeport, Permis de conduire) |
| **MRZ Parser** | ICAO 9303 complet (TD1/TD2/TD3), check digits, correction OCR |
| **Face Matching** | ML Kit Face Detection + comparaison document/selfie (grid cosine similarity) |
| **Passive Liveness** | Detection de vivacite (yeux ouverts, angle tete, anti-spoofing) |
| **Image Quality** | Detection de flou (Laplacian), luminosite, validation automatique |
| **Licence Manager** | Activation API + validation APK signature + grace period |
| **Privacy First** | Zero cloud, donnees ephemeres, tout tourne sur l'appareil |

---

## Telechargement

### SDK (Fat AAR — autonome, zero dependance)

Depuis les [Releases GitHub](https://github.com/khalilbenaz/SDMatch/releases/tag/v1.0.0) :

- **`sdkmatch-sdk-v1.0.0.aar`** (43 MB) — SDK complet avec ML Kit, modeles OCR et Face Detection embarques
- **`sdkmatch-docs-v1.0.0.zip`** — Documentation complete

> Le fat AAR contient toutes les dependances : ML Kit Text Recognition, Face Detection, modeles OCR, librairies natives (arm64, armeabi-v7a, x86, x86_64). Le client n'a besoin d'ajouter **aucune autre dependance**.

### APK Demo

- **`SDK-MATCH-Final.apk`** — Application de demonstration complete

---

## Installation rapide (Android)

### 1. Ajouter le AAR au projet

Copier `sdkmatch-sdk-v1.0.0.aar` dans `app/libs/`, puis dans `app/build.gradle.kts` :

```kotlin
dependencies {
    implementation(files("libs/sdkmatch-sdk-v1.0.0.aar"))
}
```

### 2. Permissions (AndroidManifest.xml)

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-feature android:name="android.hardware.camera" android:required="true" />
```

### 3. Utilisation

```kotlin
import com.sdkmatch.sdk.*

// --- 1. Activer la licence (une seule fois) ---
val result = LicenceManager.activate(context, "API_KEY", "API_SECRET")

// --- 2. Initialiser le SDK (obligatoire, verifie la licence) ---
SdkMatch.initialize(context)
// Toute tentative d'utiliser les classes sans initialize() leve une SecurityException

// --- 3. Scanner un document ---
val ocrText = // ... texte OCR obtenu via ML Kit TextRecognition
val docType = DocumentDetector.detect(ocrText)
val fields = DocumentDetector.extractFieldsFromText(ocrText, docType)

// --- 4. Parser la MRZ ---
val mrz = MrzParser.parse(ocrText)
// mrz.firstName, mrz.lastName, mrz.documentNumber, mrz.birthDate...

// --- 5. Comparer les visages ---
val match = FaceComparison.compare(documentBitmap, selfieBitmap)
// match.score, match.isMatch, match.livenessResult.isLive

// --- 6. Qualite image ---
val quality = ImageQuality.assess(bitmap)
// quality.isAcceptable, quality.isBlurry, quality.brightness
```

> Voir la [documentation complete](docs/) pour les guides detailles et la reference API.

---

## Architecture

```
+-------------------------------+
|   Application Android          |
|   (CameraX + UI)               |
+---------------+---------------+
                |
+---------------v---------------+
|   SDK MATCH (Fat AAR)          |
|                                |
|  DocumentDetector    MrzParser |
|  FaceComparison  ImageQuality  |
|  LicenceManager  BitmapUtils   |
|                                |
|  +---------------------------+ |
|  | ML Kit Text Recognition   | |
|  | ML Kit Face Detection     | |
|  | Modeles OCR embarques     | |
|  | Libs natives (JNI)        | |
|  +---------------------------+ |
+--------------------------------+
```

---

## Documents supportes

| Document | MRZ | OCR | Recto/Verso |
|----------|-----|-----|-------------|
| CIN Ancienne (v1) | TD1 | Oui | Oui |
| CIN Nouvelle (v2) | TD1 | Oui | Oui |
| Passeport | TD3 | Oui | N/A |
| Permis de conduire | Non | Oui | Oui |

---

## Seuils Face Matching

| Condition | Seuil |
|-----------|-------|
| Standard | 0.75 |
| Lunettes | 0.70 |
| Hijab + lunettes | 0.65 |
| Barbe | 0.68 |
| Retry (seuil bas) | 0.55 |

---

## Configuration requise

- Android API 26+ (Android 8.0)
- Camera hardware requise
- Connexion Internet (activation licence uniquement)

---

## Licence

Obtenir une licence d'essai gratuite (7 jours) sur [sdmatch.netlify.app](https://sdmatch.netlify.app/).

---

## Structure du repo

```
SDMatch/
├── README.md              # Ce fichier
├── INTEGRATION.md         # Guide d'integration rapide
├── CMakeLists.txt         # Build natif C++
├── docs/
│   ├── api-reference/     # Reference API complete
│   ├── quickstart/        # Guides par plateforme
│   └── examples/          # Exemples de code complets
└── Releases/
    ├── sdkmatch-sdk-v1.0.0.aar   # Fat AAR (43 MB)
    └── sdkmatch-docs-v1.0.0.zip  # Documentation
```
