# ADR-003: ISBN Lookup API Stratégia

**Dátum:** 2026-03-28
**Státusz:** Elfogadva

## Kontextus

A könyv felvételekor ISBN szám alapján automatikusan le kell kérni a könyv adatait
(cím, szerző, kiadó, kiadási év, kategória, borítókép) egy külső forrásból.
A megoldásnak ingyenesnek kell lennie.

## Döntés

**Kétlépéses stratégia: OpenLibrary API (elsődleges) + Google Books API (fallback)**

## API Összehasonlítás

| Szempont | OpenLibrary | Google Books | Amazon Product API |
|----------|-------------|--------------|-------------------|
| Ár | Ingyenes | Ingyenes | Fizetős |
| API kulcs | Nem kell | Kell | Kell |
| Napi limit | Nincs | 1000 req/nap | - |
| Adatminőség | Jó (közösségi) | Jó (Google) | Kiváló |
| Borítókép | ✅ | ✅ | ✅ |
| Magyar könyvek | Közepes | Jó | Jó |

## Megvalósítás

### 1. OpenLibrary API (elsődleges)
```
GET https://openlibrary.org/api/books?bibkeys=ISBN:{isbn}&format=json&jscmd=data
```
- Regisztráció nélkül, API kulcs nélkül hívható
- Ha találat van → adatok feldolgozása és visszaadása
- Ha nincs találat → tovább a fallback-re

### 2. Google Books API (fallback)
```
GET https://www.googleapis.com/books/v1/volumes?q=isbn:{isbn}&key={API_KEY}
```
- API kulcs szükséges (ingyenes, Google Cloud Console-ban igényelve)
- API kulcs SSM Parameter Store-ban tárolva
- 1000 req/nap limit – egy házi könyvtárnál bőven elegendő

### 3. Manuális bevitel (végső fallback)
Ha egyik API sem talál eredményt, a felhasználó kézzel tölti ki az adatokat.

## Backend Implementáció

A backend hívja az external API-kat (nem a frontend direktben):
- Egységes hibakezelés
- API kulcsok biztonságban maradnak
- Könnyen cserélhető/bővíthető (Strategy pattern)

```
GET /api/books/isbn/{isbn}
  → OpenLibraryService.lookup(isbn)
    → ha null: GoogleBooksService.lookup(isbn)
      → ha null: { found: false }
```

## Következmények

- Google Books API kulcs SSM Parameter Store-ban: `/homelibrary/google-books-api-key`
- A Spring Boot alkalmazásban `IsbnLookupService` interface + két implementáció
- Ha az OpenLibrary adatminősége nem megfelelő magyar könyveknél, a sorrend felcserélhető
