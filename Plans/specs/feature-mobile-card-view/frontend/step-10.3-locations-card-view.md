# Step 10.3 – Locations kártyás nézet mobilon

## Mit állít elő

- `LocationManagementPage.tsx` módosítás — breakpoint alapján Grid/Card nézet váltás a locations listánál, közös szűrő/sort állapot a kettő felett; Rooms panel mindkét nézet fölött megmarad, változatlanul; a grid nézet (AG Grid) `React.lazy()` + `Suspense` mögé kerül, hogy mobil viewporton a bundle-je se töltődjön le
- `src/components/locations/LocationCard.tsx` — kártya komponens egy helyszínhez
- `src/components/locations/LocationFilterBar.tsx` — mobil felső keresés/sort/szűrők sáv, debounce-olt kereséssel
- `src/components/locations/LocationFiltersSheet.tsx` — kiegészítő szűrők (shadcn `Sheet`)
- i18n kulcsok kiegészítése (`hu.json`, `en.json`)
- Backend: **nincs módosítás** — a meglévő `GET /api/locations` endpoint változatlan paraméterekkel
- Kapcsolódó unit tesztek

Függ: [10.1](step-10.1-shared-infra.md) (közös infra), 3.10 (Room form/delete modalok — újrahasznosítva), 3.11 (Location form/delete modalok — újrahasznosítva)

---

## Nézetváltás

`LocationManagementPage` a meglévő `useIsMobile()` (`src/hooks/use-mobile.tsx`, ld. 10.1) eredménye alapján dönt: mobil breakpoint alatt (`< 768px`) kártyalistát renderel (`InfiniteCardList` + `LocationCard`) a locations grid helyén, `≥ 768px`-nél a meglévő AG Grid-et — változatlanul, `React.lazy()` mögött.

A **Rooms panel** (step 3.9) mindkét nézetben megjelenik, felül, a szűrő/sort sáv fölött — a panel saját nyitott/csukott alapállapota (asztali: nyitva, mobil: csukva) és funkciója (room CRUD, per-room "+ Location" gomb) nem változik. Elrendezés mobilon fentről lefelé: Rooms panel → szűrő/sort sáv → kártyalista.

A szűrő és sort állapot **egy szinttel feljebb**, a `LocationManagementPage`-ben él, nem a grid vagy a kártya nézeten belül — így egy breakpoint-váltás nem veszíti el a felhasználó beállított szűrőit, csak a megjelenítési mód vált (ugyanaz a minta, mint 10.2-ben).

---

## Top szűrő/sort sáv (mindig látható, kártyás nézetben)

