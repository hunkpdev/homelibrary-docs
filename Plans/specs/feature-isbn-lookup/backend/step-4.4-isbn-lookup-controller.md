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
  "reason": "DEMO_RATE_LIMIT_EXCEEDED"
}
```

> Nem találat esetén 204 — a "nincs elérhető adat" üzleti állapot, nem hiba (ezért nem 404).
> `DemoRateLimitExceededException` → `@ExceptionHandler` 429-et ad vissza.
> A body azért szükséges, hogy a frontend meg tudja különböztetni az alkalmazás szintű DEMO limitet az infrastruktúra szintű 429-től (pl. API Gateway throttling).

## Kulcs döntések

- `@Operation` és `@ApiResponse` annotációk a Swagger UI-hoz (mindhárom response kódra)
- A controller nem validálja az ISBN-t — a service (4.3) végzi
- `IsbnLookupResult` → `IsbnLookupResponse` leképezés: MapStruct mapper vagy kézi konstruktor
- `source` lehetséges értékei: `OSZK` (találat esetén) — nem találatnál nincs response body
- `subtitle`, `pageCount` null megengedett (nem minden MARC rekordban van meg)
- A 429 válasz body-ja szándékosan strukturált (`reason: "DEMO_RATE_LIMIT_EXCEEDED"`), hogy a frontend meg tudja különböztetni az alkalmazás szintű DEMO limitet az infrastruktúra szintű throttlingtól (pl. API Gateway 429)
- Az `isbn` mező nem szerepel a response body-ban — a kliens a path paraméterből ismeri

## Elfogadási kritériumok

- `GET /api/books/isbn/{isbn}` → 200, kitöltött response body
- Nem található ISBN → 204, üres body
- Hitelesítés nélkül → 401
- VISITOR role → 403
- DEMO session limit elérve → 429, `{ "reason": "DEMO_RATE_LIMIT_EXCEEDED" }`
- DEMO napi limit elérve → 429, `{ "reason": "DEMO_RATE_LIMIT_EXCEEDED" }`
- Swagger UI-on mindhárom response kód megjelenik
