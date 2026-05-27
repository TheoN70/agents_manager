# OfficeIn — Manager d'agents Claude Code

Guide pratique pour travailler en tant que manager d'agents sur un
projet logiciel. Ce document couvre le workflow quotidien, les outils
disponibles et les rituels periodiques.

---

## Reference rapide

### Workflow d'un ticket

```
1. Rediger le ticket    → discussion avec Claude (Claude genere, tu valides)
2. Plan mode            → Shift+Tab (l'agent explore, propose un plan)
3. Valider le plan      → toi, tu approuves ou corriges
4. Execution            → l'agent code, tu surveilles
5. Review               → /review-mine
6. Tests                → execution automatique par l'agent
7. Mise a jour doc      → /update-brain
8. Commit               → /commit-changes
```

### Commandes Claude Code

| Raccourci / commande | Action |
|----------------------|--------|
| `Shift+Tab` | Entrer en plan mode (exploration + plan avant code) |
| `Shift+Tab` (en plan mode) | Sortir du plan mode, lancer l'execution |
| `Tab` | Cycle entre les modes de permission (ask / auto-edit / full-auto) |
| `Esc` | Interrompre l'agent en cours d'execution |

### Slash commands du projet

Commandes custom definies dans `.claude/commands/`. Elles s'invoquent
directement dans Claude Code avec `/nom-de-la-commande`.

| Commande | Ce qu'elle fait | Quand l'utiliser |
|----------|-----------------|------------------|
| `/install` | Setup le framework sur le repo : enregistre les projets dans `.claude/projects.json`, met a jour le `.gitignore`, scaffold la structure `.claude/` et `docs/` | Une fois a l'initialisation, ou pour ajouter un projet |
| `/review-mine` | Pour chaque projet enregistre, analyse `git -C <projet> diff`, lit les CLAUDE.md des apps touchees, verifie conventions, tests, qualite, docs et securite, produit une grille pass/fail par projet | Avant de commiter |
| `/update-brain` | Detecte ce qui a change dans chaque projet et met a jour les docs concernees : CLAUDE.md d'app, `<app>/docs/API.md`, GLOSSAIRE, ADRs | Fin de session |
| `/commit-changes` | Pour chaque projet enregistre, analyse le diff, redige un message dans le style du projet, stage les fichiers par nom et cree le commit | Quand la review et les tests passent |

---

## Partie 1 — Utiliser le framework au quotidien

### 1.1 Ton role

Tu ne codes plus — tu **specifies**, tu **orchestres** et tu **verifies**.
Ton travail se decompose en trois activites :

1. **Specifier** : transformer un besoin metier en ticket executable
   (contexte, contraintes, criteres de succes testables)
2. **Orchestrer** : piloter l'agent en plan mode, valider les plans,
   debloquer les impasses
3. **Verifier** : reviewer le code produit, lancer les checks, maintenir
   la documentation

Le code est un livrable de l'agent. Le brain (documentation) est ton
livrable a toi.

---

### 1.2 Workflow d'un ticket

```
  Besoin
    │
    ▼
┌─────────────────────┐
│  1. Rediger le ticket │  ← Claude genere depuis ton besoin, tu valides
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│  2. Plan mode         │  ← Shift+Tab, l'agent explore et propose
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│  3. Valider le plan   │  ← toi, tu approuves ou tu corriges
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│  4. Execution         │  ← l'agent code, toi tu surveilles
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│  5. Review            │  ← /review-mine
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│  6. Tests             │  ← execution automatique par l'agent
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│  7. Mise a jour doc   │  ← /update-brain
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│  8. Commit            │  ← /commit-changes
└─────────────────────┘
```

---

### 1.3 Etape par etape avec exemples

#### Etape 1 — Rediger le ticket

Un ticket bien ecrit est un bon prompt — si tu bacles la spec, l'agent
produira du code plausible mais faux.

**Claude genere, tu valides :** tu decris le besoin en langage naturel,
et l'agent genere le ticket complet.

> « Je veux permettre a la SC de signaler qu'une facture a ete
> encaissee. Genere un ticket complet structure (Objectif, Contexte,
> Fichiers concernes, Contraintes, Criteres de succes, Commande de
> validation). Appuie-toi sur les specs dans
> docs/synthesis/envoi-reception-factures.md. »

L'agent produit un brouillon structure. **Tu dois imperativement :**
- Verifier que l'objectif correspond a ce que tu veux reellement
- Verifier que les contraintes metier sont completes (l'agent ne
  connait que ce qui est documente)
