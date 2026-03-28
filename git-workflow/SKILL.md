---
name: git-workflow
description: Gestion du workflow Git complet par feature. Use when user wants to créer une branche, committer des changements, pusher, ou finir une feature. Crée une branche depuis main pour chaque feature, commit avec Conventional Commits, push. Ne merge jamais automatiquement — laisse l'utilisateur décider.
allowed-tools:
  - Bash
---

# Git Workflow — Feature Branch

Une branche par feature depuis main. Commit propre après chaque feature. Push. L'utilisateur merge quand il veut.

## État du repo

!`git branch --show-current 2>/dev/null || echo "(pas de repo git)"`
!`git log --oneline -3 2>/dev/null || echo "(pas de commits)"`

## Workflow complet — étapes dans l'ordre

### 1. Initialiser le repo (une seule fois)

```bash
git init
git add .
git commit -m "chore(setup): initial project structure"
git branch -M main
```

### 2. Démarrer une feature — créer la branche

**TOUJOURS** depuis main :

```bash
git checkout main
git checkout -b feat/<nom-feature>
```

Nommage des branches :

| Type | Format | Exemple |
|------|--------|---------|
| Nouvelle feature | `feat/<nom>` | `feat/users-crud` |
| Correction | `fix/<nom>` | `fix/users-404` |
| Config/tooling | `chore/<nom>` | `chore/docker-setup` |

### 3. Committer la feature

```bash
git add .
git commit -m "type(scope): description"
```

Format Conventional Commits :

| Type | Quand |
|------|-------|
| `feat` | Nouvelle fonctionnalité |
| `fix` | Correction de bug |
| `chore` | Config, deps, tooling |
| `refactor` | Refacto sans changement fonctionnel |
| `style` | Mise en forme, espacements |
| `docs` | Documentation |

**Scope** = la partie du projet concernée (kebab-case) :

```
feat(users): add CRUD endpoints with Prisma
feat(projects): add list and create endpoints
feat(ui-users): add users table and create dialog
feat(ui-projects): add projects page with routing
fix(users): return 404 when user not found
chore(docker): add postgres docker-compose
chore(prisma): init schema and first migration
chore(backend): configure ValidationPipe and CORS
chore(frontend): init react-router and shadcn
```

### 4. Pusher la branche

```bash
git push -u origin feat/<nom-feature>
```

### 5. Revenir sur main pour la prochaine feature

```bash
git checkout main
git checkout -b feat/<prochaine-feature>
```

> Le merge de la branche précédente se fait manuellement par l'utilisateur quand il le souhaite.

---

## Exemple — séquence complète sur un test technique 3h

```bash
# Setup initial
git init && git add . && git commit -m "chore(setup): initial project structure"
git branch -M main

# Feature 1 — infrastructure
git checkout -b chore/docker-and-prisma
# ... Claude Code génère docker-compose + schema Prisma ...
git add . && git commit -m "chore(docker): add postgres docker-compose"
git add . && git commit -m "chore(prisma): init schema with User, Project, Task"
git push -u origin chore/docker-and-prisma
git checkout main

# Feature 2 — backend users
git checkout -b feat/users-crud
# ... Claude Code génère le module users ...
git add . && git commit -m "feat(users): add CRUD endpoints with Prisma"
git push -u origin feat/users-crud
git checkout main

# Feature 3 — backend projects
git checkout -b feat/projects-crud
git add . && git commit -m "feat(projects): add CRUD endpoints with Prisma"
git push -u origin feat/projects-crud
git checkout main

# Feature 4 — frontend users
git checkout -b feat/ui-users
git add . && git commit -m "feat(ui-users): add users page with table and create dialog"
git push -u origin feat/ui-users
git checkout main

# Feature 5 — frontend projects
git checkout -b feat/ui-projects
git add . && git commit -m "feat(ui-projects): add projects page with table and create dialog"
git push -u origin feat/ui-projects
git checkout main
```

---

## Important rules

- **ALWAYS** créer la branche depuis `main` — jamais depuis une autre branche feature
- **ALWAYS** nommer la branche avec le même type que le commit (`feat/`, `fix/`, `chore/`)
- **ALWAYS** pusher la branche après le commit
- **ALWAYS** revenir sur `main` avant de créer la prochaine branche
- **NEVER** merger automatiquement — l'utilisateur décide quand merger
- **NEVER** committer sur `main` directement après le commit initial
- **NEVER** utiliser des messages génériques (`update`, `fix stuff`, `wip`)
- **NEVER** dépasser 50 caractères dans le sujet du commit

## References

- `references/commit-examples.md` — Exemples par type, comparaisons bon/mauvais
