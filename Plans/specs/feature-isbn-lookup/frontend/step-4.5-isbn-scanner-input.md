# Step 4.5 – IsbnScannerInput komponens

## Mit állít elő

- `src/components/isbn/IsbnScannerInput.tsx` — ISBN beviteli komponens (kamera + kézi)

## Működés

Két mód, automatikusan váltva:

| Mód | Feltétel | Megjelenés |
|-----|----------|------------|
| Kamera (elsődleges) | `MediaDevices.getUserMedia` elérhető és kamera van | Élő kamera nézet, vonalkód felismerés |
| Szövegmező (fallback) | Kamera nem elérhető vagy engedély megtagadva | Sima `<input>` ISBN bevitelhez |

Ha a kamera elérhető, alapból aktív — nincs szükség felhasználói váltásra.

## Kulcs döntések

- Barcode könyvtár: `react-zxing` (`useZxing` hook)
- Kamera detektálás: `navigator.mediaDevices?.getUserMedia({ video: true })` — ha kivételt dob vagy undefined → fallback módra vált
- Felismert / beírt ISBN visszaadása: `onScan: (isbn: string) => void` callback prop
- A komponens **nem** hívja a backend API-t — csak az ISBN értéket adja vissza
- Kamera módban sikeres felismerés után a komponens megáll (nem folytatja a szkennelést) — újraindítás szülő komponens általi reset-tel
- `isLoading` prop: szkennelés letiltása API hívás idejére (szülő vezérli)

## Elfogadási kritériumok

- Kamera elérhető esetén a kamera nézet jelenik meg alapból
- Kamera nem elérhető esetén szövegmező jelenik meg
- Vonalkód felismerés után `onScan` callback meghívódik az ISBN értékkel
- Kézi bevitelnél Enter / keresés gomb megnyomásra `onScan` meghívódik
- `isLoading = true` esetén az input/kamera inaktív
