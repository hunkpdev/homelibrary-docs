# Step 2.12 – Protected route wrapper

## Mit állít elő

- `src/components/ProtectedRoute.tsx` — route-szintű autentikáció + autorizáció
- `src/pages/ForbiddenPage.tsx` — 403 oldal
- `src/App.tsx` kiegészítése — auth inicializáció + route struktúra
- `src/components/ProtectedRoute.test.tsx` — unit teszt

---

## Auth inicializáció (`App.tsx`)

Az `App.tsx` mountolásakor — minden page load és F5 esetén — egyetlen explicit `POST /api/auth/refresh` hívás történik, az Axios interceptortól függetlenül:

- **Sikeres refresh** → `setAuth(token, user)` + `setInitialized()`
- **Sikertelen refresh** (nincs érvényes cookie) → `clearAuth()` + `setInitialized()`

Amíg az inicializáció fut: az alkalmazás loading spinnerben vár, semmi nem renderel.

Ez a mechanizmus állítja vissza a session-t page reload után (a HttpOnly refresh cookie böngészőben marad, a Zustand store nem).

---

## `ProtectedRoute` komponens

**Props:** `allowedRoles: Role[]`

**Döntési logika sorrendben:**

1. `!isInitialized` → loading spinner (init refresh még fut)
2. `!accessToken` → `<Navigate to="/login" />`
3. `!allowedRoles.includes(user.role)` → `<Navigate to="/403" />`
4. Minden rendben → `<Outlet />`

---

## `ForbiddenPage`

Egyszerű 403 oldal — "Nincs jogosultságod ehhez az oldalhoz" üzenet, vissza a főoldalra link.

---

## Route struktúra (`App.tsx`)

```
/login          → LoginPage (nyilvános)
/403            → ForbiddenPage (nyilvános)
/               → ProtectedRoute allowedRoles={['ADMIN', 'VISITOR']}
  /             → főoldal
/admin/*        → ProtectedRoute allowedRoles={['ADMIN']}
  /locations    → Locations oldal (feature 3, placeholder egyelőre)
  ...
```

---

## Elfogadási kritériumok

**Unit tesztek** (`ProtectedRoute.test.tsx`, React Testing Library + Vitest):
- `isInitialized=false` → loading spinner renderel
- `isInitialized=true`, `accessToken=null` → redirect `/login`
- `isInitialized=true`, `accessToken` van, role nem megfelelő → redirect `/403`
- `isInitialized=true`, `accessToken` van, role megfelelő → `<Outlet />` renderel

**Manuálisan:**
- `local` profilon bejelentkezve: védett oldal elérhető
- Oldal refresh után: session visszaáll, védett oldal elérhető marad
- VISITOR role-lal ADMIN-only oldalra navigálva → `/403` oldal jelenik meg
