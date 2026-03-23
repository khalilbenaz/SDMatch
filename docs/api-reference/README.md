# SDK MATCH — Reference API

Reference complete de l'API du SDK MATCH (Android Kotlin).

Package : `com.sdkmatch.sdk`

---

## LicenceManager

Gestion de la licence SDK. Activation, validation et renouvellement.

### Constructeur

```kotlin
LicenceManager(context: Context)
```

### Methodes

| Methode | Retour | Description |
|---------|--------|-------------|
| `activate(apiKey: String, apiSecret: String)` | `ActivationResult` | Active la licence aupres du serveur. Stocke le token localement. |
| `isLicenced()` | `Boolean` | Verifie si une licence valide existe (inclut grace period). |
| `getStatus()` | `LicenceStatus` | Retourne le statut detaille de la licence. |
| `needsRenewal()` | `Boolean` | Indique si le token doit etre renouvele. |
| `renewToken()` | `ActivationResult` | Renouvelle le token avec les credentials stockes. |
| `clearLicence()` | `Unit` | Supprime la licence locale. |

### Types

```kotlin
enum class LicenceStatus {
    VALID,          // Licence active
    EXPIRED_GRACE,  // Expiree mais dans la grace period (72h)
    EXPIRED,        // Expiree, SDK bloque
    NO_LICENCE      // Aucune licence
}

data class ActivationResult(
    val success: Boolean,
    val message: String,
    val plan: String?,        // "demo", "pro", etc.
    val features: List<String>? // ["ocr", "face_match", "mrz", ...]
)
```

### Securite

- **Validation APK** : Verifie la signature SHA-256 de l'APK (anti-tampering)
- **Device fingerprint** : Lie la licence au ANDROID_ID de l'appareil
- **Grace period** : 72 heures apres expiration du token
- **Stockage** : SharedPreferences chiffrees

---

## DocumentDetector

Detection du type de document et extraction des champs a partir du texte OCR.

### Methodes

| Methode | Retour | Description |
|---------|--------|-------------|
| `detect(ocrText: String)` | `DetectionResult` | Detecte le type de document et extrait les champs. |

### Types

```kotlin
data class DetectionResult(
    val type: String,                    // "CIN_V1", "CIN_V2", "PASSPORT", "PERMIS", "UNKNOWN"
    val fields: LinkedHashMap<String, String> // Champs extraits
)
```

### Champs extraits par type

**CIN v1 / v2 :**
- `cin` — Numero CIN (lettre + 6 chiffres, ex: "AB123456")
- `nom` — Nom de famille
- `prenom` — Prenom
- `date_naissance` — Date de naissance
- `lieu_naissance` — Lieu de naissance
- `sexe` — M ou F
- `nationalite` — Nationalite
- `adresse` — Adresse
- `date_validite` — Date de validite

**Passeport :**
- `numero` — Numero de passeport
- `nom`, `prenom`
- `date_naissance`, `lieu_naissance`
- `sexe`, `nationalite`
- `date_expiration`

**Permis de conduire :**
- `cnie` — Numero CNIE
- `nom`, `prenom`
- `date_naissance`
- `categories` — Categories du permis (A, B, C...)
- `date_delivrance`

### Logique de detection

1. **MRZ prioritaire** : Recherche de patterns MRZ (IDMAR, P<, ID)
2. **Mots-cles** : Detection par termes specifiques (PASSEPORT, PERMIS, CARTE NATIONALE...)
3. **Marqueurs v2** : Detection CIN nouvelle generation (OPID, CAN, sequences alphanumeriques longues)

---

## MrzParser

Parser MRZ conforme a la norme ICAO 9303 (Document 9303).

### Methodes

| Methode | Retour | Description |
|---------|--------|-------------|
| `parse(text: String)` | `MrzResult?` | Parse la MRZ depuis du texte brut. Retourne null si aucune MRZ trouvee. |

### Types

```kotlin
data class MrzResult(
    val format: String,         // "TD1", "TD2", "TD3"
    val documentType: String,   // "ID", "P", etc.
    val issuingCountry: String, // Code ISO 3166-1 alpha-3
    val lastName: String,
    val firstName: String,
    val documentNumber: String,
    val nationality: String,
    val birthDate: String,      // Format DD/MM/YYYY
    val sex: String,            // "M", "F", "<"
    val expiryDate: String,     // Format DD/MM/YYYY
    val cinNumber: String?,     // Numero CIN marocain (si applicable)
    val isValid: Boolean,       // Tous les check digits OK
    val rawLines: List<String>  // Lignes MRZ brutes
)
```

