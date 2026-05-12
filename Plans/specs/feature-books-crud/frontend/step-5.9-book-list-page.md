# Step 5.9 – Könyvlista oldal

## Mit állít elő

- `src/pages/BookListPage.tsx` — könyvlista AG Grid Community táblázattal
- `src/api/bookApi.ts` — API hívások könyvekhez (`GET /api/books`, `DELETE`)
- `src/store/bookStore.ts` — Zustand store, `booksRefreshTrigger` számlálóval
- `src/pages/BookListPage.test.tsx` — unit tesztek

---

## Oldal struktúra

```
[ AG Grid Community táblázat (embedded column filterekkel) ]
```

Az „+ Új könyv" gomb step 5.12-ben kerül be, amikor a form modal is elkészül. Külső szűrősor nincs.

---

## Adatlekérés

Oldal betöltésekor és `booksRefreshTrigger` változásakor:
- AG Grid Infinite Row Model datasource automatikusan: `GET /api/books?page=...&size=...&sort=...&isbn=...&title=...&authors=...&category=...&publishYear=...`

Az összevont `search` paraméter három önálló paraméterre bontva: `isbn` (prefix), `title` (contains), `authors` (contains). A `publishYear` típusa `String`, hogy prefix LIKE keresés legyen lehetséges (pl. `"202"` → 2020–2029).

Szűrő / sort / lapozás változásakor csak a grid datasource fetch fut újra.

---

## AG Grid táblázat

**Row Model:** Infinite Row Model — backend `Page<BookResponse>` válasz alapján lapoz.

**Oszlopok:**

| Oszlop | Mező | Rendezés | Embedded szűrő | Matching |
|--------|------|----------|----------------|---------|
| ISBN | `isbn` | igen | `ClearableTextFloatingFilter` | `startsWith`, case-insensitive |
| Cím | `title` | igen | szöveges | `contains`, case-insensitive |
| Szerző(k) | `authors` (pontosvesszővel elválasztva) | igen | szöveges | `contains`, case-insensitive |
| Kiadási év | `publishYear` (String) | igen | szöveges | `startsWith` |
| Kategóriák | `categories` (pontosvesszővel elválasztva) | nem | szöveges | `contains`, case-insensitive |
| Műveletek | — | nem | — | — |

**Műveletek oszlop** (csak `ADMIN` és `DEMO` látja — `DEMO`-nál `MutationButton` auto-disabled):
- **Törlés** ikon gomb → step 5.9-ben teljesen bekötve (`DeleteModal` + `deleteBook` API hívás)
- **Szerkesztés** ikon gomb → gomb megjelenik, de step 5.12-ig nem nyit modalt

**Sorra kattintás** (bárhol a műveletek oszlopon kívül) → `onRowClicked` handler step 5.10-ben kerül bekötésre; step 5.9-ben a kattintás nem vált ki eseményt.

**Téma:** `ag-theme-quartz` / `ag-theme-quartz-dark` — az alkalmazás aktuális dark/light mode állapotából, változásra automatikusan reagál.

**Default sort:** `title,asc`.

---

## State kezelés

`bookStore.ts` egyetlen `booksRefreshTrigger: number` értékkel — mutáció (létrehozás, szerkesztés, törlés) után a modalok incrementelik. Szűrő állapot helyi React state-ben él (`useState`).

`booksRefreshTrigger` változásakor: `gridApi.purgeInfiniteCache()` — a datasource objektum megmarad, a cache invalidálódik és a grid az elejéről tölt újra. Szűrőváltozáskor szintén `purgeInfiniteCache()` (egységes a Locations oldal implementációjával).

---

## Kulcs döntések

- AG Grid Infinite Row Model — egységes a Locations oldallal; szűrés kizárólag embedded column filtereken keresztül, külső szűrősor nincs
- DEMO: látja a listát, a műveletek oszlopban `MutationButton` auto-disabled tooltip-pal
- `DELETED` könyvek sosem jelennek meg — a backend Specification mindig kizárja (step 5.5), a frontend nem küld `status` paramétert

---

## Tech-debt

- **Mobilos oszlopoptimalizálás:** AG Grid oszlopok priorizálása kis képernyőn — Feature 5-ben nem prioritás, külön task-ként kezelendő (konzisztens a Locations oldal tech-debt-jével)

---

## Elfogadási kritériumok

- Grid sorok renderelve az API válasz alapján
- ADMIN: szerkesztés és törlés gomb látható és aktív
- VISITOR: műveletek oszlop nem látható
- DEMO: szerkesztés és törlés gomb látható, `MutationButton` disabled tooltip-pal
- Kategória szűrő alkalmazásakor `category` query param megjelenik az API hívásban
- Default sort: `title,asc`
