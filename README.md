# kipo-docs

Documentación técnica de software de Kipo. Cubre setup local, variables de entorno, integraciones (Stripe, Supabase), feature gating, y guías de desarrollo.

---

## Setup en Obsidian (sin terminal)

Todo ocurre dentro de Obsidian. No se necesita correr `git clone` manualmente.

### Prerequisitos

- [Obsidian](https://obsidian.md/) instalado
- [Git](https://git-scm.com/downloads) instalado en el sistema (solo el binario, no se usa la terminal)
- Token de GitHub con permiso `repo` → [generarlo aquí](https://github.com/settings/tokens)

### Paso 1 — Instalar Obsidian Git

En Obsidian:

`Settings` → `Community plugins` → desactiva **Safe mode** → `Browse` → busca **Obsidian Git** → instala y activa.

### Paso 2 — Clonar el repo desde Obsidian

1. Crea una carpeta vacía en tu computadora donde vivirá el vault (ej. `~/Documents/kipo-docs`).
2. En Obsidian: `Open folder as vault` → selecciona esa carpeta vacía.
3. Abre el command palette: `Cmd+P` (Mac) / `Ctrl+P` (Windows/Linux).
4. Ejecuta: **`Obsidian Git: Clone an existing remote repo`**.
5. Pega la URL del repo:
   ```
   https://<TU_TOKEN>@github.com/CodeZenNirvana/kipo-docs.git
   ```
   > Reemplaza `<TU_TOKEN>` con el token de GitHub generado en el prerequisito.
6. Cuando pregunte el directorio destino, deja `.` (punto) para clonar en la carpeta actual.
7. Obsidian recarga el vault automáticamente con todos los archivos y la config de `.obsidian/`.

### Paso 3 — Configurar auto-sync

En `Settings` → `Obsidian Git`:

| Setting | Valor sugerido |
|---|---|
| Vault backup interval (minutes) | `10` |
| Auto pull interval (minutes) | `10` |
| Commit message | `docs: {{date}}` |
| Pull updates on startup | activado |

### Usar

- **Pull manual**: `Cmd+P` → `Obsidian Git: Pull`
- **Commit + Push manual**: `Cmd+P` → `Obsidian Git: Commit all changes` → `Obsidian Git: Push`
- **Auto**: si configuraste el intervalo, Obsidian Git hace commit y push automáticamente cada N minutos.

---

## Estructura

```
kipo-docs/
├── docs/
│   ├── environment-variables.md
│   ├── local-development.md
│   ├── environments-setup.md
│   ├── Stripe/
│   └── Manejo de subscripciones y permisos/
└── .obsidian/          # Config del vault (no editar manualmente)
```
