# Step 2.8 – `POST /api/auth/logout` + integrációs teszt

## Mit állít elő

- `AuthService` bővítése `logout` metódussal
- `AuthController` bővítése `POST /api/auth/logout` endpointtal
- `AuthServiceTest` bővítése
- `AuthControllerTest` bővítése
- `com.homelibrary.AuthIntegrationTest` — `@SpringBootTest` integrációs teszt

---

## `AuthService` — `logout` metódus flow

1. Az autentikált user kinyerése a `SecurityContextHolder`-ből — a `JwtAuthenticationFilter` már betöltötte érvényes Bearer token esetén
2. Ha van autentikált user → DB törlés: `refreshTokenHash = null`, `refreshTokenExpiresAt = null`
3. Ha nincs autentikált user (pl. lejárt access token) → DB-t nem bántjuk, a refresh token úgyis lejár magától

A cookie törlése minden esetben megtörténik — a controlleren.

---

## `AuthController` — `POST /api/auth/logout`

- `@Operation` és `@ApiResponse` annotációk
- `AuthService.logout()` hívása
- Refresh token törlő cookie beállítása a response-ban minden esetben (a 2.6-ban definiált delete helper metódussal: `Max-Age=0`, üres érték, azonos attribútumok — `HttpOnly`, `Secure`, `SameSite=Strict`, `Path=/api/auth`)
- `204 No Content` visszaadása

---

## Integrációs teszt (`AuthIntegrationTest`)

`@SpringBootTest` — teljes Spring context, HSQLDB, valós HTTP réteg (`MockMvc`). Az auth flow end-to-end validálása:

1. `POST /api/auth/login` admin credentials-szel → `200 OK`, access token + refresh token cookie
2. `POST /api/auth/refresh` az 1. lépés refresh cookie-jával → `200 OK`, új access token + új refresh token cookie
3. `POST /api/auth/refresh` az 1. lépés **régi** refresh cookie-jával → `401` (rotation ellenőrzés — a DB-ben már az új hash van)
4. `POST /api/auth/logout` az 1. lépés access tokenjével → `204 No Content`, `Set-Cookie: Max-Age=0`
5. `POST /api/auth/refresh` a 4. lépés után → `401` (DB-ben már null a hash)

---

## Elfogadási kritériumok

**Unit tesztek** (`AuthServiceTest` bővítése):
- Autentikált userrel logout → `refreshTokenHash = null`, `refreshTokenExpiresAt = null` a DB-ben
- Nem autentikált hívás esetén logout → nem dob hibát, DB változatlan

**Unit tesztek** (`AuthControllerTest` bővítése, MockMvc):
- `POST /api/auth/logout` → `204 No Content`, `Set-Cookie` header: `Max-Age=0`, üres érték, `HttpOnly`, `SameSite=Strict`, `Path=/api/auth`

**Integrációs teszt** (`AuthIntegrationTest`):
- A fenti 5 lépéses flow hibamentesen lefut

**Manuálisan:**
- Login → refresh → logout → refresh → `401`
- `mvn clean package` hiba nélkül lefut
