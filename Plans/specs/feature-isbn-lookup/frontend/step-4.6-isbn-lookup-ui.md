# Step 4.6 – ISBN lookup UI

## Mit állít elő

- `src/components/isbn/IsbnLookupPanel.tsx` — összetett panel: scanner + API hívás + eredmény megjelenítés

---

## Működés

1. `IsbnScannerInput` (4.5) adja az ISBN-t az `onScan` callbacken keresztül
2. Panel meghívja `GET /api/books/isbn/{isbn}`
3. Eredmény alapján:
   - `found: true` → `onResult` callback meghívása a kitöltött adatokkal (szülő tölti elő a könyv formot)
   - `found: false` → figyelmeztető üzenet megjelenítése, újrapróbálkozás lehetősége
   - `rateLimitExceeded: true` (429) → tájékoztató üzenet DEMO rate limit elérésről, újrapróbálkozás nem lehetséges
   - Hálózati hiba → hibaüzenet, újrapróbálkozás

---

## Kulcs döntések

- A panel maga **nem** tartalmaz könyv form mezőket — csak az ISBN bevitelt és a találat visszajelzést kezeli
- Könyv form integráció Feature 5-ben történik: az `IsbnLookupPanel` `onResult: (result: IsbnLookupResult) => void` propot kap, amin keresztül a szülő (könyv felvétel form) kapja meg az adatokat
- API hívás közben `isLoading` állapot → `IsbnScannerInput` letiltva + spinner
- `IsbnLookupResult` TypeScript interface: megfelel a backend `IsbnLookupResponse` struktúrájának (4.4)
- Nem kerül önálló route-ra — beágyazott komponens, Feature 5 könyv felvétel formjába illesztve
- DEMO rate limit üzenet (429): egyértelmű tájékoztatás, ne legyen generikus hibaüzenet

---

## Elfogadási kritériumok

- ISBN scan / bevitel után API hívás indul, közben input letiltva
- `found: true` → `onResult` meghívódik a helyes adatokkal
- `found: false` → felhasználóbarát üzenet jelenik meg, újrapróbálkozás lehetséges
- 429 rate limit → tájékoztató üzenet jelenik meg (nem generikus hiba)
- Hálózati hiba → hibaüzenet + újrapróbálkozás gomb
- DEMO role esetén a panel és a lookup funkció elérhető (GET hívás megengedett)
