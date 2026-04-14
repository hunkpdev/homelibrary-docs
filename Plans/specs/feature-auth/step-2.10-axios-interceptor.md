# Step 2.10 – Axios interceptor

## Mit állít elő

- `src/api/axiosInstance.ts` bővítése — request interceptor + response interceptor (a fájl az 1.8-as stepben készült)

---

## Request interceptor

Minden kimenő kérésen automatikusan beállítja az `Authorization` headert:
- `useAuthStore.getState().accessToken` kiolvasása
- Ha van token: `Authorization: Bearer <token>` header hozzáadása
- Ha nincs token: header érintetlen marad

---

## Response interceptor — 401 kezelés és race condition védelem

**A probléma:** Ha egyszerre több API hívás kap 401-et (lejárt access token), mindegyik megpróbálhatja a refresh-t. A rotation miatt a második refresh hívás már érvénytelen tokennel megy → `401` → felesleges kijelentkezés.

**Megoldás:** Module-szintű `isRefreshing` flag és egyetlen `refreshPromise` — az összes várakozó kérés ezt a promise-t várja meg.

**Interceptor logika:**

1. Ha a válasz nem `401`, vagy a kérés már retry-olt (`_retry` flag) → továbbengedés / reject
2. Ha a refresh endpoint maga adta a `401`-et → session lejárt: `clearAuth()`, redirect `/login` (`window.location.href` — az interceptor React komponens fán kívül fut, nem React Router)
3. Ha már fut egy refresh (`isRefreshing === true`) → a meglévő `refreshPromise`-ra await, majd az eredeti kérés újraküldése az új access tokennel
4. Ha ez az első `401` → refresh indítása:
   - `POST /api/auth/refresh` (cookie automatikusan megy, `withCredentials: true` az 1.8-as stepből)
   - Sikeres: `setAccessToken(newToken)` a Zustand store-ban, eredeti kérés újraküldése
   - Sikertelen: `clearAuth()`, redirect `/login`
5. `finally` blokkban: `isRefreshing = false`, `refreshPromise = null`

---

## Elfogadási kritériumok

- 3 párhuzamos `401` → pontosan 1 refresh kérés indul
- Sikeres refresh után a várakozó kérések az új access tokennel újraindulnak
- Ha a refresh endpoint is `401`-et ad → redirect `/login`, nem végtelen loop
- Érvényes access tokennel küldött kérés `Authorization: Bearer <token>` headert tartalmaz
