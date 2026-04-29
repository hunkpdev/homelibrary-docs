# Step 4.2 – Liquibase: demo_isbn_daily_stats tábla

## Mit állít elő

- `db/changelog/changes/008-create-demo-isbn-daily-stats.yaml` Liquibase changeset
- `demo_isbn_daily_stats` tábla a DEMO felhasználó napi ISBN lookup limitjéhez

---

## Tábla struktúra

| Oszlop | Típus | Megszorítás | Leírás |
|--------|-------|-------------|--------|
| `id` | `UUID` | PK | |
| `lookup_date` | `DATE` | NOT NULL | Az aktuális nap (UTC) |
| `lookup_count` | `INTEGER` | NOT NULL DEFAULT 0 | Keresések száma ezen a napon |

Egyszerre egy sor van a táblában. Lazy reset logika a service rétegben van (step 4.4) — a changeset csak a táblát hozza létre.

## Kulcs döntések

- SQL szabványos típusok (ADR-006): `DATE`, `INTEGER`, `UUID`
- Liquibase master changelog-ba bekötve (`db.changelog-master.yaml`)
- HSQLDB (`local` profil) és PostgreSQL (`prod` profil) kompatibilis changeset

## Elfogadási kritériumok

- App indításkor Liquibase létrehozza a táblát hibák nélkül
- `local` profilon (HSQLDB) és `prod` profilon (Neon PostgreSQL) egyaránt lefut
