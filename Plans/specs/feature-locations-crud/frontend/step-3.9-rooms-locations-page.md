# Step 3.9 – Rooms + Locations oldal

## Mit állít elő

- `src/pages/LocationManagementPage.tsx` — rooms + locations lista oldal
- `src/api/roomApi.ts` — API hívások rooms-hoz (GET, DELETE)
- `src/api/locationApi.ts` — API hívások locations-hoz (GET, DELETE)
- `src/pages/LocationManagementPage.test.tsx` — unit teszt

---

## Funkcionális követelmények

### Táblázat — groupolt nézet (default)
- Csoportok forrása: `GET /api/rooms` — minden aktív room megjelenik fejlécként, **üres roomok is** (nincs aktív location)
- Csoporton belüli sorok forrása: `GET /api/locations?roomId=...` — az adott room locationjei
- Location sorok oszlopai: `name`, `description`, `bookCount`
- Room fejléc mutatja: room `name`, `locationCount`
- Szűrők: oszlopfejlécbe integrált, hide-olható szűrősor — location `name`, `description` szűrhető; room szinten `name` szűrhető
- Sort: location sorok `name ASC` alapértelmezetten, oszlopfejlécre kattintva váltható
- Lapozás: location szinten, backend `page`/`size` paraméterekkel

### Táblázat — flat nézet
- Egyetlen lapozott lista az összes aktív locationről, `room.name` oszloppal kiegészítve
- Szűrők és sort azonos logikával mint groupolt nézetben

### Nézet váltás
- Toggle gomb az oldal tetején: groupolt ↔ flat

### Műveletek — room szint (csak `ADMIN`)
- Room fejlécen: **szerkesztés** ikon gomb, **törlés** ikon gomb, **Új location** gomb (step 3.10 modaljai)

### Műveletek — location szint (csak `ADMIN`)
- Soronként: **szerkesztés** ikon gomb, **törlés** ikon gomb (step 3.10 modaljai)

### Műveletek — oldal szint (csak `ADMIN`)
- **Új room** gomb az oldal tetején (step 3.10 modalja)

---

## UI

shadcn/ui komponensekkel:
- `Button` — "Új room" gomb, szerkesztés/törlés ikon gombok
- TanStack Table — oszlopfejlécbe integrált szűrősor, grouping, lapozás
- `Badge` — `locationCount` és `bookCount` megjelenítéséhez

---

## Elfogadási kritériumok

**Unit tesztek** (React Testing Library + Vitest, axios mock-olva):
- Groupolt nézetben minden room fejlécként jelenik meg, beleértve az üres roomokat
- VISITOR tokennel az "Új room", szerkesztés és törlés gombok nem láthatók
- Toggle gomb flat/groupolt nézetváltáskor a táblázat struktúrája megváltozik

**Manuálisan:**
- Oldal betöltésekor a rooms csoportosítva jelennek meg, locationjeikkel
- Üres room (nincs aktív locationje) megjelenik a fejlécében, de sor nélkül
- Szűrő beírásakor a táblázat frissül (backend hívás indul)
- Lapozás működik
