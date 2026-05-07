# Step 5.12 – Könyv felvétel form

## Mit állít elő

- `src/components/books/BookAddModal.tsx` — shadcn `Dialog` alapú könyv felvétel form
- `BookListPage.tsx` kiegészítése: „Új könyv" gomb a `BookAddModal` megnyitásához

---

## UX flow

```
[ Új könyv gomb ] → Dialog megnyílik
  ┌─ ISBN fázis ──────────────────────────────┐
  │  IsbnLookupPanel (4.6)                    │
  │  → found: form mezők előtöltve (OSZK)     │
  │  → not found: ISBN előtöltve, manuális mód│
  │  → skip: „Kézi bevitel" link              │
  └───────────────────────────────────────────┘
         ↓ onResult callback
  ┌─ Form fázis ──────────────────────────────┐
  │  Könyv adatok szerkesztése / kitöltése    │
  │  → Mentés (MutationButton)                │
  └───────────────────────────────────────────┘
```

---

## ISBN fázis

Az `IsbnLookupPanel` (step 4.6) `onResult` callbackje:
- **Találat** (`result` nem null): form fázisra vált, mezők előtöltve az OSZK adatokkal; `source` = `"OSZK"`
- **Nem találat** (`result` null): form fázisra vált, csak `isbn` előtöltve, többi mező üres; `source` = `"MANUAL"`
- **„Kézi bevitel"** link (ISBN fázis átugrása): form fázis üres mezőkkel, `isbn` üres; `source` = `"MANUAL"`

DEMO user: az `IsbnLookupPanel` elérhető (GET hívás megengedett); a form submit `MutationButton`-ja auto-disabled.

---

## Form mezők

| Mező | Komponens | Kötelező | Előtöltés OSZK-ból |
|------|-----------|----------|-------------------|
| Cím | `Input` | igen | `title` |
| Alcím | `Input` | nem | `subtitle` |
| ISBN | `Input` | nem | `isbn` |
| Szerzők | tag input (vesszővel elválasztott) | nem | `authors` |
| Kiadó | `Input` | nem | `publisher` |
| Kiadási év | `Input` (number) | nem | `publishYear` |
| Oldalszám | `Input` (number) | nem | `pageCount` |
| Nyelv | `Input` | nem | `language` |
| Kategóriák | tag input (vesszővel elválasztott) | nem | — |
| Helyszín | `Select` | nem | — |
| Leírás | `Textarea` | nem | — |

> `source` rejtett mező — értékét az ISBN fázis eredménye határozza meg, a felhasználó nem szerkeszti.
> `status` nem jelenik meg a formon — a service `AT_HOME`-ra állítja alapértelmezetten.

**Helyszín dropdown:** `GET /api/locations/all` — ha a `BookListPage` már betöltötte, a `bookStore`-ból újrafelhasználható; egyébként friss fetch.

---

## Submit

- `MutationButton` (step 5.9) — DEMO esetén auto-disabled, tooltip-pal
- `POST /api/books` — sikeres válasz (201) után:
  - Dialog bezárul
  - `booksRefreshTrigger` increment → lista frissül
- Validációs hiba (400) → hibaüzenet a form alatt

---

## Kulcs döntések

- `Dialog` (nem `Sheet`) — a sok mező scrollozható Dialog-ot igényel; a részletek panel (5.11) Sheet marad, egységes megkülönböztetés: olvasás = Sheet, szerkesztés = Dialog
- Szerzők és kategóriák: vesszővel elválasztott szövegből `List<String>` — `"Tolkien, J.R.R."` → `["Tolkien, J.R.R."]`; egyszerű MVP megoldás, tag-alapú input Fázis 2-ban
- `source` automatikusan kerül be a requestbe az ISBN fázis eredménye alapján — a felhasználó nem látja

---

## Elfogadási kritériumok

- ISBN scan/bevitel → találat → form mezők előtöltve, `source = OSZK`
- ISBN scan/bevitel → nem találat → ISBN előtöltve, többi mező üres, `source = MANUAL`
- „Kézi bevitel" link → form megnyílik üres mezőkkel
- Hiányzó cím → 400, hibaüzenet a form alatt
- Sikeres mentés → Dialog bezárul, lista frissül
- DEMO: submit gomb disabled, tooltip látható; ISBN lookup elérhető
