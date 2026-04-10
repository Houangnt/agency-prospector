# Mode: pipeline — Prospect URL Inbox (Second Brain)

Process prospect/profile URLs accumulated in `data/pipeline.md`. The user can paste URLs anytime, then run `/investor-ops pipeline` to process them in batch.

## Workflow

1. **Read** `data/pipeline.md` → find `- [ ]` items in "Pending"
2. **Para cada URL pendiente**:
   a. Calcular siguiente `REPORT_NUM` secuencial (leer `reports/`, tomar el número más alto + 1)
   b. **Extract prospect context** using Playwright (browser_navigate + browser_snapshot) → WebFetch → WebSearch
   c. Si la URL no es accesible → marcar como `- [!]` con nota y continuar
   d. **Run auto-pipeline**: Eval (Blocks A–E) → Report .md → Tracker line (TSV) → merge tracker
   e. **Move from "Pending" to "Processed"**: `- [x] #NNN | URL | Name | Title at Company | Grade | Email ✅/❌`
3. **Si hay 3+ URLs pendientes**, lanzar agentes en paralelo (Agent tool con `run_in_background`) para maximizar velocidad.
4. **Al terminar**, mostrar tabla resumen:

```
| # | Name | Lead | Grade | Email | Recommended action |
```

## `data/pipeline.md` format

```markdown
## Pending
- [ ] https://jobs.example.com/posting/123
- [ ] https://www.linkedin.com/in/someone | Name | Title at Company | signal: "snippet..." | score: 11/15
- [!] https://private.url/profile — Error: login required

## Processed
- [x] #143 | https://www.linkedin.com/in/someone | Name | Title at Company | Grade A | Email ✅
- [x] #144 | https://x.com/handle/status/123 | Handle | (Tweet signal) | Grade C | Email ❌
```

## Smart extraction from URL

1. **Playwright (preferred):** `browser_navigate` + `browser_snapshot` to read profile/post content.
2. **WebFetch (fallback):** for static pages or when Playwright isn't available.
3. **WebSearch (last resort):** for lightweight context and links.

**Special cases:**
- **LinkedIn**: may require login → mark `[!]` and ask the user to paste visible text/snippets
- **`local:` prefix**: read local snapshot. Example: `local:jds/prospect-foo.md` → read `jds/prospect-foo.md`

## Numeración automática

1. Listar todos los archivos en `reports/`
2. Extraer el número del prefijo (e.g., `142-medispend...` → 142)
3. Nuevo número = máximo encontrado + 1

## Source sync check

Antes de procesar cualquier URL, verificar sync:
```bash
node cv-sync-check.mjs
```
Si hay desincronización, advertir al usuario antes de continuar.
