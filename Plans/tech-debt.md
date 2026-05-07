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

### AG Grid — mobilos oszlopoptimalizálás

**Érintett fájlok (frontend):**
- `src/pages/LocationManagementPage.tsx` — AG Grid `columnDefs`

**Teendő:** Kis képernyőn (`xs`/`sm` breakpoint) egyes oszlopok elrejtése, pl. `description`. Az AG Grid Community `hide` property responsive breakpointokhoz kötve, vagy CSS media query alapján dinamikusan állítva.

**Forrás:** Feature 3 frontend tervezés során azonosítva — step-3.9 spec.

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

### `OszkNektarClient` tesztelhetőség

**Teendő:** A natív lib betöltést és Z39.50 connection-managementet tartalmazó osztály unit teszt nélküli, és a Sonar coverage exclusion alá esik (`pom.xml` `<sonar.coverage.exclusions>`). A `loadNativeLibrary()` extrakciója + factory pattern bevezetése (`Connection` factory injektálással) javítaná a tesztelhetőséget.

**Forrás:** Feature 4 backend code review (2026-05-05). Az integrációs tesztelés MARC/Z39.50 mock nélkül komplex — pályázati átadás után újragondolandó.

---

## Lezárt

*(még üres)*
