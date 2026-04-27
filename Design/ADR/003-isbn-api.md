# ADR-003: ISBN Lookup API Stratégia

**Dátum:** 2026-03-28  
**Frissítve:** 2026-04-27  
**Státusz:** Elfogadva

## Kontextus

A könyv felvételekor ISBN szám alapján automatikusan le kell kérni a könyv adatait
(cím, szerző, kiadó, kiadási év, kategória, borítókép) egy külső forrásból.
A megoldásnak ingyenesnek kell lennie. Az alkalmazás elsősorban magyar nyelvű könyveket kezel.

## Döntés

**Kétlépéses stratégia: moly.hu (elsődleges, scraping) + OpenLibrary API (fallback)**

Google Books API-t nem használunk — API kulcs igénylése és napi limit kezelése felesleges overhead egy demo projektben. Ha a projekt kereskedelmi útra tér, az ISBNdb (fizetős) lesz a megfelelő bővítés.

## API Összehasonlítás

| Szempont | moly.hu | OpenLibrary | Google Books |
|----------|---------|-------------|--------------|
| Ár | Ingyenes | Ingyenes | Ingyenes (1000 req/nap) |
| API kulcs | Nem kell | Nem kell | Kell |
| Hozzáférés | HTML scraping | REST API | REST API |
| Magyar könyvek | Kiváló | Gyenge | Közepes |
| Stabilitás | Törékeny (UI változás) | Stabil | Stabil |

## Megvalósítás

### 1. moly.hu (elsődleges)
```
GET https://moly.hu/kereses?utf8=%E2%9C%93&query={isbn}
```
- HTML scraping Jsoup könyvtárral
- Legjobb magyar könyv lefedettség
- Nincs API kulcs, nincs rate limit
- Kockázat: UI változás esetén a selectorok frissítést igényelnek

### 2. OpenLibrary API (fallback)
```
GET https://openlibrary.org/api/books?bibkeys=ISBN:{isbn}&format=json&jscmd=data
```
- Regisztráció nélkül, API kulcs nélkül hívható
- Jó lefedettség nemzetközi könyveknél
- Gyenge magyar könyv lefedettség

### 3. Manuális bevitel (végső fallback)
Ha egyik forrás sem talál eredményt, a felhasználó kézzel tölti ki az adatokat.

## Backend Implementáció

A backend hívja a külső forrásokat (nem a frontend direktben):
- Egységes hibakezelés
- Könnyen bővíthető (új provider hozzáadása minimális változtatással)

```
GET /api/books/isbn/{isbn}
  → MolyHuClient.lookup(isbn)
    → ha null: OpenLibraryClient.lookup(isbn)
      → ha null: { found: false }
```

## Következmények

- Jsoup Maven függőség szükséges a moly.hu scrapinghez
- `IsbnSource` enum értékek: `MOLY_HU`, `OPENLIBRARY`
- Google Books API kulcs SSM-ben **nem** szükséges
- Ha a moly.hu HTML struktúrája változik, a `MolyHuClient` CSS selectorait frissíteni kell
