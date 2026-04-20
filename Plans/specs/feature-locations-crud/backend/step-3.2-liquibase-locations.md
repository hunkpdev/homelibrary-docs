# Step 3.2 – Liquibase: locations tábla

## Mit állít elő

- `backend/src/main/resources/db/changelog/changes/003-create-locations.yaml` — `locations` tábla changeset

---

## Tábla struktúra

Az ADR-006 alapján kizárólag SQL szabványos típusok. Lásd ADR-007 a rooms/locations szétválasztás indoklásakor.

| Oszlop | Típus | Megszorítás |
|--------|-------|-------------|
| `id` | `UUID` | PK |
| `name` | `VARCHAR(100)` | NOT NULL |
| `description` | `TEXT` | — |
| `room_id` | `UUID` | FK → rooms NOT NULL |
| `active` | `BOOLEAN` | NOT NULL DEFAULT true |
| `version` | `BIGINT` | NOT NULL DEFAULT 0 |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL |

---

## Kulcs döntések

- A changeset id: `003-create-locations`, author: `homelibrary`
- Be kell kötni a `db.changelog-master.yaml`-ba (a `002-create-rooms.yaml` után — FK függőség)
- `room_id` FK: `rooms(id)` — a `rooms` tábla mindig előbb jön létre
- Nincs seed adat — locationöket az admin hoz létre az UI-on keresztül

---

## Elfogadási kritériumok

- `local` profilon indítás után az HSQLDB-ben létrejön a `locations` tábla a megadott oszlopokkal
- A `room_id` FK constraint érvényes (nem létező room_id-val nem lehet locationt beszúrni)
- A changeset szerepel a `DATABASECHANGELOG` táblában
