---
name: drakbase
description: Branche les mises à jour automatiques de Drakbase (Apps Market, https://updates.drakbase.com) dans une app macOS en Swift ou une app Tauri 2, puis publie ses versions avec l'outil drakbase. À utiliser dès que l'utilisateur veut intégrer Drakbase ou Apps Market à son app, ajouter AppsMarketKit, @appsmarket/tauri ou tauri-plugin-updater pour ce serveur, lancer drakbase init, doctor, release, publish ou exec, publier une version stable ou bêta, écrire des notes de version, retirer une version, ou comprendre un message de l'outil drakbase.
---

# Drakbase : intégrer une app et publier ses versions

Drakbase (Apps Market) est un serveur de mises à jour privé, pour des apps qui se mettent à jour seules sans App Store ni Sparkle :

- **le serveur** : `https://updates.drakbase.com`, avec l'administration sous `/admin`. Cette adresse est inscrite dans les apps : elle ne change pas ;
- **l'outil `drakbase`**, sur le Mac du développeur (macOS seulement) : il crée les clés, configure l'app, signe et publie ;
- **les clients** : le paquet Swift `AppsMarketKit` (apps macOS 14 et plus), ou le plugin officiel `tauri-plugin-updater` et l'aide `@appsmarket/tauri` (apps Tauri 2 pour macOS, Linux et Windows).

L'outil et les clients viennent du dépôt privé `git@github.com:kcampion/drakbase-apps-market.git`, par SSH, et suivent ses étiquettes `vX.Y.Z`. Cette skill, elle, est publiée à part, dans le dépôt public `DrakBase/drakbase-apps-market`, qui ne contient qu'elle : ne confonds pas les deux. Cette skill décrit la version **0.2.0** de l'outil : si `drakbase --version` donne autre chose, préfère ce que dit `drakbase <commande> --help`.

Chaque version est signée sur le Mac du développeur, avec une clé propre à l'app. L'app vérifie elle-même la signature, la version signée et son identifiant : un serveur compromis ne pourrait que retenir des mises à jour.

## Règles à ne jamais enfreindre

1. **Clés privées** : `~/.appsmarket/keys/<app>.key` ne se lit pas, ne s'affiche pas, ne se copie pas dans le projet, ne se commite pas et ne s'envoie nulle part. Seules les clés publiques vont dans l'app. Ne modifie rien à la main dans `~/.appsmarket`.
2. **Secrets** : ne demande jamais à l'utilisateur son jeton d'API (`amk_…`), le mot de passe de sa base KeePassXC ou celui d'une clé, et n'en écris aucun dans une commande, un fichier ou une variable. Les commandes qui en ont besoin, `drakbase login`, `drakbase init` quand il crée les clés, `backup` et `restore`, sont lancées par l'utilisateur dans son propre terminal : donne-lui la commande exacte, puis attends son compte rendu. Le mot de passe d'une clé, lui, vient du trousseau : `release`, `exec` et `sign` n'en demandent pas. S'il y manque, ils le demanderaient au terminal : arrête-toi, et fais lancer `drakbase backup <slug>` par l'utilisateur, qui l'y range.
3. **Publier atteint les utilisateurs.** N'ajoute `--publish` qu'après un accord explicite de l'utilisateur sur cette version et ce canal. Par défaut, envoie un brouillon (sans `--publish`).
4. **Les versions ne font que monter** : SemVer canonique (`1.2.0`, `1.3.0-beta.1`), sans métadonnées `+…`. Jamais une version déjà envoyée, jamais plus basse ; `--force` seulement à la demande expresse de l'utilisateur. On ne revient pas en arrière : on corrige avec une version plus haute.
5. **Identité de l'app** : ne change jamais son slug, son identifiant de bundle ni les clés publiques qu'elle embarque (`AppsMarketPublicKeys`, `plugins.updater.pubkey`). Changer de clé demande une version de transition (`references/publication.md`).
6. **Signature de code sur macOS** : une app Swift est signée avec le certificat de signature de code du développeur, toujours le même, jamais ad hoc (`codesign --sign -`). Une app signée ad hoc ne reçoit aucune mise à jour, et `drakbase release` la refuse. Pour Tauri, mets ce certificat dans `bundle.macOS.signingIdentity`.
7. Respecte les consignes propres au projet de l'app (CLAUDE.md, README, conventions) et ne touche qu'à ce que l'intégration demande.

## Faire le point

Ces commandes ne modifient rien : lance-les toi-même.

```bash
drakbase --version
drakbase doctor            # à la racine du projet ; code de sortie 1 s'il reste un ✗
security find-identity -v -p codesigning   # certificats de signature de code (apps macOS)
```

- `drakbase` introuvable : voir « Installer l'outil ».
- **Type de projet** : `src-tauri/tauri.conf.json` (ou `tauri.conf.json`) pour une app Tauri ; `Package.swift` ou un `.xcodeproj`, avec un `Info.plist`, pour une app Swift.
- **Déjà branchée ?** `AppsMarketFeedURL` dans l'Info.plist, ou `updates.drakbase.com` dans `tauri.conf.json`.
- `doctor` marque chaque contrôle : `✓` bon, `!` avertissement, `✗` problème. Le sens de chaque message et sa correction sont dans `references/depannage.md`.

Puis suis le parcours qui correspond : A (brancher une app), B (publier une version) ou C (dépanner).

## Installer l'outil

Une fois par Mac. Il faut : Rust (`cargo`), une clé SSH qui a accès au dépôt privé, chargée dans l'agent, et KeePassXC (`keepassxc-cli`, fourni avec `/Applications/KeePassXC.app`) avec une base où l'outil sauvegardera les clés. Pour une app Swift, ajoute Xcode ou ses outils en ligne de commande, et un certificat de signature de code dans le trousseau (un certificat auto-signé suffit).

```bash
git ls-remote --tags --refs ssh://git@github.com/kcampion/drakbase-apps-market.git 'v*'   # dernière étiquette
CARGO_NET_GIT_FETCH_WITH_CLI=true cargo install --git ssh://git@github.com/kcampion/drakbase-apps-market.git --tag v0.2.0 am-cli --locked
```

Prends la plus haute étiquette. Le binaire va dans `~/.cargo/bin`. Cette installation ne demande aucun secret : tu peux la lancer, avec l'accord de l'utilisateur.

## A. Brancher une app

1. **Le slug** : l'identifiant de l'app sur le serveur, de 1 à 64 caractères parmi `a-z`, `0-9` et `-`. Il figure dans les adresses et ne change plus : propose-en un, tiré du nom de l'app, et fais-le confirmer.
2. **Le jeton et la connexion**, par l'utilisateur, une fois par Mac et à l'expiration du jeton :
   - dans l'administration, « Mon compte » › « Jetons d'API » › « Nouveau jeton » (`https://updates.drakbase.com/admin/account`), avec les droits `releases:write` et `releases:publish`. Pour une app qui n'existe pas encore sur le serveur, le jeton ne doit pas être limité à une app ;
   - dans son terminal : `drakbase login --server https://updates.drakbase.com`, qui demande le jeton et le range dans le trousseau. Avec le jeton dans le presse-papiers : `pbpaste | drakbase login --server https://updates.drakbase.com --token-stdin`.

   `drakbase doctor` montre ensuite la connexion et les droits du jeton.
3. **`drakbase init <slug>`**, par l'utilisateur, dans son terminal, à la racine du projet (pour une app Tauri macOS, ajoute `--sign-id "<nom du certificat>"`). L'outil :
   - crée les clés de l'app si elle n'en a pas, les sauvegarde dans KeePassXC (il demande le mot de passe de la base, et le chemin de la base la première fois), relit la sauvegarde, puis range le mot de passe de la clé principale dans le trousseau ;
   - écrit la configuration des mises à jour (Info.plist, ou `tauri.conf.json`, permissions, `Cargo.toml` et `package.json`), sans toucher au code ni remplacer une clé déjà embarquée ;
   - ouvre l'administration sur une page pré-remplie qui crée l'app et active sa clé. **Cette page est réservée aux superadmins** : un éditeur envoie l'adresse affichée (`…/admin/add-app#…`) à un superadmin, qui vérifie que la page montre le même identifiant de clé que le terminal, puis valide. L'outil attend 15 minutes au plus ; relancer `drakbase init <slug>` reprend là où il en était ;
   - affiche « Reste à faire », puis les contrôles de `doctor` qui ne passent pas encore.

   Demande à l'utilisateur de te coller ce compte rendu, au moins « Reste à faire » et les lignes `✗` et `!`.
4. **Le code**, par toi : suis « Reste à faire » avec `references/swift.md` ou `references/tauri.md`.
5. **Le contrôle** : `drakbase doctor <slug>` jusqu'à ce qu'il ne reste aucun `✗`. Pour une app Swift construite : `drakbase doctor <slug> --bundle <chemin>/Nom.app`, qui contrôle aussi sa signature de code et son Info.plist.
6. **La première version équipée** s'installe à la main (DMG de la page publique, ou l'app construite, dans `/Applications`) : les mises à jour automatiques partent de celle-ci.

