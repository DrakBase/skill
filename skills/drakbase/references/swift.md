# App Swift (macOS) : le paquet DrakBase

## Conditions

- macOS 14 ou plus, outils Swift 6.2 ou plus.
- **App hors sandbox.** Pour une app sandboxée, l'installation passera par un service XPC qui n'est pas encore livré : l'updater s'y dirait « indisponible ». Préviens l'utilisateur avant d'intégrer.
- Un **certificat de signature de code** dans le trousseau (`security find-identity -v -p codesigning`). Un certificat auto-signé suffit, mais toutes les versions doivent être signées par le même : l'app n'installe que des versions qui satisfont son exigence désignée. Changer de certificat demande une version de transition qui déclare le nouveau dans `DrakBaseTrustedRequirements`.
- L'app tourne depuis un dossier où l'utilisateur peut écrire, `/Applications` en général. Lancée depuis un DMG ou le dossier Téléchargements (App Translocation), elle se dit indisponible et demande d'être déplacée.

## Ce que `drakbase init` écrit

Dans l'Info.plist du projet, et nulle part ailleurs :

| Clé | Contenu |
|---|---|
| `DrakBaseFeedURL` | `https://updates.drakbase.com/api/v1/update/<slug>` |
| `DrakBasePublicKeys` | la clé publique principale, puis la clé de secours, au format Tauri |

- Il lui faut un vrai fichier Info.plist. Un projet Xcode dont l'Info.plist est entièrement généré en reçoit un : crée un petit `Info.plist` avec ces clés, et renseigne-le dans le réglage « Info.plist File » (`INFOPLIST_FILE`) de la cible ; Xcode le fusionne avec les clés générées. Puis relance `drakbase init <slug>`, avec `--plist <fichier>` si le projet en a plusieurs.
- Il ne remplace jamais des clés déjà présentes. Ne les modifie pas à la main.
- Jusqu'à la 0.2, ces clés s'appelaient `AppsMarketFeedURL`, `AppsMarketPublicKeys`, `AppsMarketTrustedRequirements` et `AppsMarketInstallerService`. Le paquet les lit encore quand la nouvelle clé manque ; `drakbase init <slug>` les renomme, sans changer leur valeur. Un script qui les lit (par exemple avec PlistBuddy) passe aux nouveaux noms en même temps.
- `CFBundleIdentifier` doit rester celui de l'app déclarée sur le serveur : le serveur refuse une archive dont l'identifiant diffère.

## La dépendance

`Package.swift` :

```swift
dependencies: [
    .package(url: "git@github.com:kcampion/drakbase-apps-market.git", from: "0.5.1"),
],
targets: [
    .executableTarget(name: "MonApp", dependencies: [
        .product(name: "DrakBase", package: "drakbase-apps-market"),
    ]),
]
```

Projet Xcode : File › Add Package Dependencies…, adresse `git@github.com:kcampion/drakbase-apps-market.git`, à partir de la version 0.5.1, produit `DrakBase`. Xcode passe par la configuration SSH du Mac.

L'app n'importe qu'un module, `DrakBase` : il donne l'updater, son interface et la vérification des signatures. Le paquet est lié statiquement et n'a aucune dépendance.

Une app d'avant la 0.3 dépend du produit `AppsMarketKit` et importe `AppsMarketCore` et `AppsMarketUI` : remplace le produit par `DrakBase` (en montant `from:` à 0.5.1) et ces imports par `import DrakBase`.

## Le code

### 1. Le point d'entrée

La nouvelle instance lancée par une mise à jour doit attendre que l'ancienne soit fermée, **avant toute autre chose** (ouverture d'une base, `App.init`). Retire `@main` de la structure `App`, et ajoute dans un fichier qui ne s'appelle pas `main.swift` :

```swift
import DrakBase

@main
@MainActor
enum Main {
    static func main() {
        RelaunchGate.waitForPreviousInstance()
        MonApp.main()
    }
}
```

App AppKit sans SwiftUI : appelle `RelaunchGate.waitForPreviousInstance()` en tête de `main.swift`, avant `NSApplicationMain` ou `NSApplication.shared.run()`.

### 2. L'updater

Un seul par app, créé après le gate :

```swift
import DrakBase
import SwiftUI

struct MonApp: App {
    @State private var updater: Updater = {
        let updater = Updater()
        updater.presenter = UpdatePrompt(updater: updater)
        updater.start()
        return updater
    }()

    var body: some Scene {
        WindowGroup { ContentView() }
            .commands { UpdatesCommands(updater: updater) }
        Settings {
            Form { UpdateSettingsSection(updater: updater) }.formStyle(.grouped)
        }
    }
}
```

