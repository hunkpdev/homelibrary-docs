# Step 3.10 – Room form modalok

## Mit állít elő

- `src/components/rooms/RoomFormModal.tsx` — room létrehozás / szerkesztés modal
- `src/components/rooms/RoomDeleteModal.tsx` — room törlés megerősítő modal
- `src/api/roomApi.ts` kiegészítés — `createRoom`, `updateRoom` hívások
- `src/components/rooms/RoomFormModal.test.tsx` — unit teszt

---

## `RoomFormModal`

**Mezők:**

| Mező | Kötelező | Leírás |
|------|----------|--------|
| `name` | igen | Helyiség neve |
| `description` | nem | Opcionális megjegyzés |

- Létrehozás és szerkesztés ugyanaz a modal — a title változik ("Új helyiség" / "Helyiség szerkesztése")
- Szerkesztéskor a meglévő értékek előre kitöltve
- Submit: `POST /api/rooms` (create) vagy `PUT /api/rooms/{id}` (edit, `version` mezővel)
- Sikeres mentés után a lista frissül, modal bezárul

---

## `RoomDeleteModal`

- Megerősítő modal: "Biztosan törlöd a(z) {name} helyiséget?"
- Submit: `DELETE /api/rooms/{id}`
- 409 esetén hibaüzenet: "A helyiséghez aktív helyszínek tartoznak"
- Sikeres törlés után a lista frissül, modal bezárul

---

## UI

shadcn/ui komponensekkel:
- `Dialog` — modal wrapper
- `Input` — szöveges mezők
- `Button` — submit, mégse gombok, loading state jelzéssel (`disabled` + spinner amíg a kérés fut)

---

## Elfogadási kritériumok

**Unit tesztek** (`RoomFormModal.test.tsx`, axios mock-olva):
- Létrehozás formban üres mezők, szerkesztéskor előre kitöltve
- Hiányzó kötelező mező esetén submit nem indul
- Sikeres submit után a modal bezárul

**Manuálisan:**
- Room létrehozás, szerkesztés, törlés működik
- Törlés 409 esetén hibaüzenet látható
