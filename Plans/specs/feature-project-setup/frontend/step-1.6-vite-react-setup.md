# Step 1.6 – Vite + React + TypeScript projekt setup

## Mit állít elő

- `frontend/` könyvtár a projekt gyökerében
- Vite + React + TypeScript projekt teljes függőség konfigurációval
- Alap könyvtárstruktúra

---

## Technológiai stack

| Csomag | Verzió | Mire |
|--------|--------|------|
| `vite` | latest | Build tool |
| `react` + `react-dom` | 19.x | UI framework |
| `typescript` | 5.x | Típusosság |
| `tailwindcss` | 4.x | Utility-first CSS |
| `shadcn/ui` | latest | Komponens könyvtár (Tailwind alapú) |
| `react-router-dom` | 7.x | Kliens oldali routing |
| `zustand` | 5.x | Globális state management |
| `axios` | 1.x | HTTP kliens |
| `i18next` + `react-i18next` | latest | Internationalizáció (hu/en) |

---

## Könyvtárstruktúra

```
frontend/
├── src/
│   ├── api/          ← Axios instance (step 1.8)
│   ├── components/   ← Újrafelhasználható UI komponensek
│   ├── pages/        ← Oldal szintű komponensek
│   ├── store/        ← Zustand store-ok
│   ├── locales/      ← i18next fordítási fájlok (hu.json, en.json)
│   ├── lib/          ← shadcn/ui utils (cn helper)
│   ├── App.tsx       ← Router konfiguráció
│   └── main.tsx      ← Belépési pont
├── public/
├── index.html
├── vite.config.ts
├── tsconfig.json
└── package.json
```

---

## i18next alap konfiguráció

- Támogatott nyelvek: `hu`, `en`
- Alapértelmezett nyelv: `hu`
- Fordítási fájlok: `src/locales/hu.json`, `src/locales/en.json` (kezdetben üres objektumok)
- Inicializálás: `src/main.tsx`-ben, az alkalmazás mountolása előtt

---

## Elfogadási kritériumok

- `npm run dev` — a fejlesztői szerver elindul, az alkalmazás elérhető `http://localhost:5173`-on
- `npm run build` — TypeScript fordítás hiba nélkül, `dist/` könyvtár generálódik
- shadcn/ui `Button` komponens renderelhető (smoke test)
