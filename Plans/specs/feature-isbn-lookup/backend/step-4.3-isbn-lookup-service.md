# Step 4.3 – IsbnLookupService

## Mit állít elő

- `IsbnLookupService` (konkrét osztály — interface/impl szétválasztás nincs, konzisztens a többi service-szel)
- `IsbnUtils` segédosztály (formátum validáció + normalizálás)
- `DemoIsbnRateLimitService` (session + napi limit logika)

---

## ISBN validáció (`IsbnUtils`)

Normalizálás és validáció: `IsbnUtils.normalize(isbn)` — kötőjelek és whitespace strip, majd formátum-ellenőrzés (10 vagy 13 jegyű numerikus string; ISBN-10: opcionálisan záró `X`). Érvénytelen formátum esetén `InvalidIsbnException`-t dob → `GlobalExceptionHandler` 422 Unprocessable Entity választ ad, hívás nélkül. Konverzió nincs — az OSZK Z39.50 `@attr 1=7` ISBN-13-mal közvetlenül keres, empirikusan igazolt.

## Orchestráció

```
lookup(isbn):
  normalized = IsbnUtils.normalize(isbn)   // dob InvalidIsbnException ha érvénytelen → 422

  result = OszkNektarClient.lookup(normalized)

  if result.present: return 200 with body
  return 204 No Content
```

Nincs külső fallback chain — egyetlen forrás az OSZK.

## DEMO rate limit (`DemoIsbnRateLimitService`)

**Session limit (5 keresés):**
- Spring Cache (ConcurrentHashMap alapú, in-memory) — Lambda instance szintű
- Kulcs: JWT `jti` claim, `SecurityContextHolder`-ből kinyerve
- Számláló növelés minden sikeres kérésnél (rate limit ellenőrzés után)
- Lambda-újraindításkor reset — DEMO kontextusban elfogadható

**Napi limit (50 keresés):**
- `demo_isbn_daily_stats` tábla olvasása
- Lazy reset: ha `lookup_date` ≠ ma (UTC) → `lookup_count = 0`, `lookup_date = ma`
- Ha `lookup_count >= 50` → limit elérve

**Ellenőrzési sorrend:** session limit → napi limit → lookup. Mindkét limit ellenőrzés a tényleges OSZK hívás előtt.

**Atomicitás:** a napi limit read-increment-write nem atomikus. Párhuzamos Lambda instance-ok esetén a limit 1-2-vel csúszhat — tudatosan elfogadott (DEMO felhasználónál párhuzamos kérés gyakorlatilag kizárt).

**`DemoRateLimitExceededException`** (vagy hasonló elnevezés) — a controller 429-gel válaszol rá.

DEMO role esetén alkalmazandó a rate limit; ADMIN-nál kihagyandó. (VISITOR az endpointot nem éri el — 403 a SecurityConfig szintjén.)

## Elfogadási kritériumok

- Kötőjeles vagy szóközös ISBN → normalizálás (strip), majd validáció — érvényes ISBN-re lookup
- Érvénytelen ISBN formátum → `InvalidIsbnException` → 422, hívás nélkül
- OSZK `Optional.empty()` → 204 No Content, nem kivétel
- DEMO: 5. keresés után session limit elérve → kivétel
- DEMO: napi 50 keresés után limit elérve → kivétel
- DEMO: új nap → lazy reset, számláló nulláról indul
- ADMIN: rate limit nem érvényesül
- Unit teszt: `IsbnUtils` validáció, orchestráció, rate limit mindkét ága
