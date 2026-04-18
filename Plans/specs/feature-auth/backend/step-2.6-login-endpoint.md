# Step 2.6 – `CookieProperties` + `POST /api/auth/login`

## Mit állít elő

- `com.homelibrary.config.CookieProperties` — `@ConfigurationProperties` az `app.cookie.*` property-khez
- `com.homelibrary.service.AuthService` — login üzleti logika (2.7 és 2.8-ban bővül)
- `com.homelibrary.controller.AuthController` — `POST /api/auth/login` endpoint (2.7 és 2.8-ban bővül)
- `com.homelibrary.dto.LoginRequest` — request DTO
- `com.homelibrary.dto.LoginResponse` — response DTO
- `com.homelibrary.service.AuthServiceTest` — unit teszt
- `com.homelibrary.controller.AuthControllerTest` — unit teszt (MockMvc)
- `application-local.yml` kiegészítés

---

## `CookieProperties`

`@ConfigurationProperties(prefix = "app.cookie")` + `@Component`.

| Property | Java mező | Típus | Default |
|---|---|---|---|
| `app.cookie.secure` | `secure` | `boolean` | `true` |

---

## `LoginRequest` / `LoginResponse` DTO-k

**`LoginRequest`:** `username`, `password` — mindkettő `@NotBlank` validációval.

**`LoginResponse`:** `accessToken` (String), `tokenType` (String, konstans `"Bearer"` — OAuth2 konvenció), `expiresIn` (long, másodpercben — `app.jwt.expiration-ms / 1000`)

---

## `AuthService` — `login` metódus

1. `authenticationManager.authenticate()` — érvénytelen credentials esetén kivételt dob (Spring Security kezeli)
2. User betöltése `userRepository.findByUsername()` alapján
3. Access token generálása: `jwtUtil.generateToken(user)`
4. Refresh token generálása a `<userId>:<secureRandomPart>` formátumban — a generálási és BCrypt hash logika egy private helper metódusba (`generateRefreshToken(User user)`) szervezendő az `AuthService`-ben, mert a step 2.7 refresh endpointja ugyanezt hívja (lásd step 2.7 — Refresh token formátum szekció)
5. BCrypt hash számítása a refresh token `secureRandomPart` részéből (`passwordEncoder.encode()`) — szintén a `generateRefreshToken` helper végzi
6. DB mentés: `user.refreshTokenHash = hash`, `user.refreshTokenExpiresAt = now() + 7 nap`
7. `LoginResponse` visszaadása + refresh token átadása a controllernek cookie beállításhoz

---

## `AuthController` — `POST /api/auth/login`

- `@Operation` és `@ApiResponse` annotációk a Swagger dokumentációhoz
- `AuthService.login()` hívása
- Refresh token cookie összeállítása és `Set-Cookie` headerbe rakása
- `LoginResponse` visszaadása `200 OK`-val

**Refresh token cookie attribútumok** (a `refresh-token-rotation.md` alapján):

| Attribútum | Érték | Miért |
|---|---|---|
| `HttpOnly` | igen | XSS védelem |
| `Secure` | `cookieProperties.isSecure()` | HTTPS-en megy csak ki; `local` profilon `false` |
| `SameSite` | `Strict` | CSRF védelem |
| `Path` | `/api/auth` | Csak az auth endpointokra megy ki a cookie |
| `Max-Age` | `jwtProperties.getRefreshTokenExpirationMs() / 1000` | Számított érték a `JwtProperties`-ből, nem hardcoded |

A cookie összeállítása `ResponseCookie` builderrel történik — önálló `@Component`: `RefreshTokenCookieUtil`. Az `AuthController` nem tartalmaz cookie logikát; a login (2.6), refresh (2.7) és logout (2.8) endpoint egyaránt ezt a bean-t használja.

---

## Application properties kiegészítések

**`application-local.yml`:**
- `app.cookie.secure: false` *(HTTP localhost-on a Secure cookie nem megy el)*

---

## Elfogadási kritériumok

**Unit tesztek** (`AuthServiceTest`):
- Érvényes credentials → access token generálva, refresh token hash elmentve a user-en, `LoginResponse` visszaadva
- Érvénytelen credentials → `AuthenticationManager` kivételt dob, nem kerül mentés a DB-be

**Unit tesztek** (`AuthControllerTest`, MockMvc):
- `POST /api/auth/login` érvényes credentials → `200 OK`, body tartalmaz `accessToken`-t és `expiresIn`-t, `Set-Cookie` header tartalmaz `refreshToken`-t `HttpOnly`, `SameSite=Strict`, `Path=/api/auth`, `Max-Age=604800` attribútumokkal
- `POST /api/auth/login` érvénytelen credentials → `401`

**Manuálisan:**
- `local` profilon `POST /api/auth/login` admin credentials-szel → `200 OK`, response body-ban `accessToken`, cookie-ban `refreshToken`
- `mvn clean package` hiba nélkül lefut
