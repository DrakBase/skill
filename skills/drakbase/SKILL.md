---
name: drakbase
description: Branche les mises à jour automatiques de DrakBase (https://updates.drakbase.com) dans une app macOS en Swift, une app Tauri 2, une stack Docker (compose, pour une machine Linux) ou un exécutable Linux (app cli), publie ses versions avec l'outil drakbase, et prépare sa fiche publique (description et captures d'écran). À utiliser dès que l'utilisateur veut intégrer DrakBase à son app ou à sa stack, ajouter le paquet Swift DrakBase, @drakbase/tauri ou tauri-plugin-updater pour ce serveur, préparer un dossier stack/ (compose.yaml, stack.json), lancer drakbase init, doctor, release (y compris release --stack), publish, exec ou listing, publier une version stable ou bêta, écrire des notes de version, rédiger la description ou prendre les captures d'écran de l'app, retirer une version, installer une stack avec drakbase-stack, ou comprendre un message de l'outil drakbase.
---

# DrakBase : intégrer une app et publier ses versions

DrakBase est un serveur de mises à jour privé, pour des apps qui se mettent à jour seules sans App Store ni Sparkle :

- **le serveur** : `https://updates.drakbase.com`, avec l'administration sous `/admin` et une page publique par app. Cette adresse est inscrite dans les apps : elle ne change pas ;
- **l'outil `drakbase`**, sur le Mac du développeur (macOS seulement) : il crée les clés, configure l'app, signe, publie et tient la fiche publique ;
- **les clients** : le paquet Swift `DrakBase` (`import DrakBase`, apps macOS 14 et plus), ou le plugin officiel `tauri-plugin-updater` et l'aide `@drakbase/tauri` (apps Tauri 2 pour macOS, Linux et Windows) ;
- **les stacks Docker** : un compose et ses images, que le serveur sert comme un registre en lecture seule ; sur la machine Linux, l'agent `drakbase-stack` les installe et les met à jour seul. L'agent est lui-même une **app cli**, un exécutable Linux publié par DrakBase (`references/stack.md`).

L'outil et les clients viennent du dépôt privé `git@github.com:kcampion/drakbase-apps-market.git`, par SSH, et suivent ses étiquettes `vX.Y.Z`. Cette skill, elle, est publiée à part, dans le dépôt public `DrakBase/skill` : ne confonds pas les deux. Elle décrit la version **0.4.0** de l'outil : si `drakbase --version` donne autre chose, préfère ce que dit `drakbase <commande> --help`.

Jusqu'à la 0.2, le paquet Swift s'appelait `AppsMarketKit`, l'aide Tauri `@appsmarket/tauri`, et les clés de l'Info.plist `AppsMarket…`. Les anciens noms marchent encore, mais `doctor` les signale : passe aux nouveaux quand tu touches à une app qui les utilise.

Chaque version est signée sur le Mac du développeur, avec une clé propre à l'app. L'app vérifie elle-même la signature, la version signée et son identifiant : un serveur compromis ne pourrait que retenir des mises à jour.

## Règles à ne jamais enfreindre

1. **Clés privées** : `~/.appsmarket/keys/<app>.key` ne se lit pas, ne s'affiche pas, ne se copie pas dans le projet, ne se commite pas et ne s'envoie nulle part. Seules les clés publiques vont dans l'app. Ne modifie rien à la main dans `~/.appsmarket`.
2. **Secrets** : ne demande jamais à l'utilisateur son jeton d'API (`amk_…`), le mot de passe de sa base KeePassXC ou celui d'une clé, et n'en écris aucun dans une commande, un fichier ou une variable. Les commandes qui en ont besoin, `drakbase login`, `drakbase init` quand il crée les clés, `backup` et `restore`, sont lancées par l'utilisateur dans son propre terminal : donne-lui la commande exacte, puis attends son compte rendu. Le mot de passe d'une clé, lui, vient du trousseau : `release`, `exec` et `sign` n'en demandent pas. S'il y manque, ils le demanderaient au terminal : arrête-toi, et fais lancer `drakbase backup <slug>` par l'utilisateur, qui l'y range.
3. **Publier atteint les utilisateurs.** N'ajoute `--publish` qu'après un accord explicite de l'utilisateur sur cette version et ce canal. Par défaut, envoie un brouillon (sans `--publish`). De même, la fiche publique (`drakbase listing`) ne change qu'une fois la description et les captures montrées à l'utilisateur et approuvées.
4. **Les versions ne font que monter** : SemVer canonique (`1.2.0`, `1.3.0-beta.1`), sans métadonnées `+…`. Jamais une version déjà envoyée, jamais plus basse ; `--force` seulement à la demande expresse de l'utilisateur. On ne revient pas en arrière : on corrige avec une version plus haute.
5. **Identité de l'app** : ne change jamais son slug, son identifiant de bundle ni les clés publiques qu'elle embarque (`DrakBasePublicKeys`, `plugins.updater.pubkey`). Changer de clé demande une version de transition (`references/publication.md`).
6. **Signature de code sur macOS** : une app Swift est signée avec le certificat de signature de code du développeur, toujours le même, jamais ad hoc (`codesign --sign -`). Une app signée ad hoc ne reçoit aucune mise à jour, et `drakbase release` la refuse. Pour Tauri, mets ce certificat dans `bundle.macOS.signingIdentity`.
7. **Captures d'écran** : jamais de données personnelles réelles (emails, noms, comptes, jetons, fichiers privés). Regarde chaque image avant de la proposer.
8. Respecte les consignes propres au projet de l'app (CLAUDE.md, README, conventions) et ne touche qu'à ce que l'intégration demande.

## Faire le point

Ces commandes ne modifient rien : lance-les toi-même.

```bash
drakbase --version
drakbase doctor            # à la racine du projet ; code de sortie 1 s'il reste un ✗
security find-identity -v -p codesigning   # certificats de signature de code (apps macOS)
```

- `drakbase` introuvable : voir « Installer l'outil ».
- **Type de projet** : `src-tauri/tauri.conf.json` (ou `tauri.conf.json`) pour une app Tauri ; `Package.swift` ou un `.xcodeproj`, avec un `Info.plist`, pour une app Swift ; `stack/compose.yaml` pour une stack Docker ; `drakbase.json` pour une app cli.
- **Déjà branchée ?** `DrakBaseFeedURL` (ou l'ancienne `AppsMarketFeedURL`) dans l'Info.plist, `updates.drakbase.com` dans `tauri.conf.json`, ou `public_keys` dans `stack/stack.json` ou `drakbase.json`.
- **Déjà publiée ?** `drakbase listing <slug>` montre la fiche publique : sans description ni captures, c'est la première publication (parcours D).
- `doctor` marque chaque contrôle : `✓` bon, `!` avertissement, `✗` problème. Le sens de chaque message et sa correction sont dans `references/depannage.md`. Pour une stack, donne toujours le slug : `drakbase doctor <slug>`.

Puis suis le parcours qui correspond : A (brancher une app), B (publier une version), C (dépanner), D (la fiche publique) ou E (une stack Docker ou une app cli).

## Installer l'outil

Une fois par Mac. Il faut : Rust (`cargo`), une clé SSH qui a accès au dépôt privé, chargée dans l'agent, et KeePassXC (`keepassxc-cli`, fourni avec `/Applications/KeePassXC.app`) avec une base où l'outil sauvegardera les clés. Pour une app Swift, ajoute Xcode ou ses outils en ligne de commande, et un certificat de signature de code dans le trousseau (un certificat auto-signé suffit).

```bash
git ls-remote --tags --refs ssh://git@github.com/kcampion/drakbase-apps-market.git 'v*'   # dernière étiquette
CARGO_NET_GIT_FETCH_WITH_CLI=true cargo install --git ssh://git@github.com/kcampion/drakbase-apps-market.git --tag v0.4.0 am-cli --locked
```

Prends la plus haute étiquette. Le binaire va dans `~/.cargo/bin`. Cette installation ne demande aucun secret : tu peux la lancer, avec l'accord de l'utilisateur.

## A. Brancher une app

1. **Le slug** : l'identifiant de l'app sur le serveur, de 1 à 64 caractères parmi `a-z`, `0-9` et `-`. Il figure dans les adresses et ne change plus : propose-en un, tiré du nom de l'app, et fais-le confirmer.
2. **Le jeton et la connexion**, par l'utilisateur, une fois par Mac et à l'expiration du jeton :
   - dans l'administration, « Mon compte » › « Jetons d'API » › « Nouveau jeton » (`https://updates.drakbase.com/admin/account`), avec les droits `releases:write`, `releases:publish` et `listing:write` (ce dernier pour la fiche publique). Pour une app qui n'existe pas encore sur le serveur, le jeton ne doit pas être limité à une app ;
   - dans son terminal : `drakbase login --server https://updates.drakbase.com`, qui demande le jeton et le range dans le trousseau. Avec le jeton dans le presse-papiers : `pbpaste | drakbase login --server https://updates.drakbase.com --token-stdin`.

   `drakbase doctor` montre ensuite la connexion et les droits du jeton.
3. **`drakbase init <slug>`**, par l'utilisateur, dans son terminal, à la racine du projet (pour une app Tauri macOS, ajoute `--sign-id "<nom du certificat>"` ; pour une stack Docker ou un exécutable Linux, voir le parcours E, `--stack` ou `--cli`). L'outil :
   - crée les clés de l'app si elle n'en a pas, les sauvegarde dans KeePassXC (il demande le mot de passe de la base, et le chemin de la base la première fois), relit la sauvegarde, puis range le mot de passe de la clé principale dans le trousseau ;
   - écrit la configuration des mises à jour (Info.plist, ou `tauri.conf.json`, permissions, `Cargo.toml` et `package.json`), sans toucher au code ni remplacer une clé déjà embarquée ; il renomme les clés `AppsMarket…` d'une app configurée avant la 0.3 ;
   - ouvre l'administration sur une page pré-remplie qui crée l'app et active sa clé, et y ajoute comme éditeur le compte du jeton s'il n'est pas superadmin (sans quoi l'outil ne verrait pas l'app). **Cette page est réservée aux superadmins** : un éditeur envoie l'adresse affichée (`…/admin/add-app#…`) à un superadmin, qui vérifie que la page montre le même identifiant de clé que le terminal, puis valide. L'outil attend 15 minutes au plus ; relancer `drakbase init <slug>` reprend là où il en était ;
   - affiche « Reste à faire », puis les contrôles de `doctor` qui ne passent pas encore.

   Demande à l'utilisateur de te coller ce compte rendu, au moins « Reste à faire » et les lignes `✗` et `!`.
4. **Le code**, par toi : suis « Reste à faire » avec `references/swift.md` ou `references/tauri.md`.
5. **Le contrôle** : `drakbase doctor <slug>` jusqu'à ce qu'il ne reste aucun `✗`. Pour une app Swift construite : `drakbase doctor <slug> --bundle <chemin>/Nom.app`, qui contrôle aussi sa signature de code et son Info.plist.
6. **La première version équipée** s'installe à la main (DMG de la page publique, ou l'app construite, dans `/Applications`) : les mises à jour automatiques partent de celle-ci.

## B. Publier une version

Le détail est dans `references/publication.md`. **À la première publication d'une app, fais aussi le parcours D** : sans description ni captures, sa page publique est vide.

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

**Versions suivantes** : si l'interface a visiblement changé, propose de refaire les captures (parcours D).

## C. Dépanner

1. `drakbase doctor [<slug>]`, à la racine du projet, éventuellement avec `--bundle Nom.app`.
2. Pour chaque `✗`, puis chaque `!` utile, cherche le message dans `references/depannage.md`.
3. Une erreur de `release`, `publish` ou `listing` dit presque toujours quoi faire. Une publication interrompue reprend avec la même commande.

## D. La fiche publique : description et captures d'écran

La page publique de l'app montre son icône, sa description, ses captures d'écran, ses installeurs et ses notes de version. Le détail est dans `references/fiche.md`.

1. **Regarde ce qui existe** : `drakbase listing <slug>`.
2. **Propose une description** : texte brut, 4000 caractères au plus, quelques courts paragraphes séparés par une ligne vide, dans la langue des utilisateurs. Tire-la du README et de l'app : ce qu'elle fait, pour qui, ses points forts. Écris-la dans un fichier du projet, par exemple `fiche/description.txt`.
3. **Prends les captures** : 3 à 6 PNG des écrans qui montrent le mieux l'app, avec des données de démonstration, rangés par exemple dans `fiche/`, chacune avec une légende courte. La méthode est dans `references/fiche.md`. Regarde chaque image toi-même, puis montre-les à l'utilisateur.
4. **Avec son accord**, essaie puis envoie :

   ```bash
   drakbase listing <slug> --description fiche/description.txt \
     --screenshot "fiche/1-principale.png=La fenêtre principale" \
     --screenshot "fiche/2-reglages.png=Les réglages" --dry-run
   ```

   La même commande sans `--dry-run` remplace la description, et fait des captures du serveur exactement cette liste, dans cet ordre. Il faut le droit `listing:write` du jeton.
5. **L'icône** et la visibilité de l'app sur la page publique se règlent dans l'administration, onglet « Informations » de l'app : propose un PNG carré de l'icône (512 Kio au plus), que l'utilisateur y dépose.

## E. Une stack Docker ou une app cli

Le détail est dans `references/stack.md`. Il faut Docker, avec buildx et Docker Compose, sur le Mac.

**Stack Docker** (des conteneurs pour une machine Linux) :
1. **Le dossier `stack/`**, par toi : `compose.yaml` (les services construits sur place avec `build:`), `.env.example` pour les réglages de la machine, `files/` pour les fichiers montés en lecture seule. Respecte les règles de `references/stack.md` : données dans des volumes nommés, montages du projet seulement depuis `files/`, en lecture seule, source variable en syntaxe longue.
2. **`drakbase init <slug>`**, par l'utilisateur, comme au parcours A : clés, `stack/stack.json` créé ou complété (`public_keys`), page d'ajout de l'app (type `stack`). Vérifie ensuite `service`, `rollback`, `pre_update` et `platforms` dans `stack.json`.
3. **`drakbase doctor <slug>`** jusqu'à ce qu'il ne reste aucun `✗` : il lit le compose avec Docker Compose et le contrôle comme à la publication.
4. **Publier** : `drakbase release --app <slug> --stack --version <version> --channel stable --notes notes.md` construit les images, les recompresse, envoie un brouillon, et affiche la commande d'installation. `--publish` seulement avec l'accord de l'utilisateur.
5. **Installer sur la machine** : l'utilisateur transmet la commande qu'affiche `release` (`curl … | sudo sh`, puis `sudo drakbase-stack install <slug> --key …`). Ne la modifie pas : la clé qu'elle contient est l'ancre de confiance de la machine.

**App cli** (un exécutable Linux) : `drakbase init <slug> --cli` écrit `drakbase.json` ; l'exécutable lit ses `public_keys` à la compilation ; chaque version se signe avec `drakbase sign`, puis s'envoie avec `drakbase publish` depuis le dossier du projet.

## Pour aller plus loin

- `references/swift.md` : le paquet DrakBase (dépendance, Info.plist, point d'entrée, interface, build et signature, script de publication).
- `references/tauri.md` : configuration, plugins, aide `@drakbase/tauri`, instance unique, build, artefacts, Linux et Windows.
- `references/publication.md` : options de `release` et `publish`, brouillons, canaux, retrait, clés, nouveau Mac.
- `references/fiche.md` : description, captures d'écran (méthode, contrôle), `drakbase listing`.
- `references/stack.md` : stacks Docker (dossier `stack/`, `compose.yaml`, `stack.json`, `release --stack`, installation sur la machine) et apps cli (`drakbase.json`, exécutables `bin`).
- `references/depannage.md` : messages de `doctor` et de l'outil, et leur correction.
- Les guides complets arrivent avec les clients, à la version installée :
  - Tauri : `node_modules/@drakbase/tauri/clients/tauri/README.md` ;
  - Swift : `clients/swift/DrakBase/README.md` dans la copie du dépôt que garde SwiftPM (`.build/checkouts/drakbase-apps-market/` pour un paquet, `SourcePackages/checkouts/drakbase-apps-market/` dans les DerivedData d'un projet Xcode).
