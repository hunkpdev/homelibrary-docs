# ADR-005: Adatbázis Migráció Eszköz Választás

**Dátum:** 2026-04-01
**Státusz:** Elfogadva

## Kontextus

Az alkalmazás PostgreSQL sémájának verziókövetett kezeléséhez migrációs eszköz szükséges.
A Spring Boot ökoszisztémában két elterjedt megoldás létezik: Flyway és Liquibase.

## Döntés

**Liquibase**

## Alternatívák

### A: Flyway
- ✅ Egyszerűbb, SQL-first megközelítés
- ✅ Spring Boot auto-konfiguráció minimális beállítással
- ❌ Rollback csak fizetős (Teams/Enterprise) verzióban
- ❌ Csak SQL changelog formátum

### B: Liquibase (választott)
- ✅ Rollback támogatás ingyenesen
- ✅ Több changelog formátum: XML, YAML, JSON, SQL
- ✅ Diff generálás (DB állapot összehasonlítás)
- ✅ Spring Boot integráció (`liquibase-core` dependency)
- ⚠️ Valamivel több kezdeti konfiguráció, mint Flyway esetén

## Megvalósítás

Liquibase changelog fájlok helye a projekt struktúrában:

```
backend/src/main/resources/db/changelog/
  db.changelog-master.yaml        ← master changelog (include-olja a többi fájlt)
  changes/
    001-create-users.yaml
    002-create-locations.yaml
    003-create-books.yaml
    004-create-book-descriptions.yaml
    005-create-loans.yaml
    006-add-indexes.yaml
```

## Következmények

- `liquibase-core` dependency a `pom.xml`-ben (Spring Boot automatikusan felismeri)
- A `spring.liquibase.change-log` property-t konfigurálni kell (`application.yml`-ben)
- Rollback lehetséges mind SQL (`rollback` tag), mind parancssorból (`liquibase rollback`)
