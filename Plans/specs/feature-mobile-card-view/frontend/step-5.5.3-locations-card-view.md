# Step 5.5.3 – Locations kártyás nézet mobilon

## Mit állít elő

- `LocationManagementPage.tsx` módosítás — breakpoint alapján Grid/Card nézet váltás a locations listánál, közös szűrő/sort állapot a kettő felett; Rooms panel mindkét nézet fölött megmarad, változatlanul
- `src/components/locations/LocationCard.tsx` — kártya komponens egy helyszínhez
- `src/components/locations/LocationFilterBar.tsx` — mobil felső keresés/sort/szűrők sáv
- `src/components/locations/LocationFiltersSheet.tsx` — kiegészítő szűrők (shadcn `Sheet`)
- i18n kulcsok kiegészítése (`hu.json`, `en.json`)
- Backend: **nincs módosítás** — a meglévő `GET /api/locations` endpoint változatlan paraméterekkel
- Kapcsolódó unit tesztek

Függ: [5.5.1](step-5.5.1-shared-infra.md) (közös infra), 3.10 (Room form/delete modalok — újrahasznosítva), 3.11 (Location form/delete modalok — újrahasznosítva)

---

## Nézetváltás

`LocationManagementPage` a `useMediaQuery` (5.5.1) eredménye alapján dönt: mobil breakpoint alatt kártyalistát renderel (`InfiniteCardList` + `LocationCard`) a locations grid helyén, felette a meglévő AG Grid-et — változatlanul.

A **Rooms panel** (step 3.9) mindkét nézetben megjelenik, felül, a szűrő/sort sáv fölött — a panel saját nyitott/csukott alapállapota (asztali: nyitva, mobil: csukva) és funkciója (room CRUD, per-room "+ Location" gomb) nem változik. Elrendezés mobilon fentről lefelé: Rooms panel → szűrő/sort sáv → kártyalista.

A szűrő és sort állapot **egy szinttel feljebb**, a `LocationManagementPage`-ben él, nem a grid vagy a kártya nézeten belül — így egy breakpoint-váltás nem veszíti el a felhasználó beállított szűrőit, csak a megjelenítési mód vált (ugyanaz a minta, mint 5.5.2-ben).

---

## Top szűrő/sort sáv (mindig látható, kártyás nézetben)

- **Keresés** szövegmező — a `name` paraméterre megy (contains, case-insensitive — a backend ma is részleges egyezést támogat, ld. API_DESIGN.md). Eltér a grid `name` oszlopfejléc dropdown-választós UX-étől (step 3.9): mobilon egyszerűbb és gyorsabb egy szabad szöveges mező, mint egy lenyíló lista az összes helyszínnévvel
- **Rendezés** select — Név A-Z/Z-A, Helyiség A-Z/Z-A, alapérték: Név A-Z (`name,asc`)
- **Szűrők** gomb — badge-dzsel jelzi az aktív (a keresésen kívüli) szűrők számát, megnyitja a `LocationFiltersSheet`-et

---

## Szűrők sheet

Shadcn `Sheet`, alulról felcsúszó, a keresésen kívüli mezőket tartalmazza:

- Helyiség (`roomId`) — dropdown, ugyanaz az adatforrás (`GET /api/rooms/all`), amit a Rooms panel is használ, nincs külön fetch
- Leírás (`description`, contains → `description` paraméter)

„Alkalmaz" gomb → bezárja a sheetet és lekérdezést indít az elejéről; „Törlés" → minden mezőt (a keresést is) alapállapotba állít.

Nincs kaszkádos "room szűkíti a location dropdown értékkészletét" viselkedés (step 3.9-ben ez a grid `name` oszlop dropdown-jának értékkészletére vonatkozott) — mivel a keresés itt szabad szöveges mező, nincs mit szűkíteni.

---

## Kártya tartalom

Ugyanazok a mezők, mint a grid oszlopai (3.9) — nincs extra vagy hiányzó adat a két nézet között:

- Név
- Helyiség neve (`room.name`, badge)
- Leírás (ha van)
- Könyvek száma (`bookCount`, badge)

**Akciók** (csak `ADMIN` látja, azonos szabályok a grid művelet oszlopával — Locations feature-ben nincs DEMO szerepkör-eset):
- Szerkesztés ikon gomb — a meglévő `LocationFormModal`-t nyitja (3.11), `roomId` előre kitöltve és disabled
- Törlés ikon gomb — csak akkor látható, ha `bookCount === 0` (ugyanaz a UX-szűrés, mint a gridben); a meglévő `LocationDeleteModal`-t nyitja (3.11)

Kártyára koppintás (akciógombokon kívül) **nem vált ki eseményt** — a Locations feature-ben nincs részletek nézet/panel, ez konzisztens a grid sorra kattintás hiányával (step 3.9-ben nincs `onRowClicked` bekötve).

---

## „+ Új helyszín" / „+ Új helyiség" hozzáférés mobilon

