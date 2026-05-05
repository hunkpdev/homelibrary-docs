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
  "isbn": "9789636091996",
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

**Response 204 (nem található):** üres body

> A 204 üzleti állapot, nem hiba — az OSZK nem találta a könyvet, vagy a rekord csonka. A frontend manuális beviteli módra vált.

**Response 422 (érvénytelen ISBN formátum):** üres body

**Response 429 (DEMO rate limit elérve):**
```json
{
  "reason": "DEMO_RATE_LIMIT_EXCEEDED"
}
```

> A 429 válasz strukturált body-t ad — szándékos kivétel az általános üres-body hibakonvenció alól, hogy a frontend megkülönböztethesse az alkalmazásszintű DEMO limitet az infrastruktúra-szintű throttlingtól.
> `DemoRateLimitExceededException` → `GlobalExceptionHandler` 429-et ad vissza strukturált body-val.

## Kulcs döntések

- `@Operation` és `@ApiResponse` annotációk a Swagger UI-hoz (200, 204, 422, 429 response kódokra)
- A controller nem validálja az ISBN-t — a service (4.3) végzi
- `IsbnLookupResult` → `IsbnLookupResponse` leképezés: MapStruct mapper vagy kézi konstruktor
- `source` lehetséges értékei: `OSZK` (találat esetén), null (nem találat esetén)
- `subtitle`, `pageCount` null megengedett (nem minden MARC rekordban van meg)
- A 429 válasz body-ja szándékosan strukturált (`{"reason": "DEMO_RATE_LIMIT_EXCEEDED"}`), ellentétben a `GlobalExceptionHandler` általános üres body-jával — a frontend-nek meg kell tudnia különböztetni a rate limit-et más hibáktól
- `source` lehetséges értéke: `OSZK` (találat esetén)

## Elfogadási kritériumok

- `GET /api/books/isbn/{isbn}` → 200, kitöltött response (nincs `found` mező)
- Nem található ISBN → 204, üres body
- Érvénytelen ISBN formátum → 422, üres body
- Hitelesítés nélkül → 401
- VISITOR role → 403
- DEMO session limit elérve → 429, `{"reason": "DEMO_RATE_LIMIT_EXCEEDED"}`
- DEMO napi limit elérve → 429, `{"reason": "DEMO_RATE_LIMIT_EXCEEDED"}`
- Swagger UI-on 200, 204, 422, 429 response kódok megjelennek
