# Tech debt & jövőbeli teendők

Olyan feladatok, amelyek nem tartoznak aktív feature-höz, de határidőre vagy opportunisztikusan elvégzendők.

---

## Nyitott

### Strukturált hibaválasz (ErrorResponse) bevezetése

**Teendő:** A `GlobalExceptionHandler` jelenleg minden hibánál üres body-jú (`ResponseEntity<Void>`) választ ad. Strukturált `ErrorResponse` record bevezetése szükséges (timestamp, status, message, path mezőkkel).

**Forrás:** API_DESIGN.md „Egységes Hibastruktúra" szekció — az előírt formátum és a tényleges implementáció eltér.

---

### DELETE /api/locations/{id} — aktív könyvek ellenőrzése hiányzik

**Teendő:** A `LocationService.delete` metódus még nem ellenőrzi, hogy a location-höz tartoznak-e aktív könyvek. Implementálandó Feature 5 step 5.4-ben: `ActiveChildException` → 409 Conflict.

**Forrás:** A `books` tábla Feature 3 idején még nem létezik; szándékosan halasztva. Kódban `// TODO(step-5.4)` comment jelöli.

---

### GitHub Actions — Node.js 24 migráció

**Határidő:** 2026-06-02 (forced Node.js 24 default)

**Érintett fájlok:**
- `.github/workflows/backend-deploy.yml`
- `.github/workflows/frontend-deploy.yml`

**Teendő:** `actions/checkout`, `actions/setup-java`, `dorny/test-reporter` action-ök Node.js 24-kompatibilis verziókra frissítendők. A jelenlegi verziók Node.js 20-at használnak, amely 2026-06-02 után deprecated lesz a GitHub Actions futtatókörnyezetben.

**Forrás:** Step 1.15 implementáció közben azonosítva.

---

### Feature 3 → Feature 5: LocationService book check és bookCount

**Érintett fájlok (Feature 5, step 5.4 után):**
- `LocationService.java` — soft delete bővítése: aktív könyv ellenőrzés (`books` tábla, `location_id` alapján) → 409 Conflict
- `LocationService.java` — `bookCount` kiszámítása: hardcoded `0`-ról tényleges GROUP BY count query-re váltás
- `LocationResponse.java` — `bookCount` valódi értékkel

**Forrás:** Feature 3 tervezés során azonosítva — a `books` tábla Feature 3 idején még nem létezik.

---

### Frontend hiba-üzenetek differenciálása

**Teendő:** Jelenleg minden API-hiba `common.errorUnexpected`-et mutat. Axios-szal technikailag megkülönböztethető a network error (`!err.response`) és a szerver 500-as hiba, de háztartási skálán a nyereség minimális — a felhasználónak mindkét esetben ugyanazt kell tennie (újratöltés). Újragondolandó, ha élesebb felhasználói bázis vagy SLA-elvárások merülnek fel.

---

### DEMO ISBN napi limit race condition

**Teendő:** A `DemoIsbnRateLimitService.incrementDaily()` `findAll → setLookupCount → save` szekvenciája nem atomikus. Két párhuzamos Lambda instance esetén a számláló 1-2-vel csúszhat (50 helyett 51-52-nek hagyhatja a napi limitet).

**Forrás:** v2 spec review (2026-04-29), elfogadott race condition pályázati átadás miatt — DEMO felhasználónál párhuzamos kérés gyakorlatilag kizárt (ADR-003 elemzés alapján). Optimistic locking (`@Version`) vagy atomic SQL UPDATE bevezetése Feature 5 utáni iterációban.

---

### Location bookCount badge — books lista előszűrése helyszínre

**Ötlet:** A Locations oldalon a `bookCount` badge kattinthatóvá tehető — navigál a Books listára, ahol a `locationId` szűrő előre be van állítva az adott helyszínre. Ez implicit helyszín szerinti szűrést ad a Books listán anélkül, hogy a Books grid szűrősorában helyszín szűrőt kellene fenntartani.

**Állapot:** Fázis 1-ben nincs szűrősor a Books gridon — ötletként rögzítve Fázis 2-re.

---

### IsbnLookupPanel reset pattern

