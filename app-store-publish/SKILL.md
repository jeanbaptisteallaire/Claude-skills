---
name: app-store-publish
description: Publish an iOS app (Flutter or native Xcode) to App Store Connect end to end — signing without registered devices, upload, TestFlight, store listing, screenshots, privacy, age rating, pricing and review submission. Use when the user asks to publish / ship / submit an app to the Apple App Store, App Store Connect, or TestFlight ("publication store apple", "mettre sur l'App Store", "TestFlight sur mon iPhone").
---

# Publish an iOS app on the App Store

Field-tested on Fusion Habits (Flutter, Oct 2026, App Store Connect in **French**). Work through the
phases in order and keep the user informed with short updates. The user speaks French: answer in French.

## 0. Before anything — confirm what ships

- **Identify the real app codebase first.** Check the memory, `CLAUDE.md`, and the folder layout. A
  repo can hold an old version (e.g. a legacy Expo app) next to the live one (e.g. Flutter in `app/`).
  Never build the wrong one. If unsure, ask.
- Never `git checkout --` / revert files with uncommitted changes to "undo" your own edit — you can
  wipe the user's work. Edit back precisely instead.
- If the folder layout changes under you mid-task, the user (or another session) did it on purpose.
  Don't "fix" it; ask.

## 1. Account prerequisites (the user must do these — never you)

| Item | Where | Notes |
|---|---|---|
| Paid Apple Developer Program | developer.apple.com | 99 $/yr |
| **Accept updated license agreement** | https://developer.apple.com/account (banner "Review Agreement") or https://appstoreconnect.apple.com/business | Shown as a yellow banner on the App Store Connect app list. Legal commitment by the account holder: tell the user where, never click it yourself. |
| Apple ID signed in to Xcode | Xcode › Settings › Accounts | Lets `-allowProvisioningUpdates` create certs/profiles and upload. |
| Apple passwords / 2FA | — | Never type them. If a CLI asks, hand the terminal to the user. |

Check signing identities: `security find-identity -v -p codesigning` (need an "Apple Distribution" one).

## 2. Project config checklist (iOS)

- Bundle id set (e.g. `com.company.app`), team id in the Xcode project (`DEVELOPMENT_TEAM`).
- `ITSAppUsesNonExemptEncryption = false` in `ios/Runner/Info.plist` (if no custom crypto) →
  skips the export-compliance question on every build:
  `/usr/libexec/PlistBuddy -c "Add :ITSAppUsesNonExemptEncryption bool false" ios/Runner/Info.plist`
- Usage strings for every permission actually used (`NSCameraUsageDescription`,
  `NSPhotoLibraryUsageDescription`, …).
