# Publier, gérer les versions et les clés

## Avant d'envoyer

- `drakbase doctor <slug>` : aucun `✗`, et la ligne de version du projet plus haute que la dernière publiée.
- La version est canonique et plus haute que la dernière du canal : `1.2.0`, `1.2.1`, `1.3.0-beta.1`. `1.2` est publiée comme `1.2.0`, et les métadonnées `+…` sont refusées.
- Les notes sont prêtes, en Markdown (voir plus bas).
- Le jeton a les droits nécessaires : `releases:write` pour créer et remplir un brouillon, `releases:publish` en plus pour publier, et `listing:write` pour la fiche publique.
- À la première publication : la description et les captures d'écran sont prêtes (`fiche.md`).

## `drakbase release` (app Swift, ou tout bundle macOS signé)

```bash
drakbase release <chemin>/Nom.app --app <slug> --channel stable|beta [--notes notes.md] [--publish]
```

- À partir du bundle signé avec le certificat, il fabrique l'archive de mise à jour (`.app.tar.gz`) et le DMG de première installation, les signe avec la clé principale de l'app (son mot de passe vient du trousseau), crée le brouillon et y envoie les fichiers.
- La version est celle du bundle (`CFBundleShortVersionString`), et l'arch celle de son exécutable : un bundle universel donne un seul artefact pour Apple Silicon et Intel.
- Le serveur est consulté avant d'empaqueter : un jeton refusé ou une version trop basse arrêtent tout avant le moindre travail.
- Il refuse un bundle non signé ou signé ad hoc.
- Options utiles : `--no-dmg` (pas de DMG), `-o <dossier>` (où écrire l'archive, sa signature et le DMG ; par défaut, à côté du bundle), `--prune` (retire du brouillon les fichiers envoyés avant qui ne font plus partie de l'envoi), `--force` (accepte une version inférieure ou égale à la dernière du canal : seulement à la demande expresse de l'utilisateur).

## `drakbase publish` (app Tauri, ou artefacts déjà signés)

```bash
drakbase publish --app <slug> --version <version> --channel stable|beta [--notes notes.md] [--publish] <artefacts…>
```

- Chaque artefact est un fichier, dont l'emplacement se déduit du nom (`.app.tar.gz`, `.dmg`, `.AppImage`, `.deb`, `.rpm`, `-setup.exe`, `.msi`), ou `<os>/<arch>/<type>=<fichier>` : `darwin/universal/app=…`, `darwin/aarch64/dmg=…`, `linux/x86_64/appimage=…`, `windows/x86_64/nsis=…`.
- La signature `<fichier>.sig` est lue à côté de chaque fichier, sauf pour un DMG. L'outil la vérifie avec la clé de l'app, et contrôle que la version signée est bien celle annoncée.
- Il déclare les clés que la version embarque, lues dans `tauri.conf.json` (ou `DrakBasePublicKeys` dans l'Info.plist pour une app Swift) : lance-le depuis le dossier du projet, ou donne `--tauri-config`. Sans configuration trouvée, il refuse `--publish`.
- Il envoie aussi les archives d'une stack (`.stack.tar.gz`, dont il lit les clés dans `stack.json`) et les exécutables d'une app cli (reconnus à leur en-tête ELF, ou `linux/<arch>/bin=<fichier>` ; leurs clés viennent de `drakbase.json`, ou de `--drakbase-json <chemin>`). Voir `stack.md`.
- Mêmes options `--publish`, `--force` et `--prune` que `release`.

## `drakbase release --stack` (stack Docker)

```bash
drakbase release --app <slug> --stack [<dossier>] --version <version> --channel stable|beta [--notes notes.md] [--publish]
```

- Il construit les images de la stack, les envoie, et publie une archive signée par arch. La version se donne avec `--version` ; le dossier vaut `stack` par défaut. Le détail est dans `stack.md`.
- `-o <dossier>` : où écrire les archives et leurs signatures (par défaut, `dist/` à côté du dossier de la stack).
- Un brouillon se publie ensuite avec la commande `drakbase publish … --publish` qu'il affiche, sans rien reconstruire.

## Brouillons et reprise

- Sans `--publish`, la version reste un **brouillon** : rien n'est proposé aux apps. On la regarde dans l'administration (page de la version : fichiers, liste de contrôle), puis on publie depuis l'administration ou en relançant la commande avec `--publish`.
- Une commande interrompue se relance telle quelle : le brouillon est repris, et les fichiers identiques déjà envoyés ne repartent pas.
- Un fichier ne se remplace que tant que la version est un brouillon. Une version publiée ne se modifie plus : on en publie une plus haute.

## Notes de version

- En Markdown, dans la langue des utilisateurs de l'app, pour eux : ce qui change dans leur usage, les corrections qu'ils remarqueront. Pas de détails internes (refactorisation, dépendances), sauf s'ils ont un effet visible.
- Titres courts et listes à puces ; le HTML brut et les images sont retirés à l'affichage.
- Elles s'affichent dans la fenêtre de mise à jour de l'app et dans l'historique de la page publique.
- Propose un brouillon de notes tiré des commits depuis la dernière version (`git log <dernière étiquette>..HEAD`), et fais-le relire.

## Canaux

- **stable** : toutes les apps. **beta** : les apps réglées sur bêta, qui voient aussi les versions stables, donc toujours la plus haute.
- Une bêta porte une version de pré-publication (`1.3.0-beta.1`, `1.3.0-beta.2`), plus basse que la stable `1.3.0` qui suivra.
- **Passer une bêta en stable** : dans l'administration, page de la version, changer son canal. Rien n'est reconstruit ni renvoyé : le canal ne fait pas partie de la signature. Une version déjà publiée en bêta ne se renvoie donc pas en stable.

## Retirer une version

Dans l'administration, page de la version : « Retirer ». Elle n'est plus proposée, ses téléchargements répondent 410, et une app qui la téléchargeait affiche que l'éditeur l'a retirée. Elle peut être remise. Les apps qui l'ont déjà installée la gardent : la correction passe par une version plus haute, jamais par un retour en arrière.

## Clés

- Chaque app a sa paire de clés : la clé principale signe, la clé de secours (embarquée par les apps Swift) sert si la principale est perdue. Les deux sont sauvegardées dans la base KeePassXC de l'utilisateur, groupe « Drakbase - Apps Market », entrées « <slug> - clé principale » et « <slug> - clé de secours ». La clé secrète de secours n'est que dans KeePassXC.
- **Nouveau Mac** : `drakbase restore <slug>` (par l'utilisateur, dans son terminal : mot de passe de la base) réinstalle la clé principale et son mot de passe dans le trousseau.
- **Clés créées avant la sauvegarde automatique** : `drakbase backup <slug>` les range dans KeePassXC, vérifie la sauvegarde et range le mot de passe de la clé principale dans le trousseau. Relancé, il vérifie seulement.
- **Changer de clé** demande une version de transition : signée avec l'ancienne clé, elle embarque déjà la nouvelle ; la nouvelle devient ensuite active sur le serveur (onglet « Clés » de l'app). Les apps plus anciennes passent d'abord par cette version. C'est une opération rare, à préparer avec un superadmin : ne la lance pas de toi-même.
- Si la clé principale et sa sauvegarde sont perdues pour une app Tauri, plus aucune mise à jour ne passe : chaque utilisateur doit réinstaller l'app depuis la page publique.

## Page publique

La page publique (`https://updates.drakbase.com/`, éventuellement protégée par un mot de passe) montre les apps listées, leur dernière version stable, les installeurs par système (le DMG pour macOS), la description, les captures d'écran et l'historique des notes. La description et les captures se règlent avec `drakbase listing` (`fiche.md`) ou dans l'administration ; l'icône et la visibilité, dans l'administration seulement, onglet « Informations ».
