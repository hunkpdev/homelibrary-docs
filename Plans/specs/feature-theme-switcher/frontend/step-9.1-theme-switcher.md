# Step 9.1 – Témaváltó

## Mit állít elő

- `src/hooks/useTheme.ts` módosítás — `colorTheme` / `setColorTheme` bővítés a meglévő `ThemeContext`-en
- `src/components/layout/ThemeSelector.tsx` — új dropdown komponens
- `src/index.css` módosítás — 5 plusz színtéma CSS változó blokkjai (light + dark variáns)
- `src/locales/en.json`, `src/locales/hu.json` módosítás — `theme.*` i18n kulcsok
- `src/components/layout/AppSidebar.tsx` módosítás — `ThemeSelector` bekötése a sidebar footer-be

---

## Témák

6 szín-téma: `zinc` (alapértelmezett), `rose`, `blue`, `orange`, `green`, `violet`.

`zinc` a shadcn/ui alapértelmezett palettája — külön CSS override nem szükséges.
A többi 5 témához `[data-color-theme="<name>"]` és `[data-color-theme="<name>"].dark` szelektorok
definiálják az OKLCH alapú CSS custom property override-okat az `index.css`-ben.

---

## Perzisztencia

- Tárolás: `localStorage`
- Kulcs: `colorTheme_${userId}` (bejelentkezett) / `colorTheme` (anonymous)
- Default: `zinc`
- Bejelentkezés / kijelentkezés esetén a `userId` változásra az állapot az adott user (vagy anonymous) kulcsáról töltődik újra

---

## CSS mechanizmus

A `useTheme` hook minden témaváltáskor beállítja a `data-color-theme` attribútumot a `<html>` elemen.
A dark/light állapot és a szín-téma egymástól független — a kettő kombinációja (`[data-color-theme="rose"].dark`) adja a sötét variáns megjelenést.

---

## ThemeSelector UI

shadcn `Select` dropdown, kisméretű (`h-8 w-[110px]`).

Minden opció: kerek szín-minta swatch + i18n témanév (`t('theme.<name>')`).  
Pozíció: sidebar footer, `DarkModeToggle` mellé kerül (egy sorban), logout gomb fölé.

```
┌─────────────────────────────────┐
│  sidebar footer                 │
│  [ ☀️/🌙 ] [ Zinc ▾ ]          │
│  [ Kijelentkezés ]              │
└─────────────────────────────────┘
```

---

## Elfogadási kritériumok

- `ThemeSelector` megjelenik a sidebar alján, `DarkModeToggle` mellé
- 6 téma érhető el, mindegyik a saját swatch-ával
- Témaváltáskor a `data-color-theme` attribútum azonnal frissül a `<html>` elemen
- Dark/light toggle és szín-téma egymástól függetlenül működik
- Kiválasztott téma localStorage-ban marad, oldal újratöltés után is megmarad
- Bejelentkezéskor a user saját mentett témája töltődik be
- Kijelentkezéskor anonymous kulcsra vált vissza; alapértelmezett: `zinc`
