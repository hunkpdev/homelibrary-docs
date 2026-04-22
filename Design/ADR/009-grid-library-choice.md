# ADR-009: Grid könyvtár — AG Grid Community (TanStack Table helyett)

**Dátum:** 2026-04-22
**Státusz:** Elfogadva

## Kontextus

A kezdeti tervekben TanStack Table szerepelt grid/táblázat könyvtárként, elsősorban azért, mert headless megközelítésével elméletileg jól illeszkedik a shadcn/ui + Tailwind stack-hez. A frontend implementáció megkezdésekor azonban kiderült, hogy a TanStack Table rendkívül nagy mennyiségű boilerplate JSX-et és hook-logikát igényel már egy alapszintű, szűrhető + lapozott + csoportosítható táblázathoz is. Mivel az alkalmazásban több listanézet lesz (locations, könyvek, kölcsönzők), ez a boilerplate minden táblánál megismétlődne.

## Döntés

**AG Grid React Community** édition váltja fel a TanStack Table-t minden táblázatos nézetben.

## Indoklás

| Szempont | TanStack Table | AG Grid Community |
|----------|---------------|-------------------|
| Boilerplate | Jelentős — hook + JSX minden funkcióhoz | Minimális — deklaratív `columnDefs` konfiguráció |
| Szűrés / rendezés / lapozás | Manuálisan kell bekötni | Beépített, `columnDef`-szintű konfiguráció |
| Infinite / server-side model | Manuális implementáció | Beépített Infinite Row Model |
| Dark mode / theming | Tailwind-natív | Saját témarendszer (ag-theme-quartz), dark módot támogat |
| Licenc | MIT | Community: MIT |
| Bundle méret | ~15 KB | ~500 KB — ennél a skálánál elfogadható |

## Korlátok (tudatos tradeoff)

- **Row Grouping** AG Grid-ben Enterprise feature — a Community kiadásban nem érhető el natívan. Az alkalmazás kliens oldali csoportosítást (rooms → locations) JavaScript szinten old meg, nem AG Grid grouping API-val. Ez a megközelítés elfogadható: az adatmennyiség kicsi, a csoportosítás logikája egyszerű.
- **Styling integráció:** Az AG Grid saját CSS tema rendszerével dolgozik — a grid belseje nem Tailwind-stílusú. A shadcn/ui komponensek (modalok, gombok, badge-ek) érintetlenek maradnak, csak a tábla maga renderelődik AG Grid témával.

## Infinite Row Model és backend kompatibilitás

A Spring Boot backend `Page<T>` alapú lapozást ad vissza (offset + limit). Az AG Grid Infinite Row Model pontosan ezt a mintát várja: `startRow` / `endRow` paraméterekkel hívja a szervert, és az eredményt cache-eli. Külön backend módosítás nem szükséges.

## Alternatívák

### A: TanStack Table megtartása
- ✅ Tailwind-natív styling
- ❌ Rengeteg boilerplate — minden táblánál újra
- ❌ Grouping, filter panel, infinite scroll mind manuálisan bekötendő

### B: AG Grid Community (választott)
- ✅ Konfigurációs API — minimális kód
- ✅ Beépített szűrés, rendezés, lapozás, infinite scroll
- ✅ Dark mode támogatás
- ⚠️ Saját témarendszer (nem Tailwind)
- ⚠️ Row Grouping csak Enterprise-ban — kliens oldali csoportosítással kiváltható

### C: AG Grid Enterprise
- ✅ Natív Row Grouping
- ❌ Fizetős — indokolatlan ennél a skálánál

## Következmények

- `ARCHITECTURE.md`: Grid/táblázat sor frissítése
- `Plans/phase1-feature-order.md`: Step 3.9 és 5.7 leírása frissítve
- `Plans/specs/feature-locations-crud/frontend/step-3.9-rooms-locations-page.md`: TanStack → AG Grid konfiguráció
- Dependency: `ag-grid-react` + `ag-grid-community` csomag a `package.json`-be
