# Step 10.2 – Books kártyás nézet mobilon

## Mit állít elő

- `BookListPage.tsx` módosítás — breakpoint alapján Grid/Card nézet váltás, közös szűrő/sort állapot a kettő felett; a grid nézet (AG Grid) `React.lazy()` + `Suspense` mögé kerül, hogy mobil viewporton a bundle-je se töltődjön le
- `src/components/books/BookCard.tsx` — kártya komponens egy könyvhöz
- `src/components/books/BookFilterBar.tsx` — mobil felső keresés/sort/szűrők sáv, debounce-olt kereséssel
- `src/components/books/BookFiltersSheet.tsx` — kiegészítő szűrők (shadcn `Sheet`)
- i18n kulcsok kiegészítése (`hu.json`, `en.json`)
- Backend: **nincs módosítás** — a meglévő `GET /api/books` endpoint változatlan paraméterekkel
- Kapcsolódó unit tesztek

Függ: [10.1](step-10.1-shared-infra.md) (közös infra), 5.10–5.12 (meglévő detail panel / add / edit / delete dialogok — újrahasznosítva, nem duplikálva)

---

## Nézetváltás

`BookListPage` a meglévő `useIsMobile()` (`src/hooks/use-mobile.tsx`, ld. 10.1) eredménye alapján dönt: mobil breakpoint alatt (`< 768px`) kártyalistát renderel (`InfiniteCardList` + `BookCard`), `≥ 768px`-nél a meglévő AG Grid-et — változatlanul.

A grid nézet komponense `React.lazy()`-vel töltődik be, `Suspense` fallback-kel — mobilon, ahol sosem renderelődik, a chunk-ja le sem töltődik. Ez a legnagyobb, legolcsóbb teljesítmény-nyereség ebben a stepben, mert az AG Grid bundle mérete jelentős (ld. ADR-009), és pont a mobil célközönségnél számít.

A szűrő és sort állapot **egy szinttel feljebb**, a `BookListPage`-ben él, nem a grid vagy a kártya nézeten belül — így egy breakpoint-váltás (pl. elforgatás asztali/tablet közeli szélességen) nem veszíti el a felhasználó beállított szűrőit, csak a megjelenítési mód vált.

---

## Top szűrő/sort sáv (mindig látható, kártyás nézetben)

- **Keresés** szövegmező — a `title` paraméterre megy (contains, ugyanaz a szemantika, mint a grid cím oszlop embedded filtere). **Debounce: 300–400 ms** — a gépelés minden leütése nem indít önálló lekérdezést/reset-et, csak a gépelés szüneteltetése után egy. Placeholder szöveg (`books.searchPlaceholder`, i18n) egyértelműsíti, hogy a keresés csak címre megy (pl. "Keresés cím szerint…" / "Search by title…"), mert a felhasználó ösztönösen szerzőt is beírhatna
- **Rendezés** select — a grid rendezhető oszlopainak egy szűkített, mobilra kurált részhalmaza: Cím A-Z/Z-A, Szerző A-Z/Z-A, Kiadási év legújabb/legrégebbi, alapérték: Cím A-Z (`title,asc`). Az ISBN szerinti rendezés (a gridben elérhető) szándékosan kimarad — mobilon a select rövid legyen, és nincs valós use-case ISBN szerinti rendezésre
- **Szűrők** gomb — badge-dzsel jelzi az aktív (a keresésen kívüli) szűrők számát, megnyitja a `BookFiltersSheet`-et
- **Találatszám** — a szűrő sáv alatt egy halvány "N találat" sor, a backend `Page<T>` válasz `totalElements` mezőjéből — ez az egyetlen visszajelzés szűrés után, hogy a szűrő hatott

A `publishYear` mező az entitásban és a `BookResponse`-ban **Integer** — a „Kiadási év legújabb/legrégebbi" rendezés numerikus, nem lexikografikus. (Csak a *szűrő* paraméter String, prefix-egyezéssel — ld. `Plans/tech-debt.md` „Books API dokumentáció" tétel.)

---

## Szűrők sheet

Shadcn `Sheet`, alulról felcsúszó, a keresésen kívüli mezőket tartalmazza:

- ISBN (prefix egyezés → `isbn` paraméter)
- Szerző (contains → `authors` paraméter)
- Kategória (contains → `category` paraméter)
- Kiadási év (prefix → `publishYear` paraméter, ugyanaz a String-alapú prefix logika, mint a gridnél)