**Teendő:** A scanner / panel UI újrapróbálkozás-állapot-átmeneteinek pattern-je nem egységes (`isLoading`, hibaüzenet, ISBN újragépelés vs. újrabeolvasás scenariók). A Feature 5 `BookAddModal` "Kézi bevitel" linkkel megkerülhető, de a panel saját retry UX-e még nincs lezárva.

**Forrás:** Feature 4 frontend code review F-W4.

---

### `OszkNektarClient` tesztelhetőség

**Teendő:** A natív lib betöltést és Z39.50 connection-managementet tartalmazó osztály unit teszt nélküli, és a Sonar coverage exclusion alá esik (`pom.xml` `<sonar.coverage.exclusions>`). A `loadNativeLibrary()` extrakciója + factory pattern bevezetése (`Connection` factory injektálással) javítaná a tesztelhetőséget.

**Forrás:** Feature 4 backend code review (2026-05-05). Az integrációs tesztelés MARC/Z39.50 mock nélkül komplex — pályázati átadás után újragondolandó.

---

### Books API dokumentáció — `search` és `publishYear` eltér a tényleges implementációtól

**Teendő:** A `Design/API_DESIGN.md` és a step-5.5 (`BookService`) / step-5.6 (`BookController`) backend specek egy kombinált `search` (LIKE cím ÉS szerző, OR) query paramétert és `publishYear`-t Integer exact-match mezőként dokumentálnak. A tényleges, step-5.9-cel (frontend, implementáció-szinkronizált) igazolt kontraktus ettől eltér: külön `isbn` (prefix), `title` (contains), `authors` (contains) paraméterek, és `publishYear` String-ként, prefix-kereséssel (`"202"` → 2020–2029). A backend-oldali doksi (API_DESIGN.md, step-5.5, step-5.6) frissítendő, hogy a tényleges implementációt tükrözze — hasonlóan a korábbi `fix/feature-5-impl-doc-sync` PR-hoz.

**Forrás:** Feature 10 (Mobil kártyás nézet, step 10.2) tervezése során azonosítva, 2026-08-07 — a kártyás nézet a step-5.9-ben rögzített, tényleges kontraktust veszi alapul, és eközben derült ki az API_DESIGN.md-vel való eltérés.

---

### CloudFront `errorResponses` 403/404 → 200 leképezése elfedi a hiányzó asseteket

**Teendő:** Az SPA deep-linkhez szükséges a 403/404 → `index.html` 200-as leképezés, de mellékhatásként egy hiányzó JS chunk kérésére HTML érkezik 200-as státusszal, ami néma fehér képernyőt okoz JS syntax error formájában. A helyes cache header-ek bevezetése (ld. step-1.14-github-actions.md) ezt a gyakorlatban megszünteti, mert elavult chunk-hash-re már nem fut rá kliens. Ha később mégis kell rá védelem, a leképezést érdemes az `/assets/*` útvonalra kizárni egy külön cache behavior-rel.

**Forrás:** Elavult bundle cache bug diagnózisa, 2026-08 (`l:\AI reviews\bug-stale-bundle-cache-headers.md`, 4. szakasz).

---

## Lezárt

### AG Grid — mobilos oszlopoptimalizálás

**Eredeti teendő volt:** Kis képernyőn (`xs`/`sm` breakpoint) egyes AG Grid oszlopok elrejtése (pl. `description`) a Locations gridnél; a Books grid analóg tétele (step-5.9 „Tech-debt" szekciója) ugyanide tartozik, sosem lett önálló bejegyzésként felvéve.

**Miért okafogyott — feltételesen (2026-08-07):** A lezárás a **tervre** vonatkozik, nem tényleges implementációra: Feature 10 (Mobil kártyás nézet, step 10.2 + 10.3, korábban „5.5") specifikációja szerint mindkét lista (Books, Locations) kártyás nézetre vált mobil breakpoint alatt, ahol az AG Grid többé nem renderelődik — így nincs mit oszlopoptimalizálni. **Feltétel:** ez csak akkor áll, ha a Feature 10 ténylegesen, a jelenlegi scope szerint elkészül és mergelődik (kód-repóban). Ha a feature descope-olódik vagy csak részben valósul meg, ez a tétel újra megnyitandó.

**Forrás:** Feature 3 frontend tervezés során azonosítva — step-3.9 spec (Locations); step-5.9 spec (Books, analóg tétel).
