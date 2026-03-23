# 🛡️ SDK MATCH — KYC Mobile SDK (On-Device)

<p align="center">
  <strong>Scanner. Vérifier. Valider.</strong><br>
  SDK de vérification d'identité 100% on-device pour les documents marocains.
</p>

---

## ✨ Fonctionnalités

| | Feature | Description |
|---|---------|-------------|
| 📄 | **OCR & MRZ** | Lecture automatique (CIN v1, CIN v2, Passeport, Permis) |
| 🧠 | **MRZ Parser** | ICAO 9303 complet (TD1/TD2/TD3), check digits |
| 👤 | **Face Matching** | ML Kit Face Detection + comparaison document/selfie |
| ✅ | **Passive Liveness** | Détection de vivacité (anti-spoofing) |
| 📸 | **CameraX** | Caméra embarquée native avec guidage temps réel |
| 🔤 | **ML Kit OCR** | Google ML Kit Text Recognition on-device |
| 🧕 | **Seuils Adaptatifs** | Lunettes (0.70), Hijab+lunettes (0.65), standard (0.75) |
| 🔒 | **Privacy First** | Zéro cloud, données éphémères |
| 🔑 | **Licence Server** | Cloudflare Workers + D1 |

---

## 📱 Application Android Native

App de démonstration en **Kotlin natif** :

- 📸 **CameraX** — Preview temps réel intégré dans l'interface
- 🔤 **ML Kit OCR** — Extraction de texte on-device
- 👤 **ML Kit Face Detection** — Détection + Liveness passif
- 📄 **Documents** — CIN v1, CIN v2, Passeport, Permis de conduire
- 🔄 **Recto/Verso** — Scan des deux côtés avec aperçu
- 🔦 **Flash** — Toggle flash sur la caméra
- 📊 **Historique** — Sauvegarde locale des scans

### 📥 Télécharger l'APK

L'APK est disponible dans les releases du projet.

---

## 🔑 Obtenir une licence

1. Rendez-vous sur **[sdmatch.netlify.app](https://sdmatch.netlify.app/)**
2. Remplissez le formulaire "Demander une démo"
3. Recevez votre **API Key** et **API Secret** (7 jours gratuit)
4. Intégrez dans l'application

---

## 🏗️ Architecture

```
┌─────────────────────────────┐
│   📱 App Android Native     │
│   Kotlin + CameraX + ML Kit │
└────────────┬────────────────┘
             │
┌────────────▼────────────────┐
│   ⚙️ C++ Core Engine        │
│   📄 MRZ Parser (ICAO 9303) │
│   👤 Face Match + Liveness   │
│   🔍 Quality Control         │
│   🔐 Licence Validator       │
└────────────┬────────────────┘
             │ FFI / JNI
┌────────────▼────────────────┐
│   🔌 Platform Wrappers      │
│   iOS │ Android │ MAUI      │
│   Flutter │ React Native    │
└─────────────────────────────┘
```

---

## 📚 Documentation

| | Doc | Lien |
|---|-----|------|
| 🍎 | Quick Start iOS | [docs/quickstart/ios.md](kyc-sdk/docs/quickstart/ios.md) |
| 🤖 | Quick Start Android | [docs/quickstart/android.md](kyc-sdk/docs/quickstart/android.md) |
| 💜 | Quick Start MAUI | [docs/quickstart/maui.md](kyc-sdk/docs/quickstart/maui.md) |
| 💙 | Quick Start Flutter | [docs/quickstart/flutter.md](kyc-sdk/docs/quickstart/flutter.md) |
| ⚛️ | Quick Start React Native | [docs/quickstart/react-native.md](kyc-sdk/docs/quickstart/react-native.md) |
| 📖 | API Reference | [docs/api-reference](kyc-sdk/docs/api-reference/README.md) |

---

## 🛠️ Tech Stack

| | Composant | Technologie |
|---|-----------|-------------|
| 📱 | App Android | Kotlin natif |
| 📸 | Caméra | Android CameraX |
| 🔤 | OCR | Google ML Kit |
| 👤 | Face Detection | Google ML Kit |
| ✅ | Liveness | ML Kit (head pose, eyes) |
| ⚙️ | Core Engine | C++17 |
| 🔧 | Build | CMake / Gradle |
| 🌐 | Site Web | HTML/CSS/JS (Netlify) |
| 🔑 | Licences | Cloudflare Workers + D1 |

---

## 🌐 Services

| | Service | URL |
|---|---------|-----|
| 🌐 | Site Vitrine | [sdmatch.netlify.app](https://sdmatch.netlify.app/) |
| 🔑 | Licence API | [sdmatch-licence-api.khalilbenaz.workers.dev](https://sdmatch-licence-api.khalilbenaz.workers.dev/) |

---

## 📄 Licence

Propriétaire — SDK MATCH © 2026

---

<p align="center">
  Made in Morocco 🇲🇦
</p>
