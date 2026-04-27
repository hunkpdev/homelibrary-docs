# Step 4.1 – OpenLibraryClient

## Mit állít elő

- `OpenLibraryClient` osztály
- `IsbnLookupResult` record (megosztott belső DTO — service és controller réteg is használja)
- `IsbnSource` enum: `OPENLIBRARY`, `GOOGLE_BOOKS`

---

## Kulcs döntések

- HTTP kliens: Spring Boot 3.2+ `RestClient` (szinkron — Lambda-n megfelelő, WebClient reaktivitása nem szükséges)
- API endpoint: `https://openlibrary.org/api/books?bibkeys=ISBN:{isbn}&format=json&jscmd=data`
- Nincs API kulcs
- Kapcsolati és olvasási timeout: `app.isbn.openlibrary.timeout-seconds` property-ből (default: 5s)
- A válasz egy JSON objektum, ahol a kulcs `"ISBN:{isbn}"` — ha a kulcs hiányzik vagy az objektum üres: `Optional.empty()`
- Válasz leképezés: dedikált `OpenLibraryBookResponse` belső DTO-ra (Jackson), majd onnan `IsbnLookupResult`-ra
- `publishYear`: a `publish_date` szövegből az évszám kinyerése (regex, első 4 jegyű szám)
- `authors`: `authors[].name` lista
- `coverImageUrl`: `cover.medium` mező, ha létezik
- `language`: `languages[0].key` alapján (pl. `/languages/hun` → `"hu"`) — ha nem szerepel: `null`

## `IsbnLookupResult` record mezői

| Mező | Típus | Leírás |
|------|-------|--------|
| `isbn` | `String` | A keresett ISBN |
| `title` | `String` | Könyv címe |
| `authors` | `List<String>` | Szerzők listája |
| `publisher` | `String` | Kiadó neve (első elem) |
| `publishYear` | `Integer` | Kiadási év |
| `language` | `String` | Könyv nyelve (ISO 639-1, pl. `"hu"`, `"en"`) |
| `categories` | `List<String>` | Kategóriák / témák |
| `coverImageUrl` | `String` | Borítókép URL |
| `source` | `IsbnSource` | Melyik API adta az adatot |
| `found` | `boolean` | `true` ha találat volt |

---

## Elfogadási kritériumok

- Ismert ISBN-re kitöltött `Optional<IsbnLookupResult>`-ot ad vissza (`found = true`, `source = OPENLIBRARY`)
- Ismeretlen ISBN-re `Optional.empty()`-t ad vissza
- Timeout vagy hálózati hiba esetén kivételt dob (a service kezeli)
- Unit teszt: WireMock-kal mockolt OpenLibrary válasz → helyes leképezés ellenőrzése
