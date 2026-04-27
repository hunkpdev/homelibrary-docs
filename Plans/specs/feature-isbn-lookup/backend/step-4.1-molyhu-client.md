# Step 4.1 – MolyHuClient

## Mit állít elő

- `MolyHuClient` osztály
- Jsoup Maven függőség (`org.jsoup:jsoup`)

---

## Kulcs döntések

- Adatforrás: moly.hu HTML scraping (nincs publikus API)
- Keresési URL: `https://moly.hu/kereses?utf8=%E2%9C%93&query={isbn}`
  - Az `utf8=✓` Rails-es form artefakt — nélküle a backend egyes esetekben eltérően viselkedik
- HTTP hívás: `RestClient` (szinkron) — `followRedirects = true` (default), a végső URL-ből derül ki melyik eset állt fenn
- Jsoup: csak a HTML parseoláshoz, nem a HTTP híváshoz
- `source = IsbnSource.MOLY_HU`
- Kapcsolati és olvasási timeout: `app.isbn.molyhu.timeout-seconds` property-ből (default: 5s)

### Request fejlécek

Minden kéréshez kötelező — felelős scraping és azonosíthatóság érdekében:

```
User-Agent: HomeLibrary/1.0 (+https://github.com/hunkpdev/homelibrary; contact: hu.nkp.dev@gmail.com)
Range: bytes=0-10000
```

> A `Range` fejléc a moly.hu oldalak `<head>` szekciójára korlátozza a letöltést — az og: meta tagek ott vannak, a közösségi tartalom (értékelések, kommentek) a `<body>`-ban. Sokat spórol forgalomból és válaszidőből. Ha a kiadások tábla nem fér bele a 10 KB-ba, a fejléc elhagyható azon a hívásra.

### Két response eset

**a) Egytalálatos eset — redirect**
A moly.hu egyetlen találatnál közvetlenül a könyv adatlapjára irányít:
`https://moly.hu/konyvek/{slug}`
Detektálás: ha a végső URL `/konyvek/` prefixű, ez az eset állt fenn — a lista parsolása kihagyható.

**b) Többtalálatos eset — listaoldal**
Lista oldal jelenik meg `<a href="/konyvek/{slug}">` linkekkel.
Az első találat linkjét kell követni → könyv adatlap URL.

### Adatbányászat a könyv adatlapról (`/konyvek/{slug}`)

**Gyors mezők — `<head>` og: meta tagekből:**

| Mező | Selector | Megjegyzés |
|------|----------|------------|
| `title` + `authors` | `<meta property="og:title">` | Formátum: `"Szerző: Cím"` — kettőspontnál szét kell vágni |
| `coverImageUrl` | `<meta property="og:image">` | Közepes méretű cover URL |
| leírás *(nem `IsbnLookupResult` mező)* | `<meta property="og:description">` | Fázis 5-ben könyv felvételnél opcionálisan használható |

**Kiadás-specifikus mezők — kiadások táblából:**
Az oldalon a `Kiadások` szekcióban (`<table class="table">` vagy `<div class="edition">` blokkok) több sor lehet — a keresett ISBN-nel megegyező sort kell azonosítani.

| Mező | Elérhetőség |
|------|-------------|
| `publishYear` | Elérhető (kiadás évszáma) |
| `publisher` | Elérhető |
| `language` | Nem biztos — ha nem scrape-elhető: `null` |
| `categories` | Nem biztos — ha nem scrape-elhető: üres lista |

> **Törékenység:** a scraping érzékeny a moly.hu UI változásaira. Ha az oldal megváltozik, a selectorokat frissíteni kell. Elfogadott kompromisszum — ez a legjobb elérhető magyar adatforrás.

---

## Elfogadási kritériumok

- Egytalálatos (redirect) esetben a végső URL alapján felismeri az esetet, parsolja az adatlapot
- Többtalálatos esetben az első találat linkjét követi
- Ismert magyar ISBN-re kitöltött `Optional<IsbnLookupResult>`-ot ad vissza (`source = MOLY_HU`)
- Ismeretlen ISBN-re `Optional.empty()`-t ad vissza
- Minden kérésben szerepel az azonosító `User-Agent` és a `Range` fejléc
- Timeout esetén kivételt dob (a service kezeli)
- Unit teszt: Jsoup-pal parsolt moly.hu HTML fixture (redirect eset + lista eset) → helyes leképezés ellenőrzése
