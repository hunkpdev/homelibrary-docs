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
- AG Grid Infinite Row Model datasource automatikusan: `GET /api/books?page=...&size=...&sort=...&search=...&category=...`

Szűrő / sort / lapozás változásakor csak a grid datasource fetch fut újra.

---

## AG Grid táblázat

**Row Model:** Infinite Row Model — backend `Page<BookResponse>` válasz alapján lapoz.

**Oszlopok:**

| Oszlop | Mező | Rendezés | Embedded szűrő |
|--------|------|----------|----------------|
| ISBN | `isbn` | igen | szöveges |
| Cím | `title` | igen | szöveges |
| Szerző(k) | `authors` (vesszővel elválasztva) | igen | szöveges |
| Kiadási év | `publishYear` | igen | szöveges |
| Kategóriák | `categories` (vesszővel elválasztva) | nem | szöveges |
| Műveletek | — | nem | — |

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

- AG Grid Infinite Row Model — egységes a Locations oldallal; szűrés kizárólag embedded column filtereken keresztül, külső szűrősor nincs
- DEMO: látja a listát, a műveletek oszlopban `MutationButton` auto-disabled tooltip-pal
- `DELETED` könyvek sosem jelennek meg — a backend Specification mindig kizárja (step 5.6), a frontend nem küld `status` paramétert

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
| Kategória szűrő alkalmazása → `category` query param megjelenik | API hívás ellenőrzése |
