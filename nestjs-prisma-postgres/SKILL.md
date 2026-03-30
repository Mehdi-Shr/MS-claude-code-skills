---
name: nestjs-prisma-postgres
description: Expert guidance for building NestJS REST APIs with Prisma ORM and PostgreSQL. Use when user wants to create a NestJS module, define a Prisma schema, write a service, controller, DTO, set up Docker Postgres, run migrations, or scaffold CRUD endpoints. Uses nest g resource for scaffolding and Prisma for all DB access.
allowed-tools:
  - Bash
  - Read
  - Write
---

# NestJS + Prisma + PostgreSQL

Builds REST APIs with NestJS CLI scaffolding and Prisma as the type-safe ORM.

## Project state

!`cat package.json 2>/dev/null | grep -E '"name"|"nestjs/core"|"prisma"' | head -5 || echo "(no package.json found)"`
!`cat prisma/schema.prisma 2>/dev/null | head -20 || echo "(no schema found)"`

## Bootstrap

> **Prérequis** : Node.js >= 20

```bash
# Créer le projet
npx @nestjs/cli@latest new backend --package-manager npm --skip-git
cd backend

# Dépendances
npm install prisma --save-dev
npm install @prisma/client @prisma/adapter-pg
npm install class-validator class-transformer @nestjs/mapped-types
npx prisma init
```

`.env` :
```
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/sagacify?schema=public"
```

## Docker PostgreSQL

`docker-compose.yml` à la racine du projet :

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: sagacify
    ports:
      - '5432:5432'
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

```bash
# Vérifier que le port 5432 est libre avant de démarrer
lsof -ti:5432 && echo "PORT OCCUPÉ — arrêter le container existant" || echo "Port libre"

docker-compose up -d
```

## main.ts (configuration globale)

```typescript
import 'dotenv/config'; // ← TOUJOURS en premier — charge le .env avant NestJS
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

## PrismaService (singleton global)

```typescript
// src/prisma/prisma.service.ts
import { Injectable } from "@nestjs/common";
import { PrismaClient } from "./generated/prisma/client.js";
import { PrismaPg } from "@prisma/adapter-pg";

@Injectable()
export class PrismaService extends PrismaClient {
  constructor() {
    const adapter = new PrismaPg({
      connectionString: process.env.DATABASE_URL as string,
    });
    super({ adapter });
  }
}
```

```typescript
// src/prisma/prisma.module.ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({ providers: [PrismaService], exports: [PrismaService] })
export class PrismaModule {}
```

Importer `PrismaModule` **une seule fois** dans `AppModule`. Jamais dans les modules features.

## Créer un module CRUD — workflow exact

### Étape 1 — Scaffolding CLI

> **ALWAYS use this — never write files manually**

```bash
# Depuis backend/
npx nest g resource <name> --no-spec
# → choisir : REST API → Yes (CRUD entry points)
```

Génère automatiquement :
```
src/users/
  dto/create-user.dto.ts
  dto/update-user.dto.ts
  entities/user.entity.ts
  users.controller.ts
  users.module.ts
  users.service.ts
```

### Étape 1b — Créer prisma.config.ts (à la racine du projet)

```typescript
import 'dotenv/config';
import { defineConfig } from 'prisma/config';

export default defineConfig({
  schema: 'prisma/schema.prisma',
  migrations: {
    path: 'prisma/migrations',
  },
  datasource: {
    url: process.env['DATABASE_URL'],
  },
});
```

### Étape 2 — Prisma schema

```prisma
generator client {
  provider     = "prisma-client"
  output       = "../src/generated/prisma"
  moduleFormat = "cjs"
}

