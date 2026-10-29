# Dépannage

`drakbase doctor [<slug>] [--bundle Nom.app]` contrôle tout sans rien modifier : `✓` bon, `!` avertissement, `✗` problème. Il sort en erreur s'il reste un `✗`. Sous chaque message, il donne le plus souvent la correction. Ce qui suit les regroupe, avec qui doit agir : **toi** (Claude), ou **l'utilisateur** quand il faut un secret, un clic dans l'administration ou un superadmin.

## Projet

| Message | Correction |
|---|---|
| aucun projet d'app dans le dossier courant | Lancer à la racine du projet, ou `--tauri-config <chemin>`. **Toi.** |
| impossible de savoir quelle app contrôler | `drakbase doctor <slug>`, ou brancher le projet avec `drakbase init <slug>`. |
| aucun Info.plist trouvé dans le projet | Ajouter un Info.plist (réglage « Info.plist File » pour un projet Xcode), ou `--plist`. **Toi**, puis `drakbase init` relancé par l'utilisateur. |
| identifiant de bundle absent, ou différent de celui du serveur | Le `CFBundleIdentifier` (ou `identifier` de Tauri) doit être celui de l'app sur le serveur. Ne change pas celui d'une app déjà distribuée : vérifie plutôt le slug. |
| dépendance AppsMarketKit absente, introuvable dans le code : … | `references/swift.md`. **Toi.** |
| `… n'est pas enregistré dans src-tauri/src`, l'interface n'utilise pas encore `@appsmarket/tauri` | `references/tauri.md`, « Ce qui reste à écrire ». **Toi.** |
| `tauri-plugin-updater` absent ou trop ancien, paquet absent de `package.json` | `drakbase init <slug>` l'ajoute (puis `npm install`), ou monte la version à 2.14 au moins, côté crate et côté npm ensemble. |
| permissions absentes de `capabilities/…` | Ajouter `updater:default`, `process:allow-restart`, `store:default` à la capacité de la fenêtre concernée. **Toi.** |
| `requireSignedVersion` ou `createUpdaterArtifacts` pas activé, installation Windows à revoir | `drakbase init <slug>` les règle ; ou à la main, comme dans `references/tauri.md`. |
| AppsMarketPublicKeys n'a pas de clé de secours | Ajouter la clé publique de secours (`~/.appsmarket/keys/<slug>.backup.key.pub`) après la principale, sans toucher à la principale. |
| clé embarquée différente de la clé principale | Les mises à jour seraient refusées. Ne remplace rien : c'est une rotation de clé (`publication.md`). Préviens l'utilisateur. |
| la version du projet n'est pas plus haute que la dernière publiée | Monter la version avant de publier. **Toi**, avec l'accord de l'utilisateur sur le numéro. |
| un script du projet lit `$VAR…` de travers | Le bash 3.2 de macOS colle le caractère accentué au nom : écrire `${VAR}…`. **Toi.** |

## Connexion et jeton

| Message | Correction |
|---|---|
| aucune connexion enregistrée | **L'utilisateur** : créer un jeton (« Mon compte » › « Jetons d'API »), puis `drakbase login --server https://updates.drakbase.com` dans son terminal. |
| le jeton ne peut pas envoyer de version (`releases:write` absente) | **L'utilisateur** : nouveau jeton avec cette portée, puis `drakbase login`. |
| le jeton ne peut pas publier (`releases:publish` absente) | Les versions resteront en brouillon : publier depuis l'administration, ou nouveau jeton. |
| le jeton ne sert qu'à l'app … / ne donne pas accès à l'app … | **L'utilisateur** : jeton pour cette app, ou pour toutes ses apps, puis `drakbase login`. |
| le jeton est refusé : il a expiré ou a été révoqué | **L'utilisateur** : nouveau jeton, puis `drakbase login`. |
| le jeton n'a pas les droits nécessaires | Sa portée, son app, ou le rôle de son propriétaire : un éditeur ne publie que pour les apps dont il est éditeur. Un superadmin l'ajoute dans l'onglet « Éditeurs » de l'app. |
| `http://` n'est permis que pour un serveur sur ce Mac | Utiliser `https://updates.drakbase.com`. |
| trop de requêtes | Attendre quelques minutes. |

## App et clé sur le serveur

| Message | Correction |
|---|---|
| l'app … n'existe pas sur le serveur, la clé principale n'est pas déclarée | **L'utilisateur** relance `drakbase init <slug>` ; la page `/admin/add-app#…` qu'il ouvre est réservée aux superadmins. Un éditeur en transmet l'adresse à un superadmin. |
| l'app a déjà une autre clé active sur le serveur | Le projet embarque une autre clé que celle du serveur : mauvais slug, ou clés d'un autre Mac (`drakbase restore <slug>`). Sinon, rotation de clé : à voir avec un superadmin. |
| l'app est de type swift sur le serveur, et le projet est une app tauri (ou l'inverse) | Mauvais slug : vérifie l'identifiant de l'app. |
| clé déclarée comme suivante, pas encore active | Normal pendant une rotation : la passer active dans l'onglet « Clés » une fois la version de transition publiée. |
| clé retirée sur le serveur | Le serveur refusera ce qu'elle signe. À voir avec un superadmin. |
| le serveur n'a pas d'identifiant de bundle pour cette app | Le renseigner dans l'administration, onglet « Informations » de l'app. |