## B. Publier une version

Le détail est dans `references/publication.md`.

**App Swift**, à partir d'un bundle signé avec le certificat :
1. monte la version (`CFBundleShortVersionString`, ou `MARKETING_VERSION` dans Xcode, ou le fichier que lit le script de build) ;
2. construis en Release, en universel (`arm64` et `x86_64`) si possible, et signe avec le certificat ;
3. écris les notes (voir plus bas) ;
4. envoie un brouillon : `drakbase release <chemin>/Nom.app --app <slug> --channel stable --notes notes.md`. L'outil empaquette l'archive de mise à jour et le DMG, les signe avec la clé de l'app, puis les envoie ;
5. avec l'accord de l'utilisateur, relance la même commande avec `--publish` (le brouillon est repris, et les fichiers identiques ne sont pas renvoyés), ou publie depuis l'administration.

**App Tauri** :
1. monte `version` dans `tauri.conf.json` (et dans `package.json` et `Cargo.toml`, pour rester cohérent) ;
2. construis avec la clé de l'app : `drakbase exec <slug> -- npx tauri build` (sur un Mac, en universel : `-- npx tauri build --target universal-apple-darwin`) ;
3. depuis le dossier du projet : `drakbase publish --app <slug> --version <version> --channel stable --notes notes.md <artefacts>`. Les artefacts et leur emplacement sont dans `references/tauri.md` ;
4. `--publish`, avec l'accord de l'utilisateur, comme pour Swift.

