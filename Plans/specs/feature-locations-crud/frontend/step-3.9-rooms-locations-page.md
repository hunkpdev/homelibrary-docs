# Step 3.9 – Rooms + Locations oldal

## Mit állít elő

- `src/pages/LocationManagementPage.tsx` — rooms + locations lista oldal
- `src/api/roomApi.ts` — API hívások rooms-hoz (GET, DELETE)
- `src/api/locationApi.ts` — API hívások locations-hoz (GET, DELETE)
- `src/pages/LocationManagementPage.test.tsx` — unit teszt

---

## Funkcionális követelmények

### Adatlekérés (mindkét nézetben)
- **2 párhuzamos fetch** oldal betöltésekor és szűrő/sort/lap változáskor:
  1. `GET /api/rooms` — összes aktív room (nem lapozott, egyetlen hívás)
  2. `GET /api/locations?page=...&size=...&sort=...&name=...` — lapozott locations, beágyazott `room` objektummal
- A toggle **nem indít új fetch-et** — csak a megjelenítési módot váltja a már betöltött adatokon

### Táblázat — groupolt nézet (default)
- Kliens csoportosít `location.room.id` alapján
- Minden aktív room megjelenik fejlécként, **üres roomok is** — a rooms fetch-ből jönnek, location nélkül
- Location sorok oszlopai: `name`, `description`, `bookCount`
- Room fejléc mutatja: room `name`, `locationCount`
- Szűrők: oszlopfejlécbe integrált, hide-olható szűrősor — location `name`, `description` szűrhető; room szinten `name` szűrhető
- Sort: location sorok `name ASC` alapértelmezetten, oszlopfejlécre kattintva váltható
- Lapozás: location szinten

### Táblázat — flat nézet
- Egyetlen lapozott lista az összes aktív locationről, `room.name` oszloppal kiegészítve
- Szűrők, sort és lapozás azonos logikával mint groupolt nézetben

### Nézet váltás
- Toggle gomb az oldal tetején: groupolt ↔ flat (csak megjelenítés vált, nincs új fetch)

### Műveletek — room szint (csak `ADMIN`)
- Room fejlécen: **szerkesztés** ikon gomb, **Új location** gomb (step 3.10 modaljai)
- **Törlés** ikon gomb csak akkor látható, ha `locationCount === 0` — backend 409 védelme ettől függetlenül megmarad

### Műveletek — location szint (csak `ADMIN`)
- Soronként: **szerkesztés** ikon gomb (step 3.11 modalja)
- **Törlés** ikon gomb csak akkor látható, ha `bookCount === 0` — backend 409 védelme ettől függetlenül megmarad

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
