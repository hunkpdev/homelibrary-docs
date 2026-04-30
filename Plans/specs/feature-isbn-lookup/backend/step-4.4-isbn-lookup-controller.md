# Step 4.4 – IsbnLookupController

## Mit állít elő

- `IsbnLookupController` osztály
- `IsbnLookupResponse` DTO

---

## Endpoint

`GET /api/books/isbn/{isbn}`

**Jogosultság:** `@PreAuthorize("hasAnyRole('ADMIN', 'DEMO')")` — VISITOR 403

**Path paraméter:** `isbn` — a service validálja a formátumot (4.3)

**Response 200 (találat):**
```json
{
  "title": "Szülői generációk harca",
  "subtitle": "hogyan értsük meg magunkat?",
  "authors": ["Steigervald Krisztián"],
  "publisher": "Partvonal",
  "publishYear": 2026,
  "language": "hun",
  "pageCount": 311,
  "source": "OSZK"
}
```

**Response 204 (nem található):** — üres body

**Response 429 (DEMO rate limit elérve):**
```json
{
  "rateLimitExceeded": true
}
```

> Nem találat esetén 204 — a "nincs elérhető adat" üzleti állapot, nem hiba (ezért nem 404).
> `DemoRateLimitExceededException` → `@ExceptionHandler` 429-et ad vissza.

## Kulcs döntések

- `@Operation` és `@ApiResponse` annotációk a Swagger UI-hoz (mindhárom response kódra)
- A controller nem validálja az ISBN-t — a service (4.3) végzi
- `IsbnLookupResult` → `IsbnLookupResponse` leképezés: MapStruct mapper vagy kézi konstruktor
- `source` lehetséges értékei: `OSZK` (találat esetén) — nem találatnál nincs response body
- `subtitle`, `pageCount` null megengedett (nem minden MARC rekordban van meg)
- A 429 válasz body-ja szándékosan strukturált (`rateLimitExceeded: true`), ellentétben a `GlobalExceptionHandler` általános üres body-jával — a frontend-nek meg kell tudnia különböztetni a rate limit-et más hibáktól
- Az `isbn` mező nem szerepel a response body-ban — a kliens a path paraméterből ismeri

## Elfogadási kritériumok

- `GET /api/books/isbn/{isbn}` → 200, kitöltött response body
- Nem található ISBN → 204, üres body
- Hitelesítés nélkül → 401
- VISITOR role → 403
- DEMO session limit elérve → 429, `{ rateLimitExceeded: true }`
- DEMO napi limit elérve → 429, `{ rateLimitExceeded: true }`
- Swagger UI-on mindhárom response kód megjelenik
