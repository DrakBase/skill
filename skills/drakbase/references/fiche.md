# La fiche publique : description et captures d'écran

Chaque app a une page publique (`https://updates.drakbase.com/apps/<slug>`) : son icône, sa description, ses captures d'écran, ses installeurs par système et l'historique de ses notes de version. À la première publication, cette page est vide de tout texte et de toute image : propose la description et les captures avant ou juste après le premier brouillon.

Tout se prépare dans le projet de l'app, dans un dossier à commiter, `fiche/` par exemple : la fiche se met ainsi à jour d'une version à l'autre.

## La description

- **Texte brut**, pas de Markdown : il s'affiche tel quel. Les paragraphes sont séparés par une ligne vide. 4000 caractères au plus ; 500 à 1200 suffisent en général.
- **Dans la langue des utilisateurs de l'app**, au ton de son README.
- **Contenu** :
  1. une première phrase qui dit ce qu'est l'app et à quoi elle sert ;
  2. un ou deux paragraphes sur ce qu'elle permet de faire, ses points forts concrets ;
  3. ce qu'il faut savoir avant de l'installer : système et version minimale, fonctionnement hors ligne, données gardées sur l'appareil, compte nécessaire ou non.
- **Sources** : le README, la description du projet, les écrans de l'app, les notes de version. N'invente aucune fonction, aucun chiffre, aucune promesse que l'app ne tient pas, et pas de superlatif sans fondement.
- Écris-la dans `fiche/description.txt` et montre-la à l'utilisateur avant de l'envoyer.

## Les captures d'écran

### Quoi montrer

- **3 à 6 captures**, 10 au plus : la fenêtre principale en usage, puis les deux ou trois fonctions qui distinguent l'app, éventuellement les réglages s'ils comptent. La première est la plus importante : elle s'affiche en tête.
- **Des données de démonstration**, réalistes et lisibles. Jamais de données personnelles réelles : emails, noms, comptes, adresses, jetons, fichiers privés, notifications. Si l'app a un mode de démonstration ou d'instantanés, utilise-le ; sinon, propose à l'utilisateur de préparer un jeu de données d'essai.
- **Une seule apparence** (claire ou sombre) pour toutes les captures, une taille de fenêtre raisonnable (environ 1200 à 1600 points de large), sans fenêtre vide ni message d'erreur.
- Une **légende** courte par capture (200 caractères au plus), qui dit ce qu'on voit : « La liste des articles, classés par flux ». Elle sert aussi de texte de remplacement.

### Comment les prendre, sur macOS

1. Construis l'app et lance-la avec ses données de démonstration (`open -n <chemin>/Nom.app`, ou `npx tauri dev` pour une app Tauri), puis amène-la sur l'écran à capturer.
2. Trouve le numéro de sa fenêtre. Ce petit script les liste :

   ```bash
   cat > "${TMPDIR:-/tmp}/fenetres.swift" <<'EOF'
   import CoreGraphics
   let owner = CommandLine.arguments.dropFirst().first ?? ""
   let windows = CGWindowListCopyWindowInfo([.optionOnScreenOnly, .excludeDesktopElements], kCGNullWindowID) as? [[String: Any]] ?? []
   for window in windows where window[kCGWindowOwnerName as String] as? String == owner
       && window[kCGWindowLayer as String] as? Int == 0 {
       let bounds = window[kCGWindowBounds as String] as? [String: Any] ?? [:]
       print(window[kCGWindowNumber as String] ?? "?", bounds["Width"] ?? "?", "x", bounds["Height"] ?? "?", window[kCGWindowName as String] ?? "")
   }
   EOF
   swift "${TMPDIR:-/tmp}/fenetres.swift" "Nom de l'app"
   ```

3. Capture cette fenêtre seule, sans ombre et sans bruit :

   ```bash
   screencapture -o -x -l <numéro> fiche/1-principale.png
   ```

   Pour passer d'un écran à l'autre, l'utilisateur clique dans l'app, ou tu l'y aides si l'app a des raccourcis ou une adresse (`open "monapp://…"`). Une capture interactive, où l'utilisateur clique lui-même sur la fenêtre, reste possible : `screencapture -o -w fiche/2-reglages.png`.
4. **Autorisation** : macOS demande l'autorisation « Enregistrement de l'écran et de l'audio système » pour le terminal qui lance `screencapture`. Sans elle, l'image ne montre que le fond d'écran ou une fenêtre vide. C'est l'utilisateur qui l'accorde, dans Réglages Système › Confidentialité et sécurité ; ne modifie pas ce réglage toi-même.

### Avant de les proposer

- **Regarde chaque image** (lis le fichier PNG) : bonne fenêtre, contenu lisible, aucune donnée personnelle, aucune autre app visible.
- **Contrôle les limites du serveur** : PNG seulement, 8 Mio au plus, de 320 à 8192 pixels de côté.

  ```bash
  sips -g pixelWidth -g pixelHeight fiche/*.png
  ls -l fiche/*.png
  ```

  Une capture Retina fait le double des points de la fenêtre : 2800 pixels de large pour une fenêtre de 1400 points, ce qui passe.
- Montre les captures et leurs légendes à l'utilisateur, avec la description.

## L'envoi : `drakbase listing`

```bash
drakbase listing <slug>                                    # la fiche actuelle, sans rien changer
drakbase listing <slug> --description fiche/description.txt \
  --screenshot "fiche/1-principale.png=La fenêtre principale" \
  --screenshot "fiche/2-reglages.png=Les réglages" \
  --dry-run                                                # ce qui changerait
```

- `--description <fichier>` remplace la description par le contenu du fichier.
- Chaque `--screenshot <fichier.png>[=<légende>]` ajoute une capture ; avec au moins un `--screenshot`, les captures du serveur deviennent exactement cette liste, dans cet ordre : les images déjà envoyées (même contenu) ne repartent pas, les légendes sont mises à jour, les autres captures sont retirées.
- Sans `--screenshot`, les captures ne changent pas ; `--clear-screenshots` les retire toutes.
- `--dry-run` montre ce qui changerait, sans rien modifier. Lance-le d'abord, puis la même commande sans `--dry-run` une fois que l'utilisateur a approuvé.
- Il faut un jeton avec le droit `listing:write`. Sans lui, l'outil le dit : l'utilisateur crée un nouveau jeton avec ce droit dans l'administration, puis relance `drakbase login`.
- La page publique ne montre l'app que si elle est visible (« Visible sur la page publique », onglet « Informations » de l'administration), et ses installeurs seulement après une version stable publiée.

## L'icône

L'icône de la page publique ne passe pas par l'outil : l'utilisateur dépose un PNG carré, 512 Kio au plus, dans l'administration, onglet « Informations » de l'app. Prépare-le depuis l'icône de l'app, par exemple `sips -s format png -z 512 512 <AppIcon>.icns --out fiche/icone.png` (ou à partir de la plus grande image de `AppIcon.appiconset`, ou de `src-tauri/icons/icon.png` pour Tauri).