**Notes de version** : en Markdown, dans la langue des utilisateurs de l'app, écrites pour eux (ce qui change pour eux, pas les détails internes). Elles s'affichent dans la fenêtre de mise à jour et sur la page publique ; le HTML brut est retiré.

**Bêta** : une version de pré-publication (`1.3.0-beta.1`) avec `--channel beta`. Seules les apps réglées sur bêta la reçoivent. Pour la passer en stable, on change son canal dans l'administration, sans rien reconstruire ni renvoyer.

**Retirer une version** : dans l'administration, page de la version. Elle n'est plus proposée, et ses téléchargements répondent 410. On corrige ensuite avec une version plus haute.

## C. Dépanner

1. `drakbase doctor [<slug>]`, à la racine du projet, éventuellement avec `--bundle Nom.app`.
2. Pour chaque `✗`, puis chaque `!` utile, cherche le message dans `references/depannage.md`.
3. Une erreur de `release` ou `publish` dit presque toujours quoi faire. Une publication interrompue reprend avec la même commande.

## Pour aller plus loin

- `references/swift.md` : AppsMarketKit (dépendance, Info.plist, point d'entrée, interface, build et signature, script de publication).
- `references/tauri.md` : configuration, plugins, aide `@appsmarket/tauri`, instance unique, build, artefacts, Linux et Windows.
- `references/publication.md` : options de `release` et `publish`, brouillons, canaux, retrait, clés, nouveau Mac.
- `references/depannage.md` : messages de `doctor` et de l'outil, et leur correction.
- Les guides complets arrivent avec les clients, à la version installée :
  - Tauri : `node_modules/@appsmarket/tauri/clients/tauri/README.md` ;
  - Swift : `clients/swift/AppsMarketKit/README.md` dans la copie du dépôt que garde SwiftPM (`.build/checkouts/drakbase-apps-market/` pour un paquet, `SourcePackages/checkouts/drakbase-apps-market/` dans les DerivedData d'un projet Xcode).
