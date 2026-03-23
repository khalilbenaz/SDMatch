# Flux KYC complet — Exemple production

Exemple complet d'un parcours KYC avec SDK MATCH en Kotlin Android. Inclut la gestion d'erreurs, les retentatives et le nettoyage des ressources.

## Flux complet

```kotlin
import android.content.Context
import android.graphics.Bitmap
import android.util.Log
import com.sdkmatch.sdk.*
import com.google.mlkit.vision.text.TextRecognition
import com.google.mlkit.vision.text.latin.TextRecognizerOptions
import com.google.mlkit.vision.common.InputImage

class KycVerificationFlow(private val context: Context) {

    private val TAG = "KYC"
    private val licence = LicenceManager(context)
    private val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)

    // ── Point d'entree ──
    suspend fun verify(
        apiKey: String,
        apiSecret: String,
        documentImagePath: String,
        documentVersoPath: String?,
        selfieBitmap: Bitmap
    ): KycVerificationResult {

        // 1. Licence
        if (!ensureLicence(apiKey, apiSecret)) {
            return KycVerificationResult.Error("Licence invalide ou expiree")
        }

        // 2. Charger et valider l'image du document
        val docBitmap = BitmapUtils.loadWithRotation(documentImagePath)
            ?: return KycVerificationResult.Error("Impossible de charger l'image du document")

        val quality = ImageQuality.assess(docBitmap)
        if (!quality.isAcceptable) {
            return KycVerificationResult.Error(
                when {
                    quality.isBlurry -> "Image floue (score: ${quality.blurScore})"
                    quality.isTooDark -> "Image trop sombre (luminosite: ${quality.brightness})"
                    quality.isTooLight -> "Image trop claire (luminosite: ${quality.brightness})"
                    else -> "Qualite image insuffisante"
                }
            )
        }

        // 3. OCR et detection du document
        val ocrText = performOcr(docBitmap)
            ?: return KycVerificationResult.Error("Echec de la reconnaissance OCR")

        val detection = DocumentDetector.detect(ocrText)
        if (detection.type == "UNKNOWN") {
            return KycVerificationResult.Error("Type de document non reconnu")
        }

        Log.d(TAG, "Document detecte: ${detection.type}")
        Log.d(TAG, "Champs: ${detection.fields}")

        // 4. Parser la MRZ (si disponible)
        val mrz = MrzParser.parse(ocrText)
        if (mrz != null && mrz.isValid) {
            Log.d(TAG, "MRZ valide: ${mrz.format} — ${mrz.lastName} ${mrz.firstName}")
        } else {
            Log.w(TAG, "MRZ non trouvee ou invalide — utilisation des champs OCR")
        }

        // 5. Verso du document (optionnel)
        var versoFields: Map<String, String>? = null
        if (documentVersoPath != null) {
            val versoBitmap = BitmapUtils.loadWithRotation(documentVersoPath)
            if (versoBitmap != null) {
                val versoQuality = ImageQuality.assess(versoBitmap)
                if (versoQuality.isAcceptable) {
                    val versoText = performOcr(versoBitmap)
                    if (versoText != null) {
                        val versoDetection = DocumentDetector.detect(versoText)
                        versoFields = versoDetection.fields
                    }
                }
                versoBitmap.recycle()
            }
        }

        // 6. Face Matching + Liveness
        val selfieQuality = ImageQuality.assess(selfieBitmap)
        if (!selfieQuality.isAcceptable) {
            return KycVerificationResult.Error("Qualite du selfie insuffisante")
        }

        val faceResult = FaceComparison.compare(context, docBitmap, selfieBitmap)

        if (!faceResult.faceDetectedInDoc) {
            return KycVerificationResult.Error("Aucun visage detecte sur le document")
        }
        if (!faceResult.faceDetectedInSelfie) {
            return KycVerificationResult.Error("Aucun visage detecte sur le selfie")
        }

        // 7. Nettoyage
        docBitmap.recycle()

        // 8. Resultat final
        return KycVerificationResult.Success(
            documentType = detection.type,
            documentFields = detection.fields,
            mrzData = mrz,
            versoFields = versoFields,
            faceMatchScore = faceResult.score,
            isMatch = faceResult.isMatch,
            isLive = faceResult.liveness.isLive,
            liveness = faceResult.liveness,
            processingTimeMs = faceResult.processingTime
        )
    }

    // ── Helpers ──

    private fun ensureLicence(apiKey: String, apiSecret: String): Boolean {
        return when (licence.getStatus()) {
            LicenceStatus.VALID -> true
            LicenceStatus.EXPIRED_GRACE -> {
                Log.w(TAG, "Licence en grace period — renouvellement...")
                licence.renewToken().success
            }
            LicenceStatus.NO_LICENCE, LicenceStatus.EXPIRED -> {
                Log.d(TAG, "Activation de la licence...")
                licence.activate(apiKey, apiSecret).success
            }
        }
    }

    private suspend fun performOcr(bitmap: Bitmap): String? {
        return try {
            val image = InputImage.fromBitmap(bitmap, 0)
            val result = com.google.android.gms.tasks.Tasks.await(
                recognizer.process(image)
            )
            result.text.takeIf { it.isNotBlank() }
        } catch (e: Exception) {
            Log.e(TAG, "Erreur OCR: ${e.message}")
            null
        }
    }
}

// ── Modele de resultat ──

sealed class KycVerificationResult {
    data class Success(
        val documentType: String,
        val documentFields: Map<String, String>,
        val mrzData: MrzParser.MrzResult?,
        val versoFields: Map<String, String>?,
        val faceMatchScore: Float,
        val isMatch: Boolean,
        val isLive: Boolean,
        val liveness: FaceComparison.LivenessResult,
        val processingTimeMs: Long
    ) : KycVerificationResult()

    data class Error(val message: String) : KycVerificationResult()
}
```

## Utilisation

```kotlin
// Dans une Activity ou un ViewModel
val flow = KycVerificationFlow(applicationContext)

lifecycleScope.launch {
    val result = flow.verify(
        apiKey = "VOTRE_API_KEY",
        apiSecret = "VOTRE_API_SECRET",
        documentImagePath = "/path/to/document_recto.jpg",
        documentVersoPath = "/path/to/document_verso.jpg",  // null si pas de verso
        selfieBitmap = selfieBitmap
    )

    when (result) {
        is KycVerificationResult.Success -> {
            Log.d("KYC", "Verifie: ${result.documentType}")
            Log.d("KYC", "Nom: ${result.documentFields["nom"]}")
            Log.d("KYC", "Score face: ${result.faceMatchScore}")
            Log.d("KYC", "Match: ${result.isMatch}, Live: ${result.isLive}")

            result.mrzData?.let {
                Log.d("KYC", "MRZ: ${it.lastName} ${it.firstName} — CIN: ${it.cinNumber}")
            }
        }
        is KycVerificationResult.Error -> {
            Log.e("KYC", "Echec KYC: ${result.message}")
        }
    }
}
```

## Notes

- **Temps de traitement** : < 1.5 seconde pour le flux complet (OCR + Face Match)
- **Memoire** : Recycler les bitmaps apres utilisation pour eviter les fuites memoire
- **CameraX** : Non inclus dans cet exemple — gerer la capture camera cote application
- **Retry** : En cas d'echec Face Match, permettre a l'utilisateur de reprendre le selfie (jusqu'a 3 essais recommandes)
