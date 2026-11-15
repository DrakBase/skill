# App Tauri 2 : tauri-plugin-updater et @drakbase/tauri

L'app utilise le plugin officiel `tauri-plugin-updater`, qui interroge le serveur, télécharge l'archive, en vérifie la signature avec la clé publique embarquée, l'installe, puis l'app se relance. L'aide `@drakbase/tauri` ajoute le canal (stable ou bêta) et l'identifiant d'installation anonyme, et range ces réglages.

## Versions

Épingle les crates et les paquets npm sur des versions exactes, les mêmes des deux côtés : `tauri build` refuse un paquet npm d'une autre version mineure que sa crate. `drakbase init` ajoute ces versions-ci, celles de l'app d'exemple :

| Crate / paquet npm | Version | Pourquoi |
|---|---|---|
| `tauri`, `@tauri-apps/api` | 2.12 (2.12.5 / 2.12.4) | |
| `@tauri-apps/cli` | 2.12 ou plus | écrit `version:` dans la signature |
| `tauri-plugin-updater`, `@tauri-apps/plugin-updater` | `=2.14.0` / `2.14.0` | `requireSignedVersion` n'existe qu'à partir de 2.14 |
| `tauri-plugin-process`, `@tauri-apps/plugin-process` | `=2.4.0` / `2.4.0` | `relaunch()` après l'installation |
| `tauri-plugin-store`, `@tauri-apps/plugin-store` | `=2.5.0` / `2.5.0` | réglages de l'aide |
| `tauri-plugin-os`, `@tauri-apps/plugin-os` | `=2.4.0` / `2.4.0` | version du système envoyée au serveur |
| `tauri-plugin-single-instance` | `=2.5.4`, facultatif | compatible avec la relance |

## Ce que `drakbase init` écrit

- **`src-tauri/tauri.conf.json`** :

  ```json
  {
    "bundle": {
      "createUpdaterArtifacts": true,
      "macOS": { "signingIdentity": "<nom du certificat>" },
      "windows": { "nsis": { "installMode": "currentUser" } }
    },
    "plugins": {
      "updater": {
        "pubkey": "<clé publique de l'app>",
        "endpoints": [
          "https://updates.drakbase.com/api/v1/update/<slug>/{{target}}/{{arch}}/{{current_version}}?bundle_type={{bundle_type}}"
        ],
        "requireSignedVersion": true,
        "windows": { "installMode": "passive" }
      }
    }
  }
  ```

  `signingIdentity` seulement avec `--sign-id`. Ne retire aucun de ces réglages : `{{bundle_type}}` fait servir le même type d'installation (une installation `deb` ne reçoit pas d'AppImage), `requireSignedVersion` refuse une archive dont la version signée diffère, `createUpdaterArtifacts: true` (jamais `"v1Compatible"`) produit les archives signées, `installMode: currentUser` évite l'UAC sous Windows.
- **Fichiers de plateforme** (`tauri.macos.conf.json`, `tauri.windows.conf.json`, `tauri.linux.conf.json`) : `init` n'écrit que `tauri.conf.json`, et ne les modifie jamais. Il les lit fusionnés, comme `tauri build`, avant d'écrire quoi que ce soit ; illisibles, ils arrêtent tout.
  - Si l'un d'eux masque un de ces réglages pour les builds de son système, `init` le signale d'une ligne `!`, qui nomme le fichier, le réglage et la valeur qu'il y prend. Corrige ce fichier, ou retires-en le réglage, puis relance `drakbase init <slug>`.
  - Avec `--sign-id`, si `tauri.macos.conf.json` donne déjà un autre certificat (`bundle.macOS.signingIdentity`), le certificat demandé n'est pas écrit dans `tauri.conf.json`, où il resterait sans effet, et `init` le dit : change-le dans `tauri.macos.conf.json`.
- `bundle.targets` : la même liste convient aux trois systèmes, par exemple `["app", "dmg", "nsis", "appimage"]`. Le DMG sert à la première installation, jamais aux mises à jour. Évite MSI : il s'installe pour toute la machine et demande l'UAC à chaque mise à jour.
- **`src-tauri/capabilities/default.json`** : `updater:default`, `process:allow-restart` et `store:default`, pour la fenêtre qui gère les mises à jour. Si l'app a d'autres fichiers de capacités, la fenêtre concernée doit avoir ces trois permissions.
- **`src-tauri/Cargo.toml`** : `tauri-plugin-updater`, `tauri-plugin-process`, `tauri-plugin-store` et `tauri-plugin-os`.
- **`package.json`** : `@drakbase/tauri` (depuis le dépôt privé, `git+ssh://git@github.com/kcampion/drakbase-apps-market.git#semver:^0.5.1`) et les quatre paquets `@tauri-apps/plugin-…`. Lance ensuite `npm install`. Jusqu'à la 0.2, l'aide s'appelait `@appsmarket/tauri` : une app qui l'utilise encore change le nom de la dépendance et de ses imports, puis lance `npm install`.

## Ce qui reste à écrire

### Les plugins, côté Rust

