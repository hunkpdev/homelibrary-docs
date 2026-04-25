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
| `name` | igen | Helyszín neve |
| `roomId` | igen | Combobox: meglévő room nevek kereshetők/szűrhetők — csak listából választható, új room az oldal tetején lévő "Új room" gombbal hozható létre — POST-nál aktív, PUT-nál disabled (lásd ADR-007) |
| `description` | nem | Opcionális megjegyzés |

- Létrehozás és szerkesztés ugyanaz a modal — a title változik ("Új helyszín" / "Helyszín szerkesztése")
- Groupolt nézetben room fejlécéről megnyitva: `roomId` előre kitöltve és disabled
- Szerkesztéskor a meglévő értékek előre kitöltve, `roomId` disabled
- Submit: `POST /api/locations` (create) vagy `PUT /api/locations/{id}` (edit, `version` mezővel)
- Sikeres mentés után a lista frissül, modal bezárul

---

## `LocationDeleteModal`

- Megerősítő modal: "Biztosan törlöd a(z) {name} helyszínt?"
- Submit: `DELETE /api/locations/{id}`
- A törlés gomb csak `bookCount === 0` esetén jelenik meg a táblázatban (UX szűrő) — a backend 409 védelme ettől függetlenül megmarad
- 409 esetén hibaüzenet: "A helyszínhez aktív könyvek tartoznak"
- Sikeres törlés után a lista frissül, modal bezárul

---

## UI

shadcn/ui komponensekkel:
- `Dialog` — modal wrapper
- `Input` — szöveges mezők
- `Combobox` — `roomId` kereshető dropdown, csak listából választható érték fogadható el
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
- Location létrehozás room fejlécéről: modal megnyílik, `roomId` előre kitöltve és disabled, sikeres mentés után a grid frissül
- Location szerkesztése (ActionCell ceruza): mezők előtöltve, `roomId` disabled, sikeres mentés után a grid frissül
- Location törlése (ActionCell kuka): megerősítő modal jelenik meg, sikeres törlés után a grid frissül
- Location törlés 409 esetén hibaüzenet látható

---

## Implementációs megjegyzés

A tervezett "Új helyszín" page-szintű gomb nem lett megvalósítva. A döntés indoka: egy location mindig konkrét roomhoz tartozik, ezért természetes belépési pont a rooms panel per-room `+` gombja. Egy második, párhuzamos létrehozási útvonal UX szempontból redundáns és zavaró lenne. A `roomId` combobox (kereshető dropdown) szintén elhagyható emiatt — a room mindig előre adott, a select disabled állapotban nyílik.