// En Prisma v7, url est dans prisma.config.ts — plus ici
datasource db {
  provider = "postgresql"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

```bash
npx prisma migrate dev --name add-user
```

### Étape 3 — DTOs (remplacer le contenu généré)

```typescript
// src/users/dto/create-user.dto.ts
import { IsEmail, IsNotEmpty, IsString } from 'class-validator';

export class CreateUserDto {
  @IsEmail() email: string;
  @IsString() @IsNotEmpty() name: string;
}
```

```typescript
// src/users/dto/update-user.dto.ts
import { PartialType } from '@nestjs/mapped-types';
import { CreateUserDto } from './create-user.dto';
export class UpdateUserDto extends PartialType(CreateUserDto) {}
```

### Étape 4 — Service (remplacer le contenu généré)

```typescript
// src/users/users.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { User, Prisma } from '../../generated/prisma/client';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  create(dto: CreateUserDto) {
    return this.prisma.user.create({ data: dto });
  }

  findAll() {
    return this.prisma.user.findMany();
  }

  async findOne(id: number) {
    const user = await this.prisma.user.findUnique({ where: { id } });
    if (!user) throw new NotFoundException(`User #${id} not found`);
    return user;
  }

  async update(id: number, dto: UpdateUserDto) {
    await this.findOne(id);
    return this.prisma.user.update({ where: { id }, data: dto });
  }

  async remove(id: number) {
    await this.findOne(id);
    return this.prisma.user.delete({ where: { id } });
  }
}
```

Le controller généré par `nest g resource` est déjà correct — ne pas le modifier.

## Authentification JWT + Guards

Approche officielle NestJS — **sans Passport**, `AuthGuard` custom avec `jwtService.verifyAsync()`.

### Installation

```bash
npm install @nestjs/jwt bcrypt
npm install --save-dev @types/bcrypt
```

### Scaffolding avec le CLI

```bash
nest g module auth --no-spec
nest g controller auth --no-spec
nest g service auth --no-spec
```

Crée `src/auth/` avec module, controller, service. Ajoute automatiquement `AuthModule` dans `AppModule`.

Créer ensuite manuellement :
- `src/auth/auth.guard.ts`
- `src/auth/public.decorator.ts`

### Structure finale

```
src/auth/
  dto/
    register.dto.ts    ← RegisterDto avec class-validator
    login.dto.ts       ← LoginDto avec class-validator
  auth.guard.ts        ← guard JWT + logique @Public()
  auth.module.ts       ← JwtModule + APP_GUARD global
  auth.controller.ts   ← POST /auth/register, POST /auth/login, GET /auth/profile
  auth.service.ts      ← register + signIn avec Prisma + bcrypt
  public.decorator.ts  ← décorateur @Public()
```

> **Toujours** créer un dossier `dto/` dans le module auth — jamais de types inline `{ email: string; password: string }` dans les paramètres `@Body()` du controller.

### Prisma schema — modèle User

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  password  String
  name      String
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
}
```

```bash
npx prisma migrate dev --name add-user
npx prisma generate
```

### `src/auth/public.decorator.ts`

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

### `src/auth/auth.guard.ts`

```typescript
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { JwtService } from '@nestjs/jwt';
import { Request } from 'express';
import { IS_PUBLIC_KEY } from './public.decorator';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private jwtService: JwtService,
    private reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // 1. Route @Public() ? → laisser passer sans vérifier le token
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    // 2. Extraire et vérifier le token JWT
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) throw new UnauthorizedException();

    try {
      const payload = await this.jwtService.verifyAsync(token);
      request['user'] = payload; // { sub, email, iat, exp }
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

### `src/auth/auth.service.ts`

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { PrismaService } from 'src/prisma/prisma.service';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private prisma: PrismaService,
    private jwtService: JwtService,
  ) {}

  async register(email: string, password: string, name: string) {
    const hashed = await bcrypt.hash(password, 10);
    return this.prisma.user.create({
      data: { email, password: hashed, name },
    });
  }

  async signIn(email: string, pass: string): Promise<{ access_token: string }> {
    const user = await this.prisma.user.findUnique({ where: { email } });
    if (!user || !(await bcrypt.compare(pass, user.password))) {
      throw new UnauthorizedException();
    }
    const payload = { sub: user.id, email: user.email };
    return { access_token: await this.jwtService.signAsync(payload) };
  }
}
```

### `src/auth/dto/register.dto.ts`

```typescript
import { IsEmail, IsNotEmpty, IsString, MinLength } from 'class-validator';

export class RegisterDto {
  @IsEmail() email: string;
  @IsString() @MinLength(6) password: string;
  @IsString() @IsNotEmpty() name: string;
}
```

### `src/auth/dto/login.dto.ts`

```typescript
import { IsEmail, IsString, MinLength } from 'class-validator';

export class LoginDto {
  @IsEmail() email: string;
  @IsString() @MinLength(6) password: string;
}
```

### `src/auth/auth.controller.ts`

