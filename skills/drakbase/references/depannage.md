# Dépannage

`drakbase doctor [<slug>] [--bundle Nom.app]` contrôle tout sans rien modifier : `✓` bon, `!` avertissement, `✗` problème. Il sort en erreur s'il reste un `✗`. Sous chaque message, il donne le plus souvent la correction. Ce qui suit les regroupe, avec qui doit agir : **toi** (Claude), ou **l'utilisateur** quand il faut un secret, un clic dans l'administration ou un superadmin.

## Projet

| Message | Correction |
|---|---|
| aucun projet d'app dans le dossier courant | Lancer à la racine du projet, ou `--tauri-config <chemin>`, `--stack <dossier>` (stack Docker) ou `--cli` (exécutable Linux). **Toi.** |
| impossible de savoir quelle app contrôler | `drakbase doctor <slug>`, ou brancher le projet avec `drakbase init <slug>`. |
| aucun Info.plist trouvé dans le projet | Ajouter un Info.plist (réglage « Info.plist File » pour un projet Xcode), ou `--plist`. **Toi**, puis `drakbase init` relancé par l'utilisateur. |
| identifiant de bundle absent, ou différent de celui du serveur | Le `CFBundleIdentifier` (ou `identifier` de Tauri) doit être celui de l'app sur le serveur. Ne change pas celui d'une app déjà distribuée : vérifie plutôt le slug. |
| dépendance DrakBase absente, introuvable dans le code : … | `references/swift.md`. **Toi.** |
| dépendance AppsMarketKit : le paquet s'appelle désormais DrakBase | Produit `DrakBase` (`from: "0.5.2"`) et `import DrakBase` à la place d'`AppsMarketCore` et `AppsMarketUI` (`swift.md`). **Toi.** |
| `AppsMarketFeedURL` (ou une autre `AppsMarket…`) : ancien nom de `DrakBase…` | `drakbase init <slug>` les renomme, valeurs inchangées. **L'utilisateur**, ou toi en éditant l'Info.plist si le slug et les valeurs restent identiques. |
| `… n'est pas enregistré dans src-tauri/src`, l'interface n'utilise pas encore `@drakbase/tauri` | `references/tauri.md`, « Ce qui reste à écrire ». **Toi.** |
| `@appsmarket/tauri` dans `package.json`, ou l'interface l'importe encore | L'aide s'appelle désormais `@drakbase/tauri` : renommer la dépendance et les imports, puis `npm install`. **Toi.** |
| `@drakbase/tauri plus ancien que l'outil : la plage ^0.4.0 de package.json s'arrête avant la version 0.5.2` | En 0.x, `^0.4.0` ne reçoit jamais la 0.5 : l'app n'a pas ce que l'aide a gagné depuis, comme l'envoi de la version du système. **L'utilisateur** relance `drakbase init <slug>`, qui monte la plage (« @drakbase/tauri monté de ^0.4.0 à ^0.5.2 ») ; puis **toi** : `npm install`, et les plugins ajoutés à enregistrer côté Rust (`references/tauri.md`). |
| `@drakbase/tauri : plage « … » illisible dans package.json` | `drakbase init` n'y touche pas, et ne modifie pas `package.json`. **Toi** : vérifier que la plage admet la version de l'outil ; sinon écrire à sa place celle que donne le message (`^0.5.2` pour l'outil 0.5.2), puis `npm install`. |
| `@drakbase/tauri plus récent que l'outil : la plage >0.6.0 de package.json commence après la version 0.5.2` | L'app demande une aide que cet outil ne connaît pas : mettre l'outil `drakbase` à jour (`SKILL.md`, « Installer l'outil »), puis relancer `drakbase doctor`. `init` ne ramène jamais une plage en arrière. |
| `tauri-plugin-updater` absent ou trop ancien, paquet absent de `package.json` | `drakbase init <slug>` l'ajoute (puis `npm install`), ou monte la version à 2.14 au moins, côté crate et côté npm ensemble. |
| permissions absentes de `capabilities/…` | Ajouter `updater:default`, `process:allow-restart`, `store:default` à la capacité de la fenêtre concernée. **Toi.** |
| `requireSignedVersion` ou `createUpdaterArtifacts` pas activé, installation Windows à revoir | `drakbase init <slug>` les règle ; ou à la main, comme dans `references/tauri.md`. |
| DrakBasePublicKeys n'a pas de clé de secours | Ajouter la clé publique de secours (`~/.appsmarket/keys/<slug>.backup.key.pub`) après la principale, sans toucher à la principale. |
| clé embarquée différente de la clé principale | Les mises à jour seraient refusées. Ne remplace rien : c'est une rotation de clé (`publication.md`). Préviens l'utilisateur. |
| la version du projet n'est pas plus haute que la dernière version stable (ou bêta) publiée | `doctor` compare comme `publish` : une pré-publication (`1.3.0-beta.1`) à la dernière bêta, une autre version à la dernière stable. Monter la version avant de publier. **Toi**, avec l'accord de l'utilisateur sur le numéro. |
| la version du projet est déjà publiée en bêta (ou en stable) | En bêta : pour la passer en stable, changer son canal dans l'administration, sans rien renvoyer. Sinon, monter la version. |
| la version du projet est une pré-publication plus basse que la dernière version stable | Elle se publierait, mais personne ne la recevrait : les apps réglées sur bêta voient aussi les versions stables. Prendre une pré-publication de la version suivante. |
| `… pour Windows (src-tauri/tauri.windows.conf.json)`, ou pour macOS ou Linux | Le fichier de plateforme nommé change ce réglage pour ses builds : `doctor` lit, OS par OS, ce que lira `tauri build`. Corriger ce fichier, ou en retirer le réglage ; `drakbase init` ne modifie que `tauri.conf.json`. **Toi**, sauf pour une clé : c'est une rotation (`publication.md`). |
| `init` : `… change « … » pour les builds …`, `… donne déjà bundle.macOS.signingIdentity` | Même cause, dite par `init` : un fichier de plateforme l'emporte sur ce qu'`init` écrit. `--sign-id` n'écrit pas dans `tauri.conf.json` un certificat que `tauri.macos.conf.json` remplacerait : le changer dans ce fichier. **Toi.** |
| un script du projet lit `$VAR…` de travers | Le bash 3.2 de macOS colle le caractère accentué au nom : écrire `${VAR}…`. **Toi.** |

## Connexion et jeton

| Message | Correction |
|---|---|
| aucune connexion enregistrée | **L'utilisateur** : créer un jeton (« Mon compte » › « Jetons d'API »), puis `drakbase login --server https://updates.drakbase.com` dans son terminal. `drakbase init <slug> --server https://updates.drakbase.com` écrit la configuration sans connexion ; l'app reste alors à déclarer, en relançant `init` une fois connecté. |
| l'outil est connecté au mauvais serveur, ou avec le jeton d'un autre compte | **L'utilisateur** : `drakbase logout` retire du trousseau l'adresse et le jeton, puis `drakbase login`. Le jeton reste valable sur le serveur : le révoquer dans l'administration s'il ne doit plus servir. |
| le jeton ne peut pas envoyer de version (`releases:write` absente) | **L'utilisateur** : nouveau jeton avec cette portée, puis `drakbase login`. |
| le jeton ne peut pas publier (`releases:publish` absente) | Les versions resteront en brouillon : publier depuis l'administration, ou nouveau jeton. |
| le jeton ne peut pas modifier la fiche (`listing:write` absente) | **L'utilisateur** : nouveau jeton avec ce droit, puis `drakbase login` ; ou la description et les captures déposées dans l'administration. |
| le jeton « … » ne sert qu'à l'app …, pas à … / ne donne pas accès à l'app … | Le jeton est limité à une autre app : il ne verrait même pas celle-ci. Le message est le même pour `publish`, `release`, `listing`, `init` et `doctor`. **L'utilisateur** : jeton pour cette app, ou pour toutes ses apps, puis `drakbase login`. |
| le jeton est refusé : il a expiré ou a été révoqué | **L'utilisateur** : nouveau jeton, puis `drakbase login`. |
| le jeton n'a pas les droits nécessaires | Sa portée, son app, ou le rôle de son propriétaire : un éditeur ne publie que pour les apps dont il est éditeur. Un superadmin l'ajoute dans l'onglet « Éditeurs » de l'app. |
| `http://` n'est permis que pour un serveur sur ce Mac | Utiliser `https://updates.drakbase.com`. |
| trop de requêtes | Attendre quelques minutes. |
| réponse du serveur illisible : il y manque le champ « … » | Le serveur est plus ancien que l'outil : il ne rend pas ce que cette version attend. À voir avec l'administrateur du serveur, qui le met à jour. Rien n'a été envoyé de travers : l'outil s'arrête plutôt que de deviner. |

## App et clé sur le serveur

| Message | Correction |
|---|---|
| l'app … n'existe pas sur le serveur, la clé principale n'est pas déclarée | **L'utilisateur** relance `drakbase init <slug>` ; la page `/admin/add-app#…` qu'il ouvre est réservée aux superadmins. Un éditeur en transmet l'adresse à un superadmin. |
| `init` reste sur « En attente de l'administration… » alors que la page dit l'app prête | Le compte du jeton n'est pas éditeur de l'app : il ne la voit pas. **L'utilisateur** l'ajoute dans l'onglet Éditeurs de l'app. Un outil 0.4 ou plus le fait proposer par la page, dans le même clic. |
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
| mot de passe de la clé principale absent du trousseau macOS : chaque signature le demandera | **L'utilisateur** : `drakbase backup <slug>`, qui le vérifie puis l'y range. |
| le mot de passe que garde le trousseau n'ouvre pas la clé | **L'utilisateur** : `drakbase restore <slug>`. |
| clés jamais sauvegardées hors de ce Mac, ou sauvegarde d'une autre clé | **L'utilisateur** : `drakbase backup <slug>`. Sans sauvegarde, une clé perdue bloque toutes les mises à jour de l'app. |
| KeePassXC n'a pas ouvert la base | Mot de passe faux, ou fichier clé (`--keepassxc-key-file`) ou YubiKey (`--keepassxc-yubikey`) manquant. |
| `keepassxc-cli` introuvable | Installer KeePassXC dans `/Applications`, ou donner son chemin dans `APPSMARKET_KEEPASSXC_CLI`. |
| `backup`, `restore` ou `init` ne voient pas des entrées existantes | Elles doivent être dans le groupe « Drakbase - Apps Market », à la racine de la base, avec des titres au tiret simple : « <slug> - clé principale ». Le renommage se fait à la main dans KeePassXC. |

## Signature et envoi

| Message | Correction |
|---|---|
| `… est signé ad hoc`, `… n'est pas signé` | Signer le bundle avec le certificat : `codesign --force --sign "<nom du certificat>" --timestamp=none Nom.app`. |
| `bundle.macOS.signingIdentity` vaut « - » ou absent, certificat absent du trousseau | `drakbase init <slug> --sign-id "<nom du certificat>"`, ou corriger `tauri.conf.json`. Si le message nomme `tauri.macos.conf.json`, c'est là que le certificat se corrige. Le nom exact vient de `security find-identity -v -p codesigning`. |
| la version … est déjà publiée en stable, la version existe déjà sur le serveur, hors brouillon (`conflict`) | Elle est publiée ou retirée : choisir une version plus haute. |
| la version … est déjà publiée en bêta | Pour la passer en stable : changer son canal dans l'administration, sans rien renvoyer. |
| `… est signé pour la version X, pas Y`, la signature ne contient pas de version | Le fichier a été construit pour une autre version : reconstruire après avoir monté la version, ou `drakbase sign <fichier> --app <slug> --version <version>` pour un fichier signé à part. |
| signature au format historique | Signer de nouveau (`drakbase exec <slug> -- npx tauri build`, ou `drakbase sign`). |
| signé par une clé que le serveur ne connaît pas pour l'app | Mauvaise clé (variables `TAURI_SIGNING_*` d'un autre projet ?) : reconstruire par `drakbase exec <slug> -- …`. |
| les artefacts d'une version sont tous signés par la même clé | Un des fichiers vient d'un autre build : tout reconstruire avec la même clé. |
| `artifact_invalid` | Le contenu ne correspond pas : identifiant de bundle, version (`CFBundleShortVersionString`), architectures du binaire ou type de fichier. |
| le fichier dépasse la taille maximale du serveur | 1 Gio par défaut. À voir avec l'administrateur du serveur. |
| `conflict` | La version n'est plus un brouillon : ses fichiers ne changent plus. Version plus haute. |

