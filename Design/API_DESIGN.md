# HomeLibrary – API Design

> **Státusz:** v1 – Draft
> **Utolsó frissítés:** 2026-03-28
> **Base URL (prod):** `https://api.[cloudfront-domain]/api`
> **Base URL (local):** `http://localhost:8080/api`

## Általános Konvenciók

- REST elvek, JSON request/response
- Minden válasz UTF-8
- Dátumok: ISO 8601 (`2026-03-28T14:30:00Z`)
- UUID-k: string formátumban (`"550e8400-e29b-41d4-a716-446655440000"`)
- Hibakezelés: egységes hibastruktúra (lásd lent)
- Autentikáció: `Authorization: Bearer <access_token>` header (kivéve login endpoint)
- Cookie: a frontend Axios `withCredentials: true` beállítással küldi a refresh token cookie-t — emiatt a szerver CORS konfigurációjában explicit origin szükséges (`Access-Control-Allow-Origin` nem lehet `*`) és `Access-Control-Allow-Credentials: true`
- Lapozás: `page` (0-tól indexelt) és `size` query paraméterekkel

## Egységes Hibastruktúra

**Jelenleg:** üres response body (Content-Length: 0). A `GlobalExceptionHandler` minden esetben `ResponseEntity<Void>`-dal tér vissza.

Strukturált hibaválasz (timestamp/status/message/path) bevezetése tech-debt — lásd `Plans/tech-debt.md`.

> Tervezett jövőbeli formátum (referenciaként):
> ```json
> {
>   "timestamp": "2026-03-28T14:30:00Z",
>   "status": 404,
>   "error": "NOT_FOUND",
>   "message": "Book not found with id: 550e8400",
>   "path": "/api/books/550e8400"
> }
> ```

---

## Auth Endpoints

### `POST /api/auth/login`
Bejelentkezés. Nincs szükség Bearer tokenre.

**Request:**
```json
{
  "username": "admin",
  "password": "secret"
}
```

**Response 200:**
```json
{
  "accessToken": "eyJhbGci...",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```
> Refresh token: HttpOnly cookie-ban érkezik (`Set-Cookie` header), nem a JSON bodyban.
>
> User adatok (id, username, role) a frontend az access token JWT payload-jából nyeri ki client-side Base64URL decode-dal — külön user objektum nem szerepel a response-ban.

**Response 401:** Hibás felhasználónév vagy jelszó

---

### `POST /api/auth/refresh`
Access token megújítása. A refresh token a cookie-ból olvasódik automatikusan.

**Response 200:**
```json
{
  "accessToken": "eyJhbGci...",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```

**Response 401:** Lejárt vagy érvénytelen refresh token

---

### `POST /api/auth/logout`
Kijelentkezés. Törli a refresh token cookie-t.

**Response 204:** No content

---

## Book Endpoints

### `GET /api/books`
Könyvek listázása szűrőkkel. Lapozott.

**Jogosultság:** `ADMIN`, `VISITOR` vagy `DEMO`

**Query paraméterek:**

| Paraméter | Típus | Leírás |
|-----------|-------|--------|
| `search` | string | Szabad szöveges (cím, szerző) |
| `status` | string | `AT_HOME`, `LOANED`, `DELETED` |
| `locationId` | UUID | Helyiség szűrő |
| `category` | string | Kategória szűrő |
| `language` | string | Könyv nyelvére szűrés |
| `publishYear` | integer | Kiadási év |
| `page` | integer | Oldalszám (default: 0) |
| `size` | integer | Oldal méret (default: 20, max: 100) |
| `sort` | string | pl. `title,asc` vagy `authors,desc` |

**Response 200:**
```json
{
  "content": [
    {
      "id": "550e8400-...",
      "isbn": "9780261102354",
      "title": "The Lord of the Rings",
      "authors": ["Tolkien, J.R.R."],
      "publisher": "HarperCollins",
      "publishYear": 1991,
      "language": "en",
      "categories": ["Fantasy", "Fiction"],
      "coverImageUrl": "https://covers.openlibrary.org/...",
      // Fázis 1: a DB cover_image_url mezőjéből (külső forrás URL). Fázis 2-től: S3 pre-signed URL, a service réteg generálja a cover_image_key alapján on-the-fly.
      "status": "AT_HOME",
      "location": {
        "id": "...",
        "name": "Bal polc",
        "room": {
          "id": "...",
          "name": "Nappali"
        }
      },
      "description": "Egy hobbit kalandjai...",
      "createdAt": "2026-03-28T14:30:00Z"
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 142,
  "totalPages": 8
}
```