**Nincs FAB.** Eltérés az 5.5.2 (Books) mintától: a Locations feature-ben tudatos tervezési döntés volt (step 3.11 implementációs megjegyzés), hogy nincs önálló, page-szintű "Új helyszín" belépési pont — egy location mindig egy roomhoz tartozik, ezért az egyetlen létrehozási útvonal a Rooms panel adott sorának "+ Location" gombja. Mivel a Rooms panel mobilon is megmarad (lásd fent), ez a belépési pont változatlanul elérhető — nincs szükség kártyás-nézet-specifikus pótlásra.

A Rooms panel saját "Új room" gombja (panel fejlécében) szintén változatlan mindkét nézetben.

---

## Adatlekérés

`useInfiniteBackendList` (5.5.1) hívja a meglévő `locationApi.ts` listázó függvényét, ugyanazokkal a paraméterekkel, amiket a grid embedded/oszlopfejléc filterei már használnak: `name`, `roomId`, `description`, `sort`, `page`, `size` (kártyás nézetben 20).

A Rooms panel adatforrása (`GET /api/rooms/all`) és a szűrők sheet room dropdown-ja nézettől függetlenül, egyszer töltődik be oldal betöltésekor / `locationsRefreshTrigger` változásakor — ez a fetch már ma is független a grid vs. kártya kérdéstől (step 3.9, 1–2. fetch).

`locationsRefreshTrigger` (meglévő Zustand store) a hook reset kulcsa — room vagy location mutáció után ugyanúgy frissül a kártyalista (és a Rooms panel), mint a grid `purgeInfiniteCache()`-e.

---

## Kulcs döntések

- **Nincs backend módosítás** — a kártyalista pontosan ugyanazokat a query paramétereket küldi, amiket a grid oszlopfejléc filterei már ma is használnak
- **Rooms panel megmarad mindkét nézetben**, változatlan viselkedéssel — nem duplikáljuk a room-kezelést kártyás-specifikus UI-ban
- **Nincs FAB** — a meglévő, tudatosan egyetlen belépési pontú location-létrehozási UX (Rooms panel per-room "+" gombja) mobilon is elég, nincs mit pótolni
- A top **keresés mező szabad szöveges `name` contains** — eltér a grid dropdown-alapú `name` szűrőjétől, mert mobilon a dropdown (összes helyszínnév felsorolva) rosszabb UX, mint egy gépelhető mező; a backend paraméter szemantikája nem változik
- Grid és kártya nézet közös szűrő/sort state-et használ (`LocationManagementPage` szinten) — nézetváltásnál a beállítások megmaradnak
- Kártya mezői 1:1 megegyeznek a grid oszlopaival
- Kártyára koppintás nem nyit semmit — nincs Locations detail nézet, konzisztensen a gridvel

---

## Megjegyzés — kapcsolódó tech-debt tétel

A `Plans/tech-debt.md`-ben nyitva álló **"AG Grid — mobilos oszlopoptimalizálás"** tétel (forrás: step-3.9) azt feltételezi, hogy a grid mobilon is renderelődik, csak kevesebb oszloppal. Az 5.5.2 (Books) és ez a step (Locations) után mobil viewporton **egyik oldal AG Grid-je sem renderelődik** — a breakpoint alatt mindig a kártyás nézet fut. Ez a tech-debt tétel emiatt várhatóan okafogyottá válik 5.5.3 lezárása után; érdemes lesz felülvizsgálni és lezárni a `Plans/tech-debt.md`-ben, de ez **nem ennek a stepnek a feladata** — itt csak jelzem, hogy ne maradjon árván a listában.

---

## Elfogadási kritériumok

**Unit tesztek** (Vitest + React Testing Library, axios mock):
- `LocationCard`: ADMIN esetén szerkesztés mindig, törlés csak `bookCount === 0` esetén aktív; VISITOR esetén egyik gomb sem látható
- `LocationFilterBar`: keresés mezőbe írás a `name` paraméterrel indít lekérdezést
- `LocationFiltersSheet`: „Alkalmaz" a megfelelő paraméterekkel (`roomId`, `description`) indít lekérdezést; „Törlés" alapállapotba állít
- `LocationManagementPage`: 767px alatt Rooms panel + kártyalista, 768px felett Rooms panel + grid renderel (breakpoint mock)
- Nézetváltás (breakpoint átlépés) nem törli a beállított szűrő/sort állapotot

**Manuálisan:**
- Rooms panel mobilon alapból csukva, kártyalista fölött jelenik meg
- Room panel "+ Location" gombja megnyitja a meglévő `LocationFormModal`-t, `roomId` előre kitöltve és disabled
- Location létrehozása/szerkesztése/törlése után a kártyalista és a Rooms panel `locationCount` badge-ei is frissülnek
- Rendezés select váltásra a lista az elejéről, új sorrendben tölt be
- Törlés gomb csak `bookCount === 0` esetén látható a kártyán
