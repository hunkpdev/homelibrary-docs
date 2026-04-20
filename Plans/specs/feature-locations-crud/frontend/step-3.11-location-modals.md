# Step 3.11 – Location form modalok

## Mit állít elő

- `src/components/locations/LocationFormModal.tsx` — location létrehozás / szerkesztés modal
- `src/components/locations/LocationDeleteModal.tsx` — location törlés megerősítő modal
- `src/api/locationApi.ts` kiegészítés — `createLocation`, `updateLocation` hívások
- `src/components/locations/LocationFormModal.test.tsx` — unit teszt

---

## `LocationFormModal`

**Mezők:**

| Mező | Kötelező | Leírás |
|------|----------|--------|
| `name` | igen | Tárolási hely neve |
| `roomId` | igen | Autocomplete dropdown meglévő room nevekből, szerkeszthető placeholder (szabad szöveg új room esetén) — POST-nál aktív, PUT-nál disabled (lásd ADR-007) |
| `description` | nem | Opcionális megjegyzés |

- Létrehozás és szerkesztés ugyanaz a modal — a title változik ("Új tárolási hely" / "Tárolási hely szerkesztése")
- Groupolt nézetben room fejlécéről megnyitva: `roomId` előre kitöltve és disabled
- Szerkesztéskor a meglévő értékek előre kitöltve, `roomId` disabled
- Submit: `POST /api/locations` (create) vagy `PUT /api/locations/{id}` (edit, `version` mezővel)
- Sikeres mentés után a lista frissül, modal bezárul

---

## `LocationDeleteModal`

- Megerősítő modal: "Biztosan törlöd a(z) {name} tárolási helyet?"
- Submit: `DELETE /api/locations/{id}`
- A törlés gomb csak `bookCount === 0` esetén jelenik meg a táblázatban (UX szűrő) — a backend 409 védelme ettől függetlenül megmarad
- 409 esetén hibaüzenet: "A tárolási helyhez aktív könyvek tartoznak"
- Sikeres törlés után a lista frissül, modal bezárul

---

## UI

shadcn/ui komponensekkel:
- `Dialog` — modal wrapper
- `Input` — szöveges mezők
- `Combobox` — `roomId` autocomplete dropdown (meglévő room nevek + szabad szöveg új roomhoz)
- `Button` — submit, mégse gombok, loading state jelzéssel (`disabled` + spinner amíg a kérés fut)

---

## Elfogadási kritériumok

**Unit tesztek** (`LocationFormModal.test.tsx`, axios mock-olva):
- Létrehozás formban üres mezők, szerkesztéskor előre kitöltve
- Groupolt nézetből megnyitva `roomId` előre kitöltve és disabled
- Szerkesztéskor `roomId` disabled
- Hiányzó kötelező mező esetén submit nem indul
- Sikeres submit után a modal bezárul

**Manuálisan:**
- Location létrehozás room fejlécéről: `roomId` előre kitöltve
- Location létrehozás "Új tárolási hely" gombbal: `roomId` autocomplete működik
- Location törlés 409 esetén hibaüzenet látható
