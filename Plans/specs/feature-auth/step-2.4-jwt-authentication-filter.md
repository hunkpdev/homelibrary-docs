# Step 2.4 – `JwtAuthenticationFilter`

## Mit állít elő

- `com.homelibrary.security.JwtAuthenticationFilter` — Spring Security filter
- `com.homelibrary.security.JwtAuthenticationFilterTest` — unit teszt

---

## Filter flow

`OncePerRequestFilter` kiterjesztése. Minden kérésnél lefut, de nem dob 401-et maga — a védett endpointok elutasítását a Security konfiguráció kezeli (step 2.5).

1. `Authorization` header kiolvasása — ha hiányzik vagy nem `Bearer` prefixű, filter chain folytatódik (pass-through)
2. Token kinyerése a headerből
3. `jwtUtil.isTokenValid(token)` — ha érvénytelen, pass-through
4. `jwtUtil.extractUserId(token)` → UUID
5. `userRepository.findById(UUID)` — ha a user nem létezik vagy `active=false`, pass-through
6. `UsernamePasswordAuthenticationToken` létrehozása a user-rel mint principal, authorities: `SimpleGrantedAuthority("ROLE_" + user.getRole().name())`
7. Authentication beállítása a `SecurityContextHolder`-ben
8. Filter chain folytatódik

---

## Elfogadási kritériumok

**Unit tesztek** (`JwtAuthenticationFilterTest`) — Mockito-val mock-olt `JwtUtil` és `UserRepository`, `MockHttpServletRequest` / `MockHttpServletResponse` / `MockFilterChain`:

- Érvényes token + aktív user → `SecurityContext`-ben van `Authentication`
- Hiányzó `Authorization` header → `SecurityContext` üres, chain folytatódik
- Nem `Bearer` prefixű header → `SecurityContext` üres, chain folytatódik
- Érvénytelen token (`isTokenValid` → `false`) → `SecurityContext` üres, chain folytatódik
- Inaktív user (`active=false`) → `SecurityContext` üres, chain folytatódik

**Manuálisan:**
- `mvn clean package` hiba nélkül lefut