- Ajouter le contexte non ecrit (deadlines, dependances avec
  d'autres equipes, decisions recentes non encore en ADR)

**Le ticket final est ta responsabilite.** L'agent t'aide a le
structurer et a identifier les fichiers — le jugement metier reste
le tien.

**Exemple de resultat : ajouter un endpoint de statut "Encaissee"**

```markdown
## Objectif
Permettre a la SC de signaler qu'une facture a ete encaissee (statut 212).

## Contexte
- Spec : docs/synthesis/envoi-reception-factures.md, section statuts obligatoires
- Le statut 212 doit declencher un F6 vers le PPF dans les 24h
- L'endpoint existe deja pour les statuts 200/210/213, il faut ajouter 212

## Fichiers concernes
- invoicing/views.py — nouvel endpoint ou extension de l'existant
- invoicing/services.py — logique de transition de statut
- invoicing/tests.py — tests du nouveau statut
- ppf_connectivity/ — declenchement F6

## Contraintes
- Non-regression sur les statuts existants (200, 210, 213)
- Le F6 doit respecter le format CDAR
- Seule une facture au statut 200 (Deposee) peut passer a 212

## Criteres de succes
- [ ] POST /api/invoices/{id}/status/encaissee/ retourne 200
- [ ] Une facture au statut 210 ne peut pas passer a 212 (retourne 400)
- [ ] Un F6 est cree dans ppf_connectivity apres le changement de statut
- [ ] Tests unitaires couvrent happy path + cas d'erreur

## Commande de validation
python manage.py test invoicing ppf_connectivity
```

#### Etape 2 — Plan mode

Ouvre Claude Code, colle le ticket, et entre en plan mode (Shift+Tab).
L'agent va :
- Lire les CLAUDE.md des apps concernees
- Lire les docs et specs referencees dans le ticket
- Proposer un plan structure (fichiers a modifier, approche, risques)

**Ce que tu regardes dans le plan :**
- Est-ce que l'agent a compris le besoin metier ? (pas juste la mecanique)
- Est-ce qu'il a identifie les bons fichiers ?
- Est-ce qu'il a prevu les cas d'erreur ?
- Est-ce qu'il propose des tests ?

**Exemple de correction a ce stade :**
> « Non, le F6 n'est pas cree dans la view — il est enqueue en async
> via ppf_connectivity. Regarde comment c'est fait pour le statut 210
> dans invoicing/services.py. »

#### Etape 3 — Valider le plan

Si le plan est bon, approuve. Si quelque chose manque, corrige avant
de laisser l'agent coder. C'est **ici** que tu exerces ton jugement —
pas apres 200 lignes de code ecrit.

#### Etape 4 — Execution

L'agent code. Pendant qu'il travaille, tu peux :
- Verifier qu'il suit le plan
- Corriger en cours de route si tu vois une derive
- Repondre a ses questions

**Ce qui doit t'alerter :**
- L'agent cree des fichiers non prevus dans le plan
- L'agent modifie des fichiers hors perimetre
- L'agent ecrit beaucoup de code sans tests

#### Etape 5 — Review

Lance `/review-mine`. La commande analyse le diff git et produit une
grille pass/fail sur 5 criteres : conventions, tests, qualite,
documentation, securite. Elle lit les CLAUDE.md des apps concernees
pour verifier la coherence.

**Exemple de sortie :**
```
| Critere          | Verdict | Detail                                      |
|------------------|---------|---------------------------------------------|
| Conventions code | OK      | Imports corrects, nommage conforme           |
| Tests            | KO      | Pas de test pour le cas facture au statut 210 |
| Documentation    | KO      | invoicing/docs/API.md non mis a jour         |
| Securite         | OK      | Permissions verifiees                        |
| Architecture     | OK      | Pattern async coherent avec l'existant       |

Verdict final : FAIL
Issues a corriger :
1. Ajouter test_encaissee_from_invalid_status
2. Mettre a jour invoicing/docs/API.md
```

→ Tu demandes a l'agent de corriger les deux points, puis tu relances.

#### Etape 6 — Tests

**L'agent lance les tests automatiquement** apres l'execution. Il
execute la suite definie dans le ticket (`Commande de validation`),
analyse la sortie et corrige les echecs avant de te rendre la main.

```bash
python manage.py test --keepdb
# ou, pour une boucle plus rapide :
python manage.py test --exclude-tag=invoice_extraction --keepdb
```

Tu interviens si l'agent boucle sans converger : tu lis la sortie,
identifies la cause et donnes la correction precise.

#### Etape 7 — Mise a jour de la documentation

Lance `/update-brain`. La commande analyse le travail effectue et met
a jour :
- `<app>/docs/API.md` si l'API a change
- `<app>/CLAUDE.md` si la structure a change
- `docs/domain/GLOSSAIRE.md` si un nouveau terme est apparu
- `docs/architecture/` si une decision a ete prise

La documentation doit suffire a donner une vision claire du code —
pas de journal de bord (DONE.md, CHANGELOG manuel) : le git log tient
ce role.

Verifie le diff de doc — l'agent peut manquer du contexte metier.

#### Etape 8 — Commit

Lance `/commit-changes`.

---

### 1.4 Quand utiliser quoi

| Situation | Action |
|-----------|--------|
| Nouveau ticket | Decrire le besoin a Claude, valider le ticket genere, demarrer en plan mode |
| Bug simple | Decrire le bug, laisser l'agent diagnostiquer et corriger |
| Refactor | Plan mode obligatoire — le perimetre doit etre clair avant de commencer |
| Exploration / spike | Pas de plan mode, conversation libre avec l'agent |
| Avant commit | `/review-mine` |
| Fin de session | `/update-brain` pour tout ce qui a change |

---

### 1.5 Comment gerer les impasses

L'agent peut se bloquer ou tourner en rond. Reactions :

**L'agent ne comprend pas le besoin metier :**
→ Reformule en pointant la spec precise.
> « Lis docs/synthesis/envoi-reception-factures.md section 4.2 — le
> statut 212 est decrit la. »

**L'agent produit du code qui ne compile pas / ne passe pas les tests :**
→ Ne le laisse pas iterer plus de 2 fois. Lis l'erreur toi-meme,
identifie le probleme, et donne-lui la correction precise.

**L'agent propose un plan trop complexe :**
→ Simplifie.
> « Non, pas besoin d'un nouveau service. Ajoute la logique dans la
> fonction existante process_status_change. »

**L'agent modifie des fichiers hors perimetre :**
→ Arrete-le.
> « Stop — on ne touche pas a authentication/ pour ce ticket. Reviens
> au perimetre defini. »

---

### 1.6 Signaux d'alerte

Tu dois reagir si tu observes :

- **Tu approuves les plans sans vraiment les lire** → reprends un
  ticket en code manuel pour te recalibrer
- **Les bugs en prod augmentent** → renforce la review, ajoute des
  tests edge case, invoque `/review-mine` systematiquement
- **Le brain n'est plus a jour** → bloque-toi une demi-journee pour
  supprimer l'obsolete, c'est de la dette qui s'accumule
- **Tu ne sais plus comment fonctionne un module** → c'est le signal
  le plus grave. Relis le module immediatement.

Voir `docs/GOVERNANCE.md` pour le plan de rollback complet.

---

---

## Partie 2 — Implementer ce framework dans un projet

Guide pas-a-pas pour deployer le framework "manager d'agents" sur un
projet existant ou nouveau. Concu pour etre suivi tel quel, dans
l'ordre — sans calendrier impose : avance au rythme du projet.

### Prerequis

- Un repo git avec du code fonctionnel
- Claude Code installe et fonctionnel
- Un developpeur volontaire pour le pilote (idealement quelqu'un qui
  connait deja le projet)

---

### Phase 0 — Setup automatique via `/install`

Avant toute chose, lance `/install` a la racine du repo. La commande :

- Te demande la liste des projets (sous-repos) qui composent le produit.
  Multiples possibles, par exemple `backend` et `frontend`. Chaque projet
  est un repo git imbrique avec son propre historique.
- Enregistre la liste dans `.claude/projects.json` :

  ```json
  { "projects": ["backend", "frontend"] }
  ```

  Ce fichier est lu par toutes les autres slash commands pour iterer
  sur les projets (diff, review, commit).
- Ajoute chaque projet au `.gitignore` racine. Le repo racine ne suit
  que le framework Claude (`.claude/`, `docs/`, `README.md`) — les
  projets imbriques ont leur propre historique git et ne doivent pas
  etre suivis par le racine.
- Scaffold les dossiers `.claude/{commands,skills,agents}/` et
  `docs/{architecture,domain,playbooks,synthesis}/` s'ils manquent.
- Cree un squelette de `CLAUDE.md` racine si absent.
- **Genere le brain initial** (Phase 1 ci-dessous) pour chaque projet :
  `CLAUDE.md` racine du projet, `CLAUDE.md` par app, `docs/ARCHITECTURE.md`,
  `docs/API.md`, `docs/AUDIT_SECURITE.md`, plus `docs/domain/GLOSSAIRE.md`
  et `docs/architecture/{README,ADR_TEMPLATE}.md` a la racine. La Phase 1
  decrit ce que chaque fichier doit contenir — `/install` produit les
  brouillons, la validation metier reste a ta charge.

La commande est idempotente : ne touche jamais a un fichier existant.
Relance-la pour ajouter un nouveau projet ou completer le brain apres
avoir supprime manuellement les fichiers a regenerer.

---

### Phase 1 — Poser le brain

**Objectif** : que l'agent ait acces a tout le contexte necessaire
pour comprendre le projet sans aide humaine.

#### 1.1 Creer le CLAUDE.md racine

C'est le point d'entree de tout agent. Il doit contenir :

```
CLAUDE.md
├── Description du projet (1 paragraphe)
├── Commandes essentielles (setup, test, run)
├── Table des apps/modules avec description courte
├── Fichiers critiques (settings, modeles centraux, specs)
├── Skills disponibles
├── Workflow (plan mode)
├── Regles pour les agents
├── Apres toute modification de code (checklist)
└── Slash commands disponibles
```

**Piege a eviter** : ne pas ecrire un roman. Le CLAUDE.md racine doit
tenir en ~100 lignes. Le detail va dans les CLAUDE.md d'apps et les
docs/.

#### 1.2 Creer un CLAUDE.md par app/module

Chaque app a son propre CLAUDE.md qui documente :
- Commandes de test specifiques
- Table des fichiers et leurs roles
- Diagrammes de flux (ASCII)
- Mecanismes cles (securite, async, validation)
- Pointeurs vers les docs detaillees

**Methode** : pour chaque app, lis le code et demande a l'agent de
generer un brouillon de CLAUDE.md. Relis-le et corrige. C'est plus
rapide que de tout ecrire a la main et ca te force a relire le code.

#### 1.3 Creer la documentation par app

Pour chaque app, creer un dossier `docs/` avec :
- `ARCHITECTURE.md` — schema, flux, modeles
- `API.md` — reference des endpoints (si applicable)
- `AUDIT_SECURITE.md` — invariants securite (permissions, validations, ISO 27001)

Convention de nommage : MAJUSCULES avec `_` comme separateur.

**Methode** : meme principe — l'agent genere, tu corriges.

#### 1.4 Creer le glossaire metier

Fichier `docs/domain/GLOSSAIRE.md`. Table alphabetique :

```
| Terme | Definition | Alias/Contexte |
```

Inclure tous les acronymes, termes metier, et conventions de nommage
du projet. C'est le document le plus sous-estime et le plus utile —
il evite les malentendus entre toi et l'agent.

#### 1.5 Creer les ADRs retroactifs

Dossier `docs/architecture/` avec :
- `README.md` — index
- `ADR_TEMPLATE.md` — template standard
- ADRs retroactifs nommes `ADR_NNN_TITRE_COURT.md` (ex. `ADR_001_TOKEN_AUTH.md`)

**Comment choisir les premiers ADRs** : quelles decisions surprendraient
un nouveau developpeur ? Pourquoi telle techno et pas une autre ?
Pourquoi tel pattern ?

---

### Phase 2 — Outiller l'agent

**Objectif** : que l'agent produise du code conforme aux standards du
projet sans rappels manuels constants.

#### 2.1 Skills presents

Dossier `.claude/skills/`. Sept skills en place :

| Skill | Trigger | Role |
|-------|---------|------|
| `python` | always | Conventions Python/Django backend : indent 4 espaces, guillemets doubles, f-strings, snake_case / PascalCase / UPPER_SNAKE_CASE, type hints 3.10+, FBV + `@extend_schema`, heritage `BaseModel`, ordre strict des imports (stdlib → third-party → django → drf → local), pas de `N+1`, pas de bare `except:` |
| `react-nextjs` | mandatory pre-condition | Doit etre invoque AVANT de creer une feature, une page, un composant, un server action, un service ou un hook frontend |
| `commit` | mots-cles + `/commit` | Seule voie autorisee pour creer un commit (jamais `git commit` manuel). Format `[FIX]/[REFACTO]/[FEATURE]/[DOCS]/[TEST]/[CHORE]` suivi d'une description en francais minuscule |
| `code_review` | `/code_review` | Lit le diff, invoque l'agent `cybersecurite`, applique les recommandations |
| `review_pr` | `/review_pr` | Review sur une PR GitHub, commentaires via `gh pr comment` |
| `trello` | mots-cles | API Trello via CLI (boards, cartes, listes, labels, membres). Board par defaut "OfficeIn Dev" |
| `work` | mot-cle "ticket" | Realise un ticket Trello de bout en bout : recuperation → analyse en plan mode → branche depuis `develop` → code (skill `python`) → commit (skill `commit`) → PR → cloture |

Le skill `always` le plus structurant est `python` — il s'injecte
automatiquement sur tout travail backend. `react-nextjs` joue le meme
role cote frontend mais en pre-condition bloquante.

**Piege a eviter** : ne pas mettre trop de contenu dans un skill.
Un skill de 500 lignes est contre-productif — l'agent le "noie" dans
le contexte. Viser 50-150 lignes par skill.

#### 2.1bis Agents presents

Dossier `.claude/agents/`. Trois agents specialises appelables
depuis les skills ou le code :

| Agent | Role |
|-------|------|
| `code_quality_python` | Audit qualite Python/Django sur six axes : lisibilite, performance, securite, architecture, dette technique, elegance. Produit un rapport priorise 🔴 haute / 🟡 moyenne / 🟢 basse |
| `code_quality_nextjs` | Audit qualite frontend Next.js/React/TypeScript sur sept axes : lisibilite, performance, securite, architecture, dette technique, accessibilite, elegance. Meme format de rapport |
| `cybersecurite` | Audit securite ; rapporte les failles detectees. Invoque par la skill `code_review` |

#### 2.2 Creer les slash commands

Dossier `.claude/commands/`. Quatre commandes couvrant tout le workflow :

| Commande | Role |
|----------|------|
| `/install` | Setup initial : enregistre les projets, met a jour `.gitignore`, scaffold la structure |
| `/review-mine` | Auto-review du diff par projet avant commit |
| `/update-brain` | Mise a jour de la documentation apres un changement |
| `/commit-changes` | Commiter dans chaque projet enregistre avec un message adapte a son style |

Les trois commandes de workflow (`review-mine`, `update-brain`,
`commit-changes`) lisent `.claude/projects.json` cree par `/install`
et iterent sur chaque projet (`git -C <projet> diff`, `git -C <projet>
commit`, etc.) — un projet sans changement est ignore.

La redaction du ticket n'a pas de commande dediee : le manager decrit
son besoin en langage naturel a Claude, qui genere le ticket structure
(cf. section 1.3, etape 1).

Chaque commande est un fichier markdown qui decrit pas-a-pas ce que
l'agent doit faire et le format de sortie attendu.

#### 2.3 Documenter les regles dans le CLAUDE.md

Ajouter au CLAUDE.md racine :
- Section **Workflow** : « tout ticket non-trivial se demarre en plan mode »
- Section **Regles pour les agents** : comportements automatiques attendus
- Section **Apres toute modification de code** : checklist post-changement
- Section **Slash commands** : table des commandes disponibles

---

### Phase 3 — Poser les processus (en parallele de la Phase 2)

**Objectif** : que le workflow soit documente et reproductible.

#### 3.1 Definir la structure de ticket attendue

Pas de fichier template a maintenir : la structure est documentee
dans le CLAUDE.md racine et appliquee par l'agent lors de la
generation du ticket. Sections imposees : Objectif, Contexte,
Fichiers concernes, Contraintes, Criteres de succes, Commande de
validation.

Le manager decrit son besoin en langage naturel ; l'agent produit
le ticket structure ; le manager valide et corrige le contexte
metier non documente. Le ticket final reste la responsabilite du
manager.

#### 3.2 Creer les playbooks operationnels

Dossier `docs/playbooks/`. Au minimum :
- Comment creer une nouvelle app/module
- Checklist pre-release

Chaque playbook est une recette pas-a-pas utilisable par un agent
ou un humain.

#### 3.3 Creer la gouvernance du brain

Fichier `docs/GOVERNANCE.md` :
- Carte d'ownership (quelle source de verite pour quel sujet)
- Regles de pruning (trimestriel, commit dedie)
- Limites de volume (seuils de restructuration)
- KPIs (velocite, qualite, sante du brain, competence)
- Plan de rollback (signaux d'alerte + procedure)

---
