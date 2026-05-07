# Step 5.7 – BookController: státusz és helyszín módosítás

## Mit állít elő

- `BookController` kiegészítése (step 5.6-ban jött létre): `PUT /api/books/{id}/status`, `PUT /api/books/{id}/location`
- `BookStatusRequest` — request DTO
- `BookLocationRequest` — request DTO
- `BookService` kiegészítése: `updateStatus` és `updateLocation` metódusok

---

## Request DTO-k

### `BookStatusRequest`

| Mező | Típus | Validáció |
|------|-------|-----------|
| `status` | `BookStatus` | `@NotNull` |

> `DELETED` értéket az endpoint visszautasítja (400) — soft delete kizárólag `DELETE /api/books/{id}` végponton keresztül történik.

### `BookLocationRequest`

| Mező | Típus | Validáció |
|------|-------|-----------|
| `locationId` | `UUID` | `@NotNull` |

---

## Endpointok

### `PUT /api/books/{id}/status`
- Jogosultság: `ADMIN`
- Request: `@Valid BookStatusRequest`
- `DELETED` status esetén → 400 (nem ezen az endpointon kezelt)
- Response 200: `BookResponse`
- Response 404: könyv nem található

### `PUT /api/books/{id}/location`
- Jogosultság: `ADMIN`
- Request: `@Valid BookLocationRequest`
- Location létezés + aktív ellenőrzés → 404 ha nem elérhető
- Response 200: `BookResponse`
- Response 404: könyv vagy location nem található

---

## Kulcs döntések

- Feature 6 (Loans) a `BookService`-t hívja közvetlenül a státuszváltáshoz (AT_HOME ↔ LOANED) — nem ezen a HTTP endpointon keresztül; az endpoint manuális admin korrekcióra való
- `@Operation` és `@ApiResponse` annotációk az endpointokon (springdoc-openapi)

---

## Elfogadási kritériumok

- `PUT /api/books/{id}/status` `{"status": "DELETED"}` → 400
- `PUT /api/books/{id}/location` nem létező `locationId`-val → 404
- Sikeres státuszváltás → 200, frissített `BookResponse`
- Sikeres áthelyezés → 200, frissített `BookResponse` az új locationnel
