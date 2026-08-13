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

**DEMO szerepkör (2026-08-07, pontosítva 2026-08-13):** a step 3.9 grid (és a step 10.3 kártyalista) mostantól `DEMO`-nak is megjeleníti a szerkesztés/törlés gombokat. A két gomb viselkedése eltér: a **Szerkesztés** plain `Button` (form-nyitó akció) — `DEMO` tokennel a `LocationFormModal` megnyílik, mezők előtöltve, csak a tényleges mentés (submit) van gate-elve a modalon belül. A **Törlés** `MutationButton` auto-disabled állapotban — DEMO tokennel a `LocationDeleteModal` soha nem nyílik meg. A modal maga nem tud/nem is kell tudnia a DEMO szerepkörről; a submit gate-elése a `LocationFormModal`-on belüli mentés-hívás felelőssége.
