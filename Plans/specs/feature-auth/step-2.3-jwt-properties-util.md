# Step 2.3 – `JwtProperties` + `JwtUtil`

## Mit állít elő

- `com.homelibrary.config.JwtProperties`
- `com.homelibrary.util.JwtUtil`
- `com.homelibrary.util.JwtUtilTest` — unit teszt
- `application.properties` kiegészítés
- `application-local.yml` kiegészítés
- `application-prod.yml` kiegészítés

---

## `JwtProperties`

`@ConfigurationProperties(prefix = "app.jwt")` + `@Component`.

| Property | Java mező | Típus |
|---|---|---|
| `app.jwt.secret` | `secret` | `String` |
| `app.jwt.expiration-ms` | `expirationMs` | `long` |

---

## `JwtUtil`

JWT library: `io.jsonwebtoken:jjwt-api` (0.12.6), algoritmus: **HS256**.

| Metódus | Leírás |
|---|---|
| `String generateToken(User user)` | Access token generálás — `sub`: user UUID, `username` claim: `user.getUsername()`, `role` claim: `user.getRole().name()`, lejárat: `now + expirationMs` |
| `boolean isTokenValid(String token)` | Aláírás és lejárat validálás — bármilyen hiba esetén `false`, nem dob kivételt |
| `UUID extractUserId(String token)` | `sub` claim kinyerése és UUID-dé konvertálása |

A `role` claim kinyerésére nincs külön metódus — a JWT filter (step 2.4) user ID alapján DB-ből tölti be a teljes `User` objektumot.

---

## Application properties kiegészítések

**`application.properties`:**
- `app.jwt.expiration-ms=900000` *(15 perc, minden profilon azonos)*

**`application-local.yml`:**
- `app.jwt.secret: local-dev-secret-for-homelibrary-min32` *(hardcoded, nem érzékeny)*

**`application-prod.yml`:**
- `app.jwt.secret: ${JWT_SECRET}` *(SSM `/homelibrary/jwt-secret` → Lambda env var — step 1.12)*

---

## Elfogadási kritériumok

**Unit tesztek** (`JwtUtilTest`) — ebben a stepben készülnek el, lokálisan futtatva zölden kell lenniük:

- `generateToken` → `isTokenValid` → `true` (happy path)
- `generateToken` → `extractUserId` → visszaadja a user UUID-ját
- Módosított tokennel `isTokenValid` → `false`
- Lejárt tokennel `isTokenValid` → `false` (tesztben `expirationMs=1` beállítással)

**Manuálisan:**
- `local` profilon az alkalmazás elindul, `JWT_SECRET` env var nélkül
- `mvn clean package` hiba nélkül lefut
