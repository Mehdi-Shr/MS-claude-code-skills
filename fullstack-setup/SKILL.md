---
name: fullstack-setup
description: Guide d'installation pas-à-pas du stack complet NestJS + React Router v7 + shadcn/ui + Prisma + PostgreSQL (Docker). À utiliser quand l'utilisateur veut démarrer un nouveau projet full stack depuis zéro. Ce skill couvre uniquement le bootstrap — utiliser nestjs-prisma-postgres et react-router-shadcn pour coder ensuite.
allowed-tools:
  - Bash
  - Read
  - Write
---

# Installation Full Stack — NestJS + React Router + shadcn/ui + Prisma + PostgreSQL

Ce skill installe l'environnement complet. Une fois terminé, utiliser :
- **`nestjs-prisma-postgres`** pour créer des modules, services, migrations Prisma
- **`react-router-shadcn`** pour créer des pages, formulaires, composants UI

## Prérequis (à vérifier à la main)

- [ ] Node.js >= 20 (`node -v`)
- [ ] Docker Desktop en cours d'exécution (`docker info`)
- [ ] npm >= 10 (`npm -v`)

---

## Étape 1 — Créer la structure du projet

```bash
mkdir <nom-projet>
cd <nom-projet>
```

Structure finale visée :

```
<nom-projet>/
  backend/          ← NestJS API
  frontend/         ← React Router + shadcn
  docker-compose.yml
```

---

## Étape 2 — PostgreSQL via Docker

Créer `docker-compose.yml` à la racine :

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: <nom-projet>
    ports:
      - '5432:5432'
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

```bash
# Vérifier que le port 5432 est libre
lsof -ti:5432 && echo "PORT OCCUPÉ" || echo "Port libre"

# Démarrer la base de données
docker-compose up -d

# Vérifier que le container tourne
docker ps
```

---

## Étape 3 — Backend NestJS

```bash
npx @nestjs/cli@latest new backend --package-manager npm --skip-git
cd backend

# Dépendances Prisma + validation
npm install prisma --save-dev
npm install @prisma/client @prisma/adapter-pg
npm install class-validator class-transformer @nestjs/mapped-types

# Initialiser Prisma
npx prisma init
```

### `.env` dans `backend/`

```
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/<nom-projet>?schema=public"
JWT_SECRET="change-this-secret-in-production"
```

### `backend/prisma/schema.prisma`

```prisma
generator client {
  provider     = "prisma-client"
  output       = "../src/generated/prisma"
  moduleFormat = "cjs"
}

datasource db {
  provider = "postgresql"
}
```

### `backend/prisma.config.ts`

```typescript
import path from 'node:path';
import { defineConfig, env } from 'prisma/config';
import 'dotenv/config';

export default defineConfig({
  schema: path.join('prisma', 'schema.prisma'),
  datasource: {
    url: env('DATABASE_URL'),
  },
});
```

### `backend/src/main.ts`

```typescript
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.setGlobalPrefix('api');
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
  app.enableCors();
  await app.listen(3001);
}
bootstrap();
```

### Créer PrismaService + PrismaModule

Créer `backend/src/prisma/prisma.service.ts` :

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaClient } from '../generated/prisma';
import { PrismaPg } from '@prisma/adapter-pg';

@Injectable()
export class PrismaService extends PrismaClient {
  constructor() {
    const adapter = new PrismaPg({ url: process.env.DATABASE_URL });
    super({ adapter });
  }
}
```

Créer `backend/src/prisma/prisma.module.ts` :

```typescript
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({ providers: [PrismaService], exports: [PrismaService] })
export class PrismaModule {}
```

### `backend/src/app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { PrismaModule } from './prisma/prisma.module';

@Module({
  imports: [PrismaModule],
})
export class AppModule {}
```

### Première migration

```bash
# Depuis backend/
npx prisma migrate dev --name init

# Vérifier que le client est généré
ls src/generated/prisma
```

### Lancer le backend

```bash
npm run start:dev
# → http://localhost:3001/api
```

---

## Étape 4 — Frontend React Router

```bash
# Depuis la racine du projet
npx create-react-router@latest frontend
cd frontend
npm install
```

### Initialiser shadcn/ui

**Via MCP (préféré) :**
```
use shadcn to init the project
```

**Via CLI (fallback) :**
```bash
npx shadcn@latest init -t react-router
```

### Installer les composants de base

**Via MCP :**
```
use shadcn to add button table dialog input label card sidebar separator avatar badge
```

**Via CLI :**
```bash
npx shadcn@latest add button table dialog input label card sidebar separator avatar badge
```

### Lancer le frontend

```bash
npm run dev
# → http://localhost:5173
```

---

## Étape 5 — Vérification

| Service    | URL                        | Attendu                  |
|------------|----------------------------|--------------------------|
| Backend    | http://localhost:3001/api  | `{ "message": "..." }`   |
| Frontend   | http://localhost:5173      | Page React Router        |
| PostgreSQL | localhost:5432             | Docker container actif   |

```bash
# Test rapide de l'API
curl http://localhost:3001/api
```

---

## Structure finale

```
<nom-projet>/
  docker-compose.yml
  backend/
    prisma/
      schema.prisma
    prisma.config.ts
    src/
      generated/prisma/    ← généré par Prisma, ne pas modifier
      prisma/
        prisma.service.ts
        prisma.module.ts
      app.module.ts
      main.ts
    .env
  frontend/
    app/
      routes/
        home.tsx
      components/
        ui/                ← shadcn, ne jamais modifier
      lib/
        api.ts             ← à créer pour les appels NestJS
      root.tsx
      routes.ts
```

---

## Après l'installation — utiliser les skills

| Tâche                              | Skill à utiliser         |
|------------------------------------|--------------------------|
| Créer un module CRUD (NestJS)      | `nestjs-prisma-postgres` |
| Ajouter un modèle Prisma           | `nestjs-prisma-postgres` |
| Créer une page / formulaire React  | `react-router-shadcn`    |
| Ajouter un composant shadcn        | `react-router-shadcn`    |
| Brancher le frontend sur l'API     | `react-router-shadcn`    |
| Committer / brancher               | `git-workflow`           |

---

## Commandes de référence rapide

```bash
# Backend
npm run start:dev                        # démarrer NestJS
npx nest g resource <name> --no-spec    # nouveau module CRUD
npx prisma migrate dev --name <name>    # nouvelle migration
npx prisma studio                        # UI base de données

# Frontend
npm run dev                              # démarrer React Router
npx shadcn@latest add <component>       # ajouter composant (fallback CLI)

# Docker
docker-compose up -d                    # démarrer PostgreSQL
docker-compose down                     # arrêter PostgreSQL
docker-compose logs postgres            # logs de la DB
```