```typescript
import { Body, Controller, Get, HttpCode, HttpStatus, Post, Request } from '@nestjs/common';
import { AuthService } from './auth.service';
import { Public } from './public.decorator';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Public()
  @Post('register')
  register(@Body() dto: RegisterDto) {
    return this.authService.register(dto.email, dto.password, dto.name);
  }

  @Public()
  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() dto: LoginDto) {
    return this.authService.signIn(dto.email, dto.password);
  }

  @Get('profile')
  getProfile(@Request() req: any) {
    return req.user; // { sub, email, iat, exp }
  }
}
```

### `src/auth/auth.module.ts` — APP_GUARD global

`APP_GUARD` enregistre le guard **globalement** : NestJS l'applique à toutes les routes sans `@UseGuards()` partout.

```typescript
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { JwtModule } from '@nestjs/jwt';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { AuthGuard } from './auth.guard';

@Module({
  imports: [
    JwtModule.register({
      global: true,
      secret: process.env.JWT_SECRET,
      signOptions: { expiresIn: '7d' },
    }),
  ],
  providers: [
    AuthService,
    {
      provide: APP_GUARD,
      useClass: AuthGuard, // protège toute l'app par défaut
    },
  ],
  controllers: [AuthController],
})
export class AuthModule {}
```

### Protéger/exposer des routes

```typescript
// Route publique — pas de token requis
@Public()
@Get()
findAll() { ... }

// Route protégée — token JWT requis (comportement par défaut)
@Get('profile')
getProfile(@Request() req: any) {
  return req.user; // { sub, email, iat, exp }
}
```

### Utiliser le token dans les requêtes

```
POST /auth/login  →  { "access_token": "eyJ..." }

Authorization: Bearer eyJ...   ← header à envoyer sur les routes protégées
```

### Hashing mot de passe

```typescript
import * as bcrypt from 'bcrypt';

const hash = await bcrypt.hash(plainText, 10);        // à la création
const match = await bcrypt.compare(plainText, hash);  // à la connexion
```

## Health Check (Terminus)

**Toujours créer un endpoint `/api/health` avec `@nestjs/terminus` — fait pro et utile en prod/exam.**

### Installation

```bash
npm install @nestjs/terminus
nest g module health --no-spec
nest g controller health --no-spec
```

### `src/health/health.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule],
  controllers: [HealthController],
})
export class HealthModule {}
```

### `src/health/health.controller.ts`

Utilise `PrismaHealthIndicator` pour vérifier que la DB est up :

```typescript
import { Controller, Get } from '@nestjs/common';
import { HealthCheck, HealthCheckService, PrismaHealthIndicator } from '@nestjs/terminus';
import { PrismaService } from '../prisma/prisma.service';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private prismaHealth: PrismaHealthIndicator,
    private prisma: PrismaService,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.prismaHealth.pingCheck('database', this.prisma),
    ]);
  }
}
```

### Réponse attendue — `GET /api/health`

```json
{
  "status": "ok",
  "info": { "database": { "status": "up" } },
  "error": {},
  "details": { "database": { "status": "up" } }
}
```

Si la DB est down → HTTP 503 automatiquement.

## Commandes essentielles

```bash
npx nest g resource <name> --no-spec   # scaffolding CRUD complet
npx prisma migrate dev --name <name>   # nouvelle migration
npx prisma migrate reset               # reset DB
npx prisma generate                    # regénérer le client après schema change
npx prisma studio                      # UI visuelle DB
npm run start:dev                      # hot-reload
```

## Important rules

- **ALWAYS** utiliser `npx nest g resource --no-spec` pour scaffolder — jamais à la main
- **ALWAYS** déclarer `PrismaModule` comme `@Global()` et l'importer uniquement dans `AppModule`
- **ALWAYS** appeler `await this.findOne(id)` avant update/delete pour avoir un 404 propre
- **ALWAYS** utiliser `PartialType` de `@nestjs/mapped-types` pour `UpdateDto`
- **NEVER** injecter `PrismaClient` directement — toujours via `PrismaService`
- **NEVER** mettre de logique métier dans les controllers — services uniquement
- **ALWAYS** inclure `generator client` avec `output`, `moduleFormat = "cjs"` dans le schema — requis pour NestJS (CommonJS)
- **ALWAYS** importer les types Prisma depuis `generated/prisma/client` — jamais depuis `@prisma/client`
- **NEVER** skip `npx prisma generate` après un changement de schema
- **ALWAYS** scaffolder auth avec `nest g module/controller/service auth --no-spec` — jamais à la main
- **ALWAYS** créer un dossier `dto/` dans `src/auth/` avec `RegisterDto` et `LoginDto` — jamais de types inline `{ email: string; ... }` dans `@Body()`
- **ALWAYS** mettre `APP_GUARD` dans `AuthModule` (pas `AppModule`) avec `useClass: AuthGuard`
- **ALWAYS** marquer les routes publiques avec `@Public()` — jamais retirer le guard
- **NEVER** importer `AuthModule` dans `AppModule` manuellement — le CLI le fait automatiquement
- **ALWAYS** mettre `auth.guard.ts` et `public.decorator.ts` à plat dans `src/auth/` — pas de sous-dossiers
- **NEVER** stocker un mot de passe en clair — toujours `bcrypt.hash(password, 10)`
- **NEVER** mettre `JWT_SECRET` en dur dans le code — toujours via `.env` + `ConfigService`
- **ALWAYS** mettre `import 'dotenv/config'` comme **toute première ligne** de `main.ts` — sinon `DATABASE_URL` est undefined au démarrage → crash SASL
- **NEVER** utiliser `ts-node` pour les scripts seed — toujours `tsx` (ts-node ne résout pas les `.js` dans les fichiers générés avec `moduleFormat = "cjs"`)
- **NEVER** créer un seed (`prisma/seed.ts`) sans que l'utilisateur le demande explicitement
- **ALWAYS** créer un endpoint `/health` avec `@nestjs/terminus` + `PrismaHealthIndicator` dans tout nouveau projet

## Seed Prisma

Installer `tsx` pour exécuter le seed (ts-node est incompatible avec `moduleFormat = "cjs"`) :

```bash
npm install tsx --save-dev
```

Ajouter dans `package.json` :
```json
"scripts": {
  "seed": "tsx prisma/seed.ts"
},
"prisma": {
  "seed": "npm run seed"
}
```

`prisma/seed.ts` :
```typescript
import { PrismaClient } from '../src/generated/prisma/client';
import { PrismaPg } from '@prisma/adapter-pg';
import 'dotenv/config';

