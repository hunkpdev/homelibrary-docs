# Step 3.9 – Rooms + Locations oldal

## Mit állít elő

- `src/pages/LocationManagementPage.tsx` — rooms panel + locations grid oldal
- `src/api/roomApi.ts` — API hívások rooms-hoz (`GET`, `DELETE`)
- `src/api/locationApi.ts` — API hívások locations-hoz (`GET`, `DELETE`)
- `src/store/locationStore.ts` — Zustand store, `locationsRefreshTrigger` számlálóval
- `src/pages/LocationManagementPage.test.tsx` — unit teszt

---

## Funkcionális követelmények

### Adatlekérés

- **2 fetch** oldal betöltésekor, illetve `locationsRefreshTrigger` változásakor (**2026-08-13, utólag módosítva** — eredetileg 3 fetch volt, ld. lent a „Name szűrő free textté alakítása" megjegyzést):
  1. `GET /api/rooms/all` — összes aktív room lapozás nélkül; a room panel adatforrása és a room dropdown szűrő feltöltéséhez
  2. AG Grid datasource init-kor automatikusan: `GET /api/locations?page=...&size=...&sort=...&name=...&roomId=...` — lapozott locations a gridhez, beágyazott `room` objektummal
- Szűrő / sort / lapozás változásakor csak a grid fetch (2.) fut újra

### State sync — Zustand refresh trigger

- `locationStore` egyetlen `locationsRefreshTrigger: number` értéket tárol
- Bármilyen room vagy location mutáció (létrehozás, szerkesztés, törlés) után a mutációt indító modal incrementeli a számlálót
- Mindkét fetch subscribe-ol rá: számláló változásakor újrafutnak
- Ezzel a rooms panel és a locations grid mindig szinkronban marad (pl. location törlés után a room `locationCount` frissül; room átnevezés után a grid `room.name` oszlopa frissül)

### Rooms panel (csak `ADMIN` látja a művelet gombokat)

- Összecsukható panel az oldal tetején (shadcn `Collapsible`)
  - Asztali nézetben alapból **nyitva**
  - Mobilon alapból **csukva**
- Kompakt lista: soronként egy room — `name`, `locationCount` badge, művelet gombok
- Panel fejlécben: **Új room** gomb (step 3.10 modalja)
- Soronként (csak `ADMIN`):
  - **Szerkesztés** ikon gomb (step 3.10 modalja)
  - **Törlés** ikon gomb — csak akkor látható, ha `locationCount === 0`; backend 409 védelme ettől függetlenül megmarad (step 3.10 modalja)
  - **+ Location** ikon gomb — step 3.11 modalját nyitja, `roomId` előre kitöltve

### Locations grid (`ADMIN` és `DEMO` látja a művelet gombokat, `DEMO` disabled)

- Flat lista, AG Grid Community Infinite Row Model
- Oszlopok: `name`, `description`, `room.name`, `bookCount`
- Szűrők:
  - `room.name` — dropdown (AG Grid custom filter, shadcn `Select`), értékek a rooms fetch-ből
  - `name` — szabad szöveges contains szűrő (AG Grid beépített szöveges filter, ugyanaz a minta, mint a `description` oszlop)
  - `description` — szöveges szűrő (AG Grid beépített)
- Sort: minden oszlopon, `name ASC` alapértelmezetten
- Lapozás: AG Grid Infinite Row Model — backend `Page<T>` válasz alapján
- Soronként:
  - **Szerkesztés** ikon gomb (step 3.11 modalja) — `ADMIN`-nál aktív, `DEMO`-nál `MutationButton` auto-disabled tooltip-pal, `VISITOR`-nál nem látható
  - **Törlés** ikon gomb — csak akkor látható, ha `bookCount === 0`; `ADMIN`-nál aktív, `DEMO`-nál `MutationButton` auto-disabled tooltip-pal, `VISITOR`-nál nem látható; backend 409 védelme ettől függetlenül megmarad (step 3.11 modalja)

**DEMO szerepkör (2026-08-07, utólag felvéve):** a globális Security szabály (step 5.2) szerint DEMO token minden `/api/**` GET végpontot elér (a `/api/users/**` kivételével), így `GET /api/locations` DEMO-nak is 200-at ad — a DEMO user tehát mindig is látta volna a listát. Kezdetben a művelet-oszlop emiatt VISITOR-nál és DEMO-nál egyaránt rejtve volt; ez a step mostantól megkülönbözteti a kettőt: DEMO látja a gombokat (disabled, `MutationButton` mintával, konzisztensen a Books listával — step 5.9), VISITOR nem lát semmit. A **Rooms panel** művelet gombjai (alább) egyelőre változatlanul `ADMIN`-only maradnak — ugyanez az aszimmetria technikailag ott is fennáll (DEMO `GET /api/rooms`-ra is 200-at kap), de ennek kiterjesztése nem volt része ennek a döntésnek; külön mérlegelendő.

**Name szűrő free textté alakítása (2026-08-13, utólag módosítva, Feature 10 M2 finding kapcsán):** eredetileg a `name` oszlop szűrője is dropdown volt (a locations fetch értékkészletéből, kaszkádosan szűkítve a kiválasztott room alapján). A step-10.3 (mobil kártyás nézet) tervezése után, a code review-ban felmerült M2 finding rámutatott, hogy a kártyás nézet (mobil, szabad szöveges keresés) és a grid (asztali, dropdown) közötti nézetváltás nem kompatibilis: egy részleges keresőszöveg nem jeleníthető meg dropdown-triggerként. A javítás a gyökérokot kezeli, nem a tünetet: mivel a backend `name` szűrője már eleve `LIKE %name%` contains match volt (sosem exact match), a grid oszlop is szabad szöveges contains szűrőre váltott, ugyanúgy, mint a `description` oszlop és a Books `title` szűrője. Ezzel a kártya↔grid kézfogás triviális (egyszerű string-átadás, a szűrés nézetváltáskor sem vész el), és a kizárólag a dropdown értékkészletét kiszolgáló `GET /api/locations/all` fetch törölve. A `room.name`/`roomId` dropdown változatlan maradt — az egy alacsony elemszámú, valódi enumerálható halmaz, ott a dropdown UX indokolt marad.

---

## UI

shadcn/ui + AG Grid Community:
- `Collapsible` — rooms panel összecsukásához
- `Button` — "Új room" gomb, szerkesztés / törlés / "+ location" ikon gombok
- `Badge` — `locationCount` és `bookCount` megjelenítéséhez
- AG Grid React Community — `room.name` oszlopon egyedi filter komponens (shadcn `Select`) oszlopfejlécbe integrálva, a `name`/`description` oszlopokon AG Grid beépített szöveges filter, lapozás (Infinite Row Model); téma (`ag-theme-quartz` / `ag-theme-quartz-dark`) az alkalmazás aktuális dark/light mode állapotából töltendő, és automatikusan reagál annak megváltozására

---

## Tech-debt

- **Mobilos oszlopoptimalizálás:** AG Grid oszlopok priorizálása kis képernyőn (pl. `description` elrejtése) — Feature 3-ban nem prioritás, külön task-ként kezelendő

---

## Elfogadási kritériumok

**Unit tesztek** (React Testing Library + Vitest, axios mock-olva):
- Rooms panel megjeleníti az összes aktív roomot `locationCount` badge-dzsel
- VISITOR tokennel a rooms panelben egyetlen művelet gomb sem látható (`locationCount` értékétől függetlenül)
- VISITOR tokennel a gridben egyetlen művelet gomb sem látható (`bookCount` értékétől függetlenül)
- ADMIN tokennel, `locationCount > 0` esetén a room törlés gombja nem látható
- ADMIN tokennel, `locationCount === 0` esetén a room törlés gombja látható
- ADMIN tokennel, `bookCount > 0` esetén a location törlés gombja nem látható
- ADMIN tokennel, `bookCount === 0` esetén a location törlés gombja látható
- DEMO tokennel a gridben a szerkesztés és törlés gomb látható, de `MutationButton` disabled tooltip-pal (a törlés láthatóságára a `bookCount === 0` feltétel változatlanul érvényes)
- DEMO tokennel a rooms panelben egyetlen művelet gomb sem látható (a Rooms panel egyelőre `ADMIN`-only marad)
- Room dropdown szűrőben az összes aktív room megjelenik
- A `name` szöveges szűrőbe gépelt részszöveg (részleges helyszínnév) contains egyezéssel szűri a listát
- Kártyás nézetből (10.3) érkező névszűrés grid nézetre váltva megmarad a `name` szűrőmezőben

**Manuálisan:**
- Light módban a grid `ag-theme-quartz`, dark módban `ag-theme-quartz-dark` témával jelenik meg
- Dark/light mode váltásakor a grid téma azonnal frissül, oldal újratöltés nélkül
- Asztali nézetben a rooms panel nyitva, mobilon csukva nyílik meg az oldal
- Room mutáció után (create/edit/delete) a locations grid automatikusan frissül
- Location mutáció után a rooms panel `locationCount` értékei automatikusan frissülnek
- Locations flat listában jelennek meg, `room.name` oszloppal
- Name szöveges szűrőbe gépelés a listát contains egyezéssel szűri; ha van kiválasztott room, a `name` és `roomId` szűrő AND kapcsolatban kombinálódik (a találatok az adott roomra szűkülnek)
- Lapozás működik