### Formats supportes

| Format | Lignes | Caracteres/ligne | Usage |
|--------|--------|-------------------|-------|
| TD1 | 3 | 30 | Cartes d'identite (CIN) |
| TD2 | 2 | 36 | Visas, certains ID |
| TD3 | 2 | 44 | Passeports |

### Validation

- **Check digits** : Algorithme poids 7-3-1 mod 10 (norme ICAO)
- **Correction OCR** : Remplacement automatique des erreurs courantes (`(` → `<`, `/` → `<`, etc.)
- **Dates** : Conversion YYMMDD → DD/MM/YYYY (00-30 = 2000s, 31-99 = 1900s)
- **Pays** : Mapping ISO alpha-3 vers noms francais

---

## FaceComparison

Comparaison faciale entre la photo du document et le selfie de l'utilisateur.

### Methodes

| Methode | Retour | Description |
|---------|--------|-------------|
| `compare(context: Context, documentBitmap: Bitmap, selfieBitmap: Bitmap)` | `FaceMatchResult` | Compare deux visages et verifie la liveness. |

### Types

```kotlin
data class FaceMatchResult(
    val score: Float,             // Score de similarite (0.0 - 1.0)
    val isMatch: Boolean,         // Score >= seuil
    val details: String,          // Description du resultat
    val faceDetectedInDoc: Boolean,
    val faceDetectedInSelfie: Boolean,
    val liveness: LivenessResult,
    val processingTime: Long      // Temps de traitement en ms
)

data class LivenessResult(
    val isLive: Boolean,          // Resultat liveness passif
    val eyeOpenScore: Float,      // Probabilite yeux ouverts
    val headAngle: Float,         // Angle de la tete
    val details: String
)
```

### Algorithme

1. **Detection** : ML Kit Face Detection sur les deux images
2. **Recadrage** : Extraction du visage avec marge de 20%
3. **Liveness passive** :
   - Probabilite yeux ouverts > 0.3
   - Angle naturel de la tete (Y/Z > 1.0 degre)
4. **Comparaison** : Grille 8x8 avec 4 features par cellule (R, G, B, variance luminance)
5. **Similarite cosinus** : Produit scalaire normalise des vecteurs de features (256 dimensions)

### Constantes

| Constante | Valeur | Description |
|-----------|--------|-------------|
| `GRID_SIZE` | 8 | Taille de la grille de features |
| `RESIZE_DIM` | 112 | Dimension de redimensionnement (112x112 px) |
| `MATCH_THRESHOLD` | 0.55 | Seuil minimum de correspondance |

---

## ImageQuality

Evaluation de la qualite d'une image capturee.

### Methodes

| Methode | Retour | Description |
|---------|--------|-------------|
| `assess(bitmap: Bitmap)` | `QualityResult` | Evalue le flou et la luminosite. |
| `isBrightnessOk(bitmap: Bitmap)` | `Boolean` | Verification rapide de la luminosite. |

### Types

```kotlin
data class QualityResult(
    val isBlurry: Boolean,    // Flou detecte (Laplacian variance basse)
    val blurScore: Double,    // Score de nettete
    val brightness: Double,   // Luminosite moyenne (0-255)
    val isTooLight: Boolean,  // Luminosite > 220
    val isTooDark: Boolean,   // Luminosite < 50
    val isAcceptable: Boolean // Non flou ET luminosite OK
)
```

### Algorithme

- **Flou** : Variance du Laplacien (noyau 3x3) sur image sous-echantillonnee (25%)
- **Luminosite** : Moyenne des pixels en niveaux de gris
- **Seuils** : MIN_BRIGHTNESS = 50, MAX_BRIGHTNESS = 220

---

## BitmapUtils

Utilitaire de chargement d'images avec correction de rotation.

### Methodes

| Methode | Retour | Description |
|---------|--------|-------------|
| `loadWithRotation(filePath: String)` | `Bitmap` | Charge un bitmap en appliquant la rotation EXIF. |

### Comportement

- Lit le tag EXIF `ORIENTATION` de l'image
- Applique la rotation correspondante (90, 180 ou 270 degres)
- Indispensable pour les images capturees par CameraX (orientation souvent incorrecte)
- Recycle automatiquement le bitmap original apres rotation