const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL as string });
const prisma = new PrismaClient({ adapter });

async function main() {
  // ... données
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

```bash
npm run seed
```

> **NEVER** utiliser `ts-node` pour le seed — utiliser `tsx` uniquement.

## Troubleshooting

### `SASL: client password must be a string` au démarrage

```
Error: SASL: SCRAM-SERVER-FIRST-MESSAGE: client password must be a string
```

**Cause** : `dotenv/config` n'est pas importé en premier dans `main.ts` — `DATABASE_URL` est `undefined` quand PrismaService s'instancie.
**Fix** : ajouter `import 'dotenv/config'` comme **toute première ligne** de `main.ts`, avant tout autre import.

```typescript
import 'dotenv/config'; // ← DOIT être la première ligne
import { NestFactory } from '@nestjs/core';
// ...
```

### `Cannot find module './internal/class.js'` dans le seed

```
Error: Cannot find module './internal/class.js'
Require stack: [.../generated/prisma/client.ts, .../seed.ts]
```

**Cause** : `ts-node` ne résout pas les imports `.js` dans les fichiers TypeScript générés par Prisma avec `moduleFormat = "cjs"`.
**Fix** : utiliser `tsx` à la place de `ts-node` pour exécuter le seed.

```bash
npm install tsx --save-dev
# package.json → "seed": "tsx prisma/seed.ts"
```

### `exports is not defined` au démarrage

```
ReferenceError: exports is not defined
    at file:///...dist/src/generated/prisma/client.js
```

**Cause** : le client Prisma est généré en ESM mais NestJS tourne en CommonJS.
**Fix** : ajouter `moduleFormat = "cjs"` dans le generator, puis `npx prisma generate`.

```prisma
generator client {
  provider     = "prisma-client"
  output       = "../src/generated/prisma"
  moduleFormat = "cjs"   # ← obligatoire avec NestJS
}
```

### `Cannot find module '../src/generated/prisma'`

**Cause** : le client n'a pas encore été généré, ou l'import pointe vers le dossier au lieu du fichier.
**Fix** : `npx prisma generate` puis importer depuis `../src/generated/prisma/client`.

## References

- `references/prisma-patterns.md` — Relations, pagination, upsert, transactions
