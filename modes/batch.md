# Mode: batch — Bulk Lead Processing

Two usage modes: **conductor --chrome** (browser-driven collection + orchestration) or **standalone** (script processes already-collected prospect URLs).

## Arquitectura

```
Claude Conductor (claude --chrome --dangerously-skip-permissions)
  │
  │  Chrome: navigates LinkedIn/Twitter/other sources (logged-in sessions)
  │  Reads DOM directly — user can watch in real time
  │
  ├─ Lead 1: reads profile/post + URL
  │    └─► claude -p worker → eval report .md + tracker-line
  │
  ├─ Lead 2: next profile/post
  │    └─► claude -p worker → eval report .md + tracker-line
  │
  └─ End: merge tracker-additions → leads.md + summary
```

Cada worker es un `claude -p` hijo con contexto limpio de 200K tokens. El conductor solo orquesta.

## Archivos

```
batch/
  batch-input.tsv               # URLs (por conductor o manual)
  batch-state.tsv               # Progreso (auto-generado, gitignored)
  batch-runner.sh               # Script orquestador standalone
  batch-prompt.md               # Prompt template para workers
  logs/                         # One log per lead (gitignored)
  tracker-additions/            # Tracker lines (gitignored)
```

## Modo A: Conductor --chrome

1. **Leer estado**: `batch/batch-state.tsv` → saber qué ya se procesó
2. **Navegar portal**: Chrome → URL de búsqueda
3. **Extraer URLs**: Leer DOM de resultados → extraer lista de URLs → append a `batch-input.tsv`
4. **For each pending URL**:
   a. Chrome: open profile/post → read visible text/context from DOM
   b. Save snapshot to `/tmp/batch-lead-{id}.txt`
   c. Compute next REPORT_NUM sequentially
   d. Run via Bash:
      ```bash
      claude -p --dangerously-skip-permissions \
        --append-system-prompt-file batch/batch-prompt.md \
        "Process this lead. URL: {url}. Snapshot: /tmp/batch-lead-{id}.txt. Report: {num}. ID: {id}"
      ```
   e. Actualizar `batch-state.tsv` (completed/failed + score + report_num)
   f. Log a `logs/{report_num}-{id}.log`
   g. Chrome: back → next lead
5. **Pagination**: if no more results → Next → repeat
6. **End**: merge `tracker-additions/` → `data/leads.md` + summary

## Modo B: Script standalone

```bash
batch/batch-runner.sh [OPTIONS]
```

Opciones:
- `--dry-run` — lista pendientes sin ejecutar
- `--retry-failed` — solo reintenta fallidas
- `--start-from N` — empieza desde ID N
- `--parallel N` — N workers en paralelo
- `--max-retries N` — intentos por oferta (default: 2)

## Formato batch-state.tsv

```
id	url	status	started_at	completed_at	report_num	score	error	retries
1	https://...	completed	2026-...	2026-...	002	4.2	-	0
2	https://...	failed	2026-...	2026-...	-	-	Error msg	1
3	https://...	pending	-	-	-	-	-	0
```

## Resumabilidad

- Si muere → re-ejecutar → lee `batch-state.tsv` → skip completadas
- Lock file (`batch-runner.pid`) previene ejecución doble
- Cada worker es independiente: fallo en oferta #47 no afecta a las demás

## Workers (claude -p)

Cada worker recibe `batch-prompt.md` como system prompt. Es self-contained.

El worker produce:
1. Report `.md` en `reports/`
2. PDF en `output/`
3. Línea de tracker en `batch/tracker-additions/{id}.tsv`
4. JSON de resultado por stdout

## Error handling

| Error | Recovery |
|-------|----------|
| URL inaccessible | Worker fails → conductor marks `failed`, continue |
| Profile behind login | Conductor tries DOM read. If fails → `failed` |
| Portal cambia layout | Conductor razona sobre HTML, se adapta |
| Worker crashes | Conductor marks `failed`, continue. Retry with `--retry-failed` |
| Conductor muere | Re-ejecutar → lee state → skip completadas |
| Email draft fails | Report .md saved. Tracker marked accordingly |