## Fiche publique (`drakbase listing`)

| Message | Correction |
|---|---|
| PNG illisible, dimensions hors limites, fichier trop gros | PNG seulement, de 320 à 8192 pixels de côté, 8 Mio au plus (`sips -g pixelWidth -g pixelHeight`). Refaire la capture, ou la réduire : `sips -Z 2560 capture.png`. |
| plus de 10 captures | Garder les plus parlantes. |
| légende ou description trop longue | 200 caractères par légende, 4000 pour la description. |
| capture vide, ou seulement le fond d'écran | Le terminal n'a pas l'autorisation « Enregistrement de l'écran » : **l'utilisateur** l'accorde dans Réglages Système › Confidentialité et sécurité, puis relance le terminal. |
| la page publique ne montre pas l'app | Elle n'est pas visible (« Visible sur la page publique », onglet « Informations »), ou n'a pas encore de version stable publiée. |

## Stacks Docker et apps cli

Les règles sont dans `references/stack.md`. Chaque règle enfreinte se termine par son code, entre parenthèses.

| Message | Correction |
|---|---|
| `docker … introuvable`, `docker buildx indisponible`, `docker compose indisponible`, Docker ne répond pas | **L'utilisateur** installe ou démarre Docker (Docker Desktop, OrbStack), avec buildx et Docker Compose. |
| `docker buildx build a échoué` | Lire la sortie de buildx au-dessus. Si le constructeur ne produit pas d'images OCI multi-plateformes : stockage d'images containerd de Docker Desktop, ou `docker buildx create --use --driver docker-container`. |
| `Docker Compose ne lit pas …/compose.yaml : …` | Le message de Docker Compose suit : corriger le compose. Une variable obligatoire (`${VAR:?…}`) manque dans `.env.example`. |
| `compose.yaml introuvable`, `renomme docker-compose.yml en compose.yaml` | Le dossier de la stack contient `compose.yaml`, sous ce nom exactement. **Toi.** |
| `stack.json : …` (`field_tool_only`, `field_unknown`, `public_keys_count`…) | `app`, `version` et `images` sont remplis par l'outil, pas à la main ; les champs permis sont dans `stack.md`. **Toi**, sans jamais toucher à `public_keys`. |
| impossible de savoir quelle app contrôler (stack) | `drakbase doctor <slug>` : `stack.json` ne dit pas le slug. |
| aucun service du compose ne se construit sur place | Une stack publie au moins une image : un service avec `build:`. |
| `« … » ne peut pas nommer une image` | Donner au service construit une `image:` en minuscules (`image: mon-app:dev`). **Toi.** |
| deux images différentes sous le même nom | Deux services construisent différemment avec la même `image:` : la même section `build`, ou des noms distincts. |
| `build.… n'est pas pris en charge` | Retirer ce champ (`secrets`, `ssh`…) du `build:`, ou construire l'image autrement. |
| le montage `${…}:/…` prend une variable pour source | L'écrire en syntaxe longue avec `type: bind` (`stack.md`). **Toi.** |
| `(bind_outside_files)`, `(bind_not_read_only)` | Un fichier du dossier de la stack se monte depuis `files/`, en lecture seule (`:ro`) ; les données vont dans un volume nommé. **Toi.** |
| `(bind_path_invalid)` | Un chemin monté (la source d'un montage, le `file` d'un `config` ou d'un `secret`) contient un composant `.` ou `..`, refusé partout : `files/../../etc` commencerait dans la stack et en sortirait. Écrire le chemin sans eux. **Toi.** |
| `(service_missing)`, `(pre_update_missing)`, `(profile_unexpected)`, `(pre_update_profile)` | `service` et `pre_update` de `stack.json` désignent des services du compose ; seul `pre_update` a un profil, exactement `drakbase-hooks`. **Toi.** |
| `(image_not_pinned)` | Une image d'un autre registre s'épingle par empreinte : `redis:7@sha256:…` (`docker buildx imagetools inspect redis:7` la donne). **Toi.** |
| `compose : « services.app.provider » n'est pas pris en charge dans une stack (compose_field_unsupported)` | Le champ nommé n'est pas dans la liste des champs admis (`stack.md`) : le retirer du `compose.yaml`. C'est le cas d'`annotations`, et de l'`ipam.config` d'un réseau. Pour `include` ou `extends`, écrire le service en entier dans `compose.yaml`. **Toi.** |
| `compose : « services.app.pid » vaut « … », qui n'est pas pris en charge dans une stack (compose_value_unsupported)` | Le champ est admis, pas cette valeur (`stack.md`) : un pilote de réseau autre que `bridge`, de volume autre que `local`, de journal autre que `json-file`, `local` ou `none`, d'ipam autre que `default` ; un `network_mode`, un `pid` ou un `ipc` hors des valeurs admises ; les `driver_opts` d'un volume (`type` autre que `none`, dont `nfs` et `cifs`, `o`, ou un `device` relatif) ; la `propagation` `shared` ou `rshared` d'un montage, ou un `bind.recursive` inconnu. Un partage NFS ou SMB se monte sur la machine, et la stack le reçoit comme un dossier réglé par `.env`. **Toi.** |
| `compose : « … » vaut « … », avec un caractère de contrôle, de sens d'écriture ou invisible (compose_text_hidden)` | Un nom (service, volume, réseau, `config`, `secret`) ou une valeur qui devient une permission (port, chemin, capacité…) contient un caractère qui ne s'affiche pas tel quel, écrit en clair dans le message (`\u{1b}`, `\u{200b}`…). Le retirer du `compose.yaml` ou du `.env.example`. **Toi.** |
| `« /chemin » : un env_file ou un label_file se lit dans files/, par un chemin relatif (env_file_outside_files)` | Ranger le fichier dans `files/` et l'écrire `./files/<nom>`, sans variable ni `..`. Pas de secret dedans : `files/` est publié. **Toi.** |
| `compose.yaml de l'archive : ce n'est pas le JSON qu'écrit l'outil (…) (compose_not_json)` | L'archive n'a pas été faite par `drakbase release --stack`, ou son compose a été retouché à la main : la refaire avec l'outil. La parenthèse dit ce qui ne va pas : « un objet JSON est attendu dès le premier caractère, sans commentaire », « caractère U+000D : seul le saut de ligne sépare les lignes », ou une erreur de lecture du JSON, comme « clé « … » en double ». |
| container_name est déconseillé | Retirer `container_name` : un nom fixe peut entrer en conflit avec un conteneur existant de la machine. |
| buildx n'a pas produit la plateforme … | Le constructeur ne sait pas construire pour cette plateforme (émulation absente) : la retirer de `platforms`, ou installer l'émulation. |
| l'app … est une app Swift sur le serveur, pas une stack Docker | Mauvais slug, ou app créée avec un autre type : vérifie l'identifiant. |
| le serveur annonce une adresse publique que l'outil ne peut pas employer | L'adresse publique du serveur (`AM_PUBLIC_URL`) entre dans les noms d'images et dans la commande d'installation : elle doit être une adresse `https://hôte` (ou `http://hôte`), sans chemin ni identifiant. À voir avec l'administrateur du serveur. |
| `requête refusée : seule une app de type stack reçoit des images` | L'app du serveur n'est pas de type `stack`. |
| `n'est pas un exécutable Linux (ELF)`, `l'exécutable est fait pour …` | Un artefact `bin` est un exécutable Linux de l'arch annoncée : vérifier la cible de compilation. |
| `… est un fichier ELF qui n'est pas un exécutable (un fichier objet .o, par exemple)` | Donner à `publish` l'exécutable lié, statique ou non, pas un fichier intermédiaire de la compilation. **Toi.** |
| `… est une app cli, et le dossier courant n'a pas de drakbase.json` | Lancer `publish` depuis le dossier du projet, ou donner `--drakbase-json <chemin>`. |

Sur la machine, côté agent `drakbase-stack` (à transmettre à l'utilisateur de la machine) :

| Message | Correction |
|---|---|
| `lecture de …/state.json : accès refusé` | L'agent se lance en root : le dossier de chaque stack n'est ouvert qu'à lui. Relancer la commande avec `sudo`. |
| une permission nouvelle arrête la mise à jour : `à accepter sur la machine : sudo drakbase-stack update <slug> --accept` | La version demande à la machine quelque chose de plus que la précédente (un port, un dossier, un volume à nom fixe…). L'utilisateur de la machine lit la liste, puis lance cette commande s'il l'accepte. |
| `<slug> : avec les réglages de .env, la version 1.1.0 demande de nouvelles permissions :`, la liste, puis `Pour les accepter : sudo drakbase-stack rollback <slug> --accept` (pour `restart` : `les réglages de .env demandent de nouvelles permissions`) | Toute relance d'une version gardée refait le contrôle du compose avec le `.env` du moment : un réglage changé depuis, un port par exemple, demande autre chose à la machine. Rien n'a été relancé (code de sortie 3). L'utilisateur de la machine lit la liste, puis lance la commande donnée s'il l'accepte. |
| `Le retour seul ne les accepte pas : rien n'a été relancé.`, puis `Pour revenir à la version 1.0.0, en acceptant ces permissions : sudo drakbase-stack rollback <slug> --accept` | Une mise à jour a échoué avec `rollback: safe`, et le retour à la version précédente demande une permission que personne n'a acceptée : la nouvelle version reste en échec, ses conteneurs en l'état. L'utilisateur de la machine lance la commande donnée. |
| `<slug> : /var/lib/drakbase-stack/<slug>/.env a changé pendant la mise à jour : vos changements restent, seuls les réglages ajoutés par la version en sont retirés : A` | Le `.env` a été modifié à la main pendant que la version démarrait, puis elle a échoué : l'agent garde ces changements, et ne retire que ce qu'elle avait ajouté. Rien à faire, sauf vérifier le `.env`. |
| `<slug> : .env remis comme avant ; réglages retirés : A, B` | Une mise à jour s'est arrêtée après avoir ajouté des réglages à `.env` : il est revenu à son contenu d'avant. Après un échec avec `rollback: manual`, l'agent ajoute que ces réglages reviendront avec leur valeur par défaut à la reprise ; pour en choisir une autre, les écrire d'abord dans le `.env` qu'il nomme. |
| `compose de la version … refusé : … (code)`, `compose refusé avec ce .env : …` | Le compose enfreint une règle de `stack.md` ; le code est celui des lignes ci-dessus. À l'installation d'une version : corriger le compose, puis publier une version plus haute. À la relance d'une version déjà installée (`rollback`, `restart`) : elle emploie un champ refusé depuis, ou le `.env` actuel la fait sortir des règles ; elle ne se relance plus telle quelle, il faut une version corrigée, que `update` installera. **Toi.** |
| `La version 1.0.0 a été refusée : son compose n'est plus admis.`, ou `rollback` qui finit par `Cette version de drakbase-stack ne la relancera pas. Ce qui reste possible :` | La version à relancer a été installée par un agent plus ancien, et emploie un champ refusé depuis (un volume `nfs`, par exemple) : `rollback` échouerait. L'utilisateur de la machine choisit dans la liste : une version plus récente (`update`), la reprise de la version en échec (`update --retry`), un réglage du `.env` corrigé si c'est lui qui est refusé, ou `uninstall`. **Toi** : publier une version corrigée. |
| `… ; les conteneurs de la version 1.1.0 restent en l'état` | Après une version restée en échec (ici la 1.1.0), la suivante s'est arrêtée avant de démarrer, au téléchargement des images par exemple : ses conteneurs n'ont pas été touchés. Le début du message dit pourquoi ; ensuite `sudo drakbase-stack status`, puis `update` ou `rollback <slug>`. |
| `<slug> ne reçoit pas de demande de son interface : la version qui tourne n'a pas de dossier de contrôle (app_control dans stack.json)` | La stack n'a pas `app_control` : ses mises à jour passent par `sudo drakbase-stack update <slug>`. Pour les valider dans l'app, ajouter `app_control` à `stack.json` (`stack.md`) et publier une version. |
| dans l'interface de l'app (`app_control`) : `la version 1.1.0 n'est pas devenue saine, et le retour à la 1.0.0 demande de nouvelles permissions ; ses conteneurs sont restés en l'état. À taper sur la machine : sudo drakbase-stack rollback <slug> --accept` | Le même arrêt, dit par `status.json` : une interface ne peut pas accepter une permission. L'app montre ce message tel quel à son administrateur, qui passe par la machine. |
| `fichier ELF qui n'est pas un exécutable` (`self-update`) | L'agent refuse de se remplacer par ce fichier : la version publiée de l'agent n'est pas un exécutable lié. À voir avec son éditeur. |

## Côté app

- **Swift, l'updater se dit « indisponible »** : configuration incomplète dans l'Info.plist du bundle construit (`drakbase doctor <slug> --bundle Nom.app`), app sandboxée, app lancée depuis un DMG ou Téléchargements, dossier non modifiable, ou app installée signée ad hoc.
- **Swift, « Relance impossible »** : la nouvelle instance n'est pas apparue ; vérifier le point d'entrée avec `RelaunchGate.waitForPreviousInstance()` et que l'app accepte de se fermer.
- **Tauri, rien ne se passe à la relance** : `relaunch()` absent (utiliser `installUpdate` ou `useUpdater`), ou `prevent_exit()` qui ne laisse pas passer `RESTART_EXIT_CODE`.
- **Tauri, « réponse du serveur interrompue, trop longue à venir ou illisible »** : pendant une installation, le téléchargement a été coupé, ou a dépassé son délai (`downloadTimeout`, 4 heures par défaut). La version reste proposée : réessayer.
- **Tauri, « réponse du serveur illisible : canal … inconnu »** (ou « champ channel absent ») : l'adresse de `plugins.updater.endpoints` n'est pas celle d'un serveur DrakBase, ou le serveur est plus récent que l'aide.
- **« à jour » alors qu'une version existe** : elle est en brouillon, en bêta (et l'app suit stable), retirée, exige un macOS plus récent, n'a pas d'artefact pour ce système, cette arch ou ce type d'installation, ou est signée par une clé que la version installée ne connaît pas.
- Pour voir ce que le serveur répond, sans rien modifier : `curl -i "https://updates.drakbase.com/api/v1/update/<slug>/darwin/aarch64/<version installée>?bundle_type=app"` (204 : rien de plus récent ; 200 : la version proposée ; 404 : app inconnue).
