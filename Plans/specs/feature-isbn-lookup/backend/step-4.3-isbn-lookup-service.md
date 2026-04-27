# Step 4.3 – IsbnLookupService

## Mit állít elő

- `IsbnLookupService` interface
- `IsbnLookupServiceImpl` implementáció
- `IsbnUtils` segédosztály (ISBN-10 ↔ ISBN-13 konverzió + formátum validáció)

---

## Kulcs döntések

### ISBN-10 ↔ ISBN-13 konverzió (`IsbnUtils`)

- ISBN-13 → ISBN-10: csak `978` prefixű ISBN-13-ból lehetséges (az első 9 érdemi számjegyből + újraszámolt ellenőrzőjegyből)
- ISBN-10 → ISBN-13: `978` prefix hozzáadása + újraszámolt ellenőrzőjegy
- `979` prefixű ISBN-13-hoz nincs ISBN-10 pár — konverzió nem lehetséges, csak az eredeti formával keres
- Ellenőrzőjegy validáció: opcionális (nem blokkoló — ha valaki rossz ISBN-t olvas be, a providers úgysem találják meg)

### Orchestráció

Minden providernél két körös keresés:
1. Eredeti ISBN-nel próbál
2. Ha nem találja → konvertált ISBN-nel próbál (ha a konverzió lehetséges)
3. Ha így sem találja → következő provider

```
lookup(isbn):
  converted = IsbnUtils.convert(isbn)   // Optional<String>

  result = MolyHuClient.lookup(isbn)
  if empty && converted.present:
    result = MolyHuClient.lookup(converted)
  if found: return result

  result = OpenLibraryClient.lookup(isbn)
  if empty && converted.present:
    result = OpenLibraryClient.lookup(converted)
  if found: return result

  return { found: false }
```

- Ha valamelyik client kivételt dob (timeout, scraping hiba): naplózás (`log.warn`), továbblép — a lookup nem bukhat meg egyetlen forrás kiesése miatt
- Ha mindkét forrás meghibásodik: `found = false` (nem 5xx)
- ISBN formátum elővalidálás: csak 10 vagy 13 jegyű numerikus string fogadható el (ISBN-10: opcionálisan záró `X`); érvénytelen formátum → `found = false`, hívás nélkül

---

## Elfogadási kritériumok

- ISBN-13 beolvasás, moly.hu csak ISBN-10-zel találja → konvertálva megtalálja
- ISBN-10 beolvasás, OpenLibrary csak ISBN-13-mal találja → konvertálva megtalálja
- `979` prefixű ISBN-13: konverzió nem történik, csak eredeti formával keres
- Moly.hu timeout → OpenLibrary-t próbálja (mindkét ISBN formával)
- Mindkét forrás, mindkét ISBN forma → `found: false`, nem kivétel
- Érvénytelen ISBN formátum → `found: false`, hívás nélkül
- Unit teszt: `IsbnUtils` konverzió mindkét irányban + `979` eset + ellenőrzőjegy helyesség
- Unit teszt: orchestráció — mind a releváns ágak lefedve
