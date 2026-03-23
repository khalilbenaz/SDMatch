# .NET MAUI — Quick Start

Integrer SDK MATCH dans votre application .NET MAUI en 5 minutes.

## Pre-requis

- .NET 8.0+
- Visual Studio 2022 17.8+ ou JetBrains Rider
- Android API 26+ / iOS 15+

## Installation

### NuGet (quand disponible)
```bash
dotnet add package SDKMatch.Maui
```

### Reference locale
```xml
<!-- .csproj -->
<ProjectReference Include="../sdk/maui/SDKMatch.Maui.csproj" />
```

## Permissions

### Android (`Platforms/Android/AndroidManifest.xml`)
```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.INTERNET" />
```

### iOS (`Platforms/iOS/Info.plist`)
```xml
<key>NSCameraUsageDescription</key>
<string>Camera requise pour le KYC</string>
```

## Utilisation

```csharp
using SDKMatch.Maui;

// 1. Activer la licence
var sdkMatch = new SdkMatchClient();
var activation = await sdkMatch.ActivateAsync("API_KEY", "API_SECRET");

if (!activation.Success)
{
    Console.WriteLine($"Erreur: {activation.Message}");
    return;
}

// 2. Detecter le document
var detector = new DocumentDetector();
var detection = await detector.DetectAsync(ocrText);
Console.WriteLine($"Type: {detection.Type}");
foreach (var (key, value) in detection.Fields)
    Console.WriteLine($"  {key} = {value}");

// 3. Parser la MRZ
var parser = new MrzParser();
var mrz = await parser.ParseAsync(ocrText);
if (mrz is { IsValid: true })
{
    Console.WriteLine($"MRZ: {mrz.LastName} {mrz.FirstName}");
    Console.WriteLine($"CIN: {mrz.CinNumber}");
}

// 4. Qualite image
var quality = new ImageQualityChecker();
var result = await quality.AssessAsync(imageBytes);
if (!result.IsAcceptable)
{
    Console.WriteLine("Image de mauvaise qualite");
    return;
}

// 5. Face Matching
var comparison = new FaceComparer();
var match = await comparison.CompareAsync(documentBytes, selfieBytes);

if (match.IsMatch && match.Liveness.IsLive)
{
    Console.WriteLine($"Identite verifiee ! Score: {match.Score:P0}");
}
```

## Voir aussi

- [Reference API](../api-reference/README.md)
- [Source MAUI SDK](../../sdk/maui/)
