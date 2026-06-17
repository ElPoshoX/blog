# Plane CRUD Skill — Design Spec

**Date**: 2026-06-11
**Author**: Az + Claude
**Status**: Draft

## 1. Invocation & Scope

**Trigger**: `/plane`, "plane issue", "create issue in plane", "list plane projects", etc.

**Natural language mapping** — user says things like:
- "list projects in plane"
- "create issue in BYT: deploy monitoring stack"
- "search BYT issues about terraform"
- "move BYT-42 to In Progress"
- "add comment to BYT-42"

Claude interprets intent → resolves IDs → builds curl → executes → formats human-readable response.

### Resources (core subset)

| Resource | Actions |
|----------|---------|
| Projects | list, get |
| Issues (work-items) | list, get, create, update, delete, search |
| States | list, get |
| Labels | list, get, create |
| Cycles | list, get |
| Modules | list, get |
| Comments | list, create |
| Links | list, create, delete |

Admin ops (create/update/delete projects, states, cycles, modules) left to UI.

### Relationship with /plane-blog-post

`/plane-blog-post` stays separate — it has BYT-specific logic (Obsidian links, bilingual drafts, blog-post label). Future refactor may have it delegate to `/plane` internals.

---

## 2. Architecture

**Pattern**: Skill is a reference doc. Claude reads it, interprets user intent, builds `/usr/bin/curl` commands, executes via Bash, parses JSON, formats output.

**Critical constraint**: Must use `/usr/bin/curl` not bare `curl` — RTK hook intercepts `curl` and corrupts JSON output for piped processing.

> **TODO (RTK)**: Add env var bypass (e.g. `RTK_RAW=1`) so `curl` can be used normally when raw output is needed. Track in RTK repo.

**Flow**:
```
User intent → Claude resolves IDs (project, state, label) → builds /usr/bin/curl command → Bash → parse JSON → format human-readable output
```

**ID Resolution**: Many operations need UUIDs (state names → state IDs, project identifiers → project UUIDs). Claude fetches these first when needed and caches within conversation.

**Authentication**: `$PLANE_API_KEY` env var. Header: `X-API-Key`.

**Base URL**: `https://plane.elposhox.dev/api/v1/workspaces/elposhox`

**Workspace**: Hardcoded `elposhox`.

---

## 3. Subcommand Syntax

```
/plane <resource> <action> [args]
```

| Command | Example |
|---------|---------|
| `projects list` | `/plane projects list` |
| `issues list <PROJECT>` | `/plane issues list BYT` |
| `issues get <PROJECT> <SEQ_ID>` | `/plane issues get BYT 24` |
| `issues create <PROJECT>` | `/plane issues create BYT "Title" --state Propuesto --priority high --labels blog-post` |
| `issues update <PROJECT> <SEQ_ID>` | `/plane issues update BYT 24 --state "In Progress"` |
| `issues delete <PROJECT> <SEQ_ID>` | `/plane issues delete BYT 24` |
| `issues search <PROJECT>` | `/plane issues search BYT "terraform"` |
| `states list <PROJECT>` | `/plane states list BYT` |
| `labels list <PROJECT>` | `/plane labels list BYT` |
| `labels create <PROJECT>` | `/plane labels create BYT "new-label" "#FF0000"` |
| `cycles list <PROJECT>` | `/plane cycles list BYT` |
| `comments list <PROJECT> <SEQ_ID>` | `/plane comments list BYT 24` |
| `comments add <PROJECT> <SEQ_ID>` | `/plane comments add BYT 24 "Comment text"` |
| `links list <PROJECT> <SEQ_ID>` | `/plane links list BYT 24` |
| `links add <PROJECT> <SEQ_ID>` | `/plane links add BYT 24 "https://..." "Title"` |
| `links delete <PROJECT> <SEQ_ID> <LINK_ID>` | `/plane links delete BYT 24 <uuid>` |

**Natural language**: Claude interprets "move BYT-24 to Draft Ready" as `issues update BYT 24 --state "Draft Ready"`.

**Project resolution**: User says `BYT` → Claude resolves to project UUID via project list.

