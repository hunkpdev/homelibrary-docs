# Step 2.11 – Login oldal

## Mit állít elő

- `src/pages/LoginPage.tsx` — login oldal komponens
- `src/pages/LoginPage.test.tsx` — unit teszt

---

## Funkcionális követelmények

**Form mezők:** username + password, mindkettő kötelező

**Submit flow:**
1. `POST /api/auth/login` az axiosInstance-on keresztül
2. Sikeres válasz esetén:
   - JWT payload dekódolása (Base64URL, könyvtár nélkül) → `sub` (userId), `username`, `role` claim kinyerése
   - `setAuth(accessToken, { id, username, role })` a Zustand store-ban
   - Redirect a főoldalra (React Router `navigate`)
3. Hiba esetén: hibaüzenet megjelenítése a form alatt (`401` → "Hibás felhasználónév vagy jelszó")

**Már bejelentkezett user:** ha a store-ban van `accessToken`, a login oldal automatikusan redirectel a főoldalra — a login oldal nem érhető el bejelentkezett állapotban.

---

## UI

shadcn/ui komponensekkel:
- `Card` — form wrapper
- `Input` — username és password mezők
- `Button` — submit, loading state jelzéssel (`disabled` + spinner amíg a kérés fut)
- Hibaüzenet: form alatt, csak hiba esetén látható

---

## Elfogadási kritériumok

**Unit tesztek** (React Testing Library + Vitest, axios mock-olva):
- Form renderelődik username és password mezőkkel
- Érvényes credentials → `setAuth` meghívva, redirect a főoldalra
- Érvénytelen credentials (`401`) → hibaüzenet megjelenik, redirect nem történik
- Submit közben a gomb disabled állapotban van

**Manuálisan:**
- `local` profilon login admin credentials-szel → redirect a főoldalra
- Hibás jelszóval → hibaüzenet látható a form alatt