Dans `src-tauri/src/lib.rs` (ou `main.rs`), dans le `Builder` :

```rust
tauri::Builder::default()
    // Si l'app l'utilise : toujours en premier, une seconde instance doit s'arrêter avant le reste.
    .plugin(tauri_plugin_single_instance::init(|app, _args, _cwd| {
        if let Some(window) = app.get_webview_window("main") {
            let _ = window.set_focus();
        }
    }))
    .plugin(tauri_plugin_updater::Builder::new().build())
    .plugin(tauri_plugin_process::init())
    .plugin(tauri_plugin_store::Builder::new().build())
    .plugin(tauri_plugin_os::init())
```

Si l'app empêche sa fermeture (`api.prevent_exit()` sur `RunEvent::ExitRequested`, pour rester dans la barre des menus par exemple), elle doit laisser passer la relance :

```rust
.run(|_app, event| {
    if let tauri::RunEvent::ExitRequested { code, api, .. } = event {
        if code != Some(tauri::RESTART_EXIT_CODE) {
            api.prevent_exit();
        }
    }
})
```

### L'interface, avec React

```tsx
import { useUpdater } from "@drakbase/tauri/react";

function Updates() {
  const updater = useUpdater(); // cherche une mise à jour au montage
  const { phase, update, progress, error, settings } = updater;

  return (
    <section>
      {phase === "available" && update ? (
        <>
          <p>La version {update.version} est disponible.</p>
          <pre>{update.notes}</pre>
          <button onClick={() => void updater.install()}>Installer et relancer</button>
        </>
      ) : null}
      {phase === "downloading" && progress ? (
        <progress max={progress.total ?? undefined} value={progress.downloaded} />
      ) : null}
      {phase === "error" ? <p>{error}</p> : null}
      <button onClick={() => void updater.check()}>Rechercher les mises à jour</button>
      <select
        value={settings?.channel ?? "stable"}
        onChange={async (event) => {
          await updater.setChannel(event.target.value === "beta" ? "beta" : "stable");
          await updater.check();
        }}
      >
        <option value="stable">Stable</option>
        <option value="beta">Bêta</option>
      </select>
      <label>
        <input
          type="checkbox"
          checked={settings?.shareStatistics ?? true}
          onChange={(event) => void updater.setShareStatistics(event.target.checked)}
        />
        Envoyer des statistiques anonymes
      </label>
    </section>
  );
}
```

- Phases : `idle`, `checking`, `upToDate`, `available`, `downloading`, `installing`, `error`.
- État : `currentVersion`, `settings` (`channel`, `shareStatistics`), `update` (`version`, `notes` en Markdown, `date`, `channel`), `progress` (`downloaded`, `total`), `error` (message en français), `lastCheck` (la dernière recherche réussie : une recherche qui échoue ne la change pas).
- Options de `useUpdater` : `checkOnMount` (vrai par défaut), `checkInterval`, `osVersion`, `timeout`, `beforeInstall`. `beforeInstall` est appelé une fois l'archive téléchargée et vérifiée, juste avant l'installation : c'est le moment d'enregistrer un travail en cours (sous Windows, l'installeur ferme l'app aussitôt).
- Le crochet cherche de nouveau à intervalle, pour qu'une app laissée ouverte voie les nouvelles versions : `checkInterval`, en millisecondes, vaut un jour par défaut, ramené entre une heure et trente jours ; `0` ou `null` pour ne chercher qu'au montage et à la demande. L'intervalle court depuis la dernière recherche, demandée ou non. Une recherche automatique qui échoue garde la version déjà proposée.
- `timeout`, en millisecondes, vaut 60 000 par défaut : passé ce délai, la recherche échoue (phase `error`) et peut être relancée. Il ne s'applique pas au téléchargement.
- Changer de canal oublie la mise à jour affichée : cherche à nouveau ensuite.
- Les statistiques sont activées par défaut : dis-le à côté de la case. Les couper efface l'identifiant d'installation.
- Rends les notes avec le composant Markdown de l'app s'il en a un ; `<pre>` suffit sinon.

### Sans React

```ts
import {
  checkForUpdate,
  describeUpdateError,
  installUpdate,
  isWithdrawnUpdateError,
  updateChannel,
} from "@drakbase/tauri";

let update = await checkForUpdate();
if (update) {
  console.log(update.version, updateChannel(update)); // « stable » ou « beta »
  try {
    await installUpdate(update, {
      onProgress: ({ downloaded, total }) => console.log(downloaded, total),
      beforeInstall: () => saveDraft(),
    });
  } catch (error) {
    showError(describeUpdateError(error)); // message en français
    if (isWithdrawnUpdateError(error)) {
      // Version retirée par l'éditeur (410) : elle ne reviendra pas, on cherche à nouveau.
      await update.close();
      update = await checkForUpdate();
    }
  }
}
```

