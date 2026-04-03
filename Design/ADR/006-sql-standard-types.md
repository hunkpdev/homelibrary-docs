# ADR-006: SQL szabványos adattípusok használata

> **Státusz:** Elfogadva
> **Dátum:** 2026-04-03

## Kontextus

A lokális fejlesztési környezetben nem akarunk sem Dockert, sem külső adatbázist telepíteni.
A megoldás HSQLDB in-memory adatbázis lokálisan, PostgreSQL élesben (Neon SaaS).
Ez csak akkor működik kétutas Liquibase script nélkül, ha az adatmodell kizárólag
mindkét motor által ismert, SQL szabványos típusokat használ.

## Döntés

**Az adatmodellben kizárólag SQL szabványos adattípusokat használunk.**
Gyártófüggő típusok tilosak – ez alól nincs kivétel.

### Érintett típusmódosítások

| Gyártófüggő (régi) | SQL szabvány (új) | Indok |
|---------------------|-------------------|-------|
| `TIMESTAMPTZ` | `TIMESTAMP WITH TIME ZONE` | PostgreSQL-specifikus rövidítés |
| `TEXT[]` | `VARCHAR(n)` JSON string-ként | PostgreSQL-specifikus tömb típus |

### Tömbök kezelése

A `books.authors` és `books.categories` mezők JSON string-ként tárolódnak (`VARCHAR`).
Példa: `["Tolkien, J.R.R."]`, `["Fantasy", "Fiction"]`

A GIN indexek (amelyek PostgreSQL tömbökhöz kellettek volna) eltávolításra kerültek.
A háztartási méretskálán (reálisan 200–2000 könyv) LIKE-alapú keresés
teljes tábla-scan esetén is érzékelhetetlen sebességű. Mérések alapján
`LIKE`-keresés ~50 000–100 000 sor felett kezd lassulni – ez a limit
ebben az alkalmazásban soha nem lesz releváns.

## Következmények

- Liquibase changelog-ok egyetlen script-tel futnak HSQL-en és PostgreSQL-en egyaránt
- Nincs `dbms`-feltételes changeset
- A fejlesztői élmény egyszerűbb: nincs külső infrastruktúra-függőség lokálisan
- Ha az adatmodell bővül, az alapelv kötelezően érvényes: új mező is csak szabványos típussal kerülhet be
