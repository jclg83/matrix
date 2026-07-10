# Synchroniser un skill partagé depuis GitHub `jclg83/matrix`

Quand Christophe demande à un agent de « mettre à jour son skill hermes-team-operations » depuis GitHub Matrix, il s'agit de synchroniser la copie locale du skill avec la source partagée.

## Source canonique

- Repo : `jclg83/matrix`
- Chemin : `hermes-team-operations/`
- Fichier principal : `hermes-team-operations/SKILL.md`
- Références : `hermes-team-operations/references/*.md`
- Raw principal : `https://raw.githubusercontent.com/jclg83/matrix/main/hermes-team-operations/SKILL.md`

## Destination locale SOS

```text
C:\Users\strap\AppData\Local\hermes-sos\skills\team\hermes-team-operations\
```

## Procédure sûre

1. Lire la copie locale via `skill_view(name="hermes-team-operations")` pour confirmer le chemin réel.
2. Interroger l'API GitHub Contents sur `repos/jclg83/matrix/contents/hermes-team-operations?ref=main` et parcourir récursivement les fichiers.
3. Faire un backup de l'ancien dossier **hors du dossier `skills/`**.
   - Exemple correct : `C:\Users\strap\hermes-sos-workspace\backups\hermes-team-operations.backup-YYYYMMDD-HHMMSS`
   - Piège : ne pas laisser `hermes-team-operations.backup-*` sous `...\skills\team\`, car le chargeur de skills détecte deux `SKILL.md` avec le même `name:` et `skill_view` devient ambigu.
4. Écrire `SKILL.md` et tous les `references/*.md` depuis les raw/download URLs GitHub.
5. Vérifier byte-à-byte que chaque fichier local correspond au remote.
6. Relire un fichier de référence via `skill_view(name="hermes-team-operations", file_path="references/...")` pour confirmer que le skill n'est pas ambigu.

## Validation minimale attendue

- `SKILL.md` existe.
- `references/` contient tous les fichiers du repo.
- Le contenu local matche le contenu GitHub (`ALL_MATCH True` ou équivalent).
- `skill_view` fonctionne sans erreur d'ambiguïté.

## Piège appris

Créer un backup directement dans le répertoire parent des skills, par exemple :

```text
...\skills\team\hermes-team-operations.backup-YYYYMMDD-HHMMSS\SKILL.md
```

provoque une collision de nom : le loader voit deux skills nommés `hermes-team-operations`. Déplacer le backup dans le workspace ou un dossier sans scan de skills avant de valider.