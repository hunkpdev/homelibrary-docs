# Step 5.5.2 – Books kártyás nézet mobilon

## Mit állít elő

- `BookListPage.tsx` módosítás — breakpoint alapján Grid/Card nézet váltás, közös szűrő/sort állapot a kettő felett
- `src/components/books/BookCard.tsx` — kártya komponens egy könyvhöz
- `src/components/books/BookFilterBar.tsx` — mobil felső keresés/sort/szűrők sáv
- `src/components/books/BookFiltersSheet.tsx` — kiegészítő szűrők (shadcn `Sheet`)
- i18n kulcsok kiegészítése (`hu.json`, `en.json`)
- Backend: **nincs módosítás** — a meglévő `GET /api/books` endpoint változatlan paraméterekkel
- Kapcsolódó unit tesztek

Függ: [5.5.1](step-5.5.1-shared-infra.md) (közös infra), 5.10–5.12 (meglévő detail panel / add / edit / delete dialogok — újrahasznosítva, nem duplikálva)

---

## Nézetváltás

`BookListPage` a `useMediaQuery` (5.5.1) eredménye alapján dönt: mobil breakpoint alatt kártyalistát renderel (`InfiniteCardList` + `BookCard`), felette a meglévő AG Grid-et — változatlanul.

A szűrő és sort állapot **egy szinttel feljebb**, a `BookListPage`-ben él, nem a grid vagy a kártya nézeten belül — így egy breakpoint-váltás (pl. elforgatás asztali/tablet közeli szélességen) nem veszíti el a felhasználó beállított szűrőit, csak a megjelenítési mód vált.

---

## Top szűrő/sort sáv (mindig látható, kártyás nézetben)

- **Keresés** szövegmező — a `title` paraméterre megy (contains, ugyanaz a szemantika, mint a grid cím oszlop embedded filtere)
- **Rendezés** select — a grid rendezhető oszlopainak megfelelő kombinációk (Cím A-Z/Z-A, Szerző A-Z/Z-A, Kiadási év legújabb/legrégebbi), alapérték: Cím A-Z (`title,asc`)
- **Szűrők** gomb — badge-dzsel jelzi az aktív (a keresésen kívüli) szűrők számát, megnyitja a `BookFiltersSheet`-et

---

## Szűrők sheet

Shadcn `Sheet`, alulról felcsúszó, a keresésen kívüli mezőket tartalmazza:

- ISBN (prefix egyezés → `isbn` paraméter)
- Szerző (contains → `authors` paraméter)
- Kategória (contains → `category` paraméter)
- Kiadási év (prefix → `publishYear` paraméter, ugyanaz a String-alapú prefix logika, mint a gridnél)

„Alkalmaz" gomb → bezárja a sheetet és lekérdezést indít az elejéről; „Törlés" → minden mezőt (a keresést is) alapállapotba állít.

---

## Kártya tartalom

Ugyanazok a mezők, mint a grid oszlopai (5.9) — nincs extra vagy hiányzó adat a két nézet között:

- Cím, szerző(k) (pontosvesszővel elválasztva)
- Kiadási év, kategóriák (badge-ek)
- ISBN (halványabb, másodlagos szöveg)

**Akciók** (csak `ADMIN` és `DEMO` látja, azonos szabályok a grid műveletek oszlopával):
- Szerkesztés / Törlés ikon gomb — `DEMO`-nál `MutationButton` auto-disabled tooltip-pal
- Kártyára koppintás (akciógombokon kívül) → megnyitja a meglévő `BookDetailPanel`-t (5.10), ugyanúgy, mint a grid sorra kattintás

---

## „+ Új könyv" hozzáférés mobilon

Floating action button (jobb alsó sarok, fixen a scroll fölött) — a meglévő hozzáadás `Dialog`-ot (5.11) nyitja meg. A grid fejléc gombja (5.12) helyett ez a mobil megfelelője, mert a felső sáv már tele van a keresés/sort/szűrők elemekkel. `VISITOR` nem látja; `DEMO`-nál `MutationButton` auto-disabled.

---

## Adatlekérés

`useInfiniteBackendList` (5.5.1) hívja a meglévő `bookApi.ts` listázó függvényét, ugyanazokkal a paraméterekkel, amiket a grid embedded filterei már használnak: `isbn`, `title`, `authors`, `category`, `publishYear`, `sort`, `page`, `size` (kártyás nézetben 20).

`booksRefreshTrigger` (meglévő Zustand store) a hook reset kulcsa — könyv mutáció után ugyanúgy frissül a kártyalista, mint a grid `purgeInfiniteCache()`-e.

---

## Kulcs döntések

- **Nincs backend módosítás** — a kártyalista pontosan ugyanazokat a query paramétereket küldi, amiket a grid embedded filterei már ma is használnak; a két nézet API-kontraktusa 100%-ban megegyezik
- A felső **keresés mező csak a `title` paraméterre megy**, nem kombinált cím+szerző OR-keresés — a backend jelenlegi paraméterei (isbn/title/authors/category/publishYear) egymástól függetlenek és AND kapcsolatban szűrnek; egy kombinált OR-keresés backend módosítást igényelne, ami ellentmond annak a célnak, hogy egy API-t használjunk módosítás nélkül. Szerző szerinti szűrés a Szűrők sheetben külön mezőként elérhető
- Grid és kártya nézet közös szűrő/sort state-et használ (`BookListPage` szinten) — nézetváltásnál a beállítások megmaradnak
- Kártya mezői 1:1 megegyeznek a grid oszlopaival
- „+ Új könyv" mobilon FAB, nem header gomb

---

## Tech-debt / megjegyzés

- **Backend doksi-inkonzisztencia észlelve:** a `Design/API_DESIGN.md` és a step-5.5/5.6 backend specek még az eredeti kombinált `search` (OR cím/szerző) paramétert dokumentálják, de a step-5.9 (implementáció-szinkronizált) frontend spec szerint a tényleges API már külön `isbn`/`title`/`authors` paramétereket vár, és a `publishYear` is String-ként (prefix) viselkedik, nem Integerként. Ez a step a step-5.9-ben rögzített, tényleges kontraktust veszi alapul. A backend-oldali doksi (API_DESIGN.md, step-5.5, step-5.6) frissítése ettől független, önálló tech-debt tétel — érdemes felvenni a `Plans/tech-debt.md`-be.

---

## Elfogadási kritériumok

**Unit tesztek** (Vitest + React Testing Library, axios mock):
- `BookCard`: ADMIN esetén szerkesztés/törlés aktív; DEMO esetén látható, de disabled (`MutationButton` tooltip); VISITOR esetén nem látható
- `BookFilterBar`: keresés mezőbe írás a `title` paraméterrel indít lekérdezést
- `BookFiltersSheet`: „Alkalmaz" a megfelelő paraméterekkel indít lekérdezést; „Törlés" alapállapotba állít
- `BookListPage`: 767px alatt kártyalista, 768px felett grid renderel (breakpoint mock)
- Nézetváltás (breakpoint átlépés) nem törli a beállított szűrő/sort állapotot

**Manuálisan:**
- Kártyára koppintás megnyitja a meglévő `BookDetailPanel`-t
- „+ Új könyv" FAB megnyitja a meglévő hozzáadás Dialogot
- Könyv létrehozása/szerkesztése/törlése után a kártyalista frissül
- Rendezés select váltásra a lista az elejéről, új sorrendben tölt be