- `Updater(allowsAnonymousStatistics: false)` pour une app qui ne doit jamais envoyer d'identifiant d'installation. Sinon, les statistiques anonymes sont activées par défaut, et l'utilisateur peut les couper dans les réglages.
- `UpdatePrompt(updater:appName:)` est la fenêtre de mise à jour : notes en Markdown, progression, « Installer et relancer », « Plus tard », « Ignorer cette version ». Elle fonctionne aussi pour une app de barre des menus.
- `UpdatesCommands` ajoute « Rechercher les mises à jour… » après « À propos ». Dans le menu d'une app de barre des menus (`MenuBarExtra`), mets `CheckForUpdatesMenuItem(updater: updater)`.
- `UpdateSettingsSection` regroupe la recherche automatique, l'installation à la fermeture, les statistiques anonymes, le canal stable ou bêta, « Rechercher maintenant », la date du dernier contrôle et la version actuelle. Pour une mise en page à soi, les mêmes contrôles existent séparément, tous avec `init(updater:)` : `AutomaticUpdateChecksToggle`, `InstallUpdatesOnQuitToggle`, `AnonymousStatisticsToggle`, `UpdateChannelPicker`, `CheckForUpdatesButton`, `CurrentVersionText`, `LastUpdateCheckText`, `UpdateStatusText`.
- Pour passer l'updater à des vues plus profondes : `.environment(updater)` (il est `@Observable`).

### 3. Avant la relance (facultatif)

Pour enregistrer un travail en cours, ou refuser la relance à ce moment :

```swift
@MainActor
final class UpdateGuard: UpdaterDelegate {
    func updaterShouldRelaunch(_ updater: Updater) async -> Bool {
        await saveDrafts()
        return true
    }
}
```

`updater.delegate = updateGuard`, où `updateGuard` est une instance gardée par l'app (une propriété de l'`App` ou du délégué d'application) : l'updater n'en garde qu'une référence faible. Après `true`, l'app doit accepter de se fermer, sinon la nouvelle instance attend en vain.

## Version, build et signature

- La version publiée est `CFBundleShortVersionString` du bundle construit (`MARKETING_VERSION` dans Xcode), sous forme SemVer : `1.2.0`, `1.3.0-beta.1`. Elle monte à chaque version.
- Construis en Release, en universel si possible : une seule archive sert alors Apple Silicon et Intel. Avec SwiftPM : `swift build -c release --arch arm64 --arch x86_64`, puis un script qui assemble le `.app`. Avec Xcode : `xcodebuild -configuration Release` avec `ARCHS="arm64 x86_64" ONLY_ACTIVE_ARCH=NO`.
- Signe le bundle complet avec le certificat, en dernier :

  ```bash
  codesign --force --sign "<nom du certificat>" --timestamp=none "dist/MonApp.app"
  codesign --verify --strict "dist/MonApp.app"
  ```

  Dans Xcode, la signature manuelle avec ce certificat (`CODE_SIGN_IDENTITY`) fait la même chose. `codesign -dv dist/MonApp.app` doit montrer `Authority=<nom du certificat>`, et jamais `Signature=adhoc`.
- Contrôle : `drakbase doctor <slug> --bundle dist/MonApp.app`.

## Un script de publication

Propose un `scripts/release.sh` qui construit, vérifie, puis appelle l'outil, pour que publier tienne en une commande :

```bash
#!/bin/bash
# Publie une version de MonApp sur Drakbase (https://updates.drakbase.com).
#   MONAPP_SIGN_ID="<nom du certificat>" ./scripts/release.sh [--channel stable|beta] [--notes <fichier.md>] [--publish]
set -euo pipefail
cd "$(dirname "$0")/.."

die() { echo "release.sh : $*" >&2; exit 1; }

CHANNEL="stable"
ARGS=()
while [ $# -gt 0 ]; do
    case $1 in
    --channel) [ $# -ge 2 ] || die "--channel attend stable ou beta"; CHANNEL=$2; shift 2 ;;
    --notes) [ $# -ge 2 ] || die "--notes attend un fichier Markdown"; ARGS+=(--notes "$2"); shift 2 ;;
    --publish) ARGS+=(--publish); shift ;;
    *) die "option inconnue : $1" ;;
    esac
done

[ -n "${MONAPP_SIGN_ID:-}" ] || die "MONAPP_SIGN_ID manque : le nom du certificat de signature de code"
command -v drakbase >/dev/null || die "drakbase introuvable"

./scripts/build-app.sh   # construit dist/MonApp.app et le signe avec "$MONAPP_SIGN_ID"
APP="dist/MonApp.app"

# Sans ses clés publiques, cette version ne pourrait plus jamais vérifier une mise à jour.
KEYS="$(/usr/libexec/PlistBuddy -c "Print :DrakBasePublicKeys" "$APP/Contents/Info.plist" 2>/dev/null | grep -c '^ ' || true)"
[ "$KEYS" -ge 2 ] || die "l'Info.plist construit n'a pas les deux clés publiques (DrakBasePublicKeys)"

drakbase release "$APP" --app monapp --channel "$CHANNEL" ${ARGS[@]+"${ARGS[@]}"}
```

Le bash de macOS (3.2) lit mal `$APP…` suivi d'un caractère accentué ou d'une ellipse dans une locale UTF-8 : écris `${APP}…`. `doctor` signale ces cas dans les scripts du projet.

## Première installation

- La première version équipée s'installe à la main, dans `/Applications`. Sans notarisation, macOS bloque l'app téléchargée au premier lancement : la page publique explique « Ouvrir quand même ».
- Une app jusque-là signée ad hoc, puis signée avec le certificat, peut redemander une dernière fois l'accès à ses données (trousseau, dossiers) ; ensuite plus jamais.
