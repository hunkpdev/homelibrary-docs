# Step 3.1 – Liquibase: rooms tábla

## Mit állít elő

- `backend/src/main/resources/db/changelog/changes/002-create-rooms.yaml` — `rooms` tábla changeset

---

## Tábla struktúra

Az ADR-006 alapján kizárólag SQL szabványos típusok.

| Oszlop | Típus | Megszorítás |
|--------|-------|-------------|
| `id` | `UUID` | PK |
| `name` | `VARCHAR(100)` | NOT NULL |
| `description` | `TEXT` | — |
| `active` | `BOOLEAN` | NOT NULL DEFAULT true |
| `version` | `BIGINT` | NOT NULL DEFAULT 0 |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL |

---

## Kulcs döntések

- A changeset id: `002-create-rooms`, author: `homelibrary`
- Be kell kötni a `db.changelog-master.yaml`-ba (az `001-create-users.yaml` után)
- Nincs seed adat — helyiségeket az admin hoz létre az UI-on keresztül

---

## Elfogadási kritériumok

- `local` profilon indítás után az HSQLDB-ben létrejön a `rooms` tábla a megadott oszlopokkal
- A changeset szerepel a `DATABASECHANGELOG` táblában