„Alkalmaz" gomb → bezárja a sheetet és lekérdezést indít az elejéről; „Törlés" → minden mezőt (a keresést is) alapállapotba állít. A sheet mezőin **nincs debounce** — az "Alkalmaz" gomb az explicit trigger, nem gépelés közben szűr.

Ha a szűrők alkalmazása után nincs találat, az `InfiniteCardList` üres-állapot üzenete utaljon a Szűrők sheetre (pl. "Nincs találat — próbálj más szűrőt").

---

## Kártya tartalom

Ugyanazok a mezők, mint a grid oszlopai (5.9) — nincs extra vagy hiányzó adat a két nézet között:

- Cím, szerző(k) (pontosvesszővel elválasztva)
- Kiadási év, kategóriák (badge-ek)
- ISBN (halványabb, másodlagos szöveg)

**Akciók** (csak `ADMIN` és `DEMO` látja, azonos szabályok a grid műveletek oszlopával):
- Szerkesztés ikon gomb — plain `Button`, `DEMO`-nak is aktív (form-nyitó akció, a tényleges mentés a modalon belül van gate-elve)
- Törlés ikon gomb — `MutationButton`, `DEMO`-nál auto-disabled tooltip-pal (a törlés maga a mutáció)
- Kártyára koppintás (akciógombokon kívül) → megnyitja a meglévő `BookDetailPanel`-t (5.10), ugyanúgy, mint a grid sorra kattintás

**Akadálymentesítés:** a kártya interaktív elemként viselkedik (`role="button"`, `tabIndex={0}`, `Enter`/`Space` a koppintással egyenértékű) — nemcsak `onClick`-es `div`, hogy billentyűzettel is elérhető legyen. A beágyazott akciógombok (szerkesztés/törlés) kattintása `stopPropagation()`-nel véd az ellen, hogy a kártya-szintű "megnyitás" is kiváltódjon.

---

## „+ Új könyv" hozzáférés mobilon

Floating action button (jobb alsó sarok, fixen a scroll fölött) — a meglévő hozzáadás `Dialog`-ot (5.11 — `BookListPage.tsx` kiegészítése az "Új könyv" gombbal) nyitja meg. A grid fejléc gombja (5.11) helyett ez a mobil megfelelője, mert a felső sáv már tele van a keresés/sort/szűrők elemekkel. `VISITOR` nem látja; a FAB plain `Button`, `DEMO`-nak is aktív — form-nyitó akció, a tényleges mentés a dialógon belül van gate-elve.

A `InfiniteCardList` alsó paddingja a FAB magasságával + margóval megnövelt, hogy a FAB ne takarja el az utolsó kártya akciógombjait vagy a "következő oldal" betöltés-jelzőt; iOS-en `env(safe-area-inset-bottom)` is figyelembe veendő a FAB pozicionálásánál.

---

## Adatlekérés

`useInfiniteBackendList` (10.1) hívja a meglévő `bookApi.ts` listázó függvényét, ugyanazokkal a paraméterekkel, amiket a grid embedded filterei már használnak: `isbn`, `title`, `authors`, `category`, `publishYear`, `sort`, `page`, `size` (kártyás nézetben 20).

**Mutáció utáni frissítés** (ld. 10.1 "Mutáció utáni frissítés" szekció):
- **Létrehozás** után: `booksRefreshTrigger` inkrementálódik, a reset kulcs része, a kártyalista az elejéről tölt újra
- **Szerkesztés** után: a modal a friss `BookResponse`-t közvetlenül átadja a hook `updateItem(id, updater)` hívásának — a kártyalista a helyén frissül, nincs teljes reset, nincs scroll-ugrás
- **Törlés** után: a modal a hook `removeItem(id)` hívásával távolítja el a kártyát a felhalmozott listából

---

## Kulcs döntések