---

### `POST /api/books`
Új könyv felvétele.

**Jogosultság:** `ADMIN`

**Request:**
```json
{
  "isbn": "9780261102354",
  "title": "The Lord of the Rings",
  "authors": ["Tolkien, J.R.R."],
  "publisher": "HarperCollins",
  "publishYear": 1991,
  "language": "en",
  "categories": ["Fantasy", "Fiction"],
  "description": "Egy hobbit kalandjai...",
  "descriptionLanguage": "hu",
  "locationId": "550e8400-...",
  "status": "AT_HOME",
  "source": "OPENLIBRARY"
}
```

**Response 201:**
```json
{
  "id": "550e8400-...",
  ...
}
```

---

### `GET /api/books/{id}`
Egy könyv részletes adatai.

**Jogosultság:** `ADMIN`, `VISITOR` vagy `DEMO`

**Response 200:** Teljes book objektum (lásd fent)
**Response 404:** Könyv nem található

---

### `PUT /api/books/{id}`
Könyv adatainak módosítása. Minden mező kötelező (teljes felülírás).

> **PUT vs PATCH:** Tudatos döntés a teljes felülírás mellett — a PATCH magasabb implementációs komplexitást és részleges update-ből adódó hibalehetőségeket hozna be, ami ennél a skálánál nem indokolt. A leggyakoribb részleges műveletek (státusz, helyszín) dedikált endpointokkal kezeltek.

**Jogosultság:** `ADMIN`

**Request:** Megegyezik a POST request struktúrával, kiegészítve a `version` mezővel:
```json
{
  "version": 3,
  ...
}
```

> **Optimistic locking:** A `version` mező értékét a szerver ellenőrzi. Ha közben más módosította a rekordot, 409 Conflict választ ad vissza. A frontend a legfrissebb adatot kell betöltse és újra próbálkozzon.

**Response 200:** Frissített book objektum (növelt `version` értékkel)
**Response 404:** Könyv nem található
**Response 409:** Conflict – konkurens módosítás (verziószám ütközés)

---

### `DELETE /api/books/{id}`
Könyv törlése (soft delete – `status` → `DELETED`).

**Jogosultság:** `ADMIN`

**Response 204:** No content
**Response 404:** Könyv nem található

---

### `PUT /api/books/{id}/status`
Státusz módosítás.

**Jogosultság:** `ADMIN`

**Request:**
```json
{
  "status": "LOANED"
}
```

**Response 200:** Frissített book objektum

---

### `PUT /api/books/{id}/location`
Könyv áthelyezése másik polcra/helyiségbe.

**Jogosultság:** `ADMIN`

**Request:**
```json
{
  "locationId": "550e8400-..."
}
```

**Response 200:** Frissített book objektum

---

## ISBN Lookup Endpoint

### `GET /api/books/isbn/{isbn}`
ISBN szám alapján adatlekérés külső API-ból (nem menti az adatbázisba).
Fázis 1-ben az OpenLibrary API-t hívja, fallbackként Google Books API-t.

**Jogosultság:** `ADMIN` vagy `DEMO`

