# Step 2.7 – `POST /api/auth/refresh`

## Mit állít elő

- `AuthService` bővítése `refresh` metódussal
- `AuthController` bővítése `POST /api/auth/refresh` endpointtal
- `AuthServiceTest` bővítése
- `AuthControllerTest` bővítése

---

## Refresh token formátum

Az opaque refresh token struktúrája: `<userId>:<secureRandomPart>`

| Rész | Tartalom |
|---|---|
| `userId` | A `users` tábla UUID PK-ja — ebből tölti be a szerver a user-t, nincs tábla-scan |
| `secureRandomPart` | `SecureRandom` által generált, Base64URL kódolt, minimum 32 byte — ez a titkos rész |

**Fontos:** A BCrypt hash kizárólag a `secureRandomPart`-ból készül, nem a teljes tokenből. A `userId` publikus adat (a JWT access tokenben is benne van), a biztonságot a random rész + BCrypt hash adja.

A token generálási logika (random rész generálás + hash + összerakás) közös helper metódusba szervezendő — a login (step 2.6) és a refresh endpoint is ezt hívja.

---

## `AuthService` — `refresh` metódus flow

1. Cookie-ból `refreshToken` kiolvasás — ha hiányzik → `401`
2. Token split `:` mentén → `userId` + `secureRandomPart`
3. `userRepository.findById(userId)` — user betöltése
4. Ellenőrzések — bármelyik kudarc → `401`:
   - User létezik és `active = true`
   - `refresh_token_expires_at > now()`
   - `BCrypt.matches(secureRandomPart, user.refreshTokenHash)`
5. Rotation:
   - Új `secureRandomPart` generálása
   - Új refresh token összerakása: `userId:újSecureRandomPart`
   - BCrypt hash számítása az új `secureRandomPart`-ból
   - DB mentés: `refreshTokenHash = új hash`, `refreshTokenExpiresAt = now() + 7 nap`
   - Új access token generálása (`jwtUtil.generateToken(user)`)
6. Visszaadás: `LoginResponse` (accessToken, tokenType, expiresIn) + új refresh token a controllernek cookie beállításhoz

---

## `AuthController` — `POST /api/auth/refresh`

- `@Operation` és `@ApiResponse` annotációk
- `AuthService.refresh()` hívása
- Új refresh token cookie beállítása (a 2.6-ban definiált helper metódussal)
- `LoginResponse` visszaadása `200 OK`-val

---

## Amit NEM kell implementálni

| Koncepció | Miért nem |
|---|---|
| Token family tracking | Túlzott komplexitás 2–5 felhasználóhoz — 401-et kap, ha lejárt tokent használnak |
| Refresh token blacklist tábla | Nem szükséges — a `users` táblában mindig csak az aktuálisan érvényes hash van, a régi implicit érvénytelen |
| Access token blacklist / revoke | Lambda-n nincs perzisztens in-memory állapot — az access token 15 perces TTL-je elfogadható kockázat |

---

## Elfogadási kritériumok

**Unit tesztek** (`AuthServiceTest` bővítése):
- Érvényes refresh token → új access token generálva, új hash mentve DB-ben, `refreshTokenExpiresAt = now() + 7 nap` (nem az eredeti login időpontjától)
- Token formátum: generált refresh token `<userId>:<randomPart>` formátumú, BCrypt hash a `<randomPart>`-ból készül
- Hash nem egyezik → `401`, DB változatlan
- `refreshTokenExpiresAt < now()` → `401`, DB változatlan
- Nem létező userId a tokenben → `401`
- Inaktív user (`active=false`) → `401`
- Rotation: refresh után a régi tokennel újra hívva → `401`

**Unit tesztek** (`AuthControllerTest` bővítése, MockMvc):
- `POST /api/auth/refresh` érvényes cookie-val → `200 OK`, új `accessToken` a body-ban, `Set-Cookie` header: `HttpOnly`, `SameSite=Strict`, `Path=/api/auth`, `Max-Age=604800`
- `POST /api/auth/refresh` hiányzó cookie-val → `401`

**Manuálisan:**
- Login után a kapott refresh cookie-val `POST /api/auth/refresh` → `200 OK`, új access token
- `mvn clean package` hiba nélkül lefut
