> **Protocole** — Avant toute tâche, lire [`../CLAUDE.md`](../CLAUDE.md) §Protocole de recherche.
> Commandes obligatoires : `Read <dossier>/CLAUDE.md` → `Grep "CLAUDE:SUMMARY"` → `Grep "CLAUDE:WARN" <fichier>`.
> **Interdit** : Bash(grep/cat/find) au lieu de Grep/Read. Ne jamais lire un fichier entier en première intention.

# CLAUDE.md — claude-vault

## Responsabilité

MCP stdio server pour analytics de sessions Claude Code. Ingère les JSONL de session (~/.claude/projects/**/*.jsonl) dans SQLite JSONB, les rend queryables par SQL, et produit des rapports structurés.

## Dépendances

- `modernc.org/sqlite` — SQLite sans CGO

## Dépendants

- Claude Code (via `.mcp.json` — serveur MCP stdio)
- Candidature NLnet (sessions filtrées/redactées)

## Fichiers clés

| Fichier | Rôle |
|---------|------|
| `cmd/claude-vault/main.go` | Server, schema SQLite, ingest JSONL, watermark, deriveProject |
| `cmd/claude-vault/mcp.go` | Serveur MCP stdio JSON-RPC 2.0, 6 outils analytics |

## Types clés

| Type | Rôle |
|------|------|
| `Server` | DB pool + claudeDir. Créé via `NewServer`. |

## Outils MCP (6)

| Outil | Usage |
|-------|-------|
| `sessions_ingest` | Scanne les JSONL, ingère incrémentalement (watermark + dédup SHA256) |
| `sessions_query` | SELECT SQL arbitrary (lecture seule, INSERT/DROP/ATTACH bloqués) |
| `sessions_stats` | Agrégations pré-cuites par projet (sessions, events, période) |
| `sessions_report` | Rapport Markdown des conversations, redaction optionnelle |
| `sessions_tag` | Tague les sessions par namespace (déduit des paths tool_use) |
| `sessions_filter` | Filtre sessions par tags include/exclude (défaut NLnet) |

## Build / test

```bash
GOWORK=off CGO_ENABLED=0 go build -o bin/claude-vault ./cmd/claude-vault/
GOWORK=off CGO_ENABLED=0 go test ./...
```

DB : `~/.claude/sessions.db` (ou `CLAUDE_VAULT_DB`).
Mode MCP : `./bin/claude-vault mcp`.
Mode ingest direct : `./bin/claude-vault` (sans args).

## Invariants et pièges connus

- **MaxOpenConns(1)** — obligatoire pour pragmas DSN per-connection
- **INSERT OR IGNORE + line_hash SHA256** — dédup cross-fichiers (symlinks, copies)
- **SELECT-only** — validation prefix `strings.HasPrefix(upper, "SELECT")`. PRAGMA table_info accessible (lecture seule, accepté)
- **Watermark** — offset fichier stocké dans `ingest_watermark`. Re-ingest du même fichier = 0 nouveau event
- **Redaction** — regex compilées (Bearer, password, API key, IP publique, home path). Appliquée uniquement quand `redact: true`
- **ts='unknown'** — events sans timestamp dans le JSONL. Filtrés dans MIN/MAX des stats
