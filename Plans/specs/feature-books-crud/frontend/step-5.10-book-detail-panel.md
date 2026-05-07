# Step 5.10 – Könyv részletek panel

## Mit állít elő

- `src/components/books/BookDetailPanel.tsx` — shadcn `Sheet` alapú slide-in részletek panel (read-only)
- `BookListPage.tsx` kiegészítése: sorra kattintáskor panel megnyitása

---

## UX döntés: panel vs. oldal

Shadcn `Sheet` (slide-in panel), nem külön route — a felhasználó a könyvlistán marad, a panel jobb oldalról csúszik be. Mobilon a Sheet full-screen overlay-ként működik. Közvetlen linkelés nem követelmény (MVP).

**Read-only:** minden szerkesztés a 5.12 edit formban (Dialog) történik — egységes, egyetlen hely minden módosításra.

---

## Panel tartalma

### Fejléc
- Borítókép (ha van) — Phase 1-ben `coverImageUrl` mindig `null`, a hely üres marad
- Cím, szerzők
- Státusz badge
- Bezárás gomb (X)

### Mezők (mind read-only)

| Mező | Megjelenítés |
|------|-------------|
| ISBN | szöveges |
| Alcím | szöveges (`subtitle`, ha van) |
| Kiadó | szöveges |
| Kiadási év | szöveges |
| Oldalszám | szöveges (`pageCount`, ha van) |
| Nyelv | szöveges |
| Kategóriák | badge-ek |
| Forrás | szöveges (`OSZK` / `MANUAL`) |
| Helyszín | `location.name` — `location.room.name` |
| Leírás | teljes szöveg (nem truncated) |
| Létrehozva | dátum |

### Akciók (csak `ADMIN` látja)

- **Szerkesztés** gomb → step 5.12 edit Dialog (panel nyitva marad mögötte)
- **Törlés** gomb → step 5.12 törlés megerősítő Dialog → sikeres törlés után panel bezárul, lista frissül

---

## Adatlekérés

- Panel megnyitáskor: `GET /api/books/{id}` — friss adat a szerverről
- Sikeres szerkesztés (5.12) után: `GET /api/books/{id}` újrahívás → panel frissül

---

## Kulcs döntések

- **Panel read-only** — státusz és helyszín módosítás az edit formba (5.12) kerül; a detail panel egyetlen felelőssége az adatok megjelenítése és a modal-ok indítása
- DEMO user: látja az összes adatot; szerkesztés és törlés gombok láthatók, de `MutationButton` auto-disabled tooltip-pal — egységes a grid műveletek oszlopával (5.9)

---

## Elfogadási kritériumok

- Sorra kattintás → panel megnyílik, `GET /api/books/{id}` adat megjelenik
- ADMIN: szerkesztés és törlés gomb látható és aktív
- DEMO: szerkesztés és törlés gomb látható, de disabled (`MutationButton` tooltip-pal)
- VISITOR: akció gombok nem láthatók
- Szerkesztés után panel automatikusan frissül
- Soft delete után panel bezárul, könyv eltűnik a gridből
