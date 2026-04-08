# Step 1.8 – Axios instance

## Mit állít elő

- `src/api/axiosInstance.ts` — konfigurált Axios instance

---

## Konfiguráció

- `baseURL`: `import.meta.env.VITE_API_BASE_URL` environment variable-ból
- `withCredentials: true` — HttpOnly refresh token cookie küldéséhez/fogadásához kötelező

---

## Environment variable

| Variable | Local érték | Prod érték |
|----------|-------------|------------|
| `VITE_API_BASE_URL` | `http://localhost:8080` | API Gateway endpoint URL (CDK outputból) |

Local értéke a `frontend/.env.local` fájlban — gitignorált.
A `frontend/.env.example` dokumentálja a szükséges variable-okat.

---

## Interceptorok

Ebben a stepben interceptor nincs — a 401 kezelés és refresh token rotation a Feature 2 (Auth) 2.9-es stepjében kerül be.

---

## Elfogadási kritériumok

- `axiosInstance` importálható, `baseURL` és `withCredentials` beállítva
- `GET /api/health` hívás az instance-on keresztül sikeres `local` profilon
