# Step 2.9 – Zustand auth store

## Mit állít elő

- `src/store/authStore.ts` — Zustand store az auth state kezeléshez

---

## Store state és actions

**State:**

| Mező | Típus | Leírás |
|---|---|---|
| `accessToken` | `string \| null` | In-memory tárolt access token — nem localStorage, nem sessionStorage |
| `user` | `AuthUser \| null` | Bejelentkezett user adatai |
| `isInitialized` | `boolean` | `false` induláskor — `true` miután az App.tsx elvégezte az első refresh kísérletet (akár sikeres, akár sikertelen). A `ProtectedRoute` ez alapján dönt, hogy renderel-e vagy loading spinnerben vár. |

**`AuthUser` típus:**

| Mező | Típus |
|---|---|
| `id` | `string` (UUID) |
| `username` | `string` |
| `role` | `'ADMIN' \| 'VISITOR'` |

**Actions:**

| Action | Leírás |
|---|---|
| `setAuth(accessToken, user)` | Login után: access token és user beállítása |
| `setAccessToken(token)` | Refresh után: csak az access token cseréje, user megmarad |
| `clearAuth()` | Logout vagy session lejárat: state törlése |
| `setInitialized()` | App.tsx hívja az első refresh kísérlet után (siker vagy kudarc egyaránt) — `isInitialized=true`. Nem kötött a `setAuth`/`clearAuth`-hoz, mert `clearAuth` mid-session is hívódhat. |

---

## Store elérése

| Kontextus | Mód |
|---|---|
| React komponensben | `useAuthStore(state => state.accessToken)` — reaktív, újrarenderel változáskor |
| React komponensen kívül (pl. Axios interceptor) | `useAuthStore.getState().accessToken` — egyszeri kiolvasás, nem reaktív |

**Javasolt selectorok:**

| Selector | Típus | Mire |
|---|---|---|
| `state => state.accessToken` | `string \| null` | Bearer token összerakásához |
| `state => state.user` | `AuthUser \| null` | User adatok megjelenítéséhez |
| `state => state.accessToken !== null` | `boolean` | Protected route ellenőrzéséhez |

---

## Fontos tervezési döntések

**Miért csak in-memory tárolás?**
Az access token nem kerül localStorage-ba vagy sessionStorage-ba — XSS támadás esetén onnan kiolvasható lenne. Az in-memory Zustand store oldal refresh esetén elvész, de az Axios interceptor ilyenkor automatikusan refresh-el a HttpOnly cookie alapján (step 2.10).

**Honnan jön az `AuthUser`?**
A login response body-ból (`accessToken`, `tokenType`, `expiresIn`) nem jön user adat — a user adatokat az access token JWT claim-jeiből kell dekódolni (`sub` = userId, `role` claim). A dekódolás client-side, könyvtár nélkül — a JWT payload Base64URL dekódolása elegendő, aláírás validálás nem szükséges (azt a szerver végzi).

---

## Elfogadási kritériumok

- `setAuth` után `accessToken` és `user` beállítva
- `setAccessToken` után csak `accessToken` változik, `user` megmarad
- `clearAuth` után `accessToken` és `user` `null`, `isInitialized` **nem változik**
- `setInitialized` után `isInitialized=true`
- Az `accessToken` nem kerül localStorage-ba vagy sessionStorage-ba
