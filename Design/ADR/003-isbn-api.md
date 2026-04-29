# ADR-003: ISBN Lookup Adatforrás-stratégia

**Dátum:** 2026-03-28  
**Frissítve:** 2026-04-29  
**Státusz:** Elfogadva (felváltja a korábbi moly.hu + OpenLibrary verziót)

## Kontextus

A könyv felvételekor ISBN szám alapján automatikusan le kell kérni a könyv adatait (cím, szerző, kiadó, kiadási év). A megoldásnak ingyenesnek kell lennie. Az alkalmazás 90%+ arányban magyar kiadású könyveket kezel, ezért a magyar lefedettség kritikus.

2026 áprilisában elvégzett manuális lefedettségi teszt (13 saját ISBN) és TOS/robots.txt ellenőrzés alapján az eredetileg tervezett moly.hu + OpenLibrary stratégia nem tartható:

- **moly.hu:** `robots.txt` tiltja a `/kereses` útvonal scrape-jét (`Disallow: /`, a `/kereses` nincs az Allow listán). A tervezett kétlépéses lookup első lépése tiltott.
- **antikvarium.hu:** TOS tiltja a tartalom felhasználását előzetes írásbeli engedély nélkül; non-commercial kivétel nincs.
- **OpenLibrary:** az Internet Archive 3–9%-os dokumentált downtime-ja és empirikus teszt (első 6 ISBN: 0 találat) alapján megbízhatatlan a magyar könyveknél.
- **Google Books:** API kulcs igényes, a 90%+ magyar lefedettségi célhoz gyenge.

## Döntés

**OSZK NEKTÁR Z39.50 protokollon mint egyetlen külső forrás; manuális bevitel mint fallback.**

Az OSZK (Országos Széchényi Könyvtár) NEKTÁR adatbázisa Z39.50 protokollon keresztül elérhető. Mért lefedettség (13-as ISBN minta): ~85% találat / ~77% teljes adattal (a 2 hiányzó ISBN mindkét esetben német kiadás — várható). Külső API fallback chain nincs.

## OSZK NEKTÁR Z39.50 endpoint

| | |
|---|---|
| Host | `tagetes2.oszk.hu` |
| Port | `1616` |
| Adatbázis | `B1` |
| Elérhetőség | 03:30–23:00 helyi idő, 7/7 |
| Rekord szintaxis | MARC21 |
| ISBN keresés | `@attr 1=7 {ISBN}` (Bib-1 attribútum) |

**Időablakon kívüli hívás (23:00–03:30):** az OSZK client azonnal `Optional.empty()`-t ad vissza → a service réteg manuális bevitelre irányít.

## MARC21 → IsbnLookupResult mapping

| HomeLibrary mező | MARC tag |
|---|---|
| Cím | 245 $a |
| Alcím | 245 $b |
| Szerző | 100 $a + $j |
| Kiadó | 260 $b |
| Kiadási év | 260 $c |
| Oldalszám | 300 $a |
| Nyelv | 041 $a (ISO 639-2, 3 betűs) |

Borítókép: MARC21 nem tartalmaz cover URL-t — Fázis 1-ben nincs.

**`hasMinimumFields()` definíció:** kötelező a `title` és `author`. Ha az OSZK csonka rekordot ad, a service `Optional.empty()`-ként kezeli → manuális bevitelre irányít.

## DEMO szerepkör korlátozás

A beégetett DEMO felhasználó ISBN lookup hívásai limitáltak:

- **Session limit: 5 keresés** — Spring Cache-ben tárolva, kulcs a JWT token ID (`jti` claim). Lambda-újraindítás esetén reset; DEMO kontextusban elfogadható.
- **Napi limit: 50 keresés** — DB-táblában tárolva, lazy reset stratégiával: ellenőrzéskor az aktuális dátumot összehasonlítja a tárolt dátummal; ha eltért, nulláz és frissíti a dátumot. Nincs szükség éjféli scheduled taskra (Lambda-n nem megbízható).

## Alternatívák elutasítva

| Forrás | Ok |
|---|---|
| moly.hu | robots.txt tiltja a `/kereses` scrape-jét |
| antikvarium.hu | TOS: előzetes írásbeli engedély kell |
| OpenLibrary | instabilitás + gyenge magyar lefedettség |
| Google Books | API kulcs + gyenge magyar lefedettség |
| ISBNdb | fizetős |

## Következmények

- Java Z39.50 client: YAZ4J + natív bináris (részletek: ADR-010)
- MARC parser: `marc4j` Maven függőség
- `IsbnSource` enum értékek: `OSZK`, `MANUAL`
- Lambda VPC outbound: TCP 1616 → `193.6.201.206/32` (OSZK statikus IP)
- DEMO napi limit: új DB tábla szükséges a napi számlálóhoz
- `cover_image_url` Fázis 1-ben nem töltjük (MARC21 nem tartalmaz)
