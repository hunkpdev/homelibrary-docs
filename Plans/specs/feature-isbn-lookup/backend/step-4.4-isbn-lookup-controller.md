# Step 4.4 – IsbnLookupController

## Mit állít elő

- `IsbnLookupController` osztály
- `IsbnLookupResponse` DTO (a `IsbnLookupResult` record API-rétegbeli megfelelője)

---

## Endpoint

`GET /api/books/isbn/{isbn}`

**Jogosultság:** `ADMIN` (DEMO role is elérheti — GET végpont)

**Path paraméter:** `isbn` — a service validálja a formátumot

**Response 200 (találat):**
```json
{
  "isbn": "9789634154679",
  "title": "Az arany ember",
  "authors": ["Jókai Mór"],
  "publisher": "Helikon",
  "publishYear": 2021,
  "language": "hu",
  "categories": ["Regény"],
  "coverImageUrl": "https://moly.hu/system/covers/...",
  "source": "MOLY_HU",
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

> Nem találat esetén is 200 a válasz (nem 404) — a "nem található" üzleti állapot, nem hiba. A frontend a `found` mező alapján dönt.

---

## Kulcs döntések

- `@Operation` és `@ApiResponse` annotációk a Swagger UI-hoz
- A controller nem validálja az ISBN-t (a service végzi — 4.3)
- `IsbnLookupResult` → `IsbnLookupResponse` leképezés: MapStruct mapper vagy kézi konstruktor
- `source` lehetséges értékei: `MOLY_HU`, `OPENLIBRARY`

---

## Elfogadási kritériumok

- `GET /api/books/isbn/{isbn}` → 200, kitöltött response
- `GET /api/books/isbn/9780000000000` → 200, `found: false`
- Hitelesítés nélkül → 401
- VISITOR role → 403
- Swagger UI-on megjelenik az endpoint
