# Refresh Token Rotation – Implementációs Specifikáció

> **Célközönség:** Claude Code (Sonnet) – ez a dokumentum a megvalósítás minden részletét tartalmazza.
> **Kapcsolódó dokumentumok:** ADR-002 (auth stratégia), ARCHITECTURE.md (biztonsági megfontolások)
> **Scope:** Backend (Spring Boot) + Frontend (React/Axios)

---

## Összefoglaló

Minden sikeres `POST /api/auth/refresh` híváskor a szerver:
1. Ellenőrzi a bejövő refresh tokent
2. Érvényteleníti a régit
3. Generál egy újat
4. Az új tokent HttpOnly cookie-ban küldi vissza

A frontend Axios interceptorja biztosítja, hogy párhuzamos 401-es válaszok esetén is csak egyetlen refresh hívás induljon.

---

## Backend módosítások

### A refresh endpoint flow-ja

**Endpoint:** `POST /api/auth/refresh`

```
1. Cookie → refresh token kiolvasás
2. BCrypt match a DB-ben tárolt hash-sel
3. Ha egyezik:
   a. Új refresh token generálása (SecureRandom, UUID vagy opaque string)
   b. BCrypt hash számítása az új refresh tokenből
   c. DB UPDATE users SET
        refresh_token_hash = <új hash>,
        refresh_token_expires_at = NOW() + 7 nap
      WHERE id = <user id>
   d. Új access token kiadása
   e. Set-Cookie header az új refresh tokennel
4. Response: { accessToken, expiresIn }
   + Set-Cookie (lásd lent)
```

### Cookie beállítások

A refresh token cookie-t **minden** alábbi attribútummal kell beállítani — a login, a refresh és a logout endpoint is ugyanezt a cookie konfigurációt használja:

```
Set-Cookie: refreshToken=<token>;
  HttpOnly;
  Secure;
  SameSite=Strict;
  Path=/api/auth;
  Max-Age=604800
```

| Attribútum | Érték | Miért |
|------------|-------|-------|
| `HttpOnly` | igen | JavaScript nem olvashatja (XSS védelem) |
| `Secure` | igen | Csak HTTPS-en megy ki |
| `SameSite` | `Strict` | Cross-origin kéréshez a böngésző nem csatolja (CSRF védelem) |
| `Path` | `/api/auth` | A cookie a login, refresh és logout endpointokra is kimegy — szükséges, mert a login beállítja, a refresh cseréli, a logout törli. Az `/api/auth` alá csak auth endpointok esnek, nincs biztonsági kockázat. |
| `Max-Age` | `604800` | 7 nap másodpercben |

### Spring Boot implementációs részletek

**Cookie létrehozás helper metódus** (használandó login és refresh endpointban is):

```java
private ResponseCookie createRefreshTokenCookie(String token) {
    return ResponseCookie.from("refreshToken", token)
        .httpOnly(true)
        .secure(true)
        .sameSite("Strict")
        .path("/api/auth")
        .maxAge(Duration.ofDays(7))
        .build();
}
```

**Cookie törlés** (logout endpointban):

```java
private ResponseCookie deleteRefreshTokenCookie() {
    return ResponseCookie.from("refreshToken", "")
        .httpOnly(true)
        .secure(true)
        .sameSite("Strict")
        .path("/api/auth")
        .maxAge(0)  // azonnal lejár → böngésző törli
        .build();
}
```

**Fontos:** A törlő cookie-nak **ugyanazokat az attribútumokat** kell tartalmaznia (path, domain, secure, sameSite), mint az eredetinek — különben a böngésző nem ismeri fel, hogy melyik cookie-t kell törölnie.

### Lokális fejlesztés (HSQLDB, local profile)

A `Secure` attribútum miatt a cookie `http://localhost`-on **nem fog elmenni**. A `local` Spring profile-ban:

```java
@Value("${app.cookie.secure:true}")
private boolean cookieSecure;

// application-local.yml:
// app.cookie.secure: false
```

A `SameSite=Strict` nem okoz gondot lokálisan, mert a frontend és backend ugyanarról az originről megy (localhost).

---

## Frontend módosítások

### Axios interceptor – refresh race condition kezelés

A probléma: ha egyszerre több API hívás kap 401-et (lejárt access token), mindegyik megpróbálhatja a refresh-t. De a rotation miatt a második refresh hívás már érvénytelen tokennel megy → 401 → felesleges kijelentkezés.

**Megoldás:** Egyetlen refresh promise, amit az összes várakozó kérés megvár.

