# Skill Claude « drakbase »

Plugin [Claude Code](https://claude.com/claude-code) pour les développeurs qui publient leurs apps sur **DrakBase** (`https://updates.drakbase.com`), un serveur de mises à jour pour les apps macOS en Swift et les apps Tauri 2 (macOS, Linux, Windows).

La skill `drakbase` guide Claude pour :

- **brancher les mises à jour** dans une app : le paquet `DrakBase` pour une app Swift, le plugin officiel `tauri-plugin-updater` et l'aide `@drakbase/tauri` pour une app Tauri ;
- **publier une version**, stable ou bêta, avec l'outil `drakbase` : construction, signature, brouillon, notes de version ;
- **préparer la fiche publique** de l'app à sa première publication : une description et des captures d'écran, envoyées avec `drakbase listing` ;
- **dépanner** : les contrôles de `drakbase doctor` et les messages de l'outil, avec leur correction.

Les secrets restent au développeur : Claude ne demande ni le jeton d'API ni les mots de passe (base KeePassXC, clés), ne touche pas aux clés privées, et ne publie rien sans son accord.

## Installation

```bash
claude plugin marketplace add DrakBase/skill
claude plugin install drakbase@drakbase
```

Dans une session Claude Code, les mêmes commandes s'écrivent `/plugin marketplace add DrakBase/skill` et `/plugin install drakbase@drakbase`.

Sans place de marché : copier le dossier `skills/drakbase` dans `~/.claude/skills/` (pour soi), ou dans `.claude/skills/` du projet de l'app (pour toute son équipe).

## Utilisation

Dans le dossier de l'app, ouvrir Claude Code et demander par exemple :

- « Branche les mises à jour DrakBase dans cette app »
- « Publie la version 1.2.0 en bêta »
- « Prépare la description et les captures d'écran de l'app »
- « drakbase doctor signale un problème, aide-moi »

La skill se déclenche d'elle-même ; `/drakbase:drakbase` l'appelle directement (`/drakbase` si elle est copiée dans un dossier de skills).

## Ce qu'il faut pour s'en servir

- un compte sur l'administration du serveur DrakBase (éditeur ou superadmin), et un jeton d'API créé dans « Mon compte » ;
- un Mac, avec KeePassXC pour la sauvegarde des clés, et, pour une app macOS, un certificat de signature de code ;
- l'outil `drakbase`, le paquet Swift `DrakBase` et `@drakbase/tauri` viennent d'un dépôt privé : il faut une clé SSH qui y a accès. La skill explique leur installation.

## Mises à jour

La skill suit les versions de l'outil `drakbase` : chaque version porte une étiquette `vX.Y.Z`. Pour la récupérer :

```bash
claude plugin marketplace update drakbase
```

ou activer la mise à jour automatique dans `/plugin`, onglet « Marketplaces ».

## Contenu

```
.claude-plugin/marketplace.json   la place de marché, dont ce dépôt est aussi le seul plugin
.claude-plugin/plugin.json        le plugin, à la version de l'outil
skills/drakbase/SKILL.md          la skill : règles, parcours (brancher, publier, dépanner)
skills/drakbase/references/       fiches lues à la demande : swift.md, tauri.md, publication.md, fiche.md, depannage.md
```

Ce dépôt est une copie publiée : ses sources sont maintenues avec l'outil, et chaque publication remplace son contenu.
