# Step 2.5 – Spring Security konfiguráció

## Mit állít elő

- `com.homelibrary.config.CorsProperties` — `@ConfigurationProperties` az `app.cors.*` property-khez
- `com.homelibrary.config.SecurityConfig` — Spring Security filter chain
- `com.homelibrary.security.UserDetailsServiceImpl` — `UserDetailsService` implementáció
- `com.homelibrary.security.UserDetailsServiceImplTest` — unit teszt
- `application-local.yml` kiegészítés
- `application-prod.yml` kiegészítés

---

## `CorsProperties`

`@ConfigurationProperties(prefix = "app.cors")` + `@Component`.

| Property | Java mező | Típus |
|---|---|---|
| `app.cors.allowed-origin` | `allowedOrigin` | `String` |

---

## `SecurityConfig`

`@Configuration` + `@EnableWebSecurity`.

**Filter chain:**
- CSRF: disabled — stateless JWT + `SameSite=Strict` cookie véd a CSRF ellen; a `SecurityConfig` osztályon `@SuppressWarnings("java:S4502")` annotáció szükséges, különben SonarQube S4502 rule-t triggerel
- Session management: `STATELESS`
- CORS: `.cors(withDefaults())` — `CorsConfigurationSource` bean-ből olvassa a konfigurációt
- `JwtAuthenticationFilter` hozzáadva `UsernamePasswordAuthenticationFilter` elé

**Authorization rules:**

| Endpoint pattern | Hozzáférés |
|---|---|
| `POST /api/auth/login` | `permitAll` |
| `POST /api/auth/refresh` | `permitAll` |
| `POST /api/auth/logout` | `permitAll` |
| `GET /api/health` | `permitAll` |
| `/swagger-ui/**` | `permitAll` |
| `/v3/api-docs/**` | `permitAll` |
| minden más | `authenticated` |

**Role hierarchy:**
- `ADMIN` > `VISITOR` — az ADMIN implicit megkapja a VISITOR jogosultságait is

**Exportált bean-ek:**
- `PasswordEncoder` — `BCryptPasswordEncoder(12)` (login és jövőbeli user management használja)
- `AuthenticationManager` — a login endpoint (step 2.6) manuális autentikációhoz
- `CorsConfigurationSource` — allowed origin: `corsProperties.getAllowedOrigin()`, allowed methods: GET, POST, PUT, DELETE, OPTIONS, allowed headers: `Authorization`, `Content-Type`, `Accept`, `allowCredentials: true`

  > `*` helyett explicit lista — a wildcard egyes böngésző implementációkban CORS preflight bypass-hoz használható lenne. Ha jövőben egyedi headerek szükségesek, itt bővítendő.

---

## `UserDetailsServiceImpl`

`UserDetailsService` implementáció a `security/` package-ben.

`loadUserByUsername(String username)`:
- `userRepository.findByUsername(username)` — ha nem találja: `UsernameNotFoundException`
- Visszaadott `UserDetails`: Spring Security beépített `User` builderével összerakva (`username`, `passwordHash`, `.disabled(!user.isActive())`, authorities: `ROLE_` + role)
- **Fontos:** `.disabled(!user.isActive())` kötelező — enélkül deaktivált user érvényes jelszóval be tud lépni

---

## Application properties kiegészítések

**`application-local.yml`:**
- `app.cors.allowed-origin: http://localhost:3000`

**`application-prod.yml`:**
- `app.cors.allowed-origin: ${CORS_ALLOWED_ORIGIN}` *(CloudFront URL → Lambda env var — CDK deploy után egyszer kézzel beállítandó)*

---

## Elfogadási kritériumok

**Unit tesztek** (`UserDetailsServiceImplTest`):
- Létező username → `UserDetails` visszaadva helyes username-mel és authority-kkel
- Nem létező username → `UsernameNotFoundException` dobva

**Manuálisan:**
- `local` profilon az alkalmazás elindul
- `GET /api/health` token nélkül → `200 OK`
- `GET /swagger-ui/index.html` token nélkül → `200 OK`
- `GET /v3/api-docs` token nélkül → `200 OK`
- `GET` bármely más `/api/**` endpoint token nélkül → `401`
- `mvn clean package` hiba nélkül lefut

> **Megjegyzés:** Az auth flow teljes integrációs tesztje (`@SpringBootTest`: login → refresh → logout) a step 2.8-ban készül el, amikor mindhárom auth endpoint létezik.