## Clés et sauvegardes

| Message | Correction |
|---|---|
| pas de clé publique principale, clé secrète principale introuvable | Clés jamais créées sur ce Mac : `drakbase init <slug>` les crée ; clés existantes ailleurs : `drakbase restore <slug>`. **L'utilisateur**, dans son terminal. |
| `… est lisible par d'autres comptes` | `chmod 600 <fichier>`. **Toi.** |
| mot de passe de la clé principale absent du trousseau | **L'utilisateur** : `drakbase backup <slug>`, qui le vérifie puis l'y range. |
| le mot de passe que garde le trousseau n'ouvre pas la clé | **L'utilisateur** : `drakbase restore <slug>`. |
| clés jamais sauvegardées hors de ce Mac, ou sauvegarde d'une autre clé | **L'utilisateur** : `drakbase backup <slug>`. Sans sauvegarde, une clé perdue bloque toutes les mises à jour de l'app. |
| KeePassXC n'a pas ouvert la base | Mot de passe faux, ou fichier clé (`--keepassxc-key-file`) ou YubiKey (`--keepassxc-yubikey`) manquant. |
| `keepassxc-cli` introuvable | Installer KeePassXC dans `/Applications`, ou donner son chemin dans `APPSMARKET_KEEPASSXC_CLI`. |
| `backup`, `restore` ou `init` ne voient pas des entrées existantes | Elles doivent être dans le groupe « Drakbase - Apps Market », à la racine de la base, avec des titres au tiret simple : « <slug> - clé principale ». Le renommage se fait à la main dans KeePassXC. |

## Signature et envoi

| Message | Correction |
|---|---|
| `… est signé ad hoc`, `… n'est pas signé` | Signer le bundle avec le certificat : `codesign --force --sign "<nom du certificat>" --timestamp=none Nom.app`. |
| `bundle.macOS.signingIdentity` vaut « - » ou absent, certificat absent du trousseau | `drakbase init <slug> --sign-id "<nom du certificat>"`, ou corriger `tauri.conf.json`. Le nom exact vient de `security find-identity -v -p codesigning`. |
| la version … est déjà publiée en stable / déjà retirée, la version existe déjà hors brouillon | Choisir une version plus haute. |
| la version … est déjà publiée en bêta | Pour la passer en stable : changer son canal dans l'administration, sans rien renvoyer. |
| `… est signé pour la version X, pas Y`, la signature ne contient pas de version | Le fichier a été construit pour une autre version : reconstruire après avoir monté la version, ou `drakbase sign <fichier> --app <slug> --version <version>` pour un fichier signé à part. |
| signature au format historique | Signer de nouveau (`drakbase exec <slug> -- npx tauri build`, ou `drakbase sign`). |
| signé par une clé que le serveur ne connaît pas pour l'app | Mauvaise clé (variables `TAURI_SIGNING_*` d'un autre projet ?) : reconstruire par `drakbase exec <slug> -- …`. |
| les artefacts d'une version sont tous signés par la même clé | Un des fichiers vient d'un autre build : tout reconstruire avec la même clé. |
| `artifact_invalid` | Le contenu ne correspond pas : identifiant de bundle, version (`CFBundleShortVersionString`), architectures du binaire ou type de fichier. |
| le fichier dépasse la taille maximale du serveur | 1 Gio par défaut. À voir avec l'administrateur du serveur. |
| `conflict` | La version n'est plus un brouillon : ses fichiers ne changent plus. Version plus haute. |

## Côté app

- **Swift, l'updater se dit « indisponible »** : configuration incomplète dans l'Info.plist du bundle construit (`drakbase doctor <slug> --bundle Nom.app`), app sandboxée, app lancée depuis un DMG ou Téléchargements, dossier non modifiable, ou app installée signée ad hoc.
- **Swift, « Relance impossible »** : la nouvelle instance n'est pas apparue ; vérifier le point d'entrée avec `RelaunchGate.waitForPreviousInstance()` et que l'app accepte de se fermer.
- **Tauri, rien ne se passe à la relance** : `relaunch()` absent (utiliser `installUpdate` ou `useUpdater`), ou `prevent_exit()` qui ne laisse pas passer `RESTART_EXIT_CODE`.
- **« à jour » alors qu'une version existe** : elle est en brouillon, en bêta (et l'app suit stable), retirée, exige un macOS plus récent, n'a pas d'artefact pour ce système, cette arch ou ce type d'installation, ou est signée par une clé que la version installée ne connaît pas.
- Pour voir ce que le serveur répond, sans rien modifier : `curl -i "https://updates.drakbase.com/api/v1/update/<slug>/darwin/aarch64/<version installée>?bundle_type=app"` (204 : rien de plus récent ; 200 : la version proposée ; 404 : app inconnue).
