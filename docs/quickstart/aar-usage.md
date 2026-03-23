# Guide complet d'utilisation du Fat AAR

Guide exhaustif pour integrer `sdkmatch-sdk-v1.0.0.aar` dans une application Android.

---

## Table des matieres

1. [Pre-requis](#pre-requis)
2. [Installation](#installation)
3. [Configuration du projet](#configuration-du-projet)
4. [Activation de la licence](#activation-de-la-licence)
5. [Capture et OCR du document](#capture-et-ocr-du-document)
6. [Detection du type de document](#detection-du-type-de-document)
7. [Parsing MRZ](#parsing-mrz)
8. [Controle qualite image](#controle-qualite-image)
9. [Face Matching et Liveness](#face-matching-et-liveness)
10. [Flux complet KYC](#flux-complet-kyc)
11. [Gestion des erreurs](#gestion-des-erreurs)
12. [ProGuard et obfuscation](#proguard-et-obfuscation)
13. [FAQ et depannage](#faq-et-depannage)

---

## Pre-requis

| Element | Minimum |
|---------|---------|
| Android API | 26 (Android 8.0 Oreo) |
| Compile SDK | 35 |
| Kotlin | 1.9+ |
| Java | 1.8+ |
| Android Studio | Hedgehog (2023.1.1) ou plus recent |
| Espace disque | ~50 MB (AAR + cache modeles) |
| Camera | Hardware camera requise |
| Internet | Uniquement pour l'activation de la licence |

---

## Installation

### Etape 1 — Telecharger le AAR

Depuis les [Releases GitHub](https://github.com/khalilbenaz/SDMatch/releases/tag/v1.0.0) :

```
sdkmatch-sdk-v1.0.0.aar (43 MB)
```

### Etape 2 — Placer dans le projet

```
votre-app/
├── app/
│   ├── libs/
│   │   └── sdkmatch-sdk-v1.0.0.aar   ← ici
│   ├── src/
│   └── build.gradle.kts
└── ...
```

### Etape 3 — Declarer la dependance

```kotlin
// app/build.gradle.kts
dependencies {
    implementation(files("libs/sdkmatch-sdk-v1.0.0.aar"))

    // CameraX — pour la capture camera (cote app)
    implementation("androidx.camera:camera-core:1.4.1")
    implementation("androidx.camera:camera-camera2:1.4.1")
    implementation("androidx.camera:camera-lifecycle:1.4.1")
    implementation("androidx.camera:camera-view:1.4.1")
}
```

> **Important** : Le fat AAR inclut deja ML Kit Text Recognition, ML Kit Face Detection, les modeles OCR, les librairies natives JNI, ExifInterface, et Core KTX. **N'ajoutez pas ces dependances** — elles sont deja dans le AAR.

### Etape 4 — Sync Gradle

Cliquez sur "Sync Now" dans Android Studio ou executez :

```bash
./gradlew app:dependencies
```

---

## Configuration du projet

### build.gradle.kts

```kotlin
android {
    compileSdk = 35

    defaultConfig {
        minSdk = 26
        targetSdk = 35
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }

    kotlinOptions {
        jvmTarget = "1.8"
    }
}
```

### AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Camera pour le scan de documents et le selfie -->
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-feature android:name="android.hardware.camera" android:required="true" />

    <!-- Internet pour l'activation de la licence uniquement -->
    <uses-permission android:name="android.permission.INTERNET" />

    <application ...>
        <!-- Votre activite -->
    </application>
</manifest>
```

### Demande de permission camera (runtime)

```kotlin
private val cameraPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { granted ->
    if (granted) {
        startCamera()
    } else {
        Toast.makeText(this, "Camera requise pour le KYC", Toast.LENGTH_LONG).show()
    }
}

// Verifier et demander
if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA)
    != PackageManager.PERMISSION_GRANTED) {
    cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
} else {
    startCamera()
}
```

---

## Activation de la licence

Avant toute utilisation du SDK, une licence valide est requise.

### Obtenir une licence

1. Aller sur [sdmatch.netlify.app](https://sdmatch.netlify.app/)
2. Remplir le formulaire "Demander une demo"
3. Recevoir votre **API Key** et **API Secret** (7 jours gratuit)

### Activer dans le code

```kotlin
import com.sdkmatch.sdk.LicenceManager

class MyApp : Application() {
    lateinit var licence: LicenceManager

    override fun onCreate() {
        super.onCreate()
        licence = LicenceManager(this)
    }
}

// Dans votre Activity ou ViewModel
val licence = (application as MyApp).licence

// Premiere activation (necessite Internet)
lifecycleScope.launch(Dispatchers.IO) {
    val result = licence.activate("sk_live_xxxxx", "secret_xxxxx")

    withContext(Dispatchers.Main) {
        if (result.success) {
            Log.d("SDK", "Licence activee !")
            Log.d("SDK", "Plan: ${result.plan}")
            Log.d("SDK", "Features: ${result.features}")
            // Demarrer le KYC
        } else {
            Log.e("SDK", "Echec: ${result.message}")
        }
    }
}
```

### Verifier la licence

```kotlin
// Verification rapide (pas besoin d'Internet)
if (licence.isLicenced()) {
    // SDK utilisable
}

// Statut detaille
when (licence.getStatus()) {
    LicenceStatus.VALID -> {
        // Licence active — tout fonctionne
    }
    LicenceStatus.EXPIRED_GRACE -> {
        // Licence expiree mais grace period de 72h active
        // Le SDK fonctionne encore, mais renouveler rapidement
        licence.renewToken()
    }
    LicenceStatus.EXPIRED -> {
        // Licence expiree — SDK bloque
        // Reactivation necessaire
    }
    LicenceStatus.NO_LICENCE -> {
        // Aucune licence — activation requise
    }
}

// Renouvellement automatique si necessaire
if (licence.needsRenewal()) {
    val renewed = licence.renewToken()
    if (!renewed.success) {
        // Forcer une reactivation
        licence.activate(apiKey, apiSecret)
    }
}
```

### Securite de la licence

Le SDK integre plusieurs protections :

- **Validation APK** : Verifie la signature SHA-256 (anti-tampering / anti-debug)
- **Device binding** : Lie la licence au ANDROID_ID de l'appareil
- **Token expiry** : Les tokens ont une duree de vie limitee
- **Grace period** : 72h apres expiration pour eviter les interruptions

---

## Capture et OCR du document

Le SDK inclut ML Kit Text Recognition. Voici comment capturer et extraire le texte :

### Configuration CameraX

```kotlin
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.camera.view.PreviewView

class ScannerActivity : AppCompatActivity() {

    private lateinit var previewView: PreviewView
    private var imageCapture: ImageCapture? = null

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            // Preview
            val preview = Preview.Builder().build().also {
                it.surfaceProvider = previewView.surfaceProvider
            }

            // Capture
            imageCapture = ImageCapture.Builder()
                .setCaptureMode(ImageCapture.CAPTURE_MODE_MAXIMIZE_QUALITY)
                .build()

            // Camera arriere pour les documents
            val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

            cameraProvider.unbindAll()
            cameraProvider.bindToLifecycle(this, cameraSelector, preview, imageCapture)

        }, ContextCompat.getMainExecutor(this))
    }

    // Capturer une photo
    private fun captureDocument() {
        val imageCapture = imageCapture ?: return

        val photoFile = File(cacheDir, "document_${System.currentTimeMillis()}.jpg")
        val outputOptions = ImageCapture.OutputFileOptions.Builder(photoFile).build()

        imageCapture.takePicture(
            outputOptions,
            ContextCompat.getMainExecutor(this),
            object : ImageCapture.OnImageSavedCallback {
                override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                    processDocument(photoFile.absolutePath)
                }
                override fun onError(exception: ImageCaptureException) {
                    Log.e("SDK", "Erreur capture: ${exception.message}")
                }
            }
        )
    }
}
```

### Extraction OCR

```kotlin
import com.sdkmatch.sdk.BitmapUtils
import com.google.mlkit.vision.text.TextRecognition
import com.google.mlkit.vision.text.latin.TextRecognizerOptions
import com.google.mlkit.vision.common.InputImage

private fun processDocument(imagePath: String) {
    // Charger avec correction de rotation EXIF
    val bitmap = BitmapUtils.loadWithRotation(imagePath)
        ?: run {
            Log.e("SDK", "Impossible de charger l'image")
            return
        }

    // ML Kit OCR (inclus dans le fat AAR)
    val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)
    val inputImage = InputImage.fromBitmap(bitmap, 0)

    recognizer.process(inputImage)
        .addOnSuccessListener { visionText ->
            val text = visionText.text
            Log.d("SDK", "Texte OCR extrait (${text.length} chars)")

            if (text.isBlank()) {
                Log.w("SDK", "Aucun texte detecte — verifier le cadrage")
                return@addOnSuccessListener
            }

            // Traiter le texte...
            analyzeDocument(text, bitmap)
        }
        .addOnFailureListener { e ->
            Log.e("SDK", "Erreur OCR: ${e.message}")
        }
}
```

---

## Detection du type de document

```kotlin
import com.sdkmatch.sdk.DocumentDetector

private fun analyzeDocument(ocrText: String, bitmap: Bitmap) {
    val result = DocumentDetector.detect(ocrText)

    Log.d("SDK", "Type: ${result.type}")
    // "CIN_V1", "CIN_V2", "PASSPORT", "PERMIS", "UNKNOWN"

    // Parcourir les champs extraits
    result.fields.forEach { (key, value) ->
        Log.d("SDK", "  $key = $value")
    }

    // Exemple de champs pour une CIN v2 :
    // cin = AB123456
    // nom = BENAZZOUZ
    // prenom = KHALIL
    // date_naissance = 15/03/1995
    // lieu_naissance = CASABLANCA
    // sexe = M
    // nationalite = Marocaine
    // date_validite = 15/03/2035
}
```

### Documents supportes et champs

**CIN Ancienne (v1) :**
`cin`, `nom`, `prenom`, `date_naissance`, `lieu_naissance`, `sexe`, `nationalite`, `adresse`, `date_validite`

**CIN Nouvelle (v2) :**
Memes champs + marqueurs biometriques (OPID, CAN)

**Passeport :**
`numero`, `nom`, `prenom`, `date_naissance`, `lieu_naissance`, `sexe`, `nationalite`, `date_expiration`

**Permis de conduire :**
`cnie`, `nom`, `prenom`, `date_naissance`, `categories`, `date_delivrance`

---

## Parsing MRZ

Le MRZ Parser extrait les donnees structurees de la zone MRZ (Machine Readable Zone).

```kotlin
import com.sdkmatch.sdk.MrzParser

val mrz = MrzParser.parse(ocrText)

if (mrz != null) {
    if (mrz.isValid) {
        // MRZ valide — check digits OK
        Log.d("SDK", "Format: ${mrz.format}")          // TD1, TD2, TD3
        Log.d("SDK", "Type: ${mrz.documentType}")       // ID, P, etc.
        Log.d("SDK", "Pays: ${mrz.issuingCountry}")     // MAR
        Log.d("SDK", "Nom: ${mrz.lastName}")
        Log.d("SDK", "Prenom: ${mrz.firstName}")
        Log.d("SDK", "N° doc: ${mrz.documentNumber}")
        Log.d("SDK", "Nationalite: ${mrz.nationality}")
        Log.d("SDK", "Naissance: ${mrz.birthDate}")     // DD/MM/YYYY
        Log.d("SDK", "Sexe: ${mrz.sex}")                // M, F, <
        Log.d("SDK", "Expiration: ${mrz.expiryDate}")
        Log.d("SDK", "CIN: ${mrz.cinNumber}")           // Numero CIN marocain
    } else {
        Log.w("SDK", "MRZ detectee mais invalide (check digits)")
    }
} else {
    Log.d("SDK", "Pas de MRZ dans le texte — utiliser les champs OCR")
}
```

### Formats MRZ

| Format | Lignes | Chars/ligne | Usage | Exemple |
|--------|--------|-------------|-------|---------|
| TD1 | 3 | 30 | CIN marocaines | `IDMAR...` |
| TD2 | 2 | 36 | Visas, certains ID | `ID...` |
| TD3 | 2 | 44 | Passeports | `P<MAR...` |

### Correction OCR automatique

Le parser corrige automatiquement les erreurs courantes de l'OCR :
- `(` → `<`
- `/` → `<`
- `O` dans les zones numeriques → `0`
- Espaces parasites supprimes

---

## Controle qualite image

Verifier la qualite avant de traiter une image evite les faux resultats.

```kotlin
import com.sdkmatch.sdk.ImageQuality

// Evaluation complete
val quality = ImageQuality.assess(bitmap)

Log.d("SDK", "Flou: ${quality.isBlurry} (score: ${quality.blurScore})")
Log.d("SDK", "Luminosite: ${quality.brightness}")
Log.d("SDK", "Trop sombre: ${quality.isTooDark}")
Log.d("SDK", "Trop clair: ${quality.isTooLight}")
Log.d("SDK", "Acceptable: ${quality.isAcceptable}")

// Guider l'utilisateur
if (!quality.isAcceptable) {
    val message = when {
        quality.isBlurry -> "Photo floue — maintenez l'appareil stable"
        quality.isTooDark -> "Photo trop sombre — augmentez la luminosite"
        quality.isTooLight -> "Photo trop claire — evitez la lumiere directe"
        else -> "Qualite insuffisante — reprenez la photo"
    }
    showGuidance(message)
    return
}

// Verification rapide (luminosite seule)
if (!ImageQuality.isBrightnessOk(bitmap)) {
    showGuidance("Luminosite inadequate")
}
```

### Seuils

| Parametre | Seuil | Description |
|-----------|-------|-------------|
| Blur | Laplacian variance | Variance basse = flou |
| Luminosite min | 50 | En dessous = trop sombre |
| Luminosite max | 220 | Au dessus = trop clair |

---

## Face Matching et Liveness

Comparer la photo du document avec le selfie de l'utilisateur.

### Capture du selfie

```kotlin
// Utiliser la camera frontale pour le selfie
val cameraSelector = CameraSelector.DEFAULT_FRONT_CAMERA
// ... meme configuration CameraX que pour le document
```

### Comparaison

```kotlin
import com.sdkmatch.sdk.FaceComparison

// documentBitmap = photo du document (recto avec photo)
// selfieBitmap = selfie capture par la camera frontale

val result = FaceComparison.compare(this, documentBitmap, selfieBitmap)

// Verification des visages
if (!result.faceDetectedInDoc) {
    Log.e("SDK", "Aucun visage detecte sur le document")
    // → Demander de rescanner le document (recto)
    return
}
if (!result.faceDetectedInSelfie) {
    Log.e("SDK", "Aucun visage detecte sur le selfie")
    // → Demander de reprendre le selfie
    return
}

// Score de correspondance
Log.d("SDK", "Score: ${result.score}")       // 0.0 - 1.0
Log.d("SDK", "Match: ${result.isMatch}")     // true si score >= seuil
Log.d("SDK", "Temps: ${result.processingTime} ms")

// Liveness passive
Log.d("SDK", "Live: ${result.liveness.isLive}")
Log.d("SDK", "Yeux ouverts: ${result.liveness.eyeOpenScore}")
Log.d("SDK", "Angle tete: ${result.liveness.headAngle}")

// Decision finale
if (result.isMatch && result.liveness.isLive) {
    // KYC REUSSI
    showSuccess("Identite verifiee avec succes !")
} else if (result.isMatch && !result.liveness.isLive) {
    // Match OK mais liveness echoue (possible spoofing)
    showWarning("Veuillez reprendre le selfie en conditions naturelles")
} else {
    // Pas de match
    showError("Le visage ne correspond pas au document")
}
```

### Seuils adaptatifs

Le SDK utilise des seuils adaptes aux conditions :

| Condition | Seuil | Usage |
|-----------|-------|-------|
| Standard | 0.75 | Conditions normales |
| Lunettes | 0.70 | Porteur de lunettes |
| Hijab + lunettes | 0.65 | Hijab et lunettes |
| Barbe | 0.68 | Barbe importante |
| Retry (permissif) | 0.55 | Seconde tentative |

### Algorithme de comparaison

1. **Detection ML Kit** : Localise le visage dans chaque image
2. **Recadrage** : Extrait la zone du visage avec 20% de marge
3. **Normalisation** : Redimensionne a 112x112 pixels
4. **Feature extraction** : Grille 8x8, 4 features par cellule (R, G, B, variance luminance)
5. **Similarite cosinus** : Comparaison des 256 features
6. **Liveness passive** : Yeux ouverts + angle naturel de la tete

---

## Flux complet KYC

Exemple end-to-end combinant toutes les etapes :

```kotlin
class KycManager(private val context: Context) {

    private val licence = LicenceManager(context)
    private val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)

    data class KycResult(
        val success: Boolean,
        val documentType: String? = null,
        val fields: Map<String, String> = emptyMap(),
        val mrz: MrzParser.MrzResult? = null,
        val faceScore: Float = 0f,
        val isLive: Boolean = false,
        val error: String? = null
    )

    suspend fun performKyc(
        documentPath: String,
        selfieBitmap: Bitmap
    ): KycResult {

        // 1. Verifier la licence
        if (!licence.isLicenced()) {
            return KycResult(false, error = "Licence invalide")
        }

        // 2. Charger le document
        val docBitmap = BitmapUtils.loadWithRotation(documentPath)
            ?: return KycResult(false, error = "Image illisible")

        // 3. Qualite
        val quality = ImageQuality.assess(docBitmap)
        if (!quality.isAcceptable) {
            docBitmap.recycle()
            return KycResult(false, error = "Qualite image insuffisante")
        }

        // 4. OCR
        val text = try {
            val image = InputImage.fromBitmap(docBitmap, 0)
            val result = Tasks.await(recognizer.process(image))
            result.text
        } catch (e: Exception) {
            docBitmap.recycle()
            return KycResult(false, error = "Erreur OCR: ${e.message}")
        }

        if (text.isBlank()) {
            docBitmap.recycle()
            return KycResult(false, error = "Aucun texte detecte")
        }

        // 5. Detection document
        val detection = DocumentDetector.detect(text)
        if (detection.type == "UNKNOWN") {
            docBitmap.recycle()
            return KycResult(false, error = "Document non reconnu")
        }

        // 6. MRZ
        val mrz = MrzParser.parse(text)

        // 7. Face Matching
        val faceResult = FaceComparison.compare(context, docBitmap, selfieBitmap)
        docBitmap.recycle()

        if (!faceResult.faceDetectedInDoc || !faceResult.faceDetectedInSelfie) {
            return KycResult(false, error = "Visage non detecte")
        }

        // 8. Resultat
        return KycResult(
            success = faceResult.isMatch && faceResult.liveness.isLive,
            documentType = detection.type,
            fields = detection.fields,
            mrz = mrz,
            faceScore = faceResult.score,
            isLive = faceResult.liveness.isLive,
            error = if (!faceResult.isMatch) "Visages ne correspondent pas"
                    else if (!faceResult.liveness.isLive) "Liveness echouee"
                    else null
        )
    }
}
```

### Utilisation dans un ViewModel

```kotlin
class KycViewModel(application: Application) : AndroidViewModel(application) {

    private val kycManager = KycManager(application)
    private val _result = MutableLiveData<KycManager.KycResult>()
    val result: LiveData<KycManager.KycResult> = _result

    fun verify(documentPath: String, selfieBitmap: Bitmap) {
        viewModelScope.launch {
            _result.value = withContext(Dispatchers.IO) {
                kycManager.performKyc(documentPath, selfieBitmap)
            }
        }
    }
}
```

---

## Gestion des erreurs

### Erreurs courantes et solutions

| Erreur | Cause | Solution |
|--------|-------|----------|
| Licence invalide | API key/secret incorrect | Verifier les credentials |
| Licence expiree | Token expire | Appeler `renewToken()` ou `activate()` |
| Aucun texte detecte | Document mal cadre ou flou | Guider le cadrage, verifier `ImageQuality` |
| Document non reconnu | Type de document non supporte | Verifier qu'il s'agit d'un document marocain |
| MRZ invalide | OCR degradee sur la zone MRZ | Reprendre la photo avec meilleur eclairage |
| Visage non detecte | Photo trop petite/sombre | Verifier la qualite et le cadrage |
| Liveness echouee | Yeux fermes ou photo statique | Reprendre en conditions naturelles |
| Score face bas | Photos tres differentes | Verifier que c'est le bon document |

### Pattern retry recommande

```kotlin
var attempts = 0
val maxAttempts = 3

while (attempts < maxAttempts) {
    val result = kycManager.performKyc(documentPath, selfieBitmap)

    if (result.success) {
        // KYC reussi
        break
    }

    attempts++
    if (attempts < maxAttempts) {
        // Guider l'utilisateur pour la prochaine tentative
        showRetryGuidance(result.error, attempts, maxAttempts)
    } else {
        showFinalError("Verification echouee apres $maxAttempts tentatives")
    }
}
```

---

## ProGuard et obfuscation

Les regles ProGuard sont **automatiquement incluses** dans le fat AAR via `consumer-rules.pro`. Aucune configuration supplementaire n'est necessaire.

Les regles incluses preservent :
- `com.sdkmatch.sdk.**` — Toutes les classes du SDK
- `com.google.mlkit.**` — ML Kit
- `org.tensorflow.lite.**` — TFLite runtime
- `androidx.exifinterface.**` — ExifInterface

### Verifier que tout fonctionne en release

```bash
./gradlew assembleRelease
# Tester l'APK release sur un appareil physique
```

---

## FAQ et depannage

### Le AAR est trop gros (43 MB) — comment reduire ?

Le AAR inclut les librairies natives pour 4 architectures. Pour reduire la taille de l'APK final, ajoutez un filtre ABI :

```kotlin
// app/build.gradle.kts
android {
    defaultConfig {
        ndk {
            abiFilters += listOf("arm64-v8a", "armeabi-v7a")
            // Exclut x86 et x86_64 (emulateurs)
        }
    }
}
```

Cela reduit la taille d'environ 40% en production.

### Conflit de dependances avec ML Kit

Si votre app utilise deja ML Kit, il peut y avoir des conflits de classes. Solutions :

```kotlin
// Option 1 : Exclure ML Kit de vos dependances (le SDK le fournit)
implementation("com.google.mlkit:text-recognition:16.0.1") {
    // Supprimer cette ligne — deja dans le SDK
}

// Option 2 : Si versions differentes, forcer la resolution
configurations.all {
    resolutionStrategy {
        force("com.google.mlkit:text-recognition:16.0.1")
        force("com.google.mlkit:face-detection:16.1.7")
    }
}
```

### L'OCR ne detecte rien

1. Verifier `ImageQuality.assess()` — l'image est-elle acceptable ?
2. Verifier le cadrage — le document doit remplir au moins 60% de l'image
3. Verifier l'eclairage — eviter les reflets et les ombres
4. Utiliser `BitmapUtils.loadWithRotation()` — CameraX peut enregistrer l'image avec une mauvaise rotation

### Le Face Matching echoue systematiquement

1. La photo du document est-elle nette ? (pas de reflet sur la photo d'identite)
2. Le selfie est-il bien eclaire ?
3. Verifier `faceDetectedInDoc` et `faceDetectedInSelfie` — si false, le visage n'est pas detecte
4. Essayer sans lunettes ou accessoires qui masquent le visage

### Erreur "Licence check failed" en debug

En mode debug, la signature APK differe de la release. Le SDK valide la signature pour la securite. Solutions :
- Tester en release build : `./gradlew installRelease`
- Ou desactiver temporairement la validation signature pendant le developpement

### App crash au demarrage

Verifier que `minSdk` est bien 26+. Le SDK utilise des API Java 8+ (LocalDateTime, etc.) non disponibles avant Android 8.0.
