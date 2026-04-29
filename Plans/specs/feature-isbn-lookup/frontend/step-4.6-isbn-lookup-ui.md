# Step 4.6 – ISBN lookup UI

## Mit állít elő

- `src/components/isbn/IsbnLookupPanel.tsx` — összetett panel: scanner + API hívás + eredmény megjelenítés

## Működés

1. `IsbnScannerInput` (4.5) adja az ISBN-t az `onScan` callbacken keresztül
2. Panel meghívja `GET /api/books/isbn/{isbn}`
3. Eredmény alapján:
   - `found: true` → `onResult` callback meghívása a kitöltött adatokkal (szülő tölti elő a könyv formot)
   - `found: false` → tájékoztató üzenet (*„Nem találtuk az adatbázisban, töltsd ki kézzel"*) + `onResult` callback meghívása `null`-lal, hogy a szülő (Feature 5 könyv felvétel form) üres állapotban, manuális kitöltésre nyitva jelenjen meg, az ISBN előtöltve. Újrapróbálkozás gomb is megjelenik (ha a felhasználó újra akarja gépelni az ISBN-t).
   - `rateLimitExceeded: true` (429) → tájékoztató üzenet DEMO rate limit elérésről, újrapróbálkozás nem lehetséges
   - Hálózati hiba → hibaüzenet, újrapróbálkozás

## Kulcs döntések

- A panel maga **nem** tartalmaz könyv form mezőket — csak az ISBN bevitelt és a találat visszajelzést kezeli
- Könyv form integráció Feature 5-ben történik: az `IsbnLookupPanel` `onResult: (result: IsbnLookupResult) => void` propot kap, amin keresztül a szülő (könyv felvétel form) kapja meg az adatokat
- API hívás közben `isLoading` állapot → `IsbnScannerInput` letiltva + spinner; az első keresés Lambda cold start + Z39.50 connection init miatt 5-7 másodpercig tarthat — a spinner mellé tájékoztató szöveg: *„Adatbázishoz csatlakozás…"*
- `IsbnLookupResult` TypeScript interface: megfelel a backend `IsbnLookupResponse` struktúrájának (4.4)
- Nem kerül önálló route-ra — beágyazott komponens, Feature 5 könyv felvétel formjába illesztve
- DEMO rate limit üzenet (429): egyértelmű tájékoztatás, ne legyen generikus hibaüzenet

## Elfogadási kritériumok

- ISBN scan / bevitel után API hívás indul, közben input letiltva
- `found: true` → `onResult` meghívódik a helyes adatokkal
- `found: false` → tájékoztató üzenet + `onResult(null)` meghívva → szülő manuális módba vált, ISBN előtöltve; újrapróbálkozás gomb elérhető
- 429 rate limit → tájékoztató üzenet jelenik meg (nem generikus hiba)
- Hálózati hiba → hibaüzenet + újrapróbálkozás gomb
- DEMO role esetén a panel és a lookup funkció elérhető (GET hívás megengedett)
