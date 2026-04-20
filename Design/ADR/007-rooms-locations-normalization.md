# ADR-007: Rooms és Locations tábla szétválasztása

**Dátum:** 2026-04-20
**Státusz:** Elfogadva

## Kontextus

Az eredeti tervben a `locations` tábla egyetlen táblában tárolta a helyiség (`room_name`) és a konkrét tárolási hely (`shelf_name`) adatait. Ez két problémát okozott:

1. **Üres csoport probléma:** Ha egy helyiség összes tárolási helye soft-deleted, a helyiség eltűnik a frontend csoportos nézetéből — és nincs lehetőség új tárolási helyet hozzáadni hozzá anélkül, hogy a room nevét kézzel begépelnénk.
2. **Adatmodell pontatlanság:** A `room_name` + `shelf_name` kombináció denormalizált — a helyiség egy önálló entitás, nem egy mező.

## Döntés

**A `locations` tábla mellé bevezettük a `rooms` táblát.** A `locations` tábla mostantól egy konkrét tárolási helyet (polc, szekrény, láda, stb.) reprezentál, és FK-val hivatkozik a `rooms` táblára.

```
rooms (id, name, description, active, ...)
  │
  └──< locations (id, name, description, active, room_id FK, ...)
          │
          └──< books (location_id FK)
```

## Alternatívák

### A: Eredeti egy-táblás megközelítés
- ✅ Egyszerűbb séma
- ❌ Denormalizált — `room_name` ismétlődik minden location sorban
- ❌ Üres szoba nem reprezentálható — ha minden shelf törölt, a szoba eltűnik
- ❌ Frontend "gányol" API-kat igényel az üres csoportok kezeléséhez

### B: Külön `rooms` és `locations` tábla (választott)
- ✅ Normalizált séma — a szoba önálló entitás
- ✅ Üres szoba természetesen reprezentálható (soft delete a `rooms` táblán)
- ✅ Tiszta CRUD mindkét entitásra
- ✅ A frontend csoportos nézete a `rooms` táblából tölti a fejléceket
- ⚠️ Két tábla, két CRUD — de a komplexitás indokolt

### C: Külön `rooms` endpoint, változatlan `locations` tábla
- ❌ Hibrid megoldás: a séma denormalizált marad, az API kettős
- ❌ Adatintegritás problémák (`room_name` eltérhet a két helyről)

## Megszorítások

- Egy location **pontosan egy roomhoz** tartozik (`room_id` NOT NULL FK) — location nem létezhet room nélkül
- Egy location **csak egy roomhoz** tartozhat — nincs many-to-many, nincs áthelyezés rooms között

## Következmények

- `DB_SCHEMA.md`: új `rooms` tábla, `locations` tábla módosítása (`room_name` + `shelf_name` → `name` + `room_id` FK)
- `API_DESIGN.md`: új `/api/rooms` CRUD endpointok, `/api/locations` frissítés
- `phase1-feature-order.md`: Feature 3 stepek bővítése (rooms és locations külön backend lépések)
- Liquibase changelog: `002-create-rooms.yaml` + `003-create-locations.yaml` (átszámozás)
- A `books` tábla `location_id` FK-ja változatlan marad