- `installUpdate` télécharge, fait vérifier la signature, installe, puis appelle `relaunch()` : sur macOS et Linux, le plugin ne relance pas l'app lui-même. Ne remplace pas l'aide par un appel direct à `check()` du plugin : il faudrait refaire les en-têtes du canal et des statistiques.
- `updateChannel(update)` rend le canal de la version proposée, `stable` ou `beta`.
- `describeUpdateError(error)` rend un message en français pour une erreur du plugin, dont les messages sont en anglais ; le texte d'origine reste à la fin, entre parenthèses. Montre ce message, pas l'erreur brute.
- `isWithdrawnUpdateError(error)` est vrai quand le téléchargement a reçu `410` : l'éditeur a retiré la version entre la recherche et le téléchargement. Réessayer ne sert à rien : libère la mise à jour (`update.close()`), puis cherche à nouveau. Le crochet, lui, libère et oublie la version tout seul ; il reste à relancer `check()`.
- Sans le crochet, c'est à l'app de chercher de nouveau de temps en temps ; `checkForUpdate` a le même délai de 60 000 ms par défaut (option `timeout`).

L'aide est livrée en TypeScript, compilé par l'app : elle suppose `"moduleResolution": "bundler"`, ce que font Vite et les modèles de Tauri.

## Construire

```bash
drakbase exec <slug> -- npx tauri build
drakbase exec <slug> -- npx tauri build --target universal-apple-darwin   # Mac universel
```

`drakbase exec` vérifie que le mot de passe du trousseau ouvre la clé, puis lance la commande avec `TAURI_SIGNING_PRIVATE_KEY` et `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`. `tauri build` signe chaque archive de mise à jour et écrit sa `.sig` à côté. La version vient de `tauri.conf.json` : écris-la sous sa forme canonique (`1.2.0`, `1.3.0-beta.1`), car le plugin lit les versions strictement.

Fichiers produits, sous `src-tauri/target/release/bundle/` (ou `src-tauri/target/<cible>/release/bundle/` avec `--target`) :

| Système | Archive de mise à jour, avec sa `.sig` | Première installation |
|---|---|---|
| macOS | `macos/<Nom>.app.tar.gz` | `dmg/<Nom>_<version>_<arch>.dmg` |
| Linux | `appimage/<nom>_<version>_<arch>.AppImage` | la même AppImage |
| Windows | `nsis/<Nom>_<version>_<arch>-setup.exe` | le même installeur |

## Publier

Depuis le dossier du projet, pour que l'outil lise la clé embarquée dans `tauri.conf.json` (sinon `--tauri-config <chemin>`) :

```bash
drakbase publish --app <slug> --version 1.2.0 --channel stable --notes notes.md \
  "src-tauri/target/release/bundle/macos/Mon App.app.tar.gz" \
  "src-tauri/target/release/bundle/dmg/Mon App_1.2.0_aarch64.dmg" \
  chemin/vers/mon-app_1.2.0_amd64.AppImage \
  "chemin/vers/Mon App_1.2.0_x64-setup.exe"
```

- La `.sig` de chaque fichier est lue à côté de lui (aucune pour un DMG).
- L'emplacement (`os/arch/type`) se déduit du nom, avec l'arch écrite par Tauri. Sinon, donne-le : `darwin/aarch64/dmg=<fichier>`.
- L'outil ne fonctionne que sur macOS : les builds Linux et Windows reviennent sur le Mac pour être publiés.
- `--publish` seulement avec l'accord de l'utilisateur ; options et reprise dans `publication.md`.

## Plateformes

- **macOS** : il faut pouvoir écrire dans le dossier de l'app. Une app lancée depuis le DMG ou Téléchargements tourne depuis une copie en lecture seule : demande de la glisser dans Applications. L'aide envoie la version du système au serveur, qui écarte les versions trop récentes pour le Mac : elle la lit par `tauri-plugin-os`, que `drakbase init` ajoute et qui ne demande aucune permission. Si `.plugin(tauri_plugin_os::init())` manque, la recherche part sans cette version, et un Mac resté sur un ancien macOS reçoit et installe une version qu'il ne peut pas lancer : `drakbase doctor` le signale. `bundle.macOS.minimumSystemVersion` fixe le minimum d'une version. L'option `osVersion` de `useUpdater` et de `checkForUpdate` impose une autre version.
- **Linux** : seule l'AppImage se met à jour en place, et seulement dans un dossier de l'utilisateur. Compte environ 90 Mo par version et par arch. `deb` et `rpm` sont servis aux installations de ce type, avec un mot de passe à chaque mise à jour.
- **Windows** : l'installeur NSIS en `currentUser` s'installe sans UAC ; à la mise à jour, il ferme l'app, s'installe en mode passif et la relance. SmartScreen avertit à la première installation d'un installeur non signé. Ce parcours n'a pas encore été essayé sur un vrai PC : propose de le vérifier.

## Où l'aide range ses réglages

`appsmarket.json`, dans le dossier de données de l'app (`~/Library/Application Support/<identifier>/` sur macOS, `~/.local/share/<identifier>/` sous Linux, `%APPDATA%\<identifier>\` sous Windows), avec les clés `channel`, `shareStatistics` et `installId`.
