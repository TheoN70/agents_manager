Setup le framework "manager d'agents" sur le projet courant.

A executer une fois, a l'initialisation du framework dans un repo. Idempotent : peut etre relance pour ajouter de nouveaux projets ou regenerer les fichiers manquants.

## Etapes

### 1. Recuperer la liste des projets

Demande a l'utilisateur la liste des projets (sous-repos) qui composent le produit. Exemples :
- Un seul : `backend`
- Deux : `backend`, `frontend`
- Trois : `api`, `web`, `mobile`

Format attendu : noms separes par des virgules ou des espaces. Les noms correspondent aux dossiers du repo racine qui contiennent chacun un repo git distinct.

### 2. Verifier la presence des dossiers

Pour chaque nom donne :
- Si `<nom>/` existe : OK, conserve
- Si `<nom>/` n'existe pas : demande a l'utilisateur si le dossier doit etre cree (vide) ou si le nom est une erreur

### 3. Ecrire `.claude/projects.json`

Cree ou met a jour `.claude/projects.json` :

```json
{
  "projects": ["backend", "frontend"]
}
```

Si le fichier existe deja, fusionne sans doublon. Conserve l'ordre fourni par l'utilisateur.

### 4. Ajouter les projets au `.gitignore` racine

Lis le `.gitignore` a la racine (cree-le s'il n'existe pas). Pour chaque projet enregistre, ajoute une ligne `<nom>/` si elle n'est pas deja presente. Conserve les lignes existantes.

Justification : le repo racine ne contient que le framework Claude. Les sous-projets sont des repos git imbriques avec leur propre historique — ils ne doivent pas etre suivis par le repo racine.

Exemple de `.gitignore` resultant :

```
backend/
frontend/
```

### 5. Verifier la structure `.claude/`

Verifie que les dossiers suivants existent, cree-les sinon :
- `.claude/commands/`
- `.claude/skills/`
- `.claude/agents/`

Ne touche PAS aux fichiers existants dans ces dossiers (skills, agents et commandes deja en place).

### 6. Verifier la structure `docs/`

Verifie que les dossiers suivants existent, cree-les sinon (avec un `.gitkeep` si vides) :
- `docs/architecture/`
- `docs/domain/`
- `docs/playbooks/`
- `docs/synthesis/`

### 7. Bootstrap du `CLAUDE.md` racine

Si `CLAUDE.md` racine n'existe pas, cree un squelette minimal :

```markdown
# <Nom du projet>

## Projets

<liste des projets enregistres avec leur role en une ligne>

## Workflow

Tout ticket non-trivial se demarre en plan mode (Shift+Tab).

## Slash commands

| Commande | Role |
|----------|------|
| `/install` | Setup ou mise a jour du framework |
| `/review-mine` | Auto-review du diff avant commit |
| `/update-brain` | Mise a jour de la doc apres un changement |
| `/commit-changes` | Commiter dans les projets enregistres |

## Apres toute modification de code

1. `/review-mine`
2. Tests
3. `/update-brain`
4. `/commit-changes`
```

Demande a l'utilisateur le nom du projet et le role de chaque sous-projet pour completer le squelette. Si `CLAUDE.md` existe deja, ne le modifie pas.

### 8. Recapitulatif

Affiche :
- Liste des projets enregistres dans `.claude/projects.json`
- Lignes ajoutees au `.gitignore`
- Fichiers/dossiers crees
- Fichiers laisses intacts (deja presents)

Termine par : « Framework installe. Prochaines etapes : initialiser le brain (`CLAUDE.md` par projet, `docs/domain/GLOSSAIRE.md`, ADRs retroactifs) — voir Partie 2 du README. »
