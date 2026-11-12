# Stacks Docker et apps cli

Une **stack** est un ensemble de conteneurs décrit par un fichier Compose, qui tourne sur une machine Linux (un Raspberry Pi, un petit serveur). DrakBase sert ses images comme un registre Docker en lecture seule, et l'agent `drakbase-stack`, sur la machine, l'installe et la met à jour seul : il ne télécharge que les couches d'image qui ont changé. Une **app cli** est un exécutable Linux publié de la même façon, comme l'agent lui-même.

Ce qui suit vaut pour l'outil `drakbase` sur le Mac du développeur. Il lui faut, en plus du reste, **Docker** avec **buildx** et **Docker Compose** (Docker Desktop ou OrbStack).

## Le dossier de la stack

À la racine du projet, un dossier `stack/` (un autre nom se donne avec `--stack <dossier>`) :

```
stack/
  compose.yaml     un compose ordinaire ; les services construits sur place ont un build:
  stack.json       la description de la stack, que drakbase init crée
  .env.example     facultatif : les réglages de la machine, avec leur valeur par défaut
  files/           facultatif : des fichiers que le compose monte en lecture seule
```

Rien d'autre du dossier n'est publié : ni `.env`, ni les sources. Le nom du fichier est exactement `compose.yaml`.

### `compose.yaml`

- Chaque image à publier vient d'un service avec `build:`. Elle prend le nom de son `image:`, sans registre ni étiquette (`image: home-manager:dev` donne l'image `home-manager`), ou à défaut le nom du service. Ce nom : minuscules et chiffres, séparés par un seul `.`, `_` ou `-`.
- Un service qui emploie la même image sans la construire (une sauvegarde, par exemple) porte la même `image:` que le service construit.
- Les autres images (Docker Hub…) sont permises, épinglées par empreinte : `redis:7@sha256:…`.
- À la publication, l'outil retire les `build:` et le `name:` de haut niveau, et remplace chaque image construite par `updates.drakbase.com/<slug>/<image>@sha256:…`. Les `${VAR}` et les chemins relatifs restent tels quels.
- **Règles**, contrôlées par `doctor` et avant toute construction par `release` :
  - le service surveillé (`service` de `stack.json`) existe et n'a pas de profil ; aucun autre service n'a de profil, sauf `pre_update`, qui a exactement `profiles: [drakbase-hooks]` ;
  - un fichier du dossier de la stack ne se monte que depuis `files/`, en lecture seule (`./files/app.toml:/etc/app.toml:ro`) ;
  - les données vivent dans des **volumes nommés**, ou dans un chemin absolu réglé par `.env` : le dossier de la stack change à chaque version ;
  - un montage dont la source est une variable s'écrit en syntaxe longue, avec son type :

    ```yaml
    volumes:
      - type: bind
        source: ${DATA_DIR:-/srv/mon-app}
        target: /data
    ```

    En syntaxe courte (`${DATA_DIR}:/data`), Docker Compose ne sait pas, avant de lire la variable, s'il s'agit d'un dossier ou d'un volume ;
  - `container_name` est déconseillé (avertissement) : le projet Compose porte le slug de l'app, et ses conteneurs en tirent leur nom.
- `build:` : `context`, `dockerfile`, `dockerfile_inline`, `args`, `target`, `labels`, `additional_contexts`, `network`, `no_cache`, `pull`, `extra_hosts`, `cache_from`, `cache_to`, `shm_size`. `tags` et `platforms` sont ignorés (les plateformes sont celles de `stack.json`) ; le reste (`secrets`, `ssh`…) est refusé.

### `stack.json`

```json
{
  "format": 1,
  "service": "home-manager",
  "health_timeout": 120,
  "rollback": "manual",
  "pre_update": "backup",
  "min_compose_version": "2.24",
  "platforms": ["linux/arm64", "linux/amd64"],
  "public_keys": ["<clé principale>", "<clé de secours>"]
}
```

