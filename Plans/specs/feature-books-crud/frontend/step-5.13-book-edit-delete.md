# Step 5.13 – Könyv szerkesztés form és soft delete megerősítés

## Mit állít elő

- `src/components/books/BookEditModal.tsx` — shadcn `Dialog` alapú szerkesztés form
- `src/components/books/BookDeleteConfirmModal.tsx` — shadcn `Dialog` alapú törlés megerősítés

Mindkettő megnyitható a grid akció oszlopából (5.10) és a részletek panelből (5.11).

---

## `BookEditModal`

### Adatok betöltése

A modal nyitásakor a hívó a `BookResponse` objektumot adja át propként — nincs külön `GET /api/books/{id}` fetch, az adat már rendelkezésre áll (gridből vagy detail panelből).

### Form mezők

Az 5.12 add formmal megegyező mezők, ISBN lookup fázis nélkül. Értékek a meglévő `BookResponse`-ból töltődnek elő:

| Mező | Előtöltés |
|------|-----------|
| Cím | `book.title` |
| Alcím | `book.subtitle` |
| ISBN | `book.isbn` |
| Szerzők | `book.authors` |
| Kiadó | `book.publisher` |
| Kiadási év | `book.publishYear` |
| Oldalszám | `book.pageCount` |
| Nyelv | `book.language` |
| Kategóriák | `book.categories` |
| Státusz | `Select` (AT_HOME / LOANED) — `book.status` |
| Helyszín | `Select` (aktív locationök) — `book.location.id` |
| Leírás | `book.description` |

**`version`** rejtett — a requestben elküldi, a felhasználó nem látja.  
**`source`** nem módosítható — a requestben a jelenlegi érték kerül be.

### Submit

- `MutationButton` — DEMO esetén auto-disabled
- `PUT /api/books/{id}` — sikeres válasz (200) után:
  - Dialog bezárul
  - `booksRefreshTrigger` increment → grid frissül
  - Ha a detail panel nyitva van: `GET /api/books/{id}` újrahívás → panel frissül
- 409 (optimistic locking): hibaüzenet — *„Valaki más közben módosította ezt a könyvet. Zárd be és nyisd újra."*
- 400: validációs hibaüzenet a form alatt

---

## `BookDeleteConfirmModal`

Kis méretű Dialog, a könyv nevével a megerősítő szövegben:

> *„Biztosan törli a következő könyvet: **[title]**? A törölt könyv nem jelenik meg többé a listában."*

| Gomb | Típus | Működés |
|------|-------|---------|
| Mégse | `Button` (outline) | Dialog bezárul |
| Törlés | `MutationButton` (destructive) | `DELETE /api/books/{id}` |

Sikeres törlés (204) után:
- Dialog bezárul
- Ha a detail panel nyitva volt: bezárul
- `booksRefreshTrigger` increment → grid frissül (könyv eltűnik)

---

## Kulcs döntések

- **Nem töltünk le friss adatot a modal nyitásakor** — a grid / detail panel adata elegendő az előtöltéshez; optimistic locking védi a stale write-ot
- **`status` és `source` a felhasználó elől rejtett** — a form a metaadatokat szerkeszti, nem a könyv életciklus-állapotát
- **409 esetén nem törekszünk automatikus merge-re** — a user maga dönt (bezárja, újraolvassa, szerkeszti)

---

## Elfogadási kritériumok

- Edit modal előtöltve nyílik meg a meglévő adatokkal
- Sikeres mentés → grid és (ha nyitva) detail panel frissül
- 409 → érthető hibaüzenet, form nyitva marad
- Törlés megerősítés → sikeres DELETE → könyv eltűnik a gridből, detail panel bezárul
- DEMO: mindkét modalban a submit / törlés gomb disabled tooltip-pal
