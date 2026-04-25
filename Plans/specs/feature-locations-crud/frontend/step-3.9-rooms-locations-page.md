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

- **3 fetch** oldal betöltésekor, illetve `locationsRefreshTrigger` változásakor:
  1–2. `Promise.all` párhuzamosan:
     - `GET /api/rooms/all` — összes aktív room lapozás nélkül; a room panel adatforrása és a room dropdown szűrő feltöltéséhez
     - `GET /api/locations/all` — összes aktív location lapozás nélkül, a location dropdown szűrő feltöltéséhez (nem a grid adata)
  3. AG Grid datasource init-kor automatikusan: `GET /api/locations?page=...&size=...&sort=...&name=...&roomId=...` — lapozott locations a gridhez, beágyazott `room` objektummal
- Szűrő / sort / lapozás változásakor csak a grid fetch (3.) fut újra

### State sync — Zustand refresh trigger

- `locationStore` egyetlen `locationsRefreshTrigger: number` értéket tárol
- Bármilyen room vagy location mutáció (létrehozás, szerkesztés, törlés) után a mutációt indító modal incrementeli a számlálót
- Mindhárom fetch subscribe-ol rá: számláló változásakor újrafutnak
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

### Locations grid (csak `ADMIN` látja a művelet gombokat)

- Flat lista, AG Grid Community Infinite Row Model
- Oszlopok: `name`, `description`, `room.name`, `bookCount`
- Szűrők: AG Grid custom filter komponensek (shadcn `Select`), oszlopfejlécbe integrálva
  - `room.name` — dropdown, értékek a rooms fetch-ből; kiválasztott room szűkíti a location dropdown értékkészletét
  - `name` — dropdown, értékek a locations fetch-ből; ha room van kiválasztva, csak az adott roomhoz tartozó locationök szerepelnek; egyébként minden
  - `description` — szöveges szűrő (AG Grid beépített)
- Sort: minden oszlopon, `name ASC` alapértelmezetten
- Lapozás: AG Grid Infinite Row Model — backend `Page<T>` válasz alapján
- Soronként (csak `ADMIN`):
  - **Szerkesztés** ikon gomb (step 3.11 modalja)
  - **Törlés** ikon gomb — csak akkor látható, ha `bookCount === 0`; backend 409 védelme ettől függetlenül megmarad (step 3.11 modalja)

---

## UI

shadcn/ui + AG Grid Community:
- `Collapsible` — rooms panel összecsukásához
- `Button` — "Új room" gomb, szerkesztés / törlés / "+ location" ikon gombok
- `Badge` — `locationCount` és `bookCount` megjelenítéséhez
- AG Grid React Community — custom filter komponensek (shadcn `Select`) oszlopfejlécbe integrálva, kaszkádos room→location szűrés, lapozás (Infinite Row Model); téma (`ag-theme-quartz` / `ag-theme-quartz-dark`) az alkalmazás aktuális dark/light mode állapotából töltendő, és automatikusan reagál annak megváltozására

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
- Room dropdown szűrőben az összes aktív room megjelenik
- Ha room van kiválasztva, a location dropdown csak az adott roomhoz tartozó locationöket mutatja
- Ha nincs room kiválasztva, a location dropdown az összes aktív locationt mutatja

**Manuálisan:**
- Light módban a grid `ag-theme-quartz`, dark módban `ag-theme-quartz-dark` témával jelenik meg
- Dark/light mode váltásakor a grid téma azonnal frissül, oldal újratöltés nélkül
- Asztali nézetben a rooms panel nyitva, mobilon csukva nyílik meg az oldal
- Room mutáció után (create/edit/delete) a locations grid automatikusan frissül
- Location mutáció után a rooms panel `locationCount` értékei automatikusan frissülnek
- Locations flat listában jelennek meg, `room.name` oszloppal
- Kaszkádos szűrés: room kiválasztása után a location dropdown értékkészlete szűkül
- Lapozás működik
