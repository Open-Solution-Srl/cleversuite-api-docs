# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository nature

Documentation-only repo for the **CleverSuite** CRM REST API. No source code, no build, no tests, no package manager. Two artifacts must stay in sync:

- `README.md` — human reference, written in **Italian** (entity tables, conventions, end-to-end examples).
- `openapi.json` — OpenAPI **3.1.0** spec, also Italian (descriptions, summaries). Rendered by the user in a [Scalar](https://github.com/scalar/scalar) container.

When adding or changing an entity, update **both files**. Entity field tables in `README.md` and the corresponding schema in `openapi.json` must agree on field names, types, required flags, and enum values.

## API model (high-level)

Three namespaces, all following the same uniform CRUD shape:

- `/CleverBase` — anagrafiche, ticketing, documenti (Cliente, Ticket, Allegato, Attivita, Utente, …)
- `/CleverCRM` — vendite/fatturazione/traffico (Articolo, Contratto, Offerta, Fatture, ChiamataCliente, …)
- `/CleverPresence` — presenze (Presenza, RichiestaAssenza, Straordinario, OrarioDiLavoro)

Per entity, **five endpoints**:

| Op | Path | Body | Success |
|---|---|---|---|
| Search | `GET /{Entita}?Campo=…` | — | `200` `{ <Key>: [...] }` (object empty if no match) |
| Read | `GET /{Entita}/{id}` | — | `200` flat object |
| Create | `POST /{Entita}` | flat object, `ID` optional | `204`, or `200` + new ID in `text/plain` |
| Update | `PUT /{Entita}` | flat object, `ID` required | `204` (partial update — only present fields are touched) |
| Delete | `DELETE /{Entita}/{id}` | — | `204` |

Auth: `username` + `password` HTTP headers on every call. In the spec these are **two separate `apiKey` security schemes** (one per header), combined with logical AND via top-level `security: [{ "username": [], "password": [] }]`. Errors: `text/plain` body with `400`/`401`/`404`/`500`. `401` is returned for missing or invalid credentials on any operation.

### Conventions hardcoded in the data model

- **Booleans are integers**: `-1` = true, `0` = false.
- **GUIDs**: canonical `XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX`. Unset foreign keys come back as `""`, not `null`.
- **Dates**: `yyyy-MM-dd` / `yyyy-MM-dd HH:mm:ss` / `HH:mm:ss`.
- **Binary**: Base64 in `ContentBytes` (or per-entity equivalent), MIME in `ContentType`.
- **Search filters**: ID/foreign-key fields (anything starting with `ID…`) match exactly; all other fields are partial (`LIKE`); multiple params combine with AND.
- **Search response array key**: usually equals the entity name, but **not always** — `/Messaggi` returns `Messaggio`, `/Fatture` returns `Fattura`, `/DettagliContratti` returns `DettaglioContratto`, `/SedeRichiestaAnagrafica` returns `SedeCliente`. When editing, copy the key from `README.md` / the existing `*SearchResponse` schema rather than inferring it.

## openapi.json structure

Three schema "shapes" per entity — keep all three when adding a new entity:

1. **`{Entita}`** — full resource. Used for GET-by-id response, GET-search array items, and POST request body. Composed via `allOf`:
   ```json
   "Allegato": {
     "allOf": [
       { "$ref": "#/components/schemas/DoFlags" },
       { "type": "object", "required": ["ID"], "properties": { ... } }
     ]
   }
   ```
2. **`{Entita}Update`** — PUT body. Plain `type: object` with `required: ["ID"]` and all-optional fields (no `allOf`, no `DoFlags`).
3. **`{Entita}SearchResponse`** — GET-search wrapper: `{ <ArrayKey>: [{$ref Entita}] }`.

Shared building blocks live under `components`:
- `schemas.DoFlags` — the four `do_*` flags, all `readOnly: true`.
- `parameters.PathId` — the `{id}` path parameter.
- `responses.PlainTextError` / `PlainTextId` / `NoContent` / `Unauthorized` — reused across every operation.

Top-level `tags`: exactly three — `CleverBase`, `CleverCRM`, `CleverPresence`. Every operation must carry `tags: ["<Namespace>"]` matching the path prefix; Scalar uses these for the sidebar grouping.

### Critical: never write `allOf` as a sibling of `properties` in entity schemas

Scalar's example generator walks only the `allOf` array when synthesizing response bodies and silently drops any sibling `properties` / `required`. Result: example responses show only the four `DoFlags` fields and none of the entity's actual columns. The whole spec was migrated away from this anti-pattern (commit-level rewrite of all 52 entity schemas). **Do not** reintroduce the form:

```json
// WRONG — Scalar shows only do_* in example responses
{ "type": "object", "allOf": [ {"$ref": ".../DoFlags"} ], "required": [...], "properties": {...} }
```

Always nest the inline subschema **inside** `allOf` as shown in the `{Entita}` example above.

## Working in this repo

- No commands to run. Validate `openapi.json` shape with `python -c "import json; json.load(open('openapi.json', encoding='utf-8'))"` after edits, or paste it into Scalar to eyeball examples and schemas.
- For bulk edits across the 50+ entity schemas, scripting via Python (`json` module preserves insertion order on 3.7+) is preferable to hand-editing — the file is ~16k lines.
- Commit messages and descriptions in `openapi.json` / `README.md` are in **Italian**; match that tone for any new content.
