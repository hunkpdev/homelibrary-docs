# HomeLibrary – Projekt Leírás

> **Dokumentáció repo:** `homelibrary-docs`
> **Kód repo:** `homelibrary`
> **Utolsó frissítés:** 2026-04-29

## Mi ez?

Webalapú elektronikus házi könyvtárkezelő alkalmazás, amellyel egy háztartás nyilvántarthatja fizikai könyveit: hol találhatók, ki olvassa őket, ki kapta kölcsön.

## Célközönség

- **Elsődleges felhasználók:** egy háztartás tagjai (tipikusan 2-5 fő)
- **Másodlagos felhasználók:** meghívott ismerősök, barátok, akik böngészési jogot kapnak

## Jogosultságok

| Szerepkör | Mit tehet |
|-----------|-----------|
| `ADMIN` | Mindent: könyv felvétel, módosítás, törlés, áthelyezés, felhasználókezelés |
| `VISITOR` | Böngészés, keresés, listázás – módosítás nem |
| `DEMO` | Portfolio bemutató szerepkör — Fázis 1 minden oldal látható (Felhasználókezelés kivételével). Csak biztonságos HTTP metódusok (GET, HEAD, OPTIONS) engedélyezve — minden state-modifying művelet (POST/PUT/PATCH/DELETE) GUI-n disabled, API-n 403-mal elutasítva. |

## Főbb funkciók

### Könyv felvétel
- Mobilos kamerával vonalkód (ISBN) beolvasás (elsődleges)
- Kézi ISBN bevitel (fallback, ha kamera nem működik)
- ISBN alapján automatikus adatlehívás külső API-ból (cím, szerző, kiadó, év, kategória, borítókép)
- Helyszín megadása: helyiség → szekrény/polc
- Kezdeti státusz beállítása

### Könyvtár kezelés
- Listázás és szűrés: szerző, kategória, kiadási év, helyiség, polc, státusz szerint
- Szabad szöveges keresés cím és szerző alapján
- Státusz módosítás: `AT_HOME` → `LOANED` (kinek?) → `AT_HOME` (visszavéve)
- Áthelyezés: helyiség/polc módosítás
- Törlés (soft delete: az adat megmarad, csak `DELETED` státuszba kerül)

### Megjelenés
- Responsive, mobil-first design
- Light / Dark mód váltás (felhasználónként megjegyzett)
- Többnyelvű felület (i18n): magyar és angol (bővíthető)
- A könyv leírása mindig a bejelentkezett felhasználó nyelvén jelenik meg

### Autentikáció (saját "mini IDM")
- Bejelentkezés felhasználónév + jelszóval
- JWT access token (rövid életű) + refresh token (HttpOnly cookie)
- Három szerepkör: `ADMIN`, `VISITOR` és `DEMO` (lásd Fázis 1 Demo role)
- Jelszó visszaállítás: Fázis 2

### Demo role (Fázis 1)
- Beégetett demo user külön `DEMO` szerepkörrel — portfólió bemutatóhoz
- Minden oldal és form látható (Felhasználókezelés kivételével)
- Mutáció gombok (mentés, törlés, küldés) GUI-n disabled
- State-modifying műveletek (POST/PUT/PATCH/DELETE) API-n 403-mal visszautasítva
- Kizárólag biztonságos HTTP metódusok (GET, HEAD, OPTIONS) engedélyezve
- Implementáció: `<MutationButton>` wrapper komponens auto-disabled DEMO role esetén; Spring Security whitelist szabály (GET/HEAD DEMO-engedélyezett, `anyRequest()` ADMIN-only)

### AI fordítás (Fázis 3)
- Ha a mentett leírás nyelve ≠ felhasználó aktuális nyelve → AI API hívással fordítás
- Jelölt megoldások: Google Gemini API vagy DeepL free tier (Fázis 3-ban döntünk)
- Cache-elés: lefordított szöveg DB-ben tárolva (ne hívjuk minden alkalommal)
- Cél: AI feature integráció tapasztalat szerzés, és általános kiadhatóság előkészítése

## Fázisok

### Fázis 1 – MVP (core funkciók)
- [ ] Auth (bejelentkezés, JWT, három szerepkör)
- [ ] ISBN lookup (OSZK NEKTÁR Z39.50 + manuális fallback) + vonalkód olvasás kamerával (react-zxing, elsődleges) / kézi bevitel (fallback)
- [ ] Könyv CRUD (felvétel, listázás, módosítás, soft delete)
- [ ] Helyiség/polc kezelés
- [ ] Státusz kezelés (AT_HOME, LOANED, DELETED)
- [ ] Kölcsönzés nyilvántartás (kinek, mikor, visszahozta-e)
- [ ] Alap responsive UI (React), dark mode, i18n (hu/en): böngésző locale alapján automatikus nyelvválasztás (magyar → hu, egyéb → en fallback), manuális váltás zászló ikonnal a menüben (az ikon mindig a másik nyelvet jelöli)
- [ ] AWS deploy (Lambda + API Gateway + S3 + CloudFront)
- [ ] CI/CD (GitHub Actions) + SonarQube Cloud
- [ ] Demo role (portfólió bemutató) — lásd Főbb funkciók / Demo role

### Fázis 2 – Kényelem
- [ ] Borítókép megjelenítés (S3 pre-signed URL)
- [ ] Jelszó visszaállítás
- [ ] Helyszín másolása: meglévő location „Másolás" gombja előtölti a létrehozó modalt (room, name, description egyaránt), a felhasználó csak a különbséget módosítja — tipikus eset: polcos szekrény több polca. Az új location `bookCount = 0`-val jön létre (a form előtölti a többi mezőt, de a könyvszám nem másolódik).
- [ ] Témaváltó: shadcn/ui gyári témák közötti váltás (a dark/light toggle megmarad, ehhez képest külön szín-téma választó) — perzisztencia helye (localStorage vs `users` tábla) Fázis 2 spec tervezésekor döntendő.

### Fázis 3 – AI & kiterjesztés
- [ ] AI alapú leírás-fordítás (Gemini vagy DeepL – tesztelés után döntés)
- [ ] Haladó szűrők, statisztikák (pl. könyvek kategóriánként)
- [ ] Natív mobilapp előkészítés (API már kész, csak UI kell)
- [ ] Általános kiadhatóság: multi-tenant vagy self-hosted opció

## Nem terjed ki (egyelőre)
- E-book kezelés (csak fizikai könyvek)
- Vásárlási javaslatok
- Olvasási előrehaladás követése
- Natív mobilapp (Fázis 3 után)
- Social login (Google, Facebook)
- MFA

## Külső API függőségek

| API / Protokoll | Mire | Limit | Kulcs kell? |
|-----------------|------|-------|-------------|
| OSZK NEKTÁR Z39.50 (`tagetes2.oszk.hu:1616`) | ISBN lookup | Ingyenes, 03:30–23:00 | Nem |
| Google Gemini API vagy DeepL | Leírás fordítás (Fázis 3) | Ingyenes tier | Igen |

## Repók

| Repo | Tartalom | Szerkeszti |
|------|----------|------------|
| `homelibrary-docs` | Tervezési dokumentumok, ADR-ek | Architekt / PO |
| `homelibrary` | Forráskód (backend, frontend, infra) | Fejlesztő |