**Response 200:**
```json
{
  "isbn": "9780261102354",
  "title": "The Lord of the Rings",
  "authors": ["Tolkien, J.R.R."],
  "publisher": "HarperCollins",
  "publishYear": 1991,
  "language": "en",
  "categories": ["Fantasy", "Fiction"],
  "coverImageUrl": "https://covers.openlibrary.org/...",
  "source": "OPENLIBRARY",
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

---

## Room Endpoints

### `GET /api/rooms`
Az összes aktív helyiség listázása.

**Jogosultság:** `ADMIN`, `VISITOR` vagy `DEMO`

**Query paraméterek:** `name` (opcionális, részleges egyezés), `page`, `size`, `sort` (default: `name,asc`)

**Response 200:** `Page<RoomResponse>`
```json
{
  "content": [
    {
      "id": "550e8400-...",
      "name": "Nappali",
      "description": "Opcionális megjegyzés",
      "locationCount": 3,
      "version": 1
    }
  ],
  ...
}
```

---

### `GET /api/rooms/all`
Lapozás nélküli teljes lista, frontend dropdown feltöltésére.

**Jogosultság:** `ADMIN`, `VISITOR` vagy `DEMO`

**Query paraméterek:** nincs

**Response 200:** `List<RoomResponse>` — összes aktív room, `name ASC` sorrendben

---

### `POST /api/rooms`
Új helyiség felvétele.

**Jogosultság:** `ADMIN`

**Request:**
```json
{
  "name": "Dolgozószoba",
  "description": "Opcionális megjegyzés"
}
```

**Response 201:** Létrehozott room objektum
**Response 400:** Validációs hiba (kötelező mező hiányzik)

---

### `PUT /api/rooms/{id}`
Helyiség módosítása. `version` mező szükséges az optimistic locking miatt.

**Jogosultság:** `ADMIN`

**Response 200:** Frissített room objektum (növelt `version` értékkel)
**Response 404:** Room nem található
**Response 409:** Conflict – optimistic locking ütközés

---

### `DELETE /api/rooms/{id}`
Helyiség törlése (soft delete, csak akkor, ha nincs hozzá aktív location rendelve).

**Jogosultság:** `ADMIN`

**Response 204:** No content
**Response 404:** Room nem található
**Response 409:** Conflict – van aktív location a roomhoz rendelve

---

## Location Endpoints

### `GET /api/locations`
Az összes aktív location listázása.

**Jogosultság:** `ADMIN`, `VISITOR` vagy `DEMO`

**Query paraméterek:** `name`, `description`, `roomId` (opcionálisak; `name` és `description` részleges egyezés, case-insensitive; `roomId` UUID szűrő), `page`, `size`, `sort` (default: `name,asc`)

**Response 200:** `Page<LocationResponse>`
```json
{
  "content": [
    {
      "id": "550e8400-...",
      "name": "Bal polc",
      "description": "Az ablak melletti polc",
      "room": {
        "id": "...",
        "name": "Nappali"
      },
      "bookCount": 23,
      "version": 1
    }
  ],
  ...
}
```

---

### `GET /api/locations/all`
Lapozás nélküli teljes lista, frontend dropdown feltöltésére.

**Jogosultság:** `ADMIN`, `VISITOR` vagy `DEMO`

**Query paraméterek:** nincs

**Response 200:** `List<LocationResponse>` — összes aktív location, `name ASC` sorrendben

---

### `POST /api/locations`
Új location felvétele.

**Jogosultság:** `ADMIN`

**Request:**
```json
{
  "name": "Felső polc",
  "roomId": "550e8400-...",
  "description": "Opcionális megjegyzés"
}
```

**Response 201:** Létrehozott location objektum
**Response 400:** Validációs hiba (kötelező mező hiányzik)
**Response 404:** Room nem található

---

### `PUT /api/locations/{id}`
Location módosítása. `version` mező szükséges az optimistic locking miatt.

**Jogosultság:** `ADMIN`

**Response 200:** Frissített location objektum (növelt `version` értékkel)
**Response 404:** Location nem található
**Response 409:** Conflict – optimistic locking ütközés

---

### `DELETE /api/locations/{id}`
Location törlése (soft delete, csak akkor, ha nincs hozzá aktív könyv rendelve).

**Jogosultság:** `ADMIN`

**Response 204:** No content
**Response 404:** Location nem található
**Response 409:** Conflict – van aktív könyv a locationhöz rendelve

---

## Loan Endpoints

### `GET /api/loans`
Aktív kölcsönzések listázása.

**Jogosultság:** `ADMIN` vagy `DEMO`

**Query paraméterek:** `active=true/false`, `bookId`, `page`, `size`

**Response 200:**
```json
{
  "content": [
    {
      "id": "550e8400-...",
      "book": {
        "id": "...",
        "title": "The Lord of the Rings",
        "isbn": "9780261102354"
      },
      "loanedToName": "Kiss Péter",
      "loanedAt": "2026-03-01T10:00:00Z",
      "returnedAt": null,
      "note": "Visszahozza március végéig"
    }
  ],
  ...
}
```

---

### `POST /api/loans`
Könyv kölcsönadása.

**Jogosultság:** `ADMIN`

**Request:**
```json
{
  "bookId": "550e8400-...",
  "loanedToName": "Kiss Péter",
  "loanedToUserId": null,
  "note": "Visszahozza március végéig"
}
```

> **Mellékhatás:** a könyv `status` mezője automatikusan `AT_HOME` → `LOANED`-ra vált.

**Response 201:** Létrehozott loan objektum
**Response 409:** A könyv már ki van kölcsönözve

---

### `PUT /api/loans/{id}/return`
Könyv visszavétele.

**Jogosultság:** `ADMIN`

> **Mellékhatás:** a könyv `status` mezője automatikusan `LOANED` → `AT_HOME`-ra vált.

**Response 200:** Frissített loan objektum (returnedAt kitöltve)

---

## User Endpoints

### `GET /api/users`
Felhasználók listázása.

**Jogosultság:** `ADMIN` (VISITOR és DEMO kizárva)

**Response 200:**
```json
[
  {
    "id": "550e8400-...",
    "username": "visitor1",
    "email": "visitor@example.com",
    "role": "VISITOR",
    "preferredLanguage": "hu",
    "active": true,
    "createdAt": "2026-01-01T00:00:00Z"
  }
]
```

---

### `GET /api/users/{id}`
Egy felhasználó adatainak lekérdezése.

**Jogosultság:** `ADMIN`, vagy a saját ID-jára `VISITOR` is (`@PreAuthorize` ownership check a controllerben) — `DEMO` kizárva

**Response 200:**
```json
{
  "id": "550e8400-...",
  "username": "visitor1",
  "email": "visitor@example.com",
  "role": "VISITOR",
  "preferredLanguage": "hu",
  "active": true,
  "createdAt": "2026-01-01T00:00:00Z"
}
```
**Response 404:** Felhasználó nem található

---

### `POST /api/users`
Új felhasználó létrehozása.

**Jogosultság:** `ADMIN`

**Request:**
```json
{
  "username": "visitor1",
  "email": "visitor@example.com",
  "password": "initialPassword123",
  "role": "VISITOR",
  "preferredLanguage": "hu"
}
```

**Response 201:** Létrehozott user objektum (jelszó nélkül)

---

### `PUT /api/users/{id}`
Felhasználó módosítása. Minden mező kötelező (teljes felülírás). `version` mező szükséges az optimistic locking miatt.

**Jogosultság:**

| Mező | `ADMIN` | `VISITOR` (csak saját) |
|------|---------|----------------------|
| `preferredLanguage` | ✅ | ✅ |
| `password` | ✅ | ✅ |
| `username` | ✅ | ❌ |
| `email` | ✅ | ❌ |
| `role` | ✅ | ❌ |
| `active` | ✅ | ❌ |

**Response 200:** Frissített user objektum (növelt `version` értékkel)
**Response 403:** Forbidden – nem saját adat vagy tiltott mező módosítása
**Response 409:** Conflict – konkurens módosítás (verziószám ütközés)

---

### `DELETE /api/users/{id}`
Felhasználó deaktiválása (soft delete – `active` → `false`).

**Jogosultság:** `ADMIN`

**Response 204:** No content

---

## HTTP Státuszkódok Összefoglalva

| Kód | Mikor |
|-----|-------|
| `200 OK` | Sikeres lekérdezés vagy módosítás |
| `201 Created` | Sikeres létrehozás |
| `204 No Content` | Sikeres törlés |
| `400 Bad Request` | Érvénytelen input (validáció hiba) |
| `401 Unauthorized` | Hiányzó vagy lejárt token |
| `403 Forbidden` | Nincs jogosultság |
| `404 Not Found` | Az erőforrás nem létezik |
| `409 Conflict` | Üzleti logika ütközés (pl. dupla kölcsönzés, konkurens módosítás – verziószám ütközés) |
| `500 Internal Server Error` | Szerver oldali hiba |
