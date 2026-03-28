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
import path from 'node:path';
import { defineConfig, env } from 'prisma/config';
import 'dotenv/config';

export default defineConfig({
  schema: path.join('prisma', 'schema.prisma'),
  datasource: {
    url: env('DATABASE_URL'), // lit DATABASE_URL depuis .env
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
import { User, Prisma } from '../../generated/prisma';
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

### Structure

```
src/auth/
  guards/
    auth.guard.ts
  decorators/
    public.decorator.ts
    current-user.decorator.ts
    roles.decorator.ts
  auth.module.ts
  auth.service.ts
  auth.controller.ts
```

### `src/auth/decorators/public.decorator.ts`

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

### `src/auth/decorators/current-user.decorator.ts`

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

### `src/auth/decorators/roles.decorator.ts`

```typescript
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

### `src/auth/guards/auth.guard.ts`

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
import { IS_PUBLIC_KEY } from '../decorators/public.decorator';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private jwtService: JwtService,
    private reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) throw new UnauthorizedException();

    try {
      const payload = await this.jwtService.verifyAsync(token);
      request['user'] = payload;
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
import { PrismaService } from '../prisma/prisma.service';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private prisma: PrismaService,
    private jwtService: JwtService,
  ) {}

  async signIn(email: string, password: string): Promise<{ access_token: string }> {
    const user = await this.prisma.user.findUnique({ where: { email } });
    if (!user || !(await bcrypt.compare(password, user.password))) {
      throw new UnauthorizedException();
    }
    const { password: _, ...result } = user;
    const payload = { sub: user.id, email: user.email };
    return { access_token: await this.jwtService.signAsync(payload) };
  }

  async register(email: string, password: string, name: string) {
    const hashed = await bcrypt.hash(password, 10);
    return this.prisma.user.create({
      data: { email, password: hashed, name },
    });
  }
}
```

### `src/auth/auth.controller.ts`

```typescript
import { Body, Controller, Get, HttpCode, HttpStatus, Post, Request } from '@nestjs/common';
import { AuthService } from './auth.service';
import { Public } from './decorators/public.decorator';
import { CurrentUser } from './decorators/current-user.decorator';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Public()
  @Post('register')
  register(@Body() body: { email: string; password: string; name: string }) {
    return this.authService.register(body.email, body.password, body.name);
  }

  @Public()
  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() body: { email: string; password: string }) {
    return this.authService.signIn(body.email, body.password);
  }

  @Get('me')
  getMe(@CurrentUser() user) {
    return user; // { sub, email, iat, exp }
  }
}
```

### `src/auth/auth.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtModule } from '@nestjs/jwt';

@Module({
  imports: [
    JwtModule.register({
      global: true,
      secret: process.env.JWT_SECRET,
      signOptions: { expiresIn: '7d' },
    }),
  ],
  providers: [AuthService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
```

### `src/app.module.ts` — guard global

```typescript
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { AuthModule } from './auth/auth.module';
import { PrismaModule } from './prisma/prisma.module';
import { AuthGuard } from './auth/guards/auth.guard';

@Module({
  imports: [
    PrismaModule,
    AuthModule,
    // ... autres modules
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: AuthGuard, // toutes les routes protégées par défaut
    },
  ],
})
export class AppModule {}
```

### Utilisation

```typescript
// Route publique — pas de token requis
@Public()
@Get('public')
publicRoute() { ... }

// Route protégée — token JWT requis (comportement par défaut)
@Get('profile')
getProfile(@CurrentUser() user) {
  return user; // { sub, email, iat, exp }
}

// Route avec rôle
@Roles(['admin'])
@Get('admin')
adminOnly() { ... }
```

### Hashing mot de passe

```typescript
import * as bcrypt from 'bcrypt';

const hash = await bcrypt.hash(plainText, 10);        // à la création
const match = await bcrypt.compare(plainText, hash);  // à la connexion
```

### Prisma schema — ajouter le champ password

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  password  String
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

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
- **ALWAYS** importer les types Prisma depuis `generated/prisma` — jamais depuis `@prisma/client`
- **NEVER** skip `npx prisma generate` après un changement de schema
- **ALWAYS** utiliser `APP_GUARD` + `JwtAuthGuard` global — pas de `@UseGuards` partout
- **ALWAYS** marquer les routes publiques avec `@Public()` — jamais retirer le guard
- **NEVER** stocker un mot de passe en clair — toujours `bcrypt.hash(password, 10)`
- **NEVER** mettre `JWT_SECRET` en dur dans le code — toujours via `.env` + `ConfigService`

## References

- `references/prisma-patterns.md` — Relations, pagination, upsert, transactions
