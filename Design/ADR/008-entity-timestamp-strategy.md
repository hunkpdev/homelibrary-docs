# ADR-008: Entitás timestamp mezők kezelési stratégiája

> **Státusz:** Elfogadva
> **Dátum:** 2026-04-21

## Kontextus

A JPA entitásokon a `created_at` és `updated_at` mezőket automatikusan kell feltölteni.
Az eredetileg tervezett megközelítés Hibernate-specifikus annotációkat alkalmazott:
`@CreationTimestamp` és `@UpdateTimestamp`.

Implementáció során kiderült, hogy ezek az annotációk `OffsetDateTime` típussal
nem működnek megbízhatóan: a Hibernate nem tölti fel automatikusan a mezőt,
vagy helytelen értéket állít be timezone-kezeléssel kapcsolatos kompatibilitási
okokból.

## Döntés

**Az összes entitáson `@PrePersist` / `@PreUpdate` JPA lifecycle callback-eket
használunk a timestamp mezők feltöltésére, explicit `ZoneOffset.UTC` timezone-nal.**

A timezone explicit módon UTC — ez nem opcionális. Felhős környezetben (Lambda,
container) a JVM timezone a deploymenttől vagy a régiótól függhet. Paraméter nélküli
`now()` hívás esetén régióváltás vagy konfiguráció-változás esetén a timestampek
inkonzisztensek lesznek a régi és új rekordok között — néma adathiba, amelyet
nehéz visszakeresni.

Az implementáció vázlata:
- `createdAt`: `OffsetDateTime`, nem frissíthető (`updatable = false`), `@PrePersist`-ben töltendő UTC értékkel
- `updatedAt`: `OffsetDateTime`, `@PrePersist` és `@PreUpdate`-ban egyaránt frissítendő UTC értékkel

## Alternatívák

### A: `@CreationTimestamp` / `@UpdateTimestamp` (elvetett)
- ✅ Tömörebb kód
- ❌ Hibernate-specifikus annotáció — nem JPA szabvány
- ❌ `OffsetDateTime` típussal kompatibilitási problémák lépnek fel
- ❌ Timezone-kezelés implicit és nehezen nyomon követhető

### B: `@PrePersist` / `@PreUpdate` (választott)
- ✅ JPA szabvány — nem Hibernate-specifikus
- ✅ Explicit UTC timezone — felhős környezetben biztonságos, régióváltástól független
- ✅ Timezone-kezelés átlátható és nyomon követhető
- ⚠️ Némileg több boilerplate — de egységes minta minden entitáson

### C: `BaseEntity` abstract osztály (ajánlott kiegészítés)

Ha az entitások száma növekszik, a timestamp mezők és callback-ek kiemelhetők
egy közös `@MappedSuperclass` alaposztályba, amelyből az összes entitás örökli
a viselkedést. Ez nem változtat a döntésen, csak csökkenti az ismétlést, és
garantálja, hogy a mintát egyetlen helyen kell karbantartani.

## Következmények

- Minden jelenlegi és jövőbeli entitás ezt a mintát követi (`User`, `Room`, `Location`, `Book`, stb.)
- A timestamp feltöltés mindig explicit UTC timezone-nal történik — ez kötelező
- `@CreationTimestamp` / `@UpdateTimestamp` tilos az egész projektben
- A `User` entitáson visszamenőlegesen javítandó, ha még a régi annotációkat használja
- A séma változatlan marad — ez kizárólag a Java oldali implementációt érinti