- **Keresés** szövegmező — a `name` paraméterre megy (contains, case-insensitive — a backend ma is részleges egyezést támogat, ld. API_DESIGN.md). Azonos szemantikájú a grid `name` oszlopfejléc szűrőjével (step 3.9, 2026-08-13 óta ugyancsak szabad szöveges contains, ld. lent „Kulcs döntések") — a kártya↔grid nézetváltás a keresést mindkét irányban egyszerű string-átadással viszi át. **Debounce: 300–400 ms**, ugyanaz a megfontolás, mint a Books keresésnél (10.2) — gépelés közben nem indul lekérdezés-áradat. Placeholder szöveg (`locations.searchPlaceholder`) jelzi, hogy a keresés a névre megy
- **Rendezés** select — Név A-Z/Z-A, Helyiség A-Z/Z-A, alapérték: Név A-Z (`name,asc`)
- **Szűrők** gomb — badge-dzsel jelzi az aktív (a keresésen kívüli) szűrők számát, megnyitja a `LocationFiltersSheet`-et
- **Találatszám** — a szűrő sáv alatt egy halvány "N találat" sor (backend `Page<T>` `totalElements`), ugyanaz a minta, mint a Books kártyás nézetben (10.2)

---

## Szűrők sheet

Shadcn `Sheet`, alulról felcsúszó, a keresésen kívüli mezőket tartalmazza:

- Helyiség (`roomId`) — dropdown, ugyanaz az adatforrás (`GET /api/rooms/all`), amit a Rooms panel is használ, nincs külön fetch
- Leírás (`description`, contains → `description` paraméter)

„Alkalmaz" gomb → bezárja a sheetet és lekérdezést indít az elejéről; „Törlés" → minden mezőt (a keresést is) alapállapotba állít. A sheet mezőin nincs debounce — az "Alkalmaz" gomb az explicit trigger.

Nincs kaszkádos "room szűkíti a name értékkészletét" viselkedés — a `name` mind a kártyás, mind a grid nézetben szabad szöveges mező (2026-08-13 óta a gridben is, ld. „Kulcs döntések"), nincs dropdown-értékkészlet, amit szűkíteni kellene.

---

## Kártya tartalom

Ugyanazok a mezők, mint a grid oszlopai (3.9) — nincs extra vagy hiányzó adat a két nézet között:

- Név
- Helyiség neve (`room.name`, badge)
- Leírás (ha van)
- Könyvek száma (`bookCount`, badge)

**Akciók** (`ADMIN` és `DEMO` látja, azonos szabályok a grid művelet oszlopával — ld. lentebb "DEMO szerepkör" szekció; `VISITOR` egyik gombot sem látja):
- Szerkesztés ikon gomb — plain `Button`, a meglévő `LocationFormModal`-t nyitja (3.11), `roomId` előre kitöltve és disabled; `DEMO`-nak is aktív (form-nyitó akció, a tényleges mentés a modalon belül van gate-elve)
- Törlés ikon gomb — csak akkor látható, ha `bookCount === 0` (ugyanaz a UX-szűrés, mint a gridben); a meglévő `LocationDeleteModal`-t nyitja (3.11); `MutationButton`, `DEMO`-nál auto-disabled tooltip-pal (a törlés maga a mutáció)

Kártyára koppintás (akciógombokon kívül) **nem vált ki eseményt** — a Locations feature-ben nincs részletek nézet/panel, ez konzisztens a grid sorra kattintás hiányával (step 3.9-ben nincs `onRowClicked` bekötve). A kártya emiatt **nem** kap interaktív szemantikát (nincs `role="button"`, nincs hover/active elevation, nincs chevron-ikon) — vizuálisan sem sugallhat kattinthatóságot, mert nincs mögötte akció.

### DEMO szerepkör

A Locations feature (Feature 3) eredetileg csak `ADMIN`/`VISITOR` jogosultságot ismert, a DEMO role a Feature 5 (step 5.2) globális Security szabálya szerint viszont a `/api/**` alá tartozó minden GET (a `/api/users/**` kivételével) elérhető DEMO tokennel is — így `GET /api/locations` DEMO-nak is 200-at ad. Eddig ez a frontend oldalon (grid: step 3.9, most: kártya) nem volt tükrözve — a DEMO user Locations oldalon eddig egyáltalán nem látott volna művelet-affordanciát, szemben a Books oldallal, ahol disabled tooltipes gombokat kap.

**Ez a step (10.3), valamint a step 3.9 és 3.11 is** frissül ennek megfelelően: a Locations grid művelet-oszlopa és a `LocationCard` akciógombjai DEMO-nak is megjelennek. A Szerkesztés (form-nyitó akció) plain `Button`-ként DEMO-nak is aktív, a tényleges mentés a modalon belül van gate-elve; a Törlés `MutationButton` mögött, DEMO-nál auto-disabled állapotban, tooltippel — ugyanúgy, mint a Books nézeteknél. Ez konzisztenciát ad a pályázati demó felhasználó számára: a Locations funkció is látszik létezni, nem tűnik el a szeme elől.

---

## „+ Új helyszín" / „+ Új helyiség" hozzáférés mobilon

**Nincs FAB.** Eltérés a 10.2 (Books) mintától: a Locations feature-ben tudatos tervezési döntés volt (step 3.11 implementációs megjegyzés), hogy nincs önálló, page-szintű "Új helyszín" belépési pont — egy location mindig egy roomhoz tartozik, ezért az egyetlen létrehozási útvonal a Rooms panel adott sorának "+ Location" gombja. Mivel a Rooms panel mobilon is megmarad (lásd fent), ez a belépési pont változatlanul elérhető — nincs szükség kártyás-nézet-specifikus pótlásra.

A Rooms panel saját "Új room" gombja (panel fejlécében) szintén változatlan mindkét nézetben.

---

## Adatlekérés

`useInfiniteBackendList` (10.1) hívja a meglévő `locationApi.ts` listázó függvényét, ugyanazokkal a paraméterekkel, amiket a grid embedded/oszlopfejléc filterei már használnak: `name`, `roomId`, `description`, `sort`, `page`, `size` (kártyás nézetben 20).

A Rooms panel adatforrása (`GET /api/rooms/all`) és a szűrők sheet room dropdown-ja nézettől függetlenül, egyszer töltődik be oldal betöltésekor / `locationsRefreshTrigger` változásakor — ez a fetch már ma is független a grid vs. kártya kérdéstől (step 3.9, 1. fetch).

**`GET /api/locations/all` nem hívódik** — sem a kártyás, sem a grid nézetben. A step 3.9-ben ez a fetch eredetileg a grid `name` oszlopfejléc dropdown-jának értékkészletét töltötte fel; miután a `name` szűrő ott is szabad szöveges contains lett (2026-08-13, ld. step 3.9 „Name szűrő free textté alakítása" megjegyzés), a hívásnak nincs többé fogyasztója, és a kódból is törölve lett.

**Mutáció utáni frissítés** (ld. 10.1 "Mutáció utáni frissítés" szekció):
- **Room vagy location létrehozása** után: `locationsRefreshTrigger` inkrementálódik, a reset kulcs része — a kártyalista az elejéről tölt újra, a Rooms panel `locationCount`-jai frissülnek
- **Location szerkesztése** után: a modal a friss `LocationResponse`-t közvetlenül átadja a hook `updateItem(id, updater)` hívásának — a kártyalista a helyén frissül, scroll-pozíció megmarad; a `locationsRefreshTrigger` emellett is inkrementálódik, hogy a Rooms panel (ami nem a kártyalista hook-ján keresztül fetch-el) frissüljön
- **Location törlése** után: a hook `removeItem(id)` távolítja el a kártyát; `locationsRefreshTrigger` inkrementálódik a Rooms panel `locationCount` frissítéséhez
- **Room szerkesztése/törlése** után: a kártyalistát nem érinti közvetlenül (a kártya `room.name` badge-je csak akkor változna, ha a szülő room átnevezve) — ezért `locationsRefreshTrigger` váltja ki a szokásos teljes reset-et, ez a ritkább művelet, nem UX-kritikus

---

## Kulcs döntések

- **Nincs backend módosítás** — a kártyalista pontosan ugyanazokat a query paramétereket küldi, amiket a grid oszlopfejléc filterei már ma is használnak
- **Rooms panel megmarad mindkét nézetben**, változatlan viselkedéssel — nem duplikáljuk a room-kezelést kártyás-specifikus UI-ban
- **Nincs FAB** — a meglévő, tudatosan egyetlen belépési pontú location-létrehozási UX (Rooms panel per-room "+" gombja) mobilon is elég, nincs mit pótolni
- A top **keresés mező szabad szöveges `name` contains** — **azonos szemantikájú a grid `name` szűrőjével** (2026-08-13-tól a grid is szabad szöveges contains, nem dropdown — ld. step 3.9), ezért a kártya↔grid kézfogás triviális string-átadás, nincs elveszett szűrés nézetváltáskor. A `roomId` grid-dropdown (alacsony elemszámú, valódi enumerálható halmaz) változatlanul dropdown maradt
- Grid és kártya nézet közös szűrő/sort state-et használ (`LocationManagementPage` szinten) — nézetváltásnál a beállítások megmaradnak
- Kártya mezői 1:1 megegyeznek a grid oszlopaival
- Kártyára koppintás nem nyit semmit, és nem is sugall interakciót — nincs Locations detail nézet, konzisztensen a griddel
- **DEMO szerepkör most már látja a művelet-affordanciákat** — a szerkesztés aktív (form-nyitó akció), a törlés `MutationButton` mögött disabled, konzisztensen a Books nézettel — ehhez a step 3.9 és 3.11 is frissül, ebben a branch-ben
- `GET /api/locations/all` teljesen kikerült (grid és kártya is) — csak a korábbi grid `name` dropdown-jához kellett, ami már nem létezik
- Keresőmező debounce-olt, sheet mezői nem
- Location szerkesztés/törlés után helyben-frissítés (`updateItem`/`removeItem`), room mutáció és location létrehozás után teljes reset marad
- Grid nézet `React.lazy()` mögött — mobilon a bundle-je le sem töltődik

---

## Megjegyzés — kapcsolódó tech-debt tétel

A `Plans/tech-debt.md` "AG Grid — mobilos oszlopoptimalizálás" tétele **lezárva (feltételesen), 2026-08-07** — miután mind a Books (10.2), mind ez a step (10.3) kártyás nézetre váltja a listákat mobil breakpoint alatt, ahol az AG Grid többé nem renderelődik. A lezárás a tervre vonatkozik: ha a Feature 10 (10.2+10.3) ténylegesen elkészül és mergelődik a kód-repóban, a tétel véglegesen okafogyott; ha a scope közben változik, a `tech-debt.md`-ben újranyitandó.

---

## Elfogadási kritériumok

**Unit tesztek** (Vitest + React Testing Library, axios mock):
- `LocationCard`: ADMIN esetén szerkesztés mindig aktív, törlés csak `bookCount === 0` esetén aktív; DEMO esetén a szerkesztés aktív (form-nyitó akció, a mentés a modalon belül gate-elt), a törlés `bookCount === 0` esetén látható, de disabled (`MutationButton` tooltip); VISITOR esetén egyik gomb sem látható
- `LocationCard`: nincs `role="button"`, kártyára kattintás nem vált ki eseményt
- `LocationFilterBar`: keresés mezőbe gyors, egymást követő gépelés csak egy lekérdezést indít (debounce)
- `LocationFiltersSheet`: „Alkalmaz" a megfelelő paraméterekkel (`roomId`, `description`) indít lekérdezést; „Törlés" alapállapotba állít
- `LocationManagementPage`: `< 768px`-nél Rooms panel + kártyalista, `≥ 768px`-nél Rooms panel + grid renderel (breakpoint mock)
- `LocationManagementPage`: kártyás nézetben `GET /api/locations/all` nem hívódik (axios mock spy)
- Nézetváltás (breakpoint átlépés) nem törli a beállított szűrő/sort állapotot
- Location szerkesztés után a kártyalista az érintett kártyát cseréli, a többi kártya és a scroll-pozíció nem változik (hook `updateItem` mock/spy)
- Location törlés után a kártyalista eltávolítja az érintett kártyát (hook `removeItem` mock/spy)

**Manuálisan:**
- Rooms panel mobilon alapból csukva, kártyalista fölött jelenik meg
- Room panel "+ Location" gombja megnyitja a meglévő `LocationFormModal`-t, `roomId` előre kitöltve és disabled
- DEMO tokennel a Locations kártyás nézetben a szerkesztés gomb aktív (form-nyitó akció), a törlés gomb (ha `bookCount === 0`) inaktív, tooltippel
- Location létrehozása/szerkesztése/törlése után a kártyalista és a Rooms panel `locationCount` badge-ei is frissülnek
- Rendezés select váltásra a lista az elejéről, új sorrendben tölt be
- Törlés gomb csak `bookCount === 0` esetén látható a kártyán
