# HomeLibrary – Projekt Leírás

> **Dokumentáció repo:** `homelibrary-docs`
> **Kód repo:** `homelibrary`
> **Utolsó frissítés:** 2026-03-28

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
- Két szerepkör: `ADMIN` és `VISITOR`
- Jelszó visszaállítás: Fázis 2

### AI fordítás (Fázis 3)
- Ha a mentett leírás nyelve ≠ felhasználó aktuális nyelve → AI API hívással fordítás
- Jelölt megoldások: Google Gemini API vagy DeepL free tier (Fázis 3-ban döntünk)
- Cache-elés: lefordított szöveg DB-ben tárolva (ne hívjuk minden alkalommal)
- Cél: AI feature integráció tapasztalat szerzés, és általános kiadhatóság előkészítése

## Fázisok

### Fázis 1 – MVP (core funkciók)
- [ ] Auth (bejelentkezés, JWT, két szerepkör)
- [ ] ISBN lookup (OpenLibrary API + Google Books fallback) + vonalkód olvasás kamerával (react-zxing, elsődleges) / kézi bevitel (fallback)
- [ ] Könyv CRUD (felvétel, listázás, módosítás, soft delete)
- [ ] Helyiség/polc kezelés
- [ ] Státusz kezelés (AT_HOME, LOANED, DELETED)
- [ ] Kölcsönzés nyilvántartás (kinek, mikor, visszahozta-e)
- [ ] Alap responsive UI (React), dark mode, i18n (hu/en): böngésző locale alapján automatikus nyelvválasztás (magyar → hu, egyéb → en fallback), manuális váltás zászló ikonnal a menüben (az ikon mindig a másik nyelvet jelöli)
- [ ] AWS deploy (Lambda + API Gateway + S3 + CloudFront)
- [ ] CI/CD (GitHub Actions) + SonarQube Cloud
- [ ] Demo role (portfólió bemutató): beégetett demo user külön `DEMO` szerepkörrel — minden oldal és form elérhető (beleértve az adatfelviteli formokat és az ISBN keresőt), de a backend mutációt indító gombok (mentés, törlés, küldés) le vannak tiltva GUI-n; API szinten POST/PUT/DELETE végpontok 403-mal visszautasítják a DEMO role-t

### Fázis 2 – Kényelem
- [ ] Borítókép megjelenítés (S3 pre-signed URL)
- [ ] Jelszó visszaállítás
- [ ] Helyszín másolása: meglévő location „Másolás" gombja előtölti a létrehozó modalt (room, name, description egyaránt), a felhasználó csak a különbséget módosítja — tipikus eset: polcos szekrény több polca
- [ ] Témaváltó: shadcn/ui gyári témák közötti váltás (a dark/light toggle megmarad, ehhez képest külön szín-téma választó)

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

| API | Mire | Limit | Kulcs kell? |
|-----|------|-------|-------------|
| [OpenLibrary](https://openlibrary.org/developers/api) | ISBN lookup (elsődleges) | Ingyenes, nincs limit | Nem |
| [Google Books API](https://developers.google.com/books) | ISBN lookup (fallback) | Ingyenes, 1000 req/nap | Igen |
| Google Gemini API vagy DeepL | Leírás fordítás (Fázis 3) | Ingyenes tier | Igen |

## Repók

| Repo | Tartalom | Szerkeszti |
|------|----------|------------|
| `homelibrary-docs` | Tervezési dokumentumok, ADR-ek | Architekt / PO |
| `homelibrary` | Forráskód (backend, frontend, infra) | Fejlesztő |
