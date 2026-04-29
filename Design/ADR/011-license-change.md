# ADR-011: Licencváltás PolyForm Personal Use 1.0.0-ra

**Dátum:** 2026-04-29  
**Státusz:** Elfogadva

## Kontextus

A `homelibrary` repón az eredeti "MIT License — View Only" szöveg valójában proprietary / All Rights Reserved jellegű, csak forráskód-olvasást engedett. Ez két problémát okozott:

1. **Library-kompatibilitás:** AGPL v3 licencű library-k (pl. JZKit2) behúzása megköveteli az alkalmazás AGPL-lé tételét — az eredeti proprietary licenccel inkompatibilis.
2. **Portfolio-szándékkal ellentétes:** a "view only" licenc megakadályozza, hogy mások tanuljanak belőle vagy fork-olják — nyílt portfolio projektnél nem kívánatos.

## Döntés

**PolyForm Personal Use 1.0.0** (https://polyformproject.org/licenses/personal-use/1.0.0/)

- Engedi: forráskód megtekintése, módosítás, fork, tanulás — non-commercial magáncélra
- Tiltja: kereskedelmi felhasználás, szervezeti felhasználás
- "Personal use" definíciója szigorú (természetes személy + magáncél) — portfolio-bemutató szándékkal kompatibilis

## Miért PolyForm Personal Use és nem más?

| Licenc | Ok |
|---|---|
| MIT / Apache 2.0 | Nem tiltja a kereskedelmi felhasználást — túl permissive |
| AGPL v3 | Forráskód publikálást követel módosítás esetén — nem cél |
| CC BY-NC 4.0 | Szoftverhez nem ajánlott (Creative Commons nem szoftver-körülményekre tervezett) |
| PolyForm Personal Use 1.0.0 | Bevett szoftver-specifikus licenc, non-commercial védelem + tanulás engedve |

## YAZ BSD-3-Clause kompatibilitás

BSD-3-Clause egyirányúan permissive — semmilyen extra megkötés nincs a saját kódra. Egyetlen kötelezettség: YAZ copyright + licensz szöveg mellékelése (`THIRD_PARTY_LICENSES.md`).

## Következmények

- `LICENSE` fájl cseréje a PolyForm Personal Use 1.0.0 standard szövegére
- `README.md` licenc-szekció frissítése
- Új `THIRD_PARTY_LICENSES.md` fájl — érintett library-k (licenc verifikálandó implementáció zárásáig):
  - `yaz4j` (BSD-3-Clause), `libyaz` upstream (BSD-3-Clause) — Index Data copyright
  - `marc4j` (Apache-2.0) — NOTICE attribution
  - `native-lib-loader` (BSD-2-Clause — verifikálandó)
  - `react-zxing` + `@zxing/library` (Apache-2.0 vagy MIT — verifikálandó) — frontend bundle-be kerül, itt is dokumentálandó
- Feature 4 implementáció zárásaikor: minden függőség licence verifikálva és attribution beillesztve
- A licencváltás a `homelibrary` (forráskód) repót érinti