```typescript
// src/api/axiosInstance.ts

let isRefreshing = false;
let refreshPromise: Promise<string> | null = null;

axiosInstance.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Csak 401-re reagálunk, és csak egyszer próbálkozunk per kérés
    if (error.response?.status !== 401 || originalRequest._retry) {
      return Promise.reject(error);
    }

    // Ha a refresh endpoint maga adta a 401-et → session lejárt, kijelentkezés
    if (originalRequest.url === '/api/auth/refresh') {
      // Redirect to login
      window.location.href = '/login';
      return Promise.reject(error);
    }

    originalRequest._retry = true;

    // Ha már fut egy refresh, várjuk meg azt
    if (isRefreshing && refreshPromise) {
      try {
        const newToken = await refreshPromise;
        originalRequest.headers['Authorization'] = `Bearer ${newToken}`;
        return axiosInstance(originalRequest);
      } catch {
        return Promise.reject(error);
      }
    }

    // Első 401 → indítjuk a refresh-t
    isRefreshing = true;
    refreshPromise = axiosInstance
      .post('/api/auth/refresh')       // cookie automatikusan megy
      .then((res) => {
        const newToken = res.data.accessToken;
        // Access token tárolása (Zustand store vagy memória)
        useAuthStore.getState().setAccessToken(newToken);
        return newToken;
      })
      .catch((refreshError) => {
        // Refresh is sikertelen → session vége
        useAuthStore.getState().clearAuth();
        window.location.href = '/login';
        throw refreshError;
      })
      .finally(() => {
        isRefreshing = false;
        refreshPromise = null;
      });

    try {
      const newToken = await refreshPromise;
      originalRequest.headers['Authorization'] = `Bearer ${newToken}`;
      return axiosInstance(originalRequest);
    } catch {
      return Promise.reject(error);
    }
  }
);
```

### Fontos: Axios withCredentials

Ahhoz, hogy az Axios egyáltalán elküldje és fogadja a cookie-kat cross-origin kéréseknél (a CloudFront domain ≠ API Gateway domain), be kell állítani:

```typescript
const axiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  withCredentials: true,  // KÖTELEZŐ a cookie küldéshez/fogadáshoz
});
```

**Figyelem:** Ha a `withCredentials: true` be van állítva, a szerver CORS konfigurációjában az `Access-Control-Allow-Origin` **nem lehet** `*` — explicit origin kell (pl. `https://homelibrary.example.com`). Az `Access-Control-Allow-Credentials: true` header is szükséges a szerver oldalon.

---

## Amit NEM kell implementálni

| Koncepció | Miért nem |
|-----------|-----------|
| Token family tracking | Túlzott komplexitás 2–5 felhasználóhoz. Egy használt token újrahasználása esetén nincs automatikus family invalidáció — egyszerűen 401-et kap. |
| Refresh token blacklist tábla | Nem szükséges, mert a `users` táblában mindig csak az aktuálisan érvényes hash van — a régi implicit érvénytelen. |
| Access token blacklist / revoke | Lambda-n nincs perzisztens in-memory állapot instance-ok között. Az access token 15 perces TTL-je elfogadható kockázat. |

---

## Tesztelési szempontok

### Unit tesztek (backend)

1. **Refresh happy path:** érvényes refresh token → új access token + új refresh token cookie + DB-ben új hash
2. **Érvénytelen refresh token:** nem egyezik a DB-ben tárolt hash → 401, DB változatlan
3. **Lejárt refresh token:** `refresh_token_expires_at` < now → 401, DB változatlan
4. **Rotation ellenőrzés:** refresh után a régi tokennel újra hívva → 401 (mert a DB-ben már az új hash van)
5. **Expiry újraszámolás:** refresh után a `refresh_token_expires_at` = now + 7 nap (nem az eredeti login időpontjától számolva)
6. **Cookie attribútumok:** a válasz `Set-Cookie` headerjében ellenőrizni: `HttpOnly`, `Secure`, `SameSite=Strict`, `Path=/api/auth`, `Max-Age=604800`
7. **Logout cookie törlés:** logout után a cookie `Max-Age=0`-val jön vissza

### Frontend tesztek

1. **Egyetlen refresh hívás:** 3 párhuzamos 401 → pontosan 1 refresh kérés indul (mock-olható)
2. **Sikeres retry:** a várakozó kérések az új access tokennel újraindulnak
3. **Refresh 401:** ha a refresh endpoint is 401-et ad → redirect /login-ra, nem végtelen loop