| Champ | Contenu |
|---|---|
| `format` | `1` |
| `service` | le service dont l'état de santé décide du succès d'une mise à jour ; un `healthcheck` (dans le compose ou l'image) le rend fiable |
| `health_timeout` | facultatif : attente de l'état sain, de 10 à 900 secondes, 120 par défaut |
| `rollback` | facultatif : `safe` si l'agent peut revenir seul à la version précédente après un échec ; `manual` (par défaut) si une version peut changer le format des données |
| `pre_update` | facultatif : un service lancé avant chaque mise à jour (une sauvegarde), distinct de `service` |
| `min_compose_version` | facultatif : version minimale de Docker Compose sur la machine |
| `platforms` | les plateformes construites, `linux/arm64` (Raspberry Pi 64 bits) et `linux/amd64` par défaut ; aussi `linux/arm/v7`, `linux/riscv64`, `linux/386` |
| `app_control` | facultatif : chemin, dans le conteneur de `service`, où l'agent monte son dossier de contrôle (`/run/drakbase`). L'app y lit `status.json` (version disponible, notes, dernière demande) et dépose ses demandes dans `request/`, pour proposer les mises à jour dans son interface et ne les installer qu'après validation d'un administrateur. Contrat : section « Mises à jour validées dans l'app » de PROTOCOLE.md, dans le dépôt de DrakBase |
| `public_keys` | écrites par `drakbase init` : ne les modifie jamais à la main |

`app`, `version` et `images` sont refusés ici : l'outil les ajoute à la publication.

### `.env.example`

Les réglages que l'utilisateur de la machine peut changer (`PORT=8080`, `TZ=Europe/Paris`, `DATA_DIR=/srv/mon-app`), avec leur valeur par défaut. L'agent en tire le `.env` de la machine à l'installation, puis y ajoute les variables nouvelles, sans jamais toucher aux autres. Pas de secret dans ce fichier : il est publié.

## Brancher une stack

1. Le dossier `stack/` avec son `compose.yaml` (**toi**), en suivant les règles ci-dessus.
2. **L'utilisateur**, dans son terminal, à la racine du projet : `drakbase init <slug>` (ou `drakbase init <slug> --stack <dossier>`). L'outil crée les clés si besoin, crée `stack.json` (service surveillé deviné dans le compose, plateformes par défaut) ou y ajoute `public_keys`, puis ouvre la page d'ajout de l'app (type `stack`) dans l'administration, comme pour une autre app.
3. **Toi** : vérifie `service`, `rollback` et `platforms` dans `stack.json`, ajoute `pre_update` s'il y a une sauvegarde, écris `.env.example`.
4. `drakbase doctor <slug>` jusqu'à ce qu'il ne reste aucun `✗`. Pour une stack, il faut toujours donner le slug : `stack.json` ne le contient pas. Il lit le compose avec Docker Compose, le contrôle comme à la publication, et montre ce que la stack demandera à la machine (ports, réseau de l'hôte, dossiers de l'hôte…).

## Publier une version de la stack

```bash
drakbase release --app <slug> --stack [<dossier>] --version <version> --channel stable|beta [--notes notes.md] [--publish]
```

L'outil enchaîne :

1. la lecture du compose et ses règles, avant toute construction ;
2. la construction de chaque image pour les plateformes de `stack.json` (`docker buildx build`, au format OCI, sans attestations), puis la recompression de ses couches en zstd -19 : une couche qui ne change pas garde son empreinte, et n'est jamais renvoyée ;
3. le compose publié, contrôlé comme le fera l'agent ; il affiche ce que la stack demande à la machine ;
4. l'archive de la version (`dist/<slug>-<version>-<arch>.stack.tar.gz`, une par arch, à côté du dossier de la stack ; `-o <dossier>` pour un autre), signée avec la clé principale de l'app ;
5. le brouillon, les couches qui manquent au serveur, les manifestes, puis les archives ;
6. avec `--publish`, la publication ; puis la **commande d'installation** à transmettre à l'utilisateur de la machine, clé publique comprise.

- Un brouillon se publie ensuite sans rien reconstruire : la commande `drakbase publish … --publish dist/…stack.tar.gz` qu'affiche `release`, ou l'administration.
- Une commande interrompue se relance telle quelle : ce qui est déjà sur le serveur ne repart pas.
- Mêmes options `--force` et `--prune` que pour les autres apps ; `--publish` seulement avec l'accord de l'utilisateur.

## Installer la stack sur une machine

La commande qu'affiche `release`, à transmettre telle quelle (par un canal de confiance : la clé publique qu'elle contient est l'ancre de confiance de la machine) :

```bash
curl -fsSL https://updates.drakbase.com/install/drakbase-stack.sh | sudo sh
sudo drakbase-stack install <slug> --key <clé publique principale>
sudo drakbase-stack auto on
```

Le premier installe l'agent, le deuxième la stack (il montre ses permissions et les fait accepter), le troisième active la vérification quotidienne. Rien ne s'installe sans validation : dans l'interface de l'app si elle le propose (`app_control`), sinon par `sudo drakbase-stack update <slug>`. Docker Engine 24 et Docker Compose 2.29 au moins sont nécessaires sur la machine. Ensuite, `sudo drakbase-stack status`, `logs <slug>`, `update`, `rollback <slug>`, et `restart <slug>` après un réglage changé dans `/var/lib/drakbase-stack/<slug>/.env`.

## App cli

Un exécutable Linux, de préférence statique (musl), pour chaque arch.

1. **L'utilisateur** : `drakbase init <slug> --cli` à la racine du projet. L'outil écrit `drakbase.json` :

   ```json
   {
     "app": "<slug>",
     "public_keys": ["<clé principale>", "<clé de secours>"]
   }
   ```

2. **Toi** : l'exécutable lit `public_keys` à la compilation (par exemple `include_str!("../drakbase.json")`) et vérifie chaque mise à jour avec ces clés, comme le fait l'agent `drakbase-stack`.
3. Construire pour chaque arch (`cargo build --release --target aarch64-unknown-linux-musl`, puis `x86_64-unknown-linux-musl`), de préférence sous le nom `<slug>-<version>-<arch>`.
4. Signer, puis publier, depuis le dossier du projet :

   ```bash
   drakbase sign dist/<slug>-<version>-aarch64 --app <slug> --version <version>
   drakbase sign dist/<slug>-<version>-x86_64 --app <slug> --version <version>
   drakbase publish --app <slug> --version <version> --channel stable [--notes notes.md] dist/<slug>-<version>-aarch64 dist/<slug>-<version>-x86_64
   ```

   Un exécutable se reconnaît à son en-tête ELF, qui donne son arch ; sinon `linux/aarch64/bin=<fichier>`. La version déclare les clés de `drakbase.json` : depuis un autre dossier, `--drakbase-json <chemin>`. Sans lui, `--publish` est refusé.
