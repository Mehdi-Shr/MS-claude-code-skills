---
name: react-router-shadcn
description: Expert guidance for building React UIs with React Router v7, shadcn/ui MCP, and fetch natif. Use when user wants to créer une page, ajouter un formulaire, afficher une liste, naviguer entre pages, appeler l'API NestJS, ou composer des composants shadcn. Pas de TanStack Query, pas de zod — fetch natif et useState uniquement.
allowed-tools:
  - Bash
  - Read
  - Write
---

# React Router v7 + shadcn/ui

Builds React SPAs with React Router v7 for routing, shadcn/ui MCP for components, and native fetch for API calls.

## Project state

!`cat package.json 2>/dev/null | grep -E '"name"|"react-router"|"shadcn"' | head -5 || echo "(no package.json found)"`
!`ls app/components/ui 2>/dev/null | head -10 || echo "(no ui components yet)"`

## Bootstrap

```bash
# Créer le projet
npx create-react-router@latest frontend
cd frontend
npm install
```

**Initialiser shadcn/ui — préférer le MCP :**

```
use shadcn to init the project
```

**Fallback CLI (si MCP non disponible) :**

```bash
npx shadcn@latest init -t react-router
```

Puis pour installer les composants de base via MCP :

```
use shadcn to add button, table, dialog, input, label, card, sidebar, separator, avatar
```

Ou via CLI :

```bash
npx shadcn@latest add button table dialog input label card sidebar separator avatar
```

## Structure React Router v7

```
app/
  routes/
    home.tsx          ← page d'accueil
    users.tsx         ← liste des users
    users.$id.tsx     ← détail d'un user
  components/
    ui/               ← shadcn — NEVER modify
    features/         ← composants métier
  lib/
    api.ts            ← fetch helpers
  root.tsx            ← layout global + navbar
```

## API layer (fetch natif — pas de librairie)

```typescript
// app/lib/api.ts
const BASE_URL = 'http://localhost:3001/api';

async function request<T>(endpoint: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${BASE_URL}${endpoint}`, {
    headers: { 'Content-Type': 'application/json' },
    ...init,
  });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}

export const api = {
  get:    <T>(url: string)               => request<T>(url),
  post:   <T>(url: string, body: unknown) => request<T>(url, { method: 'POST',   body: JSON.stringify(body) }),
  patch:  <T>(url: string, body: unknown) => request<T>(url, { method: 'PATCH',  body: JSON.stringify(body) }),
  delete: <T>(url: string)               => request<T>(url, { method: 'DELETE' }),
};
```

## Pattern page liste + création (useState + fetch)

```typescript
// app/routes/users.tsx
import { useState, useEffect } from 'react';
import { api } from '~/lib/api';

interface User { id: number; name: string; email: string; }

export default function UsersPage() {
  const [users, setUsers]     = useState<User[]>([]);
  const [loading, setLoading] = useState(true);
  const [name, setName]       = useState('');
  const [email, setEmail]     = useState('');

  const fetchUsers = async () => {
    setLoading(true);
    const data = await api.get<User[]>('/users');
    setUsers(data);
    setLoading(false);
  };

  useEffect(() => { fetchUsers(); }, []);

  const handleCreate = async (e: React.FormEvent) => {
    e.preventDefault();
    await api.post('/users', { name, email });
    setName(''); setEmail('');
    fetchUsers();
  };

  const handleDelete = async (id: number) => {
    await api.delete(`/users/${id}`);
    fetchUsers();
  };

  if (loading) return <p>Chargement...</p>;

  return (
    <div className="space-y-6">
      <form onSubmit={handleCreate} className="flex gap-2">
        <input value={name}  onChange={e => setName(e.target.value)}  placeholder="Nom"   className="border px-2 py-1 rounded" />
        <input value={email} onChange={e => setEmail(e.target.value)} placeholder="Email" className="border px-2 py-1 rounded" />
        <button type="submit" className="bg-primary text-white px-4 py-1 rounded">Créer</button>
      </form>
      <ul className="space-y-2">
        {users.map(u => (
          <li key={u.id} className="flex justify-between border p-2 rounded">
            <span>{u.name} — {u.email}</span>
            <button onClick={() => handleDelete(u.id)} className="text-red-500">Supprimer</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## Pattern avec composants shadcn (via MCP)

Demander au MCP shadcn les composants exacts :

```
use shadcn and implement a Table with Button to delete each row,
and a Dialog with a form to create a new User (name + email fields)
```

Le MCP installe les composants et génère le code adapté à ton projet.

## Routing React Router v7

```typescript
// app/root.tsx — navbar avec liens
import { Link, Outlet } from 'react-router';

export default function Root() {
  return (
    <div className="min-h-screen">
      <nav className="border-b px-4 py-3 flex gap-4">
        <Link to="/" className="font-bold">Home</Link>
        <Link to="/users">Users</Link>
        <Link to="/projects">Projects</Link>
        <Link to="/tasks">Tasks</Link>
      </nav>
      <main className="container mx-auto px-4 py-8">
        <Outlet />
      </main>
    </div>
  );
}
```

```typescript
// app/routes.ts — déclarer les routes
import { type RouteConfig, index, route } from '@react-router/dev/routes';

export default [
  index('routes/home.tsx'),
  route('users',    'routes/users.tsx'),
  route('projects', 'routes/projects.tsx'),
  route('tasks',    'routes/tasks.tsx'),
] satisfies RouteConfig;
```

## Important rules

- **BY DEFAULT** utiliser fetch natif + useState — ne pas installer TanStack Query, zod, ou react-hook-form sauf si l'utilisateur le demande explicitement
- **NEVER** modifier les fichiers dans `app/components/ui/` — utiliser le MCP shadcn pour les composants
- **PREFER** le MCP shadcn via le chat (`use shadcn to init` / `use shadcn to add ...`) — fallback CLI : `npx shadcn@latest init -t react-router` et `npx shadcn@latest add <component>`
- **ALWAYS** appeler `fetchData()` après chaque create/update/delete pour rafraîchir la liste
- **ALWAYS** gérer `e.preventDefault()` sur les formulaires
- **ALWAYS** déclarer les routes dans `app/routes.ts`
- **NEVER** faire de fetch directement dans les composants — toujours via `app/lib/api.ts`

## References

- [`references/component-catalog.md`](references/component-catalog.md) — Composants shadcn utiles avec prompts MCP
