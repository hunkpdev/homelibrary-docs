# Step 3.1 – Liquibase: locations tábla

## Mit állít elő

- `backend/src/main/resources/db/changelog/changes/002-create-locations.yaml` — `locations` tábla changeset

---

## Tábla struktúra

Az ADR-006 alapján kizárólag SQL szabványos típusok.

| Oszlop | Típus | Megszorítás |
|--------|-------|-------------|
| `id` | `UUID` | PK |
| `room_name` | `VARCHAR(100)` | NOT NULL |
| `shelf_name` | `VARCHAR(100)` | — |
| `description` | `TEXT` | — |
| `active` | `BOOLEAN` | NOT NULL DEFAULT true |
| `version` | `BIGINT` | NOT NULL DEFAULT 0 |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL |

---

## Kulcs döntések

- A changeset id: `002-create-locations`, author: `homelibrary`
- Be kell kötni a `db.changelog-master.yaml`-ba (az `001-create-users.yaml` után)
- Nincs seed adat ebben a changesetben — helyiségeket az admin hoz létre az UI-on keresztül

---

## Elfogadási kritériumok

- `local` profilon indítás után az HSQLDB-ben létrejön a `locations` tábla a megadott oszlopokkal
- A changeset szerepel a `DATABASECHANGELOG` táblában