**Issue resolution**: User says `BYT-24` (sequence_id 24) → Claude resolves to work-item UUID via `work-items/by-identifier/` or list + filter.

### Output Formatting

All responses formatted human-readable, not raw JSON:

**Create/Update**:
```
Created BYT-24: La Fábrica de Intermedios
  State:    Publicado
  Priority: medium
  Labels:   substack, ai/ml
  URL:      https://plane.elposhox.dev/elposhox/projects/.../work-items/...
```

**List**: Table format with key columns (ID, title, state, priority).

**Search**: Table with matches, highlighting relevant fields.

---

## 4. Error Handling & Safety

| Scenario | Behavior |
|----------|----------|
| `$PLANE_API_KEY` missing | Error: "Set PLANE_API_KEY env var. Get token from Plane → Profile → API Tokens" |
| 401 Unauthorized | "API key invalid or expired. Regenerate at plane.elposhox.dev" |
| 404 Not Found | "Issue/project not found. Verify identifier." |
| 429 Rate Limited | Read `X-RateLimit-Reset` header, wait, retry once. Report if still limited. |
| Delete operations | **Confirm with user before executing** — show what will be deleted |
| Network error | Report error, suggest checking plane.elposhox.dev reachability |

### Pagination

Plane uses cursor-based pagination. For list operations with many results:
- Default `per_page=100`
- If `next_page_results: true`, fetch next page automatically
- Cap at 500 results to avoid runaway pagination

---

## 5. Curl Template Patterns

```bash
# Variables
BASE="https://plane.elposhox.dev/api/v1/workspaces/elposhox"
AUTH="-H 'X-API-Key: $PLANE_API_KEY'"

# List projects
/usr/bin/curl -s -H "X-API-Key: $PLANE_API_KEY" "$BASE/projects/?fields=id,identifier,name"

# List issues
/usr/bin/curl -s -H "X-API-Key: $PLANE_API_KEY" \
  "$BASE/projects/{project_id}/work-items/?per_page=100&fields=id,sequence_id,name,state,priority,label_ids"

# Get issue by identifier
/usr/bin/curl -s -H "X-API-Key: $PLANE_API_KEY" \
  "$BASE/projects/{project_id}/work-items/by-identifier/?identifier={PROJECT}-{SEQ_ID}"

# Create issue
/usr/bin/curl -s -X POST -H "X-API-Key: $PLANE_API_KEY" -H "Content-Type: application/json" \
  "$BASE/projects/{project_id}/work-items/" \
  -d '{"name":"...","state":"<state_uuid>","priority":"medium","label_ids":["<label_uuid>"],"description_html":"<p>...</p>"}'

# Update issue (e.g., transition state)
/usr/bin/curl -s -X PATCH -H "X-API-Key: $PLANE_API_KEY" -H "Content-Type: application/json" \
  "$BASE/projects/{project_id}/work-items/{work_item_id}/" \
  -d '{"state":"<state_uuid>"}'

# Delete issue
/usr/bin/curl -s -X DELETE -H "X-API-Key: $PLANE_API_KEY" \
  "$BASE/projects/{project_id}/work-items/{work_item_id}/"

# List states
/usr/bin/curl -s -H "X-API-Key: $PLANE_API_KEY" \
  "$BASE/projects/{project_id}/states/?fields=id,name,group"

# List labels
/usr/bin/curl -s -H "X-API-Key: $PLANE_API_KEY" \
  "$BASE/projects/{project_id}/labels/?fields=id,name,color"

# Create label
/usr/bin/curl -s -X POST -H "X-API-Key: $PLANE_API_KEY" -H "Content-Type: application/json" \
  "$BASE/projects/{project_id}/labels/" \
  -d '{"name":"...","color":"#FF0000"}'

# List comments
/usr/bin/curl -s -H "X-API-Key: $PLANE_API_KEY" \
  "$BASE/projects/{project_id}/work-items/{work_item_id}/comments/"

# Add comment
/usr/bin/curl -s -X POST -H "X-API-Key: $PLANE_API_KEY" -H "Content-Type: application/json" \
  "$BASE/projects/{project_id}/work-items/{work_item_id}/comments/" \
  -d '{"comment_html":"<p>...</p>"}'

# List links
/usr/bin/curl -s -H "X-API-Key: $PLANE_API_KEY" \
  "$BASE/projects/{project_id}/work-items/{work_item_id}/links/"

# Add link
/usr/bin/curl -s -X POST -H "X-API-Key: $PLANE_API_KEY" -H "Content-Type: application/json" \
  "$BASE/projects/{project_id}/work-items/{work_item_id}/links/" \
  -d '{"url":"https://...","title":"..."}'

# Delete link
/usr/bin/curl -s -X DELETE -H "X-API-Key: $PLANE_API_KEY" \
  "$BASE/projects/{project_id}/work-items/{work_item_id}/links/{link_id}/"

# Search issues
/usr/bin/curl -s -H "X-API-Key: $PLANE_API_KEY" \
  "$BASE/projects/{project_id}/work-items/search/?search=<query>"

# List cycles
/usr/bin/curl -s -H "X-API-Key: $PLANE_API_KEY" \
  "$BASE/projects/{project_id}/cycles/?fields=id,name,start_date,end_date"

# List modules
/usr/bin/curl -s -H "X-API-Key: $PLANE_API_KEY" \
  "$BASE/projects/{project_id}/modules/?fields=id,name"
```

