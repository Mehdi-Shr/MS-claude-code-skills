# Catalogue des composants shadcn — Prompts MCP

## Installation groupée (départ projet)

```
use shadcn to add button table dialog input label card sidebar separator avatar badge
```

## Composants par usage

### Tableaux de données
```
use shadcn and implement a Table with columns [col1, col2, ...] and a Button to delete each row
```
Composants installés : `table`, `button`

### Formulaire dans Dialog (modale)
```
use shadcn and implement a Dialog with a form inside containing [champs...] and a Submit button
```
Composants installés : `dialog`, `input`, `label`, `button`

### Formulaire inline (page)
```
use shadcn and implement a form with fields [champs...] using Input, Label and a Submit Button
```
Composants installés : `input`, `label`, `button`

### Cartes
```
use shadcn and implement a Card with title, description and a footer with action buttons
```
Composants installés : `card`, `button`

### Navigation latérale
```
use shadcn and implement a Sidebar with navigation links [link1, link2, ...]
```
Composants installés : `sidebar`, `separator`

### Notifications toast
```
use shadcn and implement a Sonner toast notification for success and error states
```
Composants installés : `sonner`

### Sélecteur / Combobox
```
use shadcn and implement a Select dropdown with options [opt1, opt2, ...]
```
Composants installés : `select`

### Confirmation de suppression
```
use shadcn and implement an AlertDialog to confirm deletion before calling the delete handler
```
Composants installés : `alert-dialog`, `button`

### Badges de statut
```
use shadcn and implement Badge components with variants: default, secondary, destructive, outline
```
Composants installés : `badge`

### Skeleton de chargement
```
use shadcn and implement Skeleton placeholders while data is loading
```
Composants installés : `skeleton`

## Règle d'or

Ne jamais écrire les composants à la main. Toujours passer par le MCP ou le CLI :

```bash
npx shadcn@latest add <component>
```

Les fichiers dans `app/components/ui/` sont gérés par shadcn — ne pas les modifier.