- **iPhone-only?** Flutter defaults to `TARGETED_DEVICE_FAMILY = "1,2"` (iPhone + iPad). With iPad
  declared, Apple **requires 13" iPad screenshots** and reviews on iPad. Decide BEFORE the first
  release (iPad support can't be removed after publishing):
  `sed -i '' 's/TARGETED_DEVICE_FAMILY = "1,2";/TARGETED_DEVICE_FAMILY = 1;/' ios/Runner.xcodeproj/project.pbxproj`
  Verify in the archive: `UIDeviceFamily` must be `[1]`.
- **App icon**: 1024×1024 PNG, **no alpha**, sharp (generate from the highest-res source, never
  upscale a 512). Flutter: `flutter_launcher_icons` with `remove_alpha_ios: true`, then
  `dart run flutter_launcher_icons`. Look at the result before shipping.
- Version: bump `version: X.Y.Z+BUILD` in `pubspec.yaml`. Every upload needs a new build number.
- Run the project's checks (`dart format . && flutter analyze && flutter test`) before building.

## 3. Build, sign and upload (works with NO registered device)

`flutter build ipa` fails with *"Your team has no devices from which to generate a provisioning
profile"* because automatic signing wants a development profile. Don't override
`CODE_SIGN_IDENTITY` globally either (conflicts with Swift package targets). Instead:

```bash
cd app   # Flutter project root
flutter build ipa --release --no-codesign          # unsigned archive → build/ios/archive/Runner.xcarchive
```

`build/ios/ExportUpload.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
<key>method</key><string>app-store-connect</string>
<key>teamID</key><string>TEAM_ID</string>
<key>signingStyle</key><string>automatic</string>
<key>destination</key><string>upload</string>   <!-- "export" to only produce the .ipa -->
<key>uploadSymbols</key><true/>
</dict></plist>
```

```bash
xcodebuild -exportArchive -archivePath build/ios/archive/Runner.xcarchive \
  -exportPath build/ios/upload -exportOptionsPlist build/ios/ExportUpload.plist \
  -allowProvisioningUpdates
```
Xcode re-signs with Apple Distribution, creates the App Store profile, registers the bundle id and
uploads ("Upload succeeded"). Notes:
- Run it **outside the sandbox** and with `-exportPath` inside the project (a temp/scratch path gave
  "exportArchive Copy failed").
- Takes ~1–3 min; run long builds in the background.
- Check the archive: `plutil -p build/ios/archive/Runner.xcarchive/Products/Applications/Runner.app/Info.plist | grep -E "CFBundleVersion|CFBundleShortVersionString|UIDeviceFamily|DTSDKName"`.

## 4. App Store Connect — create the app record (first time only)

Apps › **+** › **Nouvelle app**: Plateforme iOS · Nom · Langue principale · Identifiant de lot (Xcode
already registered it, shows as "XC com…") · UGS/SKU (e.g. `appname-ios`) · Accès complet → Créer.

Browser tips (built-in browser on appstoreconnect.apple.com):
- `form_input` sets values React ignores → **click + cmd+a + type** for text fields; verify with JS
  (`[...document.querySelectorAll('main input, main textarea')].map(e=>[e.name,e.value])`).
  Native `<select>` works with `form_input`. Radios: JS `.click()` on the input works.
- Screenshots of the page can lag; trust a JS read of the values.
- **Narrow pane hides parts of the UI** (sidebar, the screenshots tab). Use
  `resize_window` 1440×1000.
- Direct URLs sometimes spin forever (e.g. `/distribution/pricing`); click the sidebar link instead.
- The built-in browser **cannot upload local files**. Either the user drags files in, or host them
  publicly and inject via JS (fetch → File → DataTransfer → input.files). Ask first.
- If pages start changing on their own, the user is driving the pane — stop and tell them.

## 5. TestFlight on the user's iPhone (fast)

1. After upload: TestFlight tab → build shows "En cours de traitement" (5–15 min).
2. **Créer un groupe** (internal) e.g. "Equipe interne", keep **Activer la distribution
   automatique** checked (future builds reach testers automatically).
3. **Inviter des testeurs** → select the account holder (internal testers = App Store Connect users).
4. When the build is "Prêt à soumettre/tester", the user installs the **TestFlight** app, signs in
   with the developer Apple ID → Installer. Builds expire after 90 days.

Wait with a background `sleep 300` (never foreground sleep / polling loops), then re-check.

## 6. Store listing — where everything is (French UI)

Left sidebar of the app: **Général** (Informations sur l'app, Vérification de l'app), **App Store /
Confiance et sécurité** (Confidentialité de l'app…), **Monétisation** (Tarifs et disponibilité).
The version page is under **App iOS › X.Y.Z À finaliser avant soumission**.

### 6a. Version page (App iOS Version X.Y.Z)
- Top of page: **Informations de la page produit** has tabs **« En‑tête et résultats de recherche »**
  (optional landscape banners, 5244×2950 etc. — leave empty) and **« Aperçus d'app et captures
  d'écran »** (the real screenshots; tab hidden in narrow viewports).
  - Device tab **« iPhone avec Dynamic Island (écran moyen) »** accepts **1179×2556, 1206×2622,
    2556×1179, 2622×1206**. Apple scales them down for smaller iPhones.
  - Counters: **« x/3 aperçus d'app »** = optional **videos** (886×1920) → can stay 0.
    **« n sur 10 captures d'écran d'app »** = screenshots, **min 1, max 10** (put 3–5: the first 3
    show in search results).
  - iPhone Duo: not required (screenshots only required from April 2027 for apps built with
    SDK ≥ iOS 27.1). iPad: only if the build declares iPad. Watch: no.
  - A grey tile with a cloud = still processing; refresh, re-upload if stuck > 15 min.
- Texte promotionnel (170) · Description (4000) · Mots-clés (100, comma-separated, no spaces needed)
  · URL de l'assistance · URL marketing · **Version** (must equal `CFBundleShortVersionString`, e.g.
  `2.0.0`) · Copyright (`2026 Full Name`).
- **Build** section (below Copyright, above Game Center): « Ajouter un build » → pick the build →
  Terminé. To swap: hover the build row → red ⊖ at the far right → remove → add the new one.
- **Informations utiles à la vérification de l'app**: uncheck **Connexion requise** if no login;
  else give a demo account. **Coordonnées**: first name, last name, **phone (international format,
  ask the user — read it back to confirm)**, email. **Remarques**: explain no-login, permissions,
  data storage.
- Publication: automatic after approval (default) or manual.
- **Enregistrer** (top right). Save is refused while a required field (e.g. phone) is empty.

### 6b. Informations sur l'app
- Sous-titre (30) · Catégorie principale / secondaire (`<select>`, e.g. HEALTH_AND_FITNESS /
  PRODUCTIVITY).
- **Droits relatifs au contenu** → « Non, elle ne contient … aucun contenu tiers » (if true) → Terminé.
- **Classifications par âge** → 7-step questionnaire: Fonctionnalités (parental controls, web,
  UGC, social, chat, ads: yes/no) · Thèmes matures · Médical ou bien‑être · Sexualité · Violence ·
  Hasard · Infos supplémentaires. Answer honestly; a habit/wellness app answering "Sujets de santé
  ou de bien-être = Oui" gets **9+**. Check the last step's radio ("Non applicable") before saving.
- **Dispositif médical réglementé**: required for Health & Fitness / Medical categories (EU/UK/US)
  → declare "No" if it isn't one.
- Enregistrer.

### 6c. Confidentialité de l'app
- Politique de confidentialité › Modifier → **URL** (required, must be live and accurate).
- Démarrer → « Collectez-vous des données ? » → answer from the code: list runtime dependencies
  (analytics, crash reporting, backend SDKs count as collection). → Enregistrer → **Publier** →
  confirm the attestation.

### 6d. Tarifs et disponibilité
- Ajouter la tarification → pays de base → Prix → **0,00 $** (free) → confirm.
- Configurer la disponibilité → « Tous les pays ou régions » (175) → Suivant → Confirmer.

### 6e. Submit
Version page → **Ajouter pour vérification** → fix anything it lists → **Soumettre pour
vérification**. Review usually takes 24–48 h.

## 7. Privacy policy & support URL (required)

- Check the URL actually responds (`curl -s -o /dev/null -w '%{http_code}' URL`, outside sandbox).
- The policy must match the shipped version (e.g. a local-only app must say "no data collected").
  Rewrite outdated generated policies. Remove false claims (AI, accounts, sync) from site and
  description — misleading metadata gets rejected.
- Quick free hosting (ask before publishing anything): a small **public** GitHub repo with the
  static site + GitHub Pages:
  ```bash
  gh repo create USER/app-site --public --source . --push
  git -c http.postBuffer=524288000 push -u origin main   # if push of images disconnects
  gh api -X POST repos/USER/app-site/pages -f "source[branch]=main" -f "source[path]=/"
  ```
  → `https://USER.github.io/app-site/privacy-policy.html` (live in ~1 min).

## 8. Screenshots — how to make them

iOS Simulator, largest iPhone (e.g. "iPhone 18 Pro Max" → 1320×2868), app in a demo/sample-data mode:
```bash
xcrun simctl status_bar <UDID> override --time "9:41" --batteryState charged --batteryLevel 100 --cellularBars 4 --wifiBars 3
xcrun simctl io <UDID> screenshot 01-home.png
for f in 0*.png; do sips -z 2622 1206 "$f" --out "iphone-6.3/$f"; done   # size the iPhone tab accepts
```
Dismiss tutorials/popups first. Navigate with the simulator `tap` tool (device points = pixels/3).

## 9. Wrap-up

- Commit the release changes on a branch (icon, device family, build number, site, listing doc),
  Conventional Commits, tag `vX.Y.Z` if the project does that.
- Save the store texts in the repo (e.g. `docs/app-store-listing.md`) for the next version.
- Tell the user exactly what is left for them and where.