### Response Shaping

Use `?fields=` query param to reduce response size. Only request fields needed for the operation.

Use `?expand=` to inline related resources when needed (e.g., `expand=state` to get state name instead of just UUID).

---

## 6. Model Selection

| Operation | Complexity | Recommended Model |
|-----------|-----------|-------------------|
| `list`, `get`, `states list`, `labels list` | Low — single curl, format output | **Haiku** (via subagent) |
| `create`, `update`, `delete` | Medium — resolve IDs, build payload | **Sonnet** (via subagent) |
| `search` + multi-step chains | High — multiple API calls, parse results | **Sonnet** (via subagent) |
| Natural language → action | Medium — intent parsing + ID resolution | **Sonnet** (inline or subagent) |

No Opus needed for any Plane operation. Work is mechanical (build curl, parse JSON), not reasoning-heavy.

**Subagent dispatch**: Skill documents model tiers. Claude decides whether to dispatch subagent or handle inline based on context. Spawning subagent has ~2-3s overhead — for a single simple curl, inline may be faster.

**Guideline**: Use subagents when the Plane operation is part of a larger task and you want to keep the main context clean. Do inline when it's a quick one-off.

---

## 7. Fallback: Django ORM (rate limit emergencies only)

If API is persistently rate-limited (429 after retry), fall back to Django shell via kubectl:

```bash
export KUBECONFIG=/Users/az/projects/homelab/talos-homelab/kubeconfig.yaml
kubectl exec -n plane deploy/plane-app-api-wl -- sh -c 'cd /code && python3 manage.py shell -c "
from plane.db.models import Issue, State, Label, Project
# Example: list issues
for i in Issue.objects.filter(project__identifier=\"BYT\").order_by(\"-created_at\")[:10]:
    print(f\"{i.project.identifier}-{i.sequence_id}: {i.name} [{i.state.name}]\")
"'
```

This bypasses API rate limits entirely but requires kubectl access to the homelab cluster. Use only when API is unavailable.

---

## Decisions Log

| Decision | Rationale |
|----------|-----------|
| Curl-based, no Python/MCP | Zero deps, zero maintenance, works everywhere |
| `/usr/bin/curl` not `curl` | RTK hook corrupts JSON output; TODO: add RTK_RAW bypass |
| Hardcoded `elposhox` workspace | Personal instance, single workspace |
| Core subset only | Admin ops in UI, CRUD covers 95% of automation needs |
| Separate from `/plane-blog-post` | Blog skill has domain-specific logic (Obsidian, bilingual) |
| Env var for API key | Shell-level, works across tools (Claude Code, pi.dev, scripts) |
| Human-readable output | Format all responses, never dump raw JSON |
| Confirm before delete | Safety — destructive ops need explicit user approval |
| Model tiers documented | Guide Claude toward efficient model selection |