- **Nincs backend módosítás** — a kártyalista pontosan ugyanazokat a query paramétereket küldi, amiket a grid embedded filterei már ma is használnak; a két nézet API-kontraktusa 100%-ban megegyezik
- A felső **keresés mező csak a `title` paraméterre megy**, nem kombinált cím+szerző OR-keresés — a backend jelenlegi paraméterei (isbn/title/authors/category/publishYear) egymástól függetlenek és AND kapcsolatban szűrnek; egy kombinált OR-keresés backend módosítást igényelne, ami ellentmond annak a célnak, hogy egy API-t használjunk módosítás nélkül. Szerző szerinti szűrés a Szűrők sheetben külön mezőként elérhető
- Grid és kártya nézet közös szűrő/sort state-et használ (`BookListPage` szinten) — nézetváltásnál a beállítások megmaradnak
- Kártya mezői 1:1 megegyeznek a grid oszlopaival
- A rendezési opciók szándékosan szűkebbek a grid sortolható oszlopainál (ISBN kimarad) — mobilon a select rövid legyen, nincs valós mobil use-case ISBN szerinti rendezésre
- A **szerző szerinti rendezés közelítés**: `Book.authors` JSON oszlop (`@JdbcTypeCode(SqlTypes.JSON)`), a `sort=authors` a szerializált tömböt szövegként rendezi — gyakorlatban ≈ első szerző szerinti ábécésorrend, nem normalizált szerző-rendezés. Ugyanez a viselkedés a griddel is (ott is sortolható az `authors` oszlop), a kártyás nézet csak explicit, jól látható select-opcióvá teszi, ezért gyakrabban fog lefutni
- „+ Új könyv" mobilon FAB, nem header gomb
- Keresőmező debounce-olt (300–400 ms), a sheet mezői nem (explicit "Alkalmaz" trigger)
- Szerkesztés/törlés után helyben-frissítés (`updateItem`/`removeItem`), csak létrehozás után teljes reset
- Grid nézet `React.lazy()` mögött — mobilon a bundle-je le sem töltődik

---

## Tech-debt / megjegyzés

- **Backend doksi-inkonzisztencia** — a `Design/API_DESIGN.md` és a step-5.5/5.6 backend specek még az eredeti kombinált `search` (OR cím/szerző) paramétert dokumentálják, de a step-5.9 (implementáció-szinkronizált) frontend spec szerint a tényleges API már külön `isbn`/`title`/`authors` paramétereket vár, és a `publishYear` is String-ként (prefix) viselkedik, nem Integerként. Ez a step a step-5.9-ben rögzített, tényleges kontraktust veszi alapul. Felvéve a `Plans/tech-debt.md`-be (2026-08-07, "Books API dokumentáció" tétel).

---

## Elfogadási kritériumok

**Unit tesztek** (Vitest + React Testing Library, axios mock):
- `BookCard`: ADMIN esetén szerkesztés/törlés aktív; DEMO esetén a szerkesztés aktív (form-nyitó akció, a mentés a modalon belül gate-elt), a törlés disabled (`MutationButton` tooltip); VISITOR esetén egyik gomb sem látható
- `BookCard`: `role="button"`, `tabIndex={0}`, `Enter`/`Space` megnyitja a `BookDetailPanel`-t; akciógombra kattintás nem váltja ki a kártya-szintű megnyitást (`stopPropagation`)
- `BookFilterBar`: keresés mezőbe gyors, egymást követő gépelés csak egy lekérdezést indít, a gépelés befejezése (debounce) után
- `BookFiltersSheet`: „Alkalmaz" a megfelelő paraméterekkel indít lekérdezést; „Törlés" alapállapotba állít; mezőbe gépelés önmagában (Alkalmaz nélkül) nem indít lekérdezést
- `BookListPage`: `< 768px`-nél kártyalista, `≥ 768px`-nél grid renderel (breakpoint mock)
- Nézetváltás (breakpoint átlépés) nem törli a beállított szűrő/sort állapotot
- Szerkesztés után a kártyalista az érintett kártyát cseréli, a többi kártya és a scroll-pozíció nem változik (hook `updateItem` mock/spy)
- Törlés után a kártyalista eltávolítja az érintett kártyát (hook `removeItem` mock/spy)

**Manuálisan:**
- Kártyára koppintás megnyitja a meglévő `BookDetailPanel`-t
- „+ Új könyv" FAB megnyitja a meglévő hozzáadás Dialogot, nem takarja el az utolsó kártyát
- Könyv létrehozása után a kártyalista az elejéről újratölt; szerkesztése/törlése után helyben frissül, scroll-pozíció megmarad
- Rendezés select váltásra a lista az elejéről, új sorrendben tölt be
- Szűrés után a "N találat" sor frissül
- Mobil hálózat-szimulációval: gyors gépelés a keresőmezőben nem indít lekérdezés-áradatot
- Hálózati hiba szimulálásával: első betöltés hibája teljes felületű hibaüzenetet mutat retry gombbal; következő oldal hibája lista alji hibasávot mutat, a meglévő kártyák megmaradnak
