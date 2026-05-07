# Step 5.10 – Könyvlista oldal

## Mit állít elő

- `src/pages/BookListPage.tsx` — könyvlista AG Grid Community táblázattal
- `src/api/bookApi.ts` — API hívások könyvekhez (`GET /api/books`, `DELETE`)
- `src/store/bookStore.ts` — Zustand store, `booksRefreshTrigger` számlálóval
- `src/pages/BookListPage.test.tsx` — unit tesztek

---

## Oldal struktúra

```
[ Szűrősor                              ] [ + Új könyv gomb ]
[ AG Grid Community táblázat            ]
```

---

## Adatlekérés

Oldal betöltésekor és `booksRefreshTrigger` változásakor:
- `GET /api/locations/all` — helyszín szűrő dropdown-hoz (párhuzamosan a grid datasource init-tel)
- AG Grid Infinite Row Model datasource automatikusan: `GET /api/books?page=...&size=...&sort=...&search=...&status=...&locationId=...`

Szűrő / sort / lapozás változásakor csak a grid datasource fetch fut újra.

---

## Szűrősor

Shadcn/ui komponensek a grid felett (a Locations oldal filter sávjával egységes vizuális elrendezés):

| Mező | Komponens | API paraméter |
|------|-----------|---------------|
| Keresés (cím, szerző) | `Input` | `search` |
| Státusz | `Select` | `status` (`AT_HOME` / `LOANED` — `DELETED` nem szerepel) |
| Helyszín | `Select` | `locationId` (értékek: `GET /api/locations/all`) |

Szűrőváltozáskor a datasource `page=0`-ra reset-el.

---

## AG Grid táblázat

**Row Model:** Infinite Row Model — backend `Page<BookResponse>` válasz alapján lapoz.

**Oszlopok:**

| Oszlop | Mező | Rendezés |
|--------|------|----------|
| ISBN | `isbn` | igen |
| Cím | `title` | igen |
| Szerző(k) | `authors` (vesszővel elválasztva) | igen |
| Kiadási év | `publishYear` | igen |
| Műveletek | — | nem |

**Műveletek oszlop** (csak `ADMIN` és `DEMO` látja — `DEMO`-nál `MutationButton` auto-disabled):
- **Szerkesztés** ikon gomb → step 5.13 form modal
- **Törlés** ikon gomb → step 5.13 törlés megerősítő modal

**Sorra kattintás** (bárhol a műveletek oszlopon kívül) → step 5.11 `BookDetailPanel` megnyitása.

**Téma:** `ag-theme-quartz` / `ag-theme-quartz-dark` — az alkalmazás aktuális dark/light mode állapotából, változásra automatikusan reagál.

**Default sort:** `title,asc`.

---

## State kezelés

`bookStore.ts` egyetlen `booksRefreshTrigger: number` értékkel — mutáció (létrehozás, szerkesztés, törlés) után a modalok incrementelik. Szűrő állapot helyi React state-ben él (`useState`).

`booksRefreshTrigger` változásakor: `gridApi.purgeInfiniteCache()` — a datasource objektum megmarad, a cache invalidálódik és a grid az elejéről tölt újra. Szűrőváltozáskor szintén `purgeInfiniteCache()` (egységes a Locations oldal implementációjával).

---

## Kulcs döntések

- AG Grid Infinite Row Model — egységes a Locations oldallal; a `category`, `language`, `publishYear` szűrők nem kerülnek a szűrősorba (a részletek panelen kereshető, MVP-ben elegendő a három fő szűrő)
- DEMO: látja a listát, a műveletek oszlopban `MutationButton` auto-disabled tooltip-pal
- `DELETED` státusz a szűrőből kizárva — konzisztens a backend Specification viselkedésével (step 5.6)

---

## Tech-debt

- **Mobilos oszlopoptimalizálás:** AG Grid oszlopok priorizálása kis képernyőn — Feature 5-ben nem prioritás, külön task-ként kezelendő (konzisztens a Locations oldal tech-debt-jével)

---

## Unit tesztek (React Testing Library + Vitest, axios mock-olva)

| Teszt | Elvárt |
|-------|--------|
| Könyvek megjelennek az API mock válasz alapján | grid sorok renderelve |
| ADMIN: szerkesztés és törlés gomb látható | gombok jelen vannak |
| VISITOR: műveletek oszlop nem látható | gombok hiányoznak |
| DEMO: gombok láthatók de disabled | `MutationButton` disabled állapot |
| Keresési szűrő alkalmazása → `search` query param megjelenik | API hívás ellenőrzése |
| Státusz szűrőben `DELETED` nem szerepel | Select options ellenőrzése |
