# Step 4.5 – IsbnLookupController

## Mit állít elő

- `IsbnLookupController` osztály
- `IsbnLookupResponse` DTO

---

## Endpoint

`GET /api/books/isbn/{isbn}`

**Jogosultság:** `@PreAuthorize("hasAnyRole('ADMIN', 'DEMO')")` — VISITOR 403

**Path paraméter:** `isbn` — a service validálja a formátumot (4.4)

**Response 200 (találat):**
```json
{
  "isbn": "9789636091996",
  "title": "Szülői generációk harca",
  "subtitle": "hogyan értsük meg magunkat?",
  "authors": ["Steigervald Krisztián"],
  "publisher": "Partvonal",
  "publishYear": 2026,
  "language": "hun",
  "pageCount": 311,
  "source": "OSZK",
  "found": true
}
```

**Response 200 (nem található):**
```json
{
  "isbn": "9780000000000",
  "found": false
}
```

**Response 429 (DEMO rate limit elérve):**
```json
{
  "found": false,
  "rateLimitExceeded": true
}
```

> Nem találat esetén is 200 a válasz (nem 404) — a "nem található" üzleti állapot, nem hiba.
> `DemoRateLimitExceededException` → `@ExceptionHandler` 429-et ad vissza.

## Kulcs döntések

- `@Operation` és `@ApiResponse` annotációk a Swagger UI-hoz (mindhárom response kódra)
- A controller nem validálja az ISBN-t — a service (4.4) végzi
- `IsbnLookupResult` → `IsbnLookupResponse` leképezés: MapStruct mapper vagy kézi konstruktor
- `source` lehetséges értékei: `OSZK` (találat esetén), null (nem találat esetén)
- `subtitle`, `pageCount` null megengedett (nem minden MARC rekordban van meg)

## Elfogadási kritériumok

- `GET /api/books/isbn/{isbn}` → 200, kitöltött response
- Nem található ISBN → 200, `found: false`
- Hitelesítés nélkül → 401
- VISITOR role → 403
- DEMO session limit elérve → 429, `rateLimitExceeded: true`
- DEMO napi limit elérve → 429, `rateLimitExceeded: true`
- Swagger UI-on mindhárom response kód megjelenik
